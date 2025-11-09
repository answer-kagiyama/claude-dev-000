# Kubernetes (kubectl) チートシート

## 基本コマンド

```bash
# バージョン確認
kubectl version
kubectl version --short

# クラスタ情報
kubectl cluster-info
kubectl cluster-info dump

# コンテキスト管理
kubectl config get-contexts
kubectl config current-context
kubectl config use-context context-name
kubectl config set-context --current --namespace=my-namespace

# Namespace
kubectl get namespaces
kubectl get ns
kubectl create namespace my-namespace
kubectl delete namespace my-namespace
```

## リソース操作

### 基本操作

```bash
# リソース一覧
kubectl get pods
kubectl get po  # 短縮形
kubectl get pods -n namespace  # Namespace指定
kubectl get pods --all-namespaces  # すべてのNamespace
kubectl get pods -A  # 短縮形
kubectl get pods -o wide  # 詳細表示
kubectl get pods -o yaml  # YAML形式
kubectl get pods -o json  # JSON形式
kubectl get pods --watch  # 監視モード
kubectl get pods -w  # 短縮形

# 複数リソース
kubectl get pods,services
kubectl get all  # Pod, Service, Deployment等

# リソース詳細
kubectl describe pod pod-name
kubectl describe pod pod-name -n namespace

# リソース作成
kubectl create -f file.yaml
kubectl apply -f file.yaml  # 更新も可能（推奨）
kubectl apply -f directory/  # ディレクトリ内すべて
kubectl apply -f https://example.com/manifest.yaml

# リソース削除
kubectl delete pod pod-name
kubectl delete -f file.yaml
kubectl delete pods --all
kubectl delete all --all  # すべてのリソース削除

# リソース編集
kubectl edit pod pod-name
kubectl edit deployment deployment-name

# ラベルセレクタ
kubectl get pods -l app=nginx
kubectl get pods -l 'environment in (production,staging)'
kubectl get pods -l app=nginx,tier=frontend
```

## Pod操作

```bash
# Pod一覧
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get pods -l app=nginx

# Pod作成
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --port=80
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Pod削除
kubectl delete pod pod-name
kubectl delete pod pod-name --grace-period=0 --force  # 強制削除

# Pod詳細
kubectl describe pod pod-name

# Podログ
kubectl logs pod-name
kubectl logs pod-name -c container-name  # コンテナ指定
kubectl logs -f pod-name  # リアルタイム表示
kubectl logs --tail=100 pod-name
kubectl logs --since=1h pod-name
kubectl logs --previous pod-name  # 前回のコンテナ

# Pod内でコマンド実行
kubectl exec pod-name -- ls /
kubectl exec -it pod-name -- bash
kubectl exec -it pod-name -c container-name -- sh

# ファイルコピー
kubectl cp pod-name:/path/to/file ./local-file
kubectl cp ./local-file pod-name:/path/to/file

# ポートフォワード
kubectl port-forward pod-name 8080:80
kubectl port-forward pod-name 8080:80 --address 0.0.0.0

# Podのトップ
kubectl top pod
kubectl top pod pod-name
```

## Deployment

```bash
# Deployment作成
kubectl create deployment nginx --image=nginx
kubectl create deployment nginx --image=nginx --replicas=3

# Deployment一覧
kubectl get deployments
kubectl get deploy

# Deployment詳細
kubectl describe deployment nginx

# スケール
kubectl scale deployment nginx --replicas=5
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# イメージ更新
kubectl set image deployment/nginx nginx=nginx:1.25

# ロールアウト
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2
kubectl rollout restart deployment/nginx

# Deployment削除
kubectl delete deployment nginx

# YAML生成
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
```

## Service

```bash
# Service作成
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Service一覧
kubectl get services
kubectl get svc

# Service詳細
kubectl describe service nginx

# Service削除
kubectl delete service nginx

# エンドポイント確認
kubectl get endpoints nginx
```

## ConfigMap & Secret

```bash
# ConfigMap作成
kubectl create configmap my-config --from-literal=key1=value1
kubectl create configmap my-config --from-file=config.txt
kubectl create configmap my-config --from-env-file=.env

# ConfigMap一覧
kubectl get configmaps
kubectl get cm

# ConfigMap詳細
kubectl describe configmap my-config
kubectl get configmap my-config -o yaml

# ConfigMap削除
kubectl delete configmap my-config

# Secret作成
kubectl create secret generic my-secret --from-literal=password=secret123
kubectl create secret generic my-secret --from-file=ssh-key=~/.ssh/id_rsa
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password

# Secret一覧
kubectl get secrets

# Secret詳細
kubectl describe secret my-secret
kubectl get secret my-secret -o yaml
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# Secret削除
kubectl delete secret my-secret
```

## マニフェストファイル

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    env:
    - name: ENV_VAR
      value: "value"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: my-config
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        env:
        - name: ENV_VAR
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: key1
        - name: SECRET_VAR
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP  # ClusterIP, NodePort, LoadBalancer

---
# NodePort
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080

---
# LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  key1: value1
  key2: value2
  config.txt: |
    multi-line
    configuration
    file
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=  # base64エンコード
  password: cGFzc3dvcmQ=
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
```

### PersistentVolume & PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

## ボリューム

```bash
# PersistentVolume
kubectl get pv
kubectl describe pv pv-name

# PersistentVolumeClaim
kubectl get pvc
kubectl describe pvc pvc-name

# StorageClass
kubectl get storageclass
kubectl get sc
```

## Namespace

```bash
# Namespace作成
kubectl create namespace dev
kubectl create namespace prod

# リソースをNamespace指定で取得
kubectl get pods -n dev
kubectl get all -n dev

# デフォルトNamespace設定
kubectl config set-context --current --namespace=dev

# Namespace削除
kubectl delete namespace dev
```

## ラベル・アノテーション

```bash
# ラベル表示
kubectl get pods --show-labels

# ラベル追加
kubectl label pod pod-name env=prod
kubectl label pod pod-name tier=frontend

# ラベル削除
kubectl label pod pod-name env-

# ラベル更新
kubectl label pod pod-name env=staging --overwrite

# アノテーション追加
kubectl annotate pod pod-name description="my description"

# アノテーション削除
kubectl annotate pod pod-name description-
```

## デバッグ・トラブルシューティング

```bash
# イベント確認
kubectl get events
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -w

# リソース使用状況
kubectl top nodes
kubectl top pods
kubectl top pod pod-name

# ログ
kubectl logs pod-name
kubectl logs -f pod-name
kubectl logs pod-name -c container-name
kubectl logs --previous pod-name

# デバッグPod起動
kubectl run debug --image=busybox -it --rm -- sh
kubectl run debug --image=nicolaka/netshoot -it --rm -- bash

# Pod内でコマンド
kubectl exec -it pod-name -- bash
kubectl exec -it pod-name -- env
kubectl exec -it pod-name -- ps aux

# ノードのコードン・ドレイン
kubectl cordon node-name  # スケジューリング停止
kubectl uncordon node-name  # スケジューリング再開
kubectl drain node-name --ignore-daemonsets  # Pod退避

# API直接呼び出し
kubectl proxy
# http://localhost:8001/api/v1/namespaces/default/pods

# リソース定義の説明
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
```

## ヘルスチェック

```yaml
# Liveness Probe（コンテナの生存確認）
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

# Readiness Probe（トラフィック受付可能確認）
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

# Startup Probe（起動完了確認）
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10

# TCP
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20

# Exec
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

## リソース管理

```yaml
resources:
  requests:  # 最小リソース
    memory: "64Mi"
    cpu: "250m"
  limits:  # 最大リソース
    memory: "128Mi"
    cpu: "500m"

# LimitRange（Namespace単位の制限）
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - max:
      memory: "1Gi"
    min:
      memory: "64Mi"
    type: Container

# ResourceQuota（Namespace単位のクォータ）
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "10"
```

## HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# HPA作成
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# HPA確認
kubectl get hpa
kubectl describe hpa nginx
```

## よく使うコマンド

```bash
# リソースの短縮形
po    # pods
svc   # services
deploy # deployments
rs    # replicasets
ns    # namespaces
cm    # configmaps
pv    # persistentvolumes
pvc   # persistentvolumeclaims

# エイリアス設定（~/.bashrc）
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kgpo='kubectl get pods'
alias kgsvc='kubectl get services'

# bash補完
source <(kubectl completion bash)
complete -F __start_kubectl k  # エイリアス用
```

## マニフェスト検証

```bash
# Dry run
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server

# 差分確認
kubectl diff -f deployment.yaml

# YAML生成
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
kubectl expose deployment nginx --port=80 --dry-run=client -o yaml
```

## RBAC（Role-Based Access Control）

```yaml
# Role（Namespace内）
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

# ClusterRole（クラスタ全体）
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]

# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## Tips

```bash
# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# カスタムカラム
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# ソート
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime

# Podの再起動（Deploymentの場合）
kubectl rollout restart deployment/nginx

# すべてのリソース削除
kubectl delete all --all
kubectl delete all --all -n namespace

# リソースのwatch
watch kubectl get pods

# 複数ファイルを一度に適用
kubectl apply -f .
kubectl apply -R -f ./manifests/
```
