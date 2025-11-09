# Kubernetes ベストプラクティス

## Pod設計

### 1コンテナ1プロセス

```yaml
# ❌ 悪い例：1つのPodに複数の異なるアプリケーション
apiVersion: v1
kind: Pod
metadata:
  name: multi-app
spec:
  containers:
  - name: web
    image: nginx
  - name: database  # Podに含めるべきでない
    image: postgres

# ✅ 良い例：サイドカーパターン
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  - name: app
    image: myapp:1.0
  - name: log-forwarder  # サポート機能
    image: fluentd
```

### リソース設定（必須）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:  # 最小リソース（スケジューリング基準）
        memory: "128Mi"
        cpu: "100m"
      limits:    # 最大リソース（制限）
        memory: "256Mi"
        cpu: "200m"
```

### リソース設定の推奨値

```yaml
# ✅ 推奨：requestsとlimitsを設定
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"  # CPUはバースト可能

# ⚠️ メモリはOOMKillerを避けるため、limits = requests * 1.5-2倍程度
# ⚠️ CPUはスロットリングを考慮してlimitsを高めに
```

## ヘルスチェック

### Liveness・Readiness・Startup Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0

    # 起動完了検知（起動が遅いアプリ用）
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 30  # 最大300秒待つ

    # 生存確認（失敗時はPod再起動）
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3

    # トラフィック受付可否（失敗時はEndpointから除外）
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
```

### Probe実装例

```go
// ヘルスチェックエンドポイント
func healthHandler(w http.ResponseWriter, r *http.Request) {
    // Liveness: アプリケーションが生きているか
    if r.URL.Path == "/health/live" {
        w.WriteHeader(http.StatusOK)
        return
    }

    // Readiness: トラフィックを受け入れられるか
    if r.URL.Path == "/health/ready" {
        if isReady() {
            w.WriteHeader(http.StatusOK)
        } else {
            w.WriteHeader(http.StatusServiceUnavailable)
        }
        return
    }
}
```

## Deployment戦略

### ローリングアップデート

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # 同時に停止できるPod数
      maxSurge: 2            # 超過して作成できるPod数
  template:
    spec:
      containers:
      - name: app
        image: myapp:2.0

        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

### PodDisruptionBudget

```yaml
# 最低限維持するPod数を保証
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2  # 最低2つは常に稼働
  selector:
    matchLabels:
      app: myapp
```

## 設定管理

### ConfigMapとSecret

```yaml
# ConfigMap: 非機密設定
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  config.json: |
    {
      "timeout": 30
    }

---
# Secret: 機密情報
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DATABASE_PASSWORD: "my-secret-password"
  API_KEY: "api-key-value"

---
# 使用例
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0

    # 環境変数として注入
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DATABASE_PASSWORD

    # ファイルとしてマウント
    volumeMounts:
    - name: config
      mountPath: /etc/config
    - name: secret
      mountPath: /etc/secret
      readOnly: true

  volumes:
  - name: config
    configMap:
      name: app-config
  - name: secret
    secret:
      secretName: app-secret
```

### ExternalSecret（推奨）

```yaml
# AWS Secrets Manager、Vault等から取得
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: app-secret
  data:
  - secretKey: database-password
    remoteRef:
      key: prod/database/password
```

## セキュリティ

### SecurityContext

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod レベル
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: app
    image: myapp:1.0

    # Container レベル
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE

    volumeMounts:
    - name: tmp
      mountPath: /tmp

  volumes:
  - name: tmp
    emptyDir: {}
```

### NetworkPolicy

```yaml
# デフォルト拒否
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# アプリケーション間通信許可
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  # データベースへの通信のみ許可
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  # DNS解決を許可
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### RBAC

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa

---
# Role（namespace内）
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: myapp-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# Podで使用
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false  # 不要なら無効化
  containers:
  - name: app
    image: myapp:1.0
```

## オートスケーリング

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # CPU使用率ベース
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # メモリ使用率ベース
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # カスタムメトリクス
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 5分間安定してから縮小
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

### VerticalPodAutoscaler

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Recreate, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 1
        memory: 1Gi
```

## Service & Ingress

### Service タイプ

```yaml
# ClusterIP（内部通信）
apiVersion: v1
kind: Service
metadata:
  name: myapp-internal
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080

---
# LoadBalancer（外部公開）
apiVersion: v1
kind: Service
metadata:
  name: myapp-external
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

## ストレージ

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3  # クラウドプロバイダー依存
  resources:
    requests:
      storage: 10Gi

---
# StatefulSet で使用
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 20Gi
```

## ロギング

### 構造化ログ（JSON）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: LOG_FORMAT
      value: "json"  # 構造化ログを推奨
```

```go
// アプリケーション側の実装例
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
logger.Info("user login",
    zap.String("user_id", "123"),
    zap.String("ip", "192.168.1.1"),
)
// {"level":"info","msg":"user login","user_id":"123","ip":"192.168.1.1"}
```

### ログ集約

```yaml
# Fluentd DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

## 命名規則

```yaml
# リソース命名規則
# {app-name}-{component}-{environment}

# ✅ 良い例
myapp-api-prod
myapp-worker-staging
myapp-db-dev

# Labels（推奨）
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: myapp-system
    app.kubernetes.io/managed-by: helm
    environment: production
    team: backend
```

## マニフェスト管理

### Kustomize

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1  # overlayで上書き
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"

---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
replicas:
- name: myapp
  count: 5
images:
- name: myapp
  newTag: v1.2.3
patchesStrategicMerge:
- resources-patch.yaml

---
# overlays/production/resources-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Helm

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

resources:
  requests:
    memory: 128Mi
    cpu: 100m
  limits:
    memory: 256Mi
    cpu: 200m

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

---
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

## モニタリング

### Prometheus & ServiceMonitor

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  ports:
  - name: http
    port: 8080
  - name: metrics  # メトリクスポート
    port: 9090
  selector:
    app: myapp

---
# ServiceMonitor（Prometheus Operator使用時）
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

## チェックリスト

### 本番環境デプロイ前
- [ ] リソースrequests/limits設定済み
- [ ] Liveness/Readiness Probe設定済み
- [ ] PodDisruptionBudget設定済み
- [ ] HPA設定済み（必要に応じて）
- [ ] SecurityContext設定済み（非root実行）
- [ ] NetworkPolicy設定済み
- [ ] RBAC最小権限設定済み
- [ ] Secret外部管理（Vault, AWS Secrets Manager等）
- [ ] ログは構造化JSON形式
- [ ] メトリクス公開（Prometheus形式）
- [ ] イメージタグは明示的（`:latest`を避ける）
- [ ] 適切なラベル付与
- [ ] Graceful shutdown実装
- [ ] ConfigMap/Secretバージョン管理

### セキュリティ
- [ ] Pod Security Standards適用
- [ ] イメージ脆弱性スキャン
- [ ] シークレットはKubernetes Secretまたは外部管理
- [ ] ServiceAccount最小権限
- [ ] NetworkPolicy有効化

### 可観測性
- [ ] 構造化ログ
- [ ] メトリクス収集
- [ ] トレーシング（分散トレーシング）
- [ ] アラート設定

## 避けるべきアンチパターン

### ❌ やってはいけないこと

```yaml
# 1. latest タグ使用
containers:
- image: myapp:latest  # ❌ バージョン不明

# 2. リソース未設定
containers:
- name: app
  image: myapp:1.0
  # resources なし ❌

# 3. ヘルスチェックなし
containers:
- name: app
  image: myapp:1.0
  # livenessProbe/readinessProbe なし ❌

# 4. root ユーザーで実行
containers:
- name: app
  image: myapp:1.0
  # securityContext なし ❌

# 5. 平文でシークレット
env:
- name: PASSWORD
  value: "my-secret"  # ❌ 平文

# 6. namespace: default 使用
metadata:
  namespace: default  # ❌ 専用namespace作成

# 7. 単一レプリカ（本番環境）
spec:
  replicas: 1  # ❌ 可用性なし
```

## 参考リソース

- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [12 Factor App](https://12factor.net/)
- [CNCF Cloud Native Trail Map](https://github.com/cncf/trailmap)
- [Kubernetes Production Best Practices](https://learnk8s.io/production-best-practices)
