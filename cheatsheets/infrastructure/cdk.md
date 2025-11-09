# AWS CDK チートシート

## セットアップ

```bash
# CDKインストール
npm install -g aws-cdk

# バージョン確認
cdk --version

# プロジェクト初期化
cdk init app --language typescript
cdk init app --language python
cdk init app --language java

# 依存関係インストール
npm install  # TypeScript
pip install -r requirements.txt  # Python

# CDK bootstrap（初回のみ）
cdk bootstrap aws://ACCOUNT-ID/REGION
cdk bootstrap aws://123456789012/ap-northeast-1
```

## 基本コマンド

```bash
# ビルド（TypeScript）
npm run build

# スタック一覧
cdk list
cdk ls

# CloudFormation テンプレート生成
cdk synth
cdk synth MyStack

# デプロイ
cdk deploy
cdk deploy MyStack
cdk deploy --all
cdk deploy --require-approval never

# 差分確認
cdk diff
cdk diff MyStack

# スタック削除
cdk destroy
cdk destroy MyStack
cdk destroy --all

# ドキュメント表示
cdk doc

# コンテキスト管理
cdk context
cdk context --clear
```

## TypeScript

### 基本構成

```typescript
// bin/my-app.ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { MyStack } from '../lib/my-stack';

const app = new cdk.App();
new MyStack(app, 'MyStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
  tags: {
    Environment: 'production',
    ManagedBy: 'CDK',
  },
});
```

```typescript
// lib/my-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class MyStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, 'MyVpc', {
      maxAzs: 2,
      natGateways: 1,
    });

    // S3バケット
    const bucket = new s3.Bucket(this, 'MyBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    // 出力
    new cdk.CfnOutput(this, 'BucketName', {
      value: bucket.bucketName,
      description: 'S3 bucket name',
    });
  }
}
```

### EC2

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';

// VPC
const vpc = new ec2.Vpc(this, 'MyVpc', {
  ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
  maxAzs: 2,
  natGateways: 1,
  subnetConfiguration: [
    {
      name: 'Public',
      subnetType: ec2.SubnetType.PUBLIC,
      cidrMask: 24,
    },
    {
      name: 'Private',
      subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
      cidrMask: 24,
    },
  ],
});

// セキュリティグループ
const sg = new ec2.SecurityGroup(this, 'WebServerSG', {
  vpc,
  description: 'Security group for web server',
  allowAllOutbound: true,
});

sg.addIngressRule(
  ec2.Peer.anyIpv4(),
  ec2.Port.tcp(80),
  'Allow HTTP traffic'
);

sg.addIngressRule(
  ec2.Peer.anyIpv4(),
  ec2.Port.tcp(443),
  'Allow HTTPS traffic'
);

// EC2インスタンス
const instance = new ec2.Instance(this, 'WebServer', {
  vpc,
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.T3,
    ec2.InstanceSize.MICRO
  ),
  machineImage: ec2.MachineImage.latestAmazonLinux2(),
  securityGroup: sg,
  keyName: 'my-key',
  userData: ec2.UserData.custom(`#!/bin/bash
    yum update -y
    yum install -y nginx
    systemctl start nginx
    systemctl enable nginx
  `),
});

// Elastic IP
const eip = new ec2.CfnEIP(this, 'WebServerEIP');
new ec2.CfnEIPAssociation(this, 'EIPAssociation', {
  eip: eip.ref,
  instanceId: instance.instanceId,
});
```

### S3

```typescript
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as s3deploy from 'aws-cdk-lib/aws-s3-deployment';

// S3バケット
const bucket = new s3.Bucket(this, 'MyBucket', {
  bucketName: 'my-app-bucket',
  versioned: true,
  encryption: s3.BucketEncryption.S3_MANAGED,
  publicReadAccess: false,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  autoDeleteObjects: true,
  lifecycleRules: [
    {
      expiration: cdk.Duration.days(90),
      transitions: [
        {
          storageClass: s3.StorageClass.INFREQUENT_ACCESS,
          transitionAfter: cdk.Duration.days(30),
        },
      ],
    },
  ],
});

// 静的ウェブサイトホスティング
const websiteBucket = new s3.Bucket(this, 'WebsiteBucket', {
  websiteIndexDocument: 'index.html',
  websiteErrorDocument: 'error.html',
  publicReadAccess: true,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

// ファイルデプロイ
new s3deploy.BucketDeployment(this, 'DeployWebsite', {
  sources: [s3deploy.Source.asset('./website')],
  destinationBucket: websiteBucket,
});
```

### Lambda

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambdaNodejs from 'aws-cdk-lib/aws-lambda-nodejs';

// Lambda関数
const fn = new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_18_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  environment: {
    TABLE_NAME: 'my-table',
  },
  timeout: cdk.Duration.seconds(30),
  memorySize: 256,
});

// Node.js Lambda（自動バンドル）
const nodejsFn = new lambdaNodejs.NodejsFunction(this, 'NodeFunction', {
  entry: 'lambda/index.ts',
  handler: 'handler',
  runtime: lambda.Runtime.NODEJS_18_X,
  bundling: {
    minify: true,
    sourceMap: true,
  },
});

// Python Lambda
const pythonFn = new lambda.Function(this, 'PythonFunction', {
  runtime: lambda.Runtime.PYTHON_3_11,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda', {
    bundling: {
      image: lambda.Runtime.PYTHON_3_11.bundlingImage,
      command: [
        'bash', '-c',
        'pip install -r requirements.txt -t /asset-output && cp -au . /asset-output'
      ],
    },
  }),
});

// Lambda Layer
const layer = new lambda.LayerVersion(this, 'MyLayer', {
  code: lambda.Code.fromAsset('layer'),
  compatibleRuntimes: [lambda.Runtime.NODEJS_18_X],
  description: 'Common utilities',
});

fn.addLayers(layer);
```

### API Gateway

```typescript
import * as apigateway from 'aws-cdk-lib/aws-apigateway';

// REST API
const api = new apigateway.RestApi(this, 'MyApi', {
  restApiName: 'My Service',
  description: 'This service serves...',
  deployOptions: {
    stageName: 'prod',
    throttlingRateLimit: 100,
    throttlingBurstLimit: 200,
  },
  defaultCorsPreflightOptions: {
    allowOrigins: apigateway.Cors.ALL_ORIGINS,
    allowMethods: apigateway.Cors.ALL_METHODS,
  },
});

// Lambda統合
const integration = new apigateway.LambdaIntegration(fn);

// リソース・メソッド
const items = api.root.addResource('items');
items.addMethod('GET', integration);
items.addMethod('POST', integration);

const item = items.addResource('{id}');
item.addMethod('GET', integration);
item.addMethod('PUT', integration);
item.addMethod('DELETE', integration);

// HTTP API
import * as apigatewayv2 from 'aws-cdk-lib/aws-apigatewayv2';
import * as integrations from 'aws-cdk-lib/aws-apigatewayv2-integrations';

const httpApi = new apigatewayv2.HttpApi(this, 'HttpApi', {
  apiName: 'my-http-api',
  corsPreflight: {
    allowOrigins: ['*'],
    allowMethods: [apigatewayv2.CorsHttpMethod.ANY],
  },
});

httpApi.addRoutes({
  path: '/items',
  methods: [apigatewayv2.HttpMethod.GET],
  integration: new integrations.HttpLambdaIntegration('GetItems', fn),
});
```

### DynamoDB

```typescript
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

// DynamoDBテーブル
const table = new dynamodb.Table(this, 'MyTable', {
  tableName: 'my-table',
  partitionKey: {
    name: 'id',
    type: dynamodb.AttributeType.STRING,
  },
  sortKey: {
    name: 'timestamp',
    type: dynamodb.AttributeType.NUMBER,
  },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  pointInTimeRecovery: true,
  stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
});

// グローバルセカンダリインデックス
table.addGlobalSecondaryIndex({
  indexName: 'email-index',
  partitionKey: {
    name: 'email',
    type: dynamodb.AttributeType.STRING,
  },
  projectionType: dynamodb.ProjectionType.ALL,
});

// Lambda に読み取り権限付与
table.grantReadData(fn);

// Lambda に読み書き権限付与
table.grantReadWriteData(fn);
```

### RDS

```typescript
import * as rds from 'aws-cdk-lib/aws-rds';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';

// データベース認証情報
const dbCredentials = new secretsmanager.Secret(this, 'DBCredentials', {
  generateSecretString: {
    secretStringTemplate: JSON.stringify({ username: 'admin' }),
    generateStringKey: 'password',
    excludePunctuation: true,
  },
});

// PostgreSQL
const postgres = new rds.DatabaseInstance(this, 'PostgresDB', {
  engine: rds.DatabaseInstanceEngine.postgres({
    version: rds.PostgresEngineVersion.VER_15_3,
  }),
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.T3,
    ec2.InstanceSize.MICRO
  ),
  vpc,
  credentials: rds.Credentials.fromSecret(dbCredentials),
  databaseName: 'mydb',
  allocatedStorage: 20,
  maxAllocatedStorage: 100,
  multiAz: false,
  publiclyAccessible: false,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
  deletionProtection: false,
});

// Aurora Serverless
const cluster = new rds.ServerlessCluster(this, 'AuroraCluster', {
  engine: rds.DatabaseClusterEngine.auroraPostgres({
    version: rds.AuroraPostgresEngineVersion.VER_13_9,
  }),
  vpc,
  credentials: rds.Credentials.fromSecret(dbCredentials),
  defaultDatabaseName: 'mydb',
  scaling: {
    minCapacity: rds.AuroraCapacityUnit.ACU_2,
    maxCapacity: rds.AuroraCapacityUnit.ACU_16,
  },
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

### ECS

```typescript
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ecsPatterns from 'aws-cdk-lib/aws-ecs-patterns';

// ECSクラスタ
const cluster = new ecs.Cluster(this, 'MyCluster', {
  vpc,
  clusterName: 'my-cluster',
});

// Fargate サービス with ALB
const fargateService = new ecsPatterns.ApplicationLoadBalancedFargateService(
  this,
  'MyFargateService',
  {
    cluster,
    cpu: 256,
    memoryLimitMiB: 512,
    desiredCount: 2,
    taskImageOptions: {
      image: ecs.ContainerImage.fromRegistry('nginx'),
      containerPort: 80,
      environment: {
        ENV: 'production',
      },
    },
    publicLoadBalancer: true,
  }
);

// オートスケーリング
const scaling = fargateService.service.autoScaleTaskCount({
  minCapacity: 1,
  maxCapacity: 10,
});

scaling.scaleOnCpuUtilization('CpuScaling', {
  targetUtilizationPercent: 70,
});

scaling.scaleOnMemoryUtilization('MemoryScaling', {
  targetUtilizationPercent: 80,
});
```

### CloudFront

```typescript
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import * as origins from 'aws-cdk-lib/aws-cloudfront-origins';

// CloudFront Distribution
const distribution = new cloudfront.Distribution(this, 'MyDistribution', {
  defaultBehavior: {
    origin: new origins.S3Origin(bucket),
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
  },
  defaultRootObject: 'index.html',
  errorResponses: [
    {
      httpStatus: 404,
      responseHttpStatus: 200,
      responsePagePath: '/index.html',
    },
  ],
});

// カスタムドメイン
import * as acm from 'aws-cdk-lib/aws-certificatemanager';
import * as route53 from 'aws-cdk-lib/aws-route53';

const certificate = acm.Certificate.fromCertificateArn(
  this,
  'Certificate',
  'arn:aws:acm:us-east-1:123456789012:certificate/xxx'
);

const distributionWithDomain = new cloudfront.Distribution(this, 'Distribution', {
  defaultBehavior: {
    origin: new origins.S3Origin(bucket),
  },
  domainNames: ['example.com', 'www.example.com'],
  certificate,
});
```

## Python

```python
# app.py
#!/usr/bin/env python3
import aws_cdk as cdk
from my_stack import MyStack

app = cdk.App()
MyStack(app, "MyStack",
    env=cdk.Environment(
        account=os.getenv('CDK_DEFAULT_ACCOUNT'),
        region=os.getenv('CDK_DEFAULT_REGION')
    ),
    tags={
        'Environment': 'production',
        'ManagedBy': 'CDK'
    }
)

app.synth()
```

```python
# my_stack.py
from aws_cdk import (
    Stack,
    aws_s3 as s3,
    aws_lambda as _lambda,
    aws_apigateway as apigw,
    RemovalPolicy,
    Duration,
)
from constructs import Construct

class MyStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)

        # S3バケット
        bucket = s3.Bucket(
            self, "MyBucket",
            versioned=True,
            encryption=s3.BucketEncryption.S3_MANAGED,
            removal_policy=RemovalPolicy.DESTROY,
            auto_delete_objects=True
        )

        # Lambda関数
        fn = _lambda.Function(
            self, "MyFunction",
            runtime=_lambda.Runtime.PYTHON_3_11,
            handler="index.handler",
            code=_lambda.Code.from_asset("lambda"),
            timeout=Duration.seconds(30),
            environment={
                "BUCKET_NAME": bucket.bucket_name
            }
        )

        # S3への権限付与
        bucket.grant_read_write(fn)

        # API Gateway
        api = apigw.LambdaRestApi(
            self, "MyApi",
            handler=fn,
            proxy=False
        )

        items = api.root.add_resource("items")
        items.add_method("GET")
        items.add_method("POST")
```

## Constructs

### カスタムConstruct

```typescript
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';

export interface StaticWebsiteProps {
  domainName?: string;
}

export class StaticWebsite extends Construct {
  public readonly bucket: s3.Bucket;
  public readonly distribution: cloudfront.Distribution;

  constructor(scope: Construct, id: string, props?: StaticWebsiteProps) {
    super(scope, id);

    this.bucket = new s3.Bucket(this, 'Bucket', {
      websiteIndexDocument: 'index.html',
      publicReadAccess: true,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    this.distribution = new cloudfront.Distribution(this, 'Distribution', {
      defaultBehavior: {
        origin: new origins.S3Origin(this.bucket),
      },
      domainNames: props?.domainName ? [props.domainName] : undefined,
    });
  }
}

// 使用
const website = new StaticWebsite(this, 'MyWebsite', {
  domainName: 'example.com',
});
```

## コンテキスト・パラメータ

```typescript
// cdk.json
{
  "context": {
    "environment": "production",
    "vpc-id": "vpc-12345678"
  }
}

// コード内で取得
const environment = this.node.tryGetContext('environment');
const vpcId = this.node.tryGetContext('vpc-id');

// コマンドラインから指定
cdk deploy -c environment=staging

// パラメータ
const param = new cdk.CfnParameter(this, 'InstanceType', {
  type: 'String',
  default: 't3.micro',
  allowedValues: ['t3.micro', 't3.small', 't3.medium'],
});

new ec2.Instance(this, 'Instance', {
  instanceType: new ec2.InstanceType(param.valueAsString),
  // ...
});
```

## アスペクト

```typescript
import { IAspect, Annotations } from 'aws-cdk-lib';
import { IConstruct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';

class BucketVersioningChecker implements IAspect {
  public visit(node: IConstruct): void {
    if (node instanceof s3.CfnBucket) {
      if (!node.versioningConfiguration ||
          !node.versioningConfiguration.status === 'Enabled') {
        Annotations.of(node).addWarning('Bucket versioning is not enabled');
      }
    }
  }
}

// 適用
Aspects.of(this).add(new BucketVersioningChecker());
```

## Testing

```typescript
// test/my-stack.test.ts
import * as cdk from 'aws-cdk-lib';
import { Template } from 'aws-cdk-lib/assertions';
import { MyStack } from '../lib/my-stack';

test('S3 Bucket Created', () => {
  const app = new cdk.App();
  const stack = new MyStack(app, 'MyTestStack');
  const template = Template.fromStack(stack);

  template.resourceCountIs('AWS::S3::Bucket', 1);
});

test('Bucket has versioning enabled', () => {
  const app = new cdk.App();
  const stack = new MyStack(app, 'MyTestStack');
  const template = Template.fromStack(stack);

  template.hasResourceProperties('AWS::S3::Bucket', {
    VersioningConfiguration: {
      Status: 'Enabled',
    },
  });
});

// テスト実行
npm test
```

## Tips

```bash
# 環境変数でAWS認証情報設定
export AWS_PROFILE=myprofile
export AWS_REGION=ap-northeast-1

# CDKコンテキストクリア
cdk context --clear

# 特定のスタックのみデプロイ
cdk deploy MyStack

# パラメータ指定
cdk deploy --parameters InstanceType=t3.large

# ホットスワップ（開発時のみ）
cdk deploy --hotswap

# ロールバック
cdk deploy --rollback true

# CloudFormation テンプレート出力
cdk synth > template.yaml

# バージョン固定（package.json）
{
  "dependencies": {
    "aws-cdk-lib": "2.100.0",
    "constructs": "^10.0.0"
  }
}

# 複数スタックのデプロイ順序制御
stack2.addDependency(stack1);
```

## よく使うパターン

```typescript
// 環境別設定
const isProd = app.node.tryGetContext('environment') === 'production';

const stack = new MyStack(app, 'MyStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
  instanceType: isProd ? 't3.large' : 't3.micro',
  replicaCount: isProd ? 3 : 1,
});

// タグの一括付与
cdk.Tags.of(app).add('Project', 'MyProject');
cdk.Tags.of(stack).add('Environment', 'production');

// リソース名の命名規則
const resourceName = (name: string) =>
  `${stackName}-${environment}-${name}`;
```
