# Helm チートシート

## 基本コマンド

```bash
# バージョン確認
helm version

# ヘルプ
helm help
helm install --help

# リポジトリ追加
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami

# リポジトリ一覧
helm repo list

# リポジトリ更新
helm repo update

# リポジトリ削除
helm repo remove stable

# チャート検索
helm search repo nginx
helm search repo stable/
helm search hub wordpress

# チャート情報表示
helm show chart bitnami/nginx
helm show values bitnami/nginx
helm show readme bitnami/nginx
helm show all bitnami/nginx
```

## インストール・アンインストール

```bash
# インストール
helm install my-release bitnami/nginx
helm install my-release bitnami/nginx --namespace my-namespace
helm install my-release bitnami/nginx --create-namespace --namespace my-namespace

# カスタム値で��ンストール
helm install my-release bitnami/nginx --set replicaCount=3
helm install my-release bitnami/nginx --set image.tag=1.25
helm install my-release bitnami/nginx -f values.yaml

# ドライラン
helm install my-release bitnami/nginx --dry-run --debug

# リリース名自動生成
helm install bitnami/nginx --generate-name

# ローカルチャートからインストール
helm install my-release ./my-chart

# tarファイルからインストール
helm install my-release my-chart-0.1.0.tgz

# URLからインストール
helm install my-release https://example.com/charts/my-chart-0.1.0.tgz

# アンインストール
helm uninstall my-release
helm uninstall my-release --namespace my-namespace
helm uninstall my-release --keep-history
```

## リリース管理

```bash
# リリース一覧
helm list
helm list -n my-namespace
helm list --all-namespaces
helm list -a  # 削除済みも含む

# リリース詳細
helm status my-release
helm status my-release -n my-namespace

# リリース履歴
helm history my-release

# アップグレード
helm upgrade my-release bitnami/nginx
helm upgrade my-release bitnami/nginx --set replicaCount=5
helm upgrade my-release bitnami/nginx -f values.yaml
helm upgrade my-release ./my-chart
helm upgrade my-release bitnami/nginx --reuse-values
helm upgrade my-release bitnami/nginx --reset-values

# インストールまたはアップグレード
helm upgrade --install my-release bitnami/nginx

# ロールバック
helm rollback my-release
helm rollback my-release 1  # リビジョン1へ
helm rollback my-release 0  # 直前のリビジョンへ

# デプロイ済みマニフェスト確認
helm get manifest my-release
helm get values my-release
helm get notes my-release
helm get hooks my-release
helm get all my-release
```

## チャート作成

```bash
# チャート作成
helm create my-chart

# チャート構造
my-chart/
├── Chart.yaml          # チャートメタデータ
├── values.yaml         # デフォルト値
├── charts/             # 依存チャート
├── templates/          # テンプレート
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # ヘルパーテンプレート
│   ├── NOTES.txt       # インストール後のメッセージ
│   └── tests/
│       └── test-connection.yaml
└── .helmignore

# チャートのバリデーション
helm lint my-chart

# チャートのパッケージング
helm package my-chart
helm package my-chart --version 1.0.0
helm package my-chart --app-version 2.0.0

# テンプレートのレンダリング
helm template my-release my-chart
helm template my-release my-chart -f values.yaml
helm template my-release my-chart --set replicaCount=3

# 依存関係の更新
helm dependency update my-chart
helm dependency build my-chart
helm dependency list my-chart
```

## Chart.yaml

```yaml
apiVersion: v2
name: my-chart
description: A Helm chart for Kubernetes
type: application  # または library

# チャートバージョン（SemVer 2）
version: 0.1.0

# アプリケーションバージョン
appVersion: "1.0"

# 依存関係
dependencies:
  - name: postgresql
    version: 12.1.0
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: 17.0.0
    repository: https://charts.bitnami.com/bitnami
    alias: cache

# メタデータ
keywords:
  - web
  - application
home: https://example.com
sources:
  - https://github.com/example/my-app
maintainers:
  - name: John Doe
    email: john@example.com
icon: https://example.com/icon.png
deprecated: false
```

## values.yaml

```yaml
# レプリカ数
replicaCount: 1

# イメージ設定
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# サービスアカウント
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Pod設定
podAnnotations: {}
podSecurityContext: {}

securityContext: {}

# サービス設定
service:
  type: ClusterIP
  port: 80

# Ingress設定
ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

# リソース制限
resources: {}
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

# オートスケーリング
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# Node選択
nodeSelector: {}

tolerations: []

affinity: {}
```

## templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-chart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 12 }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: http
        readinessProbe:
          httpGet:
            path: /
            port: http
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

## templates/_helpers.tpl

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-chart.labels" -}}
helm.sh/chart: {{ include "my-chart.chart" . }}
{{ include "my-chart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

## テンプレート関数

```yaml
# 文字列操作
{{ .Values.name | quote }}           # クォート
{{ .Values.name | upper }}           # 大文字
{{ .Values.name | lower }}           # 小文字
{{ .Values.name | trim }}            # トリム
{{ .Values.name | trunc 63 }}        # 切り詰め
{{ printf "%s-%s" .Release.Name .Chart.Name }}  # フォーマット

# デフォルト値
{{ .Values.name | default "default-name" }}

# 条件分岐
{{- if .Values.enabled }}
enabled: true
{{- else }}
enabled: false
{{- end }}

{{- if and .Values.enabled .Values.external }}
...
{{- end }}

{{- if or .Values.enabled .Values.force }}
...
{{- end }}

{{- if not .Values.disabled }}
...
{{- end }}

# ループ
{{- range .Values.items }}
- {{ . }}
{{- end }}

{{- range $key, $value := .Values.map }}
{{ $key }}: {{ $value }}
{{- end }}

# YAML出力
{{- toYaml .Values.resources | nindent 12 }}
{{- toJson .Values.data }}

# インクルード
{{- include "my-chart.labels" . | nindent 4 }}

# 変数
{{- $name := .Values.name }}
name: {{ $name }}

# with（スコープ変更）
{{- with .Values.service }}
type: {{ .type }}
port: {{ .port }}
{{- end }}
```

## カスタム値ファイル

```yaml
# values-dev.yaml
replicaCount: 1
image:
  tag: "dev"
ingress:
  enabled: true
  hosts:
    - host: dev.example.com

# values-prod.yaml
replicaCount: 3
image:
  tag: "1.0.0"
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
ingress:
  enabled: true
  hosts:
    - host: example.com
  tls:
    - secretName: example-tls
      hosts:
        - example.com
```

```bash
# 使用
helm install my-release ./my-chart -f values-dev.yaml
helm install my-release ./my-chart -f values-prod.yaml
```

## フック

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-chart.fullname" . }}-migration
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: migration
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
```

### フックの種類

```
pre-install     # インストール前
post-install    # インストール後
pre-delete      # 削除前
post-delete     # 削除後
pre-upgrade     # アップグレード前
post-upgrade    # アップグレード後
pre-rollback    # ロールバック前
post-rollback   # ロールバック後
test            # テスト
```

## テスト

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "my-chart.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "my-chart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

```bash
# テスト実行
helm test my-release
```

## よく使うコマンド

```bash
# 開発中のワークフロー
helm create my-chart
helm lint my-chart
helm template my-release my-chart
helm install my-release my-chart --dry-run --debug
helm install my-release my-chart
helm upgrade my-release my-chart
helm uninstall my-release

# 値の上書き
helm install my-release my-chart \
  --set replicaCount=3 \
  --set image.tag=2.0 \
  --set service.type=LoadBalancer

# 複数の値ファイル
helm install my-release my-chart \
  -f values.yaml \
  -f values-prod.yaml

# デバッグ
helm install my-release my-chart --dry-run --debug
helm template my-release my-chart --debug

# 特定のテンプレートのみ表示
helm template my-release my-chart -s templates/deployment.yaml

# リリース情報
helm get values my-release
helm get manifest my-release
helm get notes my-release
helm get hooks my-release

# 依存関係
helm dependency update my-chart
helm dependency list my-chart

# チャートリポジトリ
helm repo add myrepo https://charts.example.com
helm repo update
helm search repo myrepo/
helm pull myrepo/my-chart
helm pull myrepo/my-chart --untar

# チャートのプッシュ（ChartMuseum）
helm push my-chart-0.1.0.tgz myrepo
```

## プラグイン

```bash
# プラグイン一覧
helm plugin list

# プラグインインストール
helm plugin install https://github.com/databus23/helm-diff

# よく使うプラグイン

# helm-diff（変更差分表示）
helm diff upgrade my-release my-chart

# helm-secrets（シークレット管理）
helm secrets install my-release my-chart -f secrets.yaml

# helm-push（ChartMuseum用）
helm push my-chart myrepo
```

## Tips

```bash
# 環境変数でKubeconfig指定
export KUBECONFIG=~/.kube/config

# Namespaceデフォルト設定
export HELM_NAMESPACE=my-namespace

# カスタムリポジトリキャッシュ
export HELM_REPOSITORY_CACHE=~/.helm/repository

# デバッグモード
export HELM_DEBUG=true

# リリース名の自動生成
helm install my-chart --generate-name

# アップグレード時の待機
helm upgrade my-release my-chart --wait --timeout 5m

# ロールバック時の待機
helm rollback my-release --wait

# 失敗時のクリーンアップ
helm upgrade my-release my-chart --atomic

# 履歴保持数
helm upgrade my-release my-chart --history-max 5

# リリース情報をJSONで出力
helm list -o json
helm get values my-release -o json
```

## .helmignore

```
# VCS
.git/
.gitignore

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Temporary files
*.tmp
*.bak

# CI/CD
.gitlab-ci.yml
.github/
```

## ベストプラクティス

```yaml
# 1. ラベルの一貫性
metadata:
  labels:
    app.kubernetes.io/name: {{ include "my-chart.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}

# 2. セキュリティコンテキスト
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

# 3. リソース制限
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# 4. ヘルスチェック
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5

# 5. イメージタグの明示
image:
  repository: nginx
  tag: "1.25.0"  # latest は避ける
  pullPolicy: IfNotPresent
```
