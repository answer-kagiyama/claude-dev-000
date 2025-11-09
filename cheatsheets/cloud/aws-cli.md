# AWS CLI チートシート

## セットアップ

```bash
# インストール確認
aws --version

# 設定
aws configure
# AWS Access Key ID: YOUR_ACCESS_KEY
# AWS Secret Access Key: YOUR_SECRET_KEY
# Default region name: ap-northeast-1
# Default output format: json

# プロファイル別設定
aws configure --profile dev
aws configure --profile prod

# 設定確認
aws configure list
aws configure get region

# 認証情報確認
cat ~/.aws/credentials
cat ~/.aws/config

# 環境変数で設定
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=ap-northeast-1
export AWS_PROFILE=dev
```

## 基本オプション

```bash
# プロファイル指定
aws s3 ls --profile prod

# リージョン指定
aws ec2 describe-instances --region us-east-1

# 出力形式
aws ec2 describe-instances --output json
aws ec2 describe-instances --output yaml
aws ec2 describe-instances --output table
aws ec2 describe-instances --output text

# クエリ（JMESPath）
aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId'

# ドライラン
aws ec2 run-instances --dry-run ...

# ヘルプ
aws help
aws s3 help
aws s3 cp help
```

## EC2

```bash
# インスタンス一覧
aws ec2 describe-instances
aws ec2 describe-instances --instance-ids i-1234567890abcdef0
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]' --output table

# インスタンス起動
aws ec2 run-instances \
  --image-id ami-xxxxxxxxx \
  --instance-type t2.micro \
  --key-name my-key \
  --security-group-ids sg-xxxxxxxxx \
  --subnet-id subnet-xxxxxxxxx \
  --count 1

# インスタンス停止・起動
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 reboot-instances --instance-ids i-1234567890abcdef0

# インスタンス終了
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0

# AMI一覧
aws ec2 describe-images --owners self
aws ec2 describe-images --filters "Name=name,Values=ubuntu*"

# AMI作成
aws ec2 create-image --instance-id i-1234567890abcdef0 --name "My AMI"

# キーペア
aws ec2 describe-key-pairs
aws ec2 create-key-pair --key-name my-key --query 'KeyMaterial' --output text > my-key.pem
aws ec2 delete-key-pair --key-name my-key

# セキュリティグループ
aws ec2 describe-security-groups
aws ec2 create-security-group --group-name my-sg --description "My security group"
aws ec2 authorize-security-group-ingress --group-id sg-xxx --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 revoke-security-group-ingress --group-id sg-xxx --protocol tcp --port 22 --cidr 0.0.0.0/0

# タグ
aws ec2 create-tags --resources i-1234567890abcdef0 --tags Key=Name,Value=MyInstance
```

## S3

```bash
# バケット一覧
aws s3 ls
aws s3 ls s3://bucket-name/
aws s3 ls s3://bucket-name/prefix/ --recursive

# ファイルコピー
aws s3 cp file.txt s3://bucket-name/
aws s3 cp s3://bucket-name/file.txt ./
aws s3 cp s3://bucket-name/file.txt s3://another-bucket/
aws s3 cp . s3://bucket-name/ --recursive
aws s3 cp s3://bucket-name/ . --recursive

# ファイル移動
aws s3 mv file.txt s3://bucket-name/
aws s3 mv s3://bucket-name/file.txt ./

# ファイル削除
aws s3 rm s3://bucket-name/file.txt
aws s3 rm s3://bucket-name/prefix/ --recursive

# 同期
aws s3 sync . s3://bucket-name/
aws s3 sync s3://bucket-name/ .
aws s3 sync s3://source-bucket/ s3://dest-bucket/

# バケット作成
aws s3 mb s3://bucket-name
aws s3 mb s3://bucket-name --region ap-northeast-1

# バケット削除
aws s3 rb s3://bucket-name
aws s3 rb s3://bucket-name --force  # 中身も削除

# バケット情報（s3api使用）
aws s3api list-buckets
aws s3api get-bucket-location --bucket bucket-name
aws s3api get-bucket-versioning --bucket bucket-name

# オブジェクト一覧（s3api）
aws s3api list-objects-v2 --bucket bucket-name
aws s3api list-objects-v2 --bucket bucket-name --prefix prefix/

# 署名付きURL生成
aws s3 presign s3://bucket-name/file.txt --expires-in 3600
```

## IAM

```bash
# ユーザー一覧
aws iam list-users
aws iam get-user --user-name username

# ユーザー作成
aws iam create-user --user-name username

# ユーザー削除
aws iam delete-user --user-name username

# アクセスキー作成
aws iam create-access-key --user-name username

# アクセスキー削除
aws iam delete-access-key --user-name username --access-key-id AKIAIOSFODNN7EXAMPLE

# グループ
aws iam list-groups
aws iam create-group --group-name groupname
aws iam add-user-to-group --user-name username --group-name groupname
aws iam remove-user-from-group --user-name username --group-name groupname

# ポリシー
aws iam list-policies
aws iam list-attached-user-policies --user-name username
aws iam attach-user-policy --user-name username --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam detach-user-policy --user-name username --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# ロール
aws iam list-roles
aws iam get-role --role-name rolename
aws iam create-role --role-name rolename --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name rolename --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# 現在の認証情報
aws sts get-caller-identity
```

## RDS

```bash
# インスタンス一覧
aws rds describe-db-instances
aws rds describe-db-instances --db-instance-identifier mydb

# インスタンス作成
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password password \
  --allocated-storage 20

# インスタンス削除
aws rds delete-db-instance --db-instance-identifier mydb --skip-final-snapshot
aws rds delete-db-instance --db-instance-identifier mydb --final-db-snapshot-identifier mydb-final-snapshot

# インスタンス起動・停止
aws rds start-db-instance --db-instance-identifier mydb
aws rds stop-db-instance --db-instance-identifier mydb

# スナップショット一覧
aws rds describe-db-snapshots
aws rds describe-db-snapshots --db-instance-identifier mydb

# スナップショット作成
aws rds create-db-snapshot --db-instance-identifier mydb --db-snapshot-identifier mydb-snapshot

# スナップショットから復元
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-restored \
  --db-snapshot-identifier mydb-snapshot
```

## Lambda

```bash
# 関数一覧
aws lambda list-functions

# 関数作成
aws lambda create-function \
  --function-name my-function \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip

# 関数更新
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# 関数削除
aws lambda delete-function --function-name my-function

# 関数実行
aws lambda invoke \
  --function-name my-function \
  --payload '{"key": "value"}' \
  response.json

# 関数情報
aws lambda get-function --function-name my-function

# 環境変数設定
aws lambda update-function-configuration \
  --function-name my-function \
  --environment "Variables={KEY1=value1,KEY2=value2}"

# ログ確認（CloudWatch Logs）
aws logs tail /aws/lambda/my-function --follow
```

## ECS

```bash
# クラスタ一覧
aws ecs list-clusters
aws ecs describe-clusters --clusters cluster-name

# タスク定義一覧
aws ecs list-task-definitions
aws ecs describe-task-definition --task-definition task-name:1

# サービス一覧
aws ecs list-services --cluster cluster-name
aws ecs describe-services --cluster cluster-name --services service-name

# タスク一覧
aws ecs list-tasks --cluster cluster-name
aws ecs describe-tasks --cluster cluster-name --tasks task-arn

# サービス更新
aws ecs update-service --cluster cluster-name --service service-name --force-new-deployment
aws ecs update-service --cluster cluster-name --service service-name --desired-count 3

# タスク実行
aws ecs run-task \
  --cluster cluster-name \
  --task-definition task-name \
  --count 1
```

## CloudWatch

```bash
# ロググループ一覧
aws logs describe-log-groups

# ログストリーム一覧
aws logs describe-log-streams --log-group-name /aws/lambda/my-function

# ログ表示
aws logs tail /aws/lambda/my-function
aws logs tail /aws/lambda/my-function --follow
aws logs tail /aws/lambda/my-function --since 1h

# メトリクス
aws cloudwatch list-metrics
aws cloudwatch list-metrics --namespace AWS/EC2

# アラーム一覧
aws cloudwatch describe-alarms
aws cloudwatch describe-alarms --alarm-names alarm-name
```

## VPC

```bash
# VPC一覧
aws ec2 describe-vpcs

# サブネット一覧
aws ec2 describe-subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxx"

# ルートテーブル
aws ec2 describe-route-tables

# インターネットゲートウェイ
aws ec2 describe-internet-gateways

# NATゲートウェイ
aws ec2 describe-nat-gateways

# Elastic IP
aws ec2 describe-addresses
aws ec2 allocate-address --domain vpc
aws ec2 release-address --allocation-id eipalloc-xxx
```

## CloudFormation

```bash
# スタック一覧
aws cloudformation list-stacks
aws cloudformation describe-stacks
aws cloudformation describe-stacks --stack-name stack-name

# スタック作成
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=Key1,ParameterValue=Value1

# スタック更新
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml

# スタック削除
aws cloudformation delete-stack --stack-name my-stack

# スタックイベント
aws cloudformation describe-stack-events --stack-name my-stack

# テンプレート検証
aws cloudformation validate-template --template-body file://template.yaml
```

## その他のサービス

### DynamoDB

```bash
# テーブル一覧
aws dynamodb list-tables

# テーブル情報
aws dynamodb describe-table --table-name table-name

# アイテム取得
aws dynamodb get-item \
  --table-name table-name \
  --key '{"id": {"S": "123"}}'

# アイテム追加
aws dynamodb put-item \
  --table-name table-name \
  --item '{"id": {"S": "123"}, "name": {"S": "Alice"}}'

# スキャン
aws dynamodb scan --table-name table-name

# クエリ
aws dynamodb query \
  --table-name table-name \
  --key-condition-expression "id = :id" \
  --expression-attribute-values '{":id": {"S": "123"}}'
```

### SQS

```bash
# キュー一覧
aws sqs list-queues

# メッセージ送信
aws sqs send-message \
  --queue-url https://sqs.region.amazonaws.com/account-id/queue-name \
  --message-body "Hello World"

# メッセージ受信
aws sqs receive-message \
  --queue-url https://sqs.region.amazonaws.com/account-id/queue-name

# メッセージ削除
aws sqs delete-message \
  --queue-url https://sqs.region.amazonaws.com/account-id/queue-name \
  --receipt-handle receipt-handle
```

### SNS

```bash
# トピック一覧
aws sns list-topics

# トピック作成
aws sns create-topic --name topic-name

# サブスクリプション
aws sns subscribe \
  --topic-arn arn:aws:sns:region:account-id:topic-name \
  --protocol email \
  --notification-endpoint email@example.com

# メッセージ送信
aws sns publish \
  --topic-arn arn:aws:sns:region:account-id:topic-name \
  --message "Hello World"
```

### Route 53

```bash
# ホストゾーン一覧
aws route53 list-hosted-zones

# レコード一覧
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC
```

### ElastiCache

```bash
# クラスタ一覧
aws elasticache describe-cache-clusters
aws elasticache describe-replication-groups
```

## よく使うクエリパターン

```bash
# 実行中のEC2インスタンスのID一覧
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text

# インスタンスのIPアドレス一覧
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,PrivateIpAddress]' \
  --output table

# 特定タグのリソース
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=production"

# S3バケットのサイズ
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name BucketSizeBytes \
  --dimensions Name=BucketName,Value=my-bucket Name=StorageType,Value=StandardStorage \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 86400 \
  --statistics Average

# Lambda関数のログストリーム最新10件
aws logs tail /aws/lambda/my-function --since 1h | head -10
```

## Tips

```bash
# エイリアス
alias awsp='aws --profile prod'
alias awsd='aws --profile dev'

# 複数プロファイルの切り替え
export AWS_PROFILE=prod

# MFA使用時の一時認証情報取得
aws sts get-session-token \
  --serial-number arn:aws:iam::123456789012:mfa/user \
  --token-code 123456

# JSONファイルから読み込み
aws ec2 run-instances --cli-input-json file://instance.json

# スケルトンJSON生成
aws ec2 run-instances --generate-cli-skeleton

# ページネーション
aws s3api list-objects-v2 --bucket bucket-name --max-items 100

# 並列実行（xargs）
aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId' --output text | \
  xargs -n 1 -P 10 aws ec2 stop-instances --instance-ids

# 出力をjqで整形
aws ec2 describe-instances | jq '.Reservations[].Instances[] | {id: .InstanceId, state: .State.Name}'

# 認証情報のキャッシュクリア
rm -rf ~/.aws/cli/cache/

# デバッグモード
aws s3 ls --debug
```

## AWS CLI設定ファイル

### ~/.aws/config

```ini
[default]
region = ap-northeast-1
output = json

[profile dev]
region = us-east-1
output = yaml

[profile prod]
region = ap-northeast-1
output = table
role_arn = arn:aws:iam::123456789012:role/ProductionRole
source_profile = default
```

### ~/.aws/credentials

```ini
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY

[dev]
aws_access_key_id = DEV_ACCESS_KEY
aws_secret_access_key = DEV_SECRET_KEY

[prod]
aws_access_key_id = PROD_ACCESS_KEY
aws_secret_access_key = PROD_SECRET_KEY
```
