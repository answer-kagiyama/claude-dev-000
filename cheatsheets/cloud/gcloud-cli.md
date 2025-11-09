# gcloud CLI チートシート

## セットアップ

```bash
# インストール確認
gcloud version

# 初期設定
gcloud init

# 認証
gcloud auth login
gcloud auth application-default login

# 認証情報確認
gcloud auth list

# ログアウト
gcloud auth revoke account@example.com

# 設定確認
gcloud config list
gcloud config list --all

# プロジェクト設定
gcloud config set project PROJECT_ID
gcloud config get-value project

# リージョン・ゾーン設定
gcloud config set compute/region asia-northeast1
gcloud config set compute/zone asia-northeast1-a

# アカウント切り替え
gcloud config set account account@example.com
```

## 構成管理

```bash
# 構成一覧
gcloud config configurations list

# 構成作成
gcloud config configurations create dev
gcloud config configurations create prod

# 構成切り替え
gcloud config configurations activate dev

# 構成削除
gcloud config configurations delete dev

# プロパティ設定
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
gcloud config set project my-project

# プロパティ削除
gcloud config unset compute/region
```

## プロジェクト

```bash
# プロジェクト一覧
gcloud projects list

# プロジェクト情報
gcloud projects describe PROJECT_ID

# プロジェクト作成
gcloud projects create PROJECT_ID --name="My Project"

# プロジェクト削除
gcloud projects delete PROJECT_ID

# 現在のプロジェクト確認
gcloud config get-value project
```

## Compute Engine (GCE)

```bash
# インスタンス一覧
gcloud compute instances list
gcloud compute instances list --filter="zone:asia-northeast1-a"
gcloud compute instances list --format="table(name,zone,machineType,status)"

# インスタンス作成
gcloud compute instances create my-instance \
  --machine-type=e2-micro \
  --zone=asia-northeast1-a \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud

# 追加オプション付き作成
gcloud compute instances create my-instance \
  --machine-type=e2-medium \
  --zone=asia-northeast1-a \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-standard \
  --tags=http-server,https-server \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y nginx'

# インスタンス起動・停止
gcloud compute instances start my-instance
gcloud compute instances stop my-instance
gcloud compute instances reset my-instance

# インスタンス削除
gcloud compute instances delete my-instance
gcloud compute instances delete my-instance --zone=asia-northeast1-a

# インスタンス詳細
gcloud compute instances describe my-instance

# SSH接続
gcloud compute ssh my-instance
gcloud compute ssh my-instance --zone=asia-northeast1-a

# SCPでファイル転送
gcloud compute scp local-file my-instance:~/
gcloud compute scp my-instance:~/remote-file ./

# マシンタイプ一覧
gcloud compute machine-types list
gcloud compute machine-types list --filter="zone:asia-northeast1-a"

# イメージ一覧
gcloud compute images list
gcloud compute images list --project=ubuntu-os-cloud
gcloud compute images list --filter="family:ubuntu-2204-lts"

# ディスク一覧
gcloud compute disks list

# ディスク作成
gcloud compute disks create my-disk --size=100GB --zone=asia-northeast1-a

# ディスク削除
gcloud compute disks delete my-disk --zone=asia-northeast1-a

# スナップショット一覧
gcloud compute snapshots list

# スナップショット作成
gcloud compute disks snapshot my-disk --snapshot-names=my-snapshot

# ファイアウォールルール一覧
gcloud compute firewall-rules list

# ファイアウォールルール作成
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server

# ファイアウォールルール削除
gcloud compute firewall-rules delete allow-http

# 外部IPアドレス一覧
gcloud compute addresses list

# 外部IPアドレス予約
gcloud compute addresses create my-ip --region=asia-northeast1

# 外部IPアドレス削除
gcloud compute addresses delete my-ip --region=asia-northeast1
```

## Google Kubernetes Engine (GKE)

```bash
# クラスタ一覧
gcloud container clusters list

# クラスタ作成
gcloud container clusters create my-cluster \
  --zone=asia-northeast1-a \
  --num-nodes=3 \
  --machine-type=e2-medium

# Autopilotクラスタ作成
gcloud container clusters create-auto my-cluster \
  --region=asia-northeast1

# クラスタ削除
gcloud container clusters delete my-cluster --zone=asia-northeast1-a

# クラスタ情報
gcloud container clusters describe my-cluster --zone=asia-northeast1-a

# kubectl認証情報取得
gcloud container clusters get-credentials my-cluster --zone=asia-northeast1-a

# ノードプール一覧
gcloud container node-pools list --cluster=my-cluster

# ノードプール作成
gcloud container node-pools create my-pool \
  --cluster=my-cluster \
  --zone=asia-northeast1-a \
  --num-nodes=2 \
  --machine-type=e2-medium

# ノードプール削除
gcloud container node-pools delete my-pool --cluster=my-cluster

# クラスタのアップグレード
gcloud container clusters upgrade my-cluster --zone=asia-northeast1-a

# ノード数変更
gcloud container clusters resize my-cluster --num-nodes=5 --zone=asia-northeast1-a
```

## Cloud Storage (GCS)

```bash
# バケット一覧
gcloud storage buckets list
gsutil ls

# バケット作成
gcloud storage buckets create gs://my-bucket --location=asia-northeast1
gsutil mb -l asia-northeast1 gs://my-bucket

# バケット削除
gcloud storage buckets delete gs://my-bucket
gsutil rb gs://my-bucket

# オブジェクト一覧
gcloud storage ls gs://my-bucket/
gsutil ls gs://my-bucket/
gsutil ls -r gs://my-bucket/**

# ファイルコピー
gcloud storage cp file.txt gs://my-bucket/
gsutil cp file.txt gs://my-bucket/
gsutil cp gs://my-bucket/file.txt ./
gsutil cp -r directory/ gs://my-bucket/

# ファイル移動
gsutil mv gs://my-bucket/old.txt gs://my-bucket/new.txt

# ファイル削除
gcloud storage rm gs://my-bucket/file.txt
gsutil rm gs://my-bucket/file.txt
gsutil rm -r gs://my-bucket/directory/

# 同期
gsutil rsync -r ./local-dir gs://my-bucket/remote-dir
gsutil rsync -r gs://my-bucket/remote-dir ./local-dir

# バケット情報
gcloud storage buckets describe gs://my-bucket
gsutil ls -L -b gs://my-bucket

# 公開設定
gsutil iam ch allUsers:objectViewer gs://my-bucket

# 署名付きURL生成
gsutil signurl -d 1h key.json gs://my-bucket/file.txt
```

## Cloud SQL

```bash
# インスタンス一覧
gcloud sql instances list

# インスタンス作成（PostgreSQL）
gcloud sql instances create my-instance \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=asia-northeast1

# インスタンス作成（MySQL）
gcloud sql instances create my-instance \
  --database-version=MYSQL_8_0 \
  --tier=db-f1-micro \
  --region=asia-northeast1

# インスタンス削除
gcloud sql instances delete my-instance

# インスタンス情報
gcloud sql instances describe my-instance

# データベース一覧
gcloud sql databases list --instance=my-instance

# データベース作成
gcloud sql databases create mydb --instance=my-instance

# ユーザー一覧
gcloud sql users list --instance=my-instance

# ユーザー作成
gcloud sql users create myuser --instance=my-instance --password=mypassword

# 接続
gcloud sql connect my-instance --user=postgres

# バックアップ一覧
gcloud sql backups list --instance=my-instance

# バックアップ作成
gcloud sql backups create --instance=my-instance
```

## Cloud Functions

```bash
# 関数一覧
gcloud functions list

# 関数デプロイ（第1世代）
gcloud functions deploy my-function \
  --runtime=python311 \
  --trigger-http \
  --entry-point=hello_world \
  --source=.

# 関数デプロイ（第2世代）
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python311 \
  --trigger-http \
  --entry-point=hello_world \
  --source=.

# 環境変数付きデプロイ
gcloud functions deploy my-function \
  --runtime=python311 \
  --trigger-http \
  --set-env-vars=KEY1=value1,KEY2=value2

# 関数削除
gcloud functions delete my-function

# 関数情報
gcloud functions describe my-function

# 関数呼び出し
gcloud functions call my-function --data='{"name":"World"}'

# ログ確認
gcloud functions logs read my-function
gcloud functions logs read my-function --limit=50
```

## Cloud Run

```bash
# サービス一覧
gcloud run services list

# デプロイ
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-image:latest \
  --platform=managed \
  --region=asia-northeast1 \
  --allow-unauthenticated

# CPU・メモリ指定
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-image:latest \
  --platform=managed \
  --region=asia-northeast1 \
  --cpu=2 \
  --memory=1Gi \
  --min-instances=0 \
  --max-instances=10

# 環境変数設定
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-image:latest \
  --set-env-vars=KEY1=value1,KEY2=value2

# サービス削除
gcloud run services delete my-service --region=asia-northeast1

# サービス情報
gcloud run services describe my-service --region=asia-northeast1

# URL取得
gcloud run services describe my-service --region=asia-northeast1 --format="value(status.url)"

# ログ確認
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=my-service" --limit=50
```

## Container Registry / Artifact Registry

```bash
# Container Registry

# イメージ一覧
gcloud container images list

# タグ一覧
gcloud container images list-tags gcr.io/PROJECT_ID/my-image

# イメージ削除
gcloud container images delete gcr.io/PROJECT_ID/my-image:tag

# Docker認証設定
gcloud auth configure-docker

# Artifact Registry

# リポジトリ一覧
gcloud artifacts repositories list

# リポジトリ作成
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=asia-northeast1

# リポジトリ削除
gcloud artifacts repositories delete my-repo --location=asia-northeast1

# Docker認証設定
gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```

## IAM

```bash
# サービスアカウント一覧
gcloud iam service-accounts list

# サービスアカウント作成
gcloud iam service-accounts create my-sa \
  --display-name="My Service Account"

# サービスアカウント削除
gcloud iam service-accounts delete my-sa@PROJECT_ID.iam.gserviceaccount.com

# キー作成
gcloud iam service-accounts keys create key.json \
  --iam-account=my-sa@PROJECT_ID.iam.gserviceaccount.com

# キー一覧
gcloud iam service-accounts keys list \
  --iam-account=my-sa@PROJECT_ID.iam.gserviceaccount.com

# ロール付与
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:my-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/viewer

# ロール一覧
gcloud iam roles list

# プロジェクトのIAMポリシー確認
gcloud projects get-iam-policy PROJECT_ID
```

## Logging

```bash
# ログ確認
gcloud logging read "resource.type=gce_instance" --limit=10
gcloud logging read "resource.type=cloud_run_revision" --limit=10
gcloud logging read 'timestamp>="2024-01-01T00:00:00Z"' --limit=50

# リアルタイム表示
gcloud logging tail "resource.type=cloud_run_revision"

# フィルタ
gcloud logging read "severity>=ERROR" --limit=50
gcloud logging read 'resource.labels.service_name="my-service"' --limit=50
```

## その他のサービス

### App Engine

```bash
# アプリケーションデプロイ
gcloud app deploy

# バージョン一覧
gcloud app versions list

# サービス一覧
gcloud app services list

# ログ確認
gcloud app logs tail
```

### Pub/Sub

```bash
# トピック一覧
gcloud pubsub topics list

# トピック作成
gcloud pubsub topics create my-topic

# トピック削除
gcloud pubsub topics delete my-topic

# サブスクリプション一覧
gcloud pubsub subscriptions list

# サブスクリプション作成
gcloud pubsub subscriptions create my-sub --topic=my-topic

# メッセージ送信
gcloud pubsub topics publish my-topic --message="Hello World"

# メッセージ受信
gcloud pubsub subscriptions pull my-sub --auto-ack
gcloud pubsub subscriptions pull my-sub --limit=10
```

### Firestore

```bash
# データベース一覧
gcloud firestore databases list

# インデックス一覧
gcloud firestore indexes composite list

# エクスポート
gcloud firestore export gs://my-bucket/firestore-backup
```

## ネットワーク

```bash
# VPC一覧
gcloud compute networks list

# サブネット一覧
gcloud compute networks subnets list

# VPC作成
gcloud compute networks create my-vpc --subnet-mode=custom

# サブネット作成
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=asia-northeast1 \
  --range=10.0.0.0/24

# ロードバランサー一覧
gcloud compute forwarding-rules list
gcloud compute backend-services list
```

## よく使うコマンド

```bash
# フォーマット指定
gcloud compute instances list --format=json
gcloud compute instances list --format=yaml
gcloud compute instances list --format="table(name,zone,status)"
gcloud compute instances list --format="value(name)"

# フィルタ
gcloud compute instances list --filter="zone:asia-northeast1-a"
gcloud compute instances list --filter="status:RUNNING"
gcloud compute instances list --filter="name~^my-.*"

# ソート
gcloud compute instances list --sort-by=name
gcloud compute instances list --sort-by=~creationTimestamp

# リミット
gcloud compute instances list --limit=10

# プロジェクト指定
gcloud compute instances list --project=PROJECT_ID

# 対話形式無効化
gcloud compute instances delete my-instance --quiet
gcloud compute instances delete my-instance -q
```

## Tips

```bash
# エイリアス
alias g='gcloud'
alias gce='gcloud compute'
alias gke='gcloud container'
alias gcr='gcloud container images'

# bash補完
source /path/to/google-cloud-sdk/completion.bash.inc

# 複数プロジェクト管理
gcloud config configurations create dev
gcloud config configurations create prod
gcloud config configurations activate dev

# デフォルト値確認
gcloud config list

# APIの有効化
gcloud services enable compute.googleapis.com
gcloud services enable container.googleapis.com
gcloud services enable run.googleapis.com

# 有効なAPI一覧
gcloud services list --enabled

# クォータ確認
gcloud compute project-info describe --project=PROJECT_ID

# 利用料金確認（Cloud Console必要）
# https://console.cloud.google.com/billing

# デバッグログ
gcloud compute instances list --log-http

# ヘルプ
gcloud help
gcloud compute help
gcloud compute instances help
gcloud compute instances create --help
```

## 設定ファイル

### ~/.config/gcloud/configurations/config_dev

```ini
[core]
account = dev@example.com
project = my-dev-project

[compute]
region = asia-northeast1
zone = asia-northeast1-a
```

### サービスアカウントキーで認証

```bash
gcloud auth activate-service-account \
  --key-file=key.json

export GOOGLE_APPLICATION_CREDENTIALS=key.json
```
