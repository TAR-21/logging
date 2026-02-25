# OpenShift Logging & LokiStack Complete Deployment Guide

## 1. Preparation

In this architecture, `ClusterLogForwarder` (Vector) collects logs and forwards them to `LokiStack` (Loki).

### ① Configure S3 (NooBaa) Bucket Connection for Log Storage

Store the object storage information used by Loki in a Secret.

```bash
# Retrieve bucket connection information
BUCKET_NAME=$(oc get configmap loki-bucket -n openshift-storage -o jsonpath='{.data.BUCKET_NAME}')
BUCKET_HOST=$(oc get configmap loki-bucket -n openshift-storage -o jsonpath='{.data.BUCKET_HOST}')
ACCESS_KEY=$(oc get secret loki-bucket -n openshift-storage -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SECRET_KEY=$(oc get secret loki-bucket -n openshift-storage -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)

# Create Secret for Loki
oc create secret generic logging-loki-s3 -n openshift-logging \
  --from-literal=access_key_id=$ACCESS_KEY \
  --from-literal=access_key_secret=$SECRET_KEY \
  --from-literal=bucketnames=$BUCKET_NAME \
  --from-literal=endpoint=https://$BUCKET_HOST \
  --from-literal=region=noobaa
```

---

## 2. Storage Layer: Deploy LokiStack

Create and apply `lokistack.yaml`.

### `lokistack.yaml`

```yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  size: 1x.extra-small
  storage:
    schemas:
    - version: v13
      effectiveDate: "2026-01-01"
    secret:
      name: logging-loki-s3
      type: s3
  storageClassName: ocs-storagecluster-ceph-rbd
  tenants:
    mode: openshift-logging
```

**Apply command:**
`oc apply -f lokistack.yaml`

---

## 3. Collector Layer: Configure Permissions (RBAC)

Manually grant permissions so the Loki Gateway can receive logs.

```bash
# ① Create ServiceAccount for collector
oc create sa log-collector -n openshift-logging

# ② Create and assign manual role for Loki write access (403 workaround)
cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manual-loki-write
rules:
- apiGroups: ["loki.grafana.com"]
  resources: ["application", "infrastructure", "audit"]
  resourceNames: ["logs"]
  verbs: ["create"]
EOF

oc adm policy add-cluster-role-to-user manual-loki-write -z log-collector -n openshift-logging

# ③ Assign required standard collection roles
oc adm policy add-cluster-role-to-user cluster-logging-collector -z log-collector -n openshift-logging
oc adm policy add-cluster-role-to-user collect-application-logs -z log-collector -n openshift-logging
oc adm policy add-cluster-role-to-user collect-infrastructure-logs -z log-collector -n openshift-logging
```

---

## 4. Forwarding Configuration: Deploy ClusterLogForwarder

Create and apply `forwarder.yaml`.

### `forwarder.yaml`

> During initial setup, processing historical logs may consume significant memory. Apply the memory expansion patch (`memory: 32Gi`) described below if necessary.

```yaml
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
    - name: default-loki
      type: loki
      loki:
        url: 'https://logging-loki-gateway-http.openshift-logging.svc:8080/api/logs/v1/infrastructure' # Target infrastructure logs
        authentication:
          token:
            from: serviceAccount
      tls:
        insecureSkipVerify: true
  pipelines:
    - name: all-logs-to-loki
      inputRefs:
        - infrastructure
      outputRefs:
        - default-loki
  serviceAccount:
    name: log-collector
```

**Apply command:**
`oc apply -f forwarder.yaml`

### Memory Expansion Patch (OOMKilled mitigation)

```bash
oc patch clusterlogforwarder instance -n openshift-logging --type=merge \
  -p '{"spec":{"collector":{"resources":{"limits":{"memory":"32Gi"}}}}}'
```

---

## 5. UI Configuration: Enable Console Plugin

Create and apply `ui-plugin.yaml` to display the **Logs** tab in the OpenShift Web Console.

### `ui-plugin.yaml`

```yaml
apiVersion: observability.openshift.io/v1alpha1
kind: UIPlugin
metadata:
  name: logging
spec:
  type: Logging
  logging:
    lokiStack:
      name: logging-loki
```

**Apply command:**
`oc apply -f ui-plugin.yaml`

### Grant View Permissions (for yourself)

```bash
oc adm policy add-cluster-role-to-user lokistack-logging-loki-application-view $(oc whoami)
oc adm policy add-cluster-role-to-user lokistack-logging-loki-infrastructure-view $(oc whoami)
```

---

## 6. Final Verification

After all Pods are stable, verify that Loki recognizes the logs with the following command:

```bash
# Success if label data is returned
oc exec -n openshift-logging \
  $(oc get pod -l app.kubernetes.io/component=gateway -n openshift-logging -o name | head -n 1) \
  -c gateway -- \
  curl -sk -H "Authorization: Bearer $(oc whoami -t)" \
  https://localhost:8080/api/logs/v1/infrastructure/loki/api/v1/labels
```
