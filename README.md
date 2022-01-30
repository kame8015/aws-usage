**Note to self: How to use AWS Services**
# AWS Services

## Table Of Contents
- [Lambda](#Lambda)
- [API Gateway](#Lambda)
- EC2
- Lightsail
- [Auto Scaling](#AutoScaling)
- [Fargate](#Container)
- [Elastic Container Service (ECS)](#Container)
- [Elastic Container Registry (ECR)](#Container)
- [AWS Batch](#Container)
- [IAM](#IAM)
- Cognito
- CloudFormation
- DynamoDB
- RDS (Aurora)
- CloudWatch (Logs / Event / Dashboard)
- CodeCommit
- CodeBuild
- CodeDeploy
- CodePipeline
- S3
- VPC
- Route 53
- CloudFront
- Systems Manager
- Secrets Manager
- Key Management Service (KMS)
- Simple Queue Service (SQS)
- Amazon Simple Email Service (SES)
- WAF
- SNS (Simple Notification Service)
- CloudShell
- Amazon SageMaker


## Lambda
### Features
サーバレスの FaaS(Function as a Service)。サーバレスであるので価格が安い。  
リクエスト数に応じて自動的にスケーリングしてくれる。  
基本的にはコールドスタートするので、最初に起動時は即時実行とはならない点に注意。  
また実行時間上限が15分であることにも注意。  
実行するためにはトリガーが必要。  
バージョンやエイリアスの概念があり、Lambda 内でバージョン管理できる。  
トリガーから Lambda のハンドラーにイベントを渡せる。
### Usage
#### API Gateway + Lambda + DynamoDB
![](img/apigateway_lambda.svg)

例えば API Gateway と Lambda のプロキシ統合により、すぐ API が作成できる。
API Gateway から Lambda ハンドラーのイベントに渡ってくる形式は以下の通り。
```json
{
    "resource": "/my/path",
    "path": "/my/path",
    "httpMethod": "GET",
    "headers": {
        "header1": "value1",
        "header2": "value2"
    },
    "multiValueHeaders": {
        "header1": [
        "value1"
        ],
        "header2": [
        "value1",
        "value2"
        ]
    },
    "queryStringParameters": {
        "parameter1": "value1",
        "parameter2": "value"
    },
    "multiValueQueryStringParameters": {
        "parameter1": [
        "value1",
        "value2"
        ],
        "parameter2": [
        "value"
        ]
    },
    "requestContext": {
        "accountId": "123456789012",
        "apiId": "id",
        "authorizer": {
        "claims": null,
        "scopes": null
        },
        "domainName": "id.execute-api.us-east-1.amazonaws.com",
        "domainPrefix": "id",
        "extendedRequestId": "request-id",
        "httpMethod": "GET",
        "identity": {
        "accessKey": null,
        "accountId": null,
        "caller": null,
        "cognitoAuthenticationProvider": null,
        "cognitoAuthenticationType": null,
        "cognitoIdentityId": null,
        "cognitoIdentityPoolId": null,
        "principalOrgId": null,
        "sourceIp": "IP",
        "user": null,
        "userAgent": "user-agent",
        "userArn": null,
        "clientCert": {
            "clientCertPem": "CERT_CONTENT",
            "subjectDN": "www.example.com",
            "issuerDN": "Example issuer",
            "serialNumber": "a1:a1:a1:a1:a1:a1:a1:a1:a1:a1:a1:a1:a1:a1:a1:a1",
            "validity": {
            "notBefore": "May 28 12:30:02 2019 GMT",
            "notAfter": "Aug  5 09:36:04 2021 GMT"
            }
        }
        },
        "path": "/my/path",
        "protocol": "HTTP/1.1",
        "requestId": "id=",
        "requestTime": "04/Mar/2020:19:15:17 +0000",
        "requestTimeEpoch": 1583349317135,
        "resourceId": null,
        "resourcePath": "/my/path",
        "stage": "$default"
    },
    "pathParameters": null,
    "stageVariables": null,
    "body": "Hello from Lambda!",
    "isBase64Encoded": false
}
```
例えば Lambda でクエリパラメータを取得して、DynamoDB からレコードを取得したかったら、
```python
# index.py
import boto3
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    query_param_1 = event["queryStringParameters"]["parameter1"]
    dynamodb = boto3.resource("dynamodb")
    table = dynamodb.Table("tableName")
    try:
        response = table.get_item(Key={"name": query_param_1})
    except ClientError as e:
        print(e.response["Error"]["Message"])
    else:
        return response["Item"]
```
こうすれば、API Gateway のレスポンスボディに json 形式でデータが渡される。

#### CodePipeline + Lambda
![](img/codepipeline_lambda.svg)

CI/CD 環境の構築でパイプラインを作成することがあり、  
その際に CodePipeline から Lambda をトリガーする場合。  
Lambda の event に json イベントを渡す。  
CodePipeline から "UserParameters" に値を渡すことが可能。
```json
{
    "CodePipeline.job": {
        "id": "11111111-abcd-1111-abcd-111111abcdef",
        "accountId": "111111111111",
        "data": {
            "actionConfiguration": {
                "configuration": {
                    "FunctionName": "MyLambdaFunctionForAWSCodePipeline",
                    "UserParameters": "some-input-such-as-a-URL"
                }
            },
            "inputArtifacts": [
                {
                    "location": {
                        "s3Location": {
                            "bucketName": "the name of the bucket configured as the pipeline artifact store in Amazon S3, for example codepipeline-us-east-2-1234567890",
                            "objectKey": "the name of the application, for example CodePipelineDemoApplication.zip"
                        },
                        "type": "S3"
                    },
                    "revision": null,
                    "name": "ArtifactName"
                }
            ],
            "outputArtifacts": [],
            "artifactCredentials": {
                "secretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
                "sessionToken": "MIICiTCCAfICCQD6m7oRw0uXOjANBgkqhkiG9w~~~",
                "accessKeyId": "AKIAIOSFODNN7EXAMPLE"
            },
            "continuationToken": "A continuation token if continuing job",
            "encryptionKey": {
            "id": "arn:aws:kms:us-west-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab",
            "type": "KMS"
            }
        }
    }
}
```

### LambdaからAuroraに接続する方法
従来、LambdaからRDSに接続するのはアンチパターンであった。  
具体的な接続方法はIAM認証トークンを使ってRDSに接続するというもの。

[IAM 認証および AWS SDK for Python (Boto3) を使用した DB clusterへの接続](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/UsingWithRDS.IAMDBAuth.Connecting.Python.html)

その理由は、Lambdaの実行のたびにRDSのコネクションを貼り直してしまうため、  
コネクション数が増加してすぐDBコネクションエラーになってしまっていたからである。  
しかし、RDS Proxyが登場したことにより、LambdaとRDSのコネクションプールの役目をRDS Proxyが担うことができるようになり、  コネクションを有効活用してRDSにアクセスできるようになった。  
料金体系としては DBインスタンスのvCPU1つにつき1時間0.018USDである。(ただし最低料金として2つのvCPU分の料金がかかる)  
なのでvCPUが4の場合、1ヶ月約5,700円ほどかかる見込みである。  
よってLambdaはクーロンジョブで実行するだけだから、**そこまでコネクションを貼らないということが想定されるのであれば**使用することは検討の余地がありそう。

実際の構築は下記のQiitaの記事が参考になりそう。

- [祝GA‼︎【Go】Lambda + RDS 接続にRDS Proxyを使ってみた](https://qiita.com/maika_kamada/items/6eb6a40c17b4b8995acb)
- [【AWS】RDS Proxyを使用してLambdaからAuroraに接続する](https://ichi-station.com/aws-rds-proxy-lambda-aurora/)

注意点としては、ざっくり下記の点が挙げられる。
- LambdaをAuroraとRDS Proxyと同じVPCに入れること
- Lambdaを踏み台サーバと同じパブリックのサブネットに入れること
- Lambdaに与えるセキュリティグループを作成して、RDS Proxyのインバウンドルールに与えること
- RDS ProxyをAuroraと同じプライベートサブネットに入れること
- RDS Proxyに与えるセキュリティグループを差k末井して、Auroraのインバウンドルールに与えること
- LambdaからDB接続する際はpymysqlというライブラリを使用するが、Lambdaのライブラリに入っていないので、zipでまとめてインポートする必要がある

## AutoScaling
EC2インスタンスの数を監視し、自動的にスケールアウトしたりスケールインしたりする。  
インスタンスの数は、最大サイズ、最小サイズの範囲で希望サイズの分だけ指定したAMIから起動する設定ができる。  
ELBに紐付けることもでき、ELBからのリクエストをAutoScalingGroup管轄のインスタンスに振り分けることができる。  
既に起動しているインスタンスをAutoScalingGroupにアタッチすることもできるし、その逆もできる。またインスタンスを複数のアベイラビリティーゾーンで使用する設定も可能。  
またEC2の起動設定と同様に、インスタンス起動時に実行するユーザデータも設定することができる。  
希望数量を増減するにはいくつか方法があり、手動で行うか、スケジュールで自動的に行うか、標準メトリクスやカスタムメトリクスをCloudWatchで収集してアラームを発火することでスケールアウト・スケールインを行うことができる。
![](img/autoscalinggroup.svg)

## Container
ローカルPCでDockerコンテナを起動するように、AWS上で行う方法を記載する。

### ECS
![](img/ecs.svg)
Dockerでイメージを作成して、ECR(Elastic Container Registry)にプッシュして登録する。  
登録されたイメージをECSにpullしてきてコンテナを起動する。

ECSオブジェクトの関連図
![](img/ecs-object.png)

ECSのエンティティにはクラスター、サービス、タスク定義、コンテナ定義がある。  
それぞれのエンティティに関する説明は下記の記事が参考になる。

[Amazon EC2 Container Service(ECS)の概念整理](https://qiita.com/NewGyu/items/9597ed2eda763bd504d7)

ざっくりと説明すると、クラスターの上でサービスを動作させ、ECRからイメージをプルしてきてタスクの中でコンテナを起動する、というもの。

例えばLambdaからECSのタスクを起動することができ、ついでにパラメータも渡すことができる。

### AWS Batch
AWS BatchはざっくりというとECSにQueueの機能があるイメージ。

AWS Batchのエンティティにはジョブ、ジョブ定義、ジョブキュー、コンピュータ環境がある。  
これも以下の記事が参考になる。

[AWS Batchに入門してみた](https://zenn.dev/nameless_gyoza/articles/what-is-aws-batch-20210327)

ジョブの状態がRUNNABLEで止まることがあり、それはコンピュータ環境やIAMの設定が問題であることが多いのでその都度調べる。

## IAM
Identity and Access Managementというもので、AWSのサービスで「認証」と「認可」の設定を行うことができる。
- 認証: 相手が誰(何)なのか確認すること
- 認可: リソースへのアクセス権限を与えること

IAMでは以下の機能がある。
- IAMユーザ(グループ)
- IAMポリシー
- IAMロール

### IAMユーザ
AWSを利用するアカウント。また、IAMグループを作成して複数のIAMユーザをまとめて管理することができる。部署単位で権限を分けたい場合などに利用する。

### IAMポリシー
IAMユーザやIAMロールにアタッチすることができる。AWSリソースへの操作権限を設定する機能。
- AWS管理ポリシー
    - AWSが提供するIAMポリシー。各サービスに対して大まかな制御ポリシーが設定できる
- カスタマー管理ポリシー
    - JSONファイルなどを利用して設定するポリシー。AWS管理ポリシーよりも細やかな制御が可能
- インラインポリシー
    - 特定のIAMユーザやIAMロール専用に作成されるポリシー

### IAMロール
ユーザやグループではなく、EC2などのAWSのサービスや他のアカウントに対してAWSの操作権限を付与するための仕組み。  
イメージとしてはIAMユーザがかぶる帽子のようなもの。例えば複数のIAMロールを作っておいて、スイッチロールすることによって一つのIAMユーザで権限を使い分けることができる。

IAMのベストプラクティス
- IAMユーザやIAMロールに設定する権限は最小限にする
- 多要素認証(MFA)をできるだけ行う
- CloudTrailでモニタリングする
    - パスワードが漏洩したときにどのような操作が行われたかを記録しておくことができる
    - SNSで通知を行うことができる
- アクセスキーは極力使わない
    - Gitにアクセスキーをあげてしまうと、クローラーに引っかかって漏洩する可能性が高い

引用: [【AWS IAMとは？】初心者にもわかりやすく解説](https://www.wafcharm.com/blog/aws-iam-for-beginners/)