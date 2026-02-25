# OpenShift Logging & LokiStack 完全構築ガイド

## 1. 準備

この構成では、`ClusterLogForwarder` (Vector) がログを収集し、`LokiStack` (Loki) へ転送します。

### ① ログ保存用S3(NooBaa)バケットの接続設定

Lokiが使用するオブジェクトストレージの情報をSecretに格納します。

```bash
# バケット接続情報の取得
BUCKET_NAME=$(oc get configmap loki-bucket -n openshift-storage -o jsonpath='{.data.BUCKET_NAME}')
BUCKET_HOST=$(oc get configmap loki-bucket -n openshift-storage -o jsonpath='{.data.BUCKET_HOST}')
ACCESS_KEY=$(oc get secret loki-bucket -n openshift-storage -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SECRET_KEY=$(oc get secret loki-bucket -n openshift-storage -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)

# Loki用Secretの作成
oc create secret generic logging-loki-s3 -n openshift-logging \
  --from-literal=access_key_id=$ACCESS_KEY \
  --from-literal=access_key_secret=$SECRET_KEY \
  --from-literal=bucketnames=$BUCKET_NAME \
  --from-literal=endpoint=https://$BUCKET_HOST \
  --from-literal=region=noobaa

```

---

## 2. ストレージ層：LokiStack のデプロイ

`lokistack.yaml` を作成・適用します。

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

**適用コマンド:** `oc apply -f lokistack.yaml`

---

## 3. コレクター層：権限(RBAC)の設定

Loki Gatewayがログを受け取れるよう、手動で権限を付与します。

```bash
# ① コレクター用ServiceAccount作成
oc create sa log-collector -n openshift-logging

# ② Loki書き込み用手動ロールの作成と付与 (403対策)
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

# ③ 収集に必要な標準ロールの付与
oc adm policy add-cluster-role-to-user cluster-logging-collector -z log-collector -n openshift-logging
oc adm policy add-cluster-role-to-user collect-application-logs -z log-collector -n openshift-logging
oc adm policy add-cluster-role-to-user collect-infrastructure-logs -z log-collector -n openshift-logging

```

---

## 4. 転送設定：ClusterLogForwarder のデプロイ

`forwarder.yaml` を作成・適用します。

### `forwarder.yaml`

※ 構築初期は過去ログの処理でメモリを消費するため、`memory: 32Gi` への増強パッチを後述の手順で行います。

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
        url: 'https://logging-loki-gateway-http.openshift-logging.svc:8080/api/logs/v1/infrastructure' # インフラログを狙い撃ち
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

**適用コマンド:** `oc apply -f forwarder.yaml`

**メモリ増強パッチ (OOMKilled対策):**

```bash
oc patch clusterlogforwarder instance -n openshift-logging --type=merge -p '{"spec":{"collector":{"resources":{"limits":{"memory":"32Gi"}}}}}'

```

---

## 5. UI設定：コンソールプラグインの有効化

`ui-plugin.yaml` を作成・適用し、OpenShift Webコンソールに「Logs」タブを表示します。

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

**適用コマンド:** `oc apply -f ui-plugin.yaml`

**閲覧権限の付与 (自分用):**

```bash
oc adm policy add-cluster-role-to-user lokistack-logging-loki-application-view $(oc whoami)
oc adm policy add-cluster-role-to-user lokistack-logging-loki-infrastructure-view $(oc whoami)

```

---

## 6. 最終確認

全てのPodが安定したのち、以下のコマンドでLokiがログを認識しているか確認します。

```bash
# labels にデータがあれば成功
oc exec -n openshift-logging $(oc get pod -l app.kubernetes.io/component=gateway -n openshift-logging -o name | head -n 1) -c gateway -- curl -sk -H "Authorization: Bearer $(oc whoami -t)" https://localhost:8080/api/logs/v1/infrastructure/loki/api/v1/labels

