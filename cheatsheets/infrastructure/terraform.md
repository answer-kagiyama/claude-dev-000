# Terraform チートシート

## 基本コマンド

```bash
# バージョン確認
terraform version

# 初期化
terraform init
terraform init -upgrade  # プロバイダー更新

# フォーマット
terraform fmt
terraform fmt -recursive

# バリデーション
terraform validate

# プラン（実行計画）
terraform plan
terraform plan -out=tfplan

# 適用
terraform apply
terraform apply tfplan
terraform apply -auto-approve

# 破棄
terraform destroy
terraform destroy -auto-approve

# 状態確認
terraform show
terraform state list
terraform state show aws_instance.example

# 出力確認
terraform output
terraform output instance_ip

# ワークスペース
terraform workspace list
terraform workspace new dev
terraform workspace select dev

# グラフ
terraform graph | dot -Tsvg > graph.svg
```

## 基本構文

### main.tf

```hcl
# Terraformバージョン指定
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "ap-northeast-1"
  }
}

# プロバイダー設定
provider "aws" {
  region = "ap-northeast-1"

  default_tags {
    tags = {
      Environment = "production"
      ManagedBy   = "Terraform"
    }
  }
}

# リソース
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "example-instance"
  }
}

# データソース
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# 出力
output "instance_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.example.public_ip
}
```

### variables.tf

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Instance type must be t2.* or t3.*"
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["ap-northeast-1a", "ap-northeast-1c"]
}

variable "instance_config" {
  description = "Instance configuration"
  type = object({
    instance_type = string
    ami_id        = string
    key_name      = string
  })
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}
```

### terraform.tfvars

```hcl
environment      = "production"
instance_type    = "t3.medium"
instance_count   = 3
enable_monitoring = true

tags = {
  Project = "MyApp"
  Owner   = "DevOps"
}

availability_zones = [
  "ap-northeast-1a",
  "ap-northeast-1c",
  "ap-northeast-1d"
]

instance_config = {
  instance_type = "t3.medium"
  ami_id        = "ami-0c55b159cbfafe1f0"
  key_name      = "my-key"
}
```

### outputs.tf

```hcl
output "instance_ids" {
  description = "IDs of EC2 instances"
  value       = aws_instance.example[*].id
}

output "instance_public_ips" {
  description = "Public IP addresses"
  value       = aws_instance.example[*].public_ip
}

output "instance_private_ips" {
  description = "Private IP addresses"
  value       = aws_instance.example[*].private_ip
}

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "db_endpoint" {
  description = "Database endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}
```

## リソース

### EC2インスタンス

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  key_name      = aws_key_pair.deployer.key_name

  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public.id

  user_data = <<-EOF
              #!/bin/bash
              apt-get update
              apt-get install -y nginx
              EOF

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
  }

  tags = {
    Name = "web-server"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# 複数インスタンス
resource "aws_instance" "workers" {
  count = var.instance_count

  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = {
    Name = "worker-${count.index + 1}"
  }
}

# for_each を使用
resource "aws_instance" "servers" {
  for_each = toset(["web", "api", "worker"])

  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = {
    Name = each.key
  }
}
```

### VPC

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-northeast-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

### セキュリティグループ

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description     = "SSH from bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-sg"
  }
}
```

### S3バケット

```hcl
resource "aws_s3_bucket" "assets" {
  bucket = "my-app-assets-${var.environment}"

  tags = {
    Name = "Assets bucket"
  }
}

resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "assets" {
  bucket = aws_s3_bucket.assets.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## モジュール

### モジュール作成

```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = {
    Name = var.vpc_name
  }
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.vpc_name}-public-${count.index + 1}"
  }
}

# modules/vpc/variables.tf
variable "vpc_name" {
  description = "VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
}

variable "public_subnet_cidrs" {
  description = "Public subnet CIDR blocks"
  type        = list(string)
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

# modules/vpc/outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}
```

### モジュール使用

```hcl
module "vpc" {
  source = "./modules/vpc"

  vpc_name             = "production-vpc"
  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  availability_zones   = ["ap-northeast-1a", "ap-northeast-1c"]
}

# モジュールの出力参照
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_ids[0]
  # ...
}

# リモートモジュール
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-northeast-1a", "ap-northeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
}
```

## ローカル値

```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = "MyApp"
  }

  instance_name = "${var.environment}-instance"

  availability_zones = [
    for az in data.aws_availability_zones.available.names :
    az if length(regexall("ap-northeast-1[ac]", az)) > 0
  ]
}

resource "aws_instance" "example" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = merge(
    local.common_tags,
    {
      Name = local.instance_name
    }
  )
}
```

## データソース

```hcl
# AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# アベイラビリティゾーン
data "aws_availability_zones" "available" {
  state = "available"
}

# 現在のリージョン
data "aws_region" "current" {}

# 現在のアカウントID
data "aws_caller_identity" "current" {}

# VPC
data "aws_vpc" "selected" {
  id = var.vpc_id
}

# サブネット
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }

  tags = {
    Tier = "Private"
  }
}
```

## 条件式・ループ

```hcl
# 条件式
resource "aws_instance" "example" {
  count = var.create_instance ? 1 : 0

  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
}

# 三項演算子
instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

# for式（リスト）
variable "users" {
  default = ["alice", "bob", "charlie"]
}

locals {
  user_arns = [
    for user in var.users : "arn:aws:iam::123456789012:user/${user}"
  ]
}

# for式（マップ）
locals {
  instance_tags = {
    for k, v in var.tags : k => upper(v)
  }
}

# for_each
resource "aws_iam_user" "users" {
  for_each = toset(var.user_names)
  name     = each.value
}

# dynamic ブロック
resource "aws_security_group" "example" {
  name = "example"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

## 関数

```hcl
# 文字列操作
upper("hello")                    # "HELLO"
lower("HELLO")                    # "hello"
title("hello world")              # "Hello World"
format("Hello, %s!", "World")     # "Hello, World!"
join(",", ["a", "b", "c"])        # "a,b,c"
split(",", "a,b,c")               # ["a", "b", "c"]
replace("hello", "l", "L")        # "heLLo"
trimspace("  hello  ")            # "hello"

# コレクション操作
length([1, 2, 3])                 # 3
concat([1, 2], [3, 4])            # [1, 2, 3, 4]
contains([1, 2, 3], 2)            # true
merge({a=1}, {b=2})               # {a=1, b=2}
keys({a=1, b=2})                  # ["a", "b"]
values({a=1, b=2})                # [1, 2]

# 数値操作
max(1, 2, 3)                      # 3
min(1, 2, 3)                      # 1
floor(1.9)                        # 1
ceil(1.1)                         # 2

# 型変換
tostring(123)                     # "123"
tonumber("123")                   # 123
tolist(toset([1, 2, 2, 3]))       # [1, 2, 3]
tomap({a=1, b=2})

# ファイル操作
file("path/to/file.txt")
templatefile("template.tpl", {name = "World"})
fileexists("path/to/file")

# エンコーディング
base64encode("hello")
base64decode("aGVsbG8=")
jsonencode({name = "value"})
jsondecode('{"name":"value"}')
```

## バックエンド

### S3バックエンド

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### リモート状態参照

```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "ap-northeast-1"
  }
}

resource "aws_instance" "example" {
  subnet_id = data.terraform_remote_state.vpc.outputs.subnet_id
}
```

## ワークスペース

```bash
# ワークスペース一覧
terraform workspace list

# 作成
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# 切り替え
terraform workspace select dev

# 削除
terraform workspace delete dev

# 現在のワークスペース
terraform workspace show
```

```hcl
# コード内でワークスペース参照
resource "aws_instance" "example" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"

  tags = {
    Name        = "instance-${terraform.workspace}"
    Environment = terraform.workspace
  }
}
```

## State管理

```bash
# 状態一覧
terraform state list

# リソース詳細
terraform state show aws_instance.example

# リソース削除（状態のみ）
terraform state rm aws_instance.example

# リソース移動
terraform state mv aws_instance.old aws_instance.new

# 状態の取り込み
terraform import aws_instance.example i-1234567890abcdef0

# 状態のプル
terraform state pull

# 状態のプッシュ
terraform state push

# リソースの状態削除後に再作成
terraform taint aws_instance.example
terraform untaint aws_instance.example

# リソースのリフレッシュ
terraform refresh
```

## プロビジョナー

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  # ファイルコピー
  provisioner "file" {
    source      = "script.sh"
    destination = "/tmp/script.sh"

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  # リモート実行
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/script.sh",
      "/tmp/script.sh",
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  # ローカル実行
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt"
  }

  # 破棄時の処理
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance destroyed'"
  }
}
```

## よく使うコマンド

```bash
# フォーマット＋バリデーション
terraform fmt -recursive && terraform validate

# プラン保存＋適用
terraform plan -out=tfplan
terraform apply tfplan

# 特定リソースのみ適用
terraform apply -target=aws_instance.example

# 変数指定
terraform apply -var="instance_type=t3.large"
terraform apply -var-file="production.tfvars"

# 状態確認
terraform show
terraform state list
terraform output

# グラフ生成
terraform graph | dot -Tsvg > graph.svg

# プロバイダー更新
terraform init -upgrade

# モジュール更新
terraform get -update
```

## Tips

```bash
# 環境変数で認証情報設定
export AWS_ACCESS_KEY_ID="xxx"
export AWS_SECRET_ACCESS_KEY="xxx"
export AWS_DEFAULT_REGION="ap-northeast-1"

# Terraform変数を環境変数で設定
export TF_VAR_instance_type="t3.medium"
export TF_VAR_environment="production"

# ログレベル設定
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log

# 並列実行数制限
terraform apply -parallelism=10

# 状態ロック無効化（非推奨）
terraform apply -lock=false

# 入力プロンプトスキップ
terraform apply -auto-approve -input=false
```
