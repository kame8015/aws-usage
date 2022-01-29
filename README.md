**Note to self: How to use AWS Services**

# AWS Services

## Table Of Contents
- [Lambda](#Lambda)


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
