# wildrydes-site

## 概要
AWSでのアプリ構築の自己学習を目的として構築しました。<br>
参考：[サーバーレスのウェブアプリケーション構築](https://aws.amazon.com/jp/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/)

## 技術
- AWS Amplify
- Amazon Cognito
- AWS Lambda
- Amazon DynamoDB
- Amazon API Gateway



## 手順

### 事前設定
- AWS CLIインストール<br>
[AWS公式ガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

- AWS CLIの認証情報設定<br>
[AWS公式ガイド](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sign-in.html)


- ArcGIS 登録アカウント<br>
[登録サイト](https://www.arcgis.com/sharing/rest/oauth2/authorize?client_id=arcgisonline&response_type=token&display=default&state=%7B%22portalUrl%22%3A%22https%3A%2F%2Fwww.arcgis.com%22%2C%22useLandingPage%22%3Atrue%2C%22clientId%22%3A%22arcgisonline%22%7D&expiration=20160&locale=ja&redirect_uri=https%3A%2F%2Fwww.arcgis.com%2Fhome%2Faccountswitcher-callback.html&force_login=true&hideCancel=true&showSignupOption=true&canHandleCrossOrgSignIn=true&signuptype=esri&redirectToUserOrgUrl=true&allow_verification=true)

### モジュール 1: 継続的デプロイを使用した静的ウェブホスティング
- リージョンの選択：ap-northeast-1
- Gitリポジトリ作成
- Gitリポジトリの事前設定
  - このチュートリアルに関連付けられた一般がアクセスできる既存の S3 バケットからウェブサイトのコンテンツをコピーし、リポジトリにコンテンツを追加
- AWS Amplify コンソールでウェブホスティングを有効にする

## エラーログ
### 発生場所：S3 から静的ファイルをコピー
`aws s3 cp s3://wildrydes-ap-northeast-1/WebApplication/1_StaticWebHosting/website ./ --recursive`を実行したところエラー発生。

エラー文：`fatal error: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied`

[AWS re:POST](https://repost.aws/questions/QUKwoXr7h-RdO9uKdYYzfwjw/build-a-serverless-web-application-errors-on-copying-s3-file-wildrydes)より、
- IAMユーザーが使用しているIAMポリシーでS3への「s3:ListBucket」が許可されているかどうかを確認してください。
- 対象のS3バケットのバケットポリシーでも「s3:ListBucket」が許可されているかどうかを確認してください。

とのことだったので
- IAMポリシーを以下の記事を参考に設定
  - [ポリシーの例: &Amazon S3](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_examples.html#policy_library_S3)の中の[Amazon S3 バケットオブジェクトへの読み書きアクセスを付与する](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_examples_s3_rw-bucket.html)
  - [S3のポリシーに定義した「Resource」を正しく理解できていますか？](https://dev.classmethod.jp/articles/how-to-write-resource-of-s3-bucket-policy-and-iam-policy/)

```
{
    "Version":"2012-10-17",		 	 	 
    "Statement": [
        {
            "Sid": "ListObjectsInBucket",
            "Effect": "Allow",
            "Action": ["s3:ListBucket"],
            "Resource": ["arn:aws:s3:::wildrydes-us-east-1"]
        },
        {
            "Sid": "AllObjectActions",
            "Effect": "Allow",
            "Action": "s3:*Object",
            "Resource": ["arn:aws:s3:::wildrydes-us-east-1/*"]
        }
    ]
}
```
しかしこれではエラー解決せず。<br>
次にGitHubのaws-samples公式リポジトリのIssue「[S3からWildrydesファイルをコピーしようとするとアクセスが拒否され失敗する](https://github.com/aws-samples/aws-serverless-workshops/issues/292)」を参照し、IAMユーザーにAmazonS3ReadOnlyAccessポリシーを付与することで解決したようだが、私のIAMではすでにそのポリシーは存在した。<br>
[最新のIssue](https://github.com/aws-samples/aws-serverless-workshops/issues/292#issuecomment-1942911448)にて、「the problem is that the s3 bucketv isn't publicly accessible anymore. in order to find it check it here`aws s3 cp s3://ttt-wildrydes/wildrydes-site ./ --recursive`」とあり、これで解決。

