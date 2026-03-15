# wildrydes-site

## 目次
- [このリポジトリについて](#このリポジトリについて)
- [学んだこと・気づき](#学んだこと気づき)
- [アプリケーションのアーキテクチャ](#アプリケーションのアーキテクチャ)
- [機能](#機能)
- [手順](#手順)
- [エラー対応](#エラー対応)

## このリポジトリについて
以下のハンズオンを参考に、AWSのサーバーレス構成を実際に手を動かして理解するために取り組みました。<br>
チュートリアルの通りに手を動かした形ですが、現在は動作しない箇所は都度調べて修正しながら完走しました。<br>
[サーバーレスのウェブアプリケーション構築](https://aws.amazon.com/jp/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/)

## 学んだこと・気づき
- Lambda + DynamoDB + API Gateway の連携の流れ
- Amazon Cognitoを使ってAPI Gatewayでリクエストを認証する仕組みの把握
- フロント（GitHub/Amplify）とバックエンド（AWS）を分離して管理・接続する構成の把握
- IAMロールの権限設計の考え方
- 上手く動かないとき、公式ドキュメントだけでなくIssueやコミュニティを調べる重要性

## アプリケーションのアーキテクチャ
- AWS Amplify
  - 静的なウェブリソースのホストと継続的デプロイ
- Amazon Cognito
  - バックエンド API を保護するためのユーザー管理機能と認証機能
- AWS Lambda
  - バックエンド API の関数を定義
- Amazon DynamoDB
  - API の Lambda 関数実行時にデータを格納
- Amazon API Gateway
  - バックエンド API からデータを送受信

![](/images/README/architecture.jpg)

## 機能
- メールアドレスで認証しアカウント作成
- マップ上を押下、その後実行ボタン押下により、その地点にユニコーンが移動

## 操作イメージ
https://github.com/user-attachments/assets/4bcb0f4b-3cd6-4a72-beef-d48188e83e58

## 手順

<details>
<summary>ここをクリックで詳細手順を開く</summary>

### 事前設定
- AWS CLIインストール<br>
[AWS公式ガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

- AWS CLIの認証情報設定<br>
[AWS公式ガイド](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sign-in.html)


- ArcGIS 登録アカウント<br>
[登録サイト](https://www.arcgis.com/sharing/rest/oauth2/authorize?client_id=arcgisonline&response_type=token&display=default&state=%7B%22portalUrl%22%3A%22https%3A%2F%2Fwww.arcgis.com%22%2C%22useLandingPage%22%3Atrue%2C%22clientId%22%3A%22arcgisonline%22%7D&expiration=20160&locale=ja&redirect_uri=https%3A%2F%2Fwww.arcgis.com%2Fhome%2Faccountswitcher-callback.html&force_login=true&hideCancel=true&showSignupOption=true&canHandleCrossOrgSignIn=true&signuptype=esri&redirectToUserOrgUrl=true&allow_verification=true)

### モジュール 1: 継続的デプロイを使用した静的ウェブホスティング
- 使用技術：AWS Amplify
- 概要：継続的デプロイのビルドでウェブアプリケーションの静的リソースをホストするように AWS Amplify を設定
<details>
<summary>ここをクリックで詳細手順を開く</summary>
  
- リージョンの選択：ap-northeast-1
- Gitリポジトリ作成
- Gitリポジトリの事前設定(S3バケットからコードをコピー)
  - ~~このチュートリアルに関連付けられた一般がアクセスできる既存の S3 バケットからウェブサイトのコンテンツをコピーし、リポジトリにコンテンツを追加~~
  - [aws-samples公式リポジトリのIssue](https://github.com/aws-samples/aws-serverless-workshops/issues/292#issuecomment-1942911448)を参考に以下を実行してS3をclone（[エラー対応](#エラー対応)を参照）<br>
  `aws s3 cp s3://ttt-wildrydes/wildrydes-site ./ --recursive`
  - originを自分のリポジトリに変更し、強制push
- AWS Amplify コンソールでウェブホスティングを有効にする
  -  AWS Amplify コンソールを使用してGitHubコードをデプロイ
- サイトを変更する
  -  Amplifyでリポジトリ変更時にアプリの再ビルド、再デプロイが成功することを確認



#### アプリビルド時のタイトル画面
![アプリビルド時のタイトル画面](./images/README/title.jpg)

</details>


### モジュール 2: ユーザーを管理する
- 使用技術：Amazon Cognito
- 概要：
  - ユーザーのアカウントを管理するための Amazon Cognito ユーザープールを作成する
  - 顧客が新しいユーザーとして登録し、自分の E メールアドレスを確認して、サイトにサインインできるようにするページを配置
    - ユーザーが登録を送信した後、Amazon Cognito は提供されたアドレスに確認コードを記載した確認メールを送信
    - ユーザーはサイトに戻り、自分の E メールアドレスと受け取った確認コードを入力、サインインするときに、ユーザー名 (または E メール) とパスワードを入力
  - JavaScript 関数が Amazon Cognito と通信し、セキュアリモートパスワードプロトコル (SRP) を使用して認証して、一連の JSON ウェブトークン (JWT) を受け取る
    - JWT にはユーザーの ID に関する認証トークンが含まれており、Amazon API Gateway で構築した RESTful API に対して認証するために使用するのでメモしておく​

<details>
<summary>ここをクリックで詳細手順を開く</summary>
  
- Amazon Cognito ユーザープールを作成し、アプリをユーザープールと統合する
  - ユーザープール名：WildRydes
  - アプリクライアント：WildRydesWebApp
  - ユーザープール ID をコピーして保存
  - クライアント ID をコピーして保存
- ウェブサイトの設定ファイルを更新する
  - wildryde-site/js/config.js ファイルを開く
  -  [ユーザープール ID] と [アプリクライアント ID] の正しい値に更新
  -  region の値は、ユーザープールを作成した AWS リージョンのコードを参照
  -  ``new_config``というコミット名でコミットしプッシュ
- 実装を検証する
  -  サイトのホームページ (`index.html` ページ) の [Giddy Up] ボタンをクリック
  -  登録フォームに入力して、[Let's Ryde] を選択。今回は自分の E メールを使用
  -  電子メールで受け取った認証コードを入力して、アカウントの検証プロセスを完了
  -  Cognito コンソールを使用して新しいユーザーを確認
  -  登録ステップで入力した E メールアドレスとパスワードを使用してログイン
  -  成功した場合、`/ride.html` にリダイレクト。API が設定されていないという通知が表示される。
  - **重要**: 次のモジュールで Amazon Cognito ユーザープールオーソライザーを作成するために、認証トークンをコピーして保存
 #### <sign up画面>
![](./images/README/signup.jpg)

#### <APIがまだ設定されていないというアラート>
![](./images/README/arrert.jpg)
  
</details>



### モジュール 3: サーバーレスサービスバックエンド
- 使用技術：AWS Lambda , Amazon DynamoDB
- 概要：
  - AWS Lambda と Amazon DynamoDB を使用して、ウェブアプリケーションのリクエストを処理するためのバックエンドプロセスを構築する
    - 最初のモジュールにデプロイしたブラウザアプリケーションを使用すると、ユーザーはユニコーンを自分の好きな場所に送信するように要求できる​
    - これらのリクエストを満たすには、ブラウザで実行されている JavaScript がクラウドで実行されているサービスを呼び出す必要がある​
  - ユーザーがユニコーンを要求するたびに呼び出される Lambda 関数を実装​
    - この関数は、フリートからユニコーンを選択し、その要求を DynamoDB テーブルに記録してから、ディスパッチされているユニコーンに関する詳細を使用して、フロントエンドアプリケーションに応答する
    - この関数は、Amazon API Gateway を使用してブラウザから呼び出される
    - 関数は単独でテストを行う​

<details>
<summary>ここをクリックで詳細手順を開く</summary>

- Amazon DynamoDBテーブルを作成する
  - テーブル名：「Rides」
  - パーティションキー に「RideId」と入力
  - ARN をコピー
- Lambda 関数の IAM ロールを作成する
  - 概要：
    - すべての Lambda 関数には、それに関連付けられた IAM ロールがあり、その関数が他のどの AWS のサービスとやり取りできるかを定義する
    - このチュートリアルの目的のために、ログを Amazon CloudWatch Logs に書き込むアクセス許可と、項目を DynamoDB テーブルに書き込むアクセス許可を Lambda 関数に付与する IAM ロールを作成する
  -  ロール名：「WildRydesLambda」
  -  ポリシー名：「DynamoDBWriteAccess」
- リクエスト処理のためにLambda関数を作成する
  - 概要：
    - AWS Lambda は、HTTP リクエストなどのイベントに応答してコードを実行する
    - ウェブアプリケーションからの API リクエストを処理してユニコーンをディスパッチするコア機能を構築する​
  - [関数名] フィールドに「RequestUnicorn」と入力
  - ~~[ランタイム] には [Node.js 16.x] を選択 (新しいバージョンの Node.js はこのチュートリアルでは動作しません)。~~
  - [Node.js 16.x]は選択できず、選択できる最も古いバージョンである[Node.js 21.x]を選択
  - [既存のロール] ドロップダウンで [WildRydesLambda] を選択
  - [関数を作成] をクリック
  - ~~[コードソース] セクションまでスクロールし、index.js コードエディタの既存のコードを requestUnicorn.js の内容に置き換える~~
  - チュートリアル通りのコードではエラーになったので、index.js コードエディタの既存コードをNode.js 21でも動くように置き換える（[エラー対応](#エラー対応)を参照）

  
- 実装を検証する(関数のテスト)
    - 概要：AWS Lambda コンソールを使用して構築した関数をテストする
    -  [イベント名] フィールドに「TestRequestEvent」と入力
    -  次のテストイベントをコピーして、[イベント JSON] セクションに貼り付ける

  <details>
  <summary>コード詳細を開く</summary>


  ```
    {
        "path": "/ride",
        "httpMethod": "POST",
        "headers": {
            "Accept": "*/*",
            "Authorization": "eyJraWQiOiJLTzRVMWZs",
            "content-type": "application/json; charset=UTF-8"
        },
        "queryStringParameters": null,
        "pathParameters": null,
        "requestContext": {
            "authorizer": {
                "claims": {
                    "cognito:username": "the_username"
                }
            }
        },
        "body": "{\"PickupLocation\":{\"Latitude\":47.6174755835663,\"Longitude\":-122.28837066650185}}"
        }
  ```


  </details>


- 関数の [コードソース] セクションで [テスト] を選択し、ドロップダウンから [TestRequestEvent] を選択
- テスト実行
- 表示される「関数の実行: 成功」メッセージで、[詳細] ドロップダウンを展開

```
{
    "statusCode": 201,
    "body": "{\"RideId\":\"SvLnijIAtg6inAFUBRT+Fg==\",\"Unicorn\":{\"Name\":\"Rocinante\",\"Color\":\"Yellow\",\"Gender\":\"Female\"},\"Eta\":\"30 seconds\"}",
    "headers": {
        "Access-Control-Allow-Origin": "*"
    }
}
```

</details>

### モジュール 4: RESTful API をデプロイする
- 使用技術：Amazon API Gateway
- 概要：Amazon API Gateway を使用して、前のモジュールで RESTful API として構築した Lambda 関数を公開する
- この API はパブリックインターネットでアクセス可能になり、前のモジュールで作成した Amazon Cognito ユーザープールを使用して保護される
- 次に、この設定を使用して、公開された API に対して AJAX 呼び出しを行うクライアント側 JavaScript を追加することにより、静的にホストされたウェブサイトを動的なウェブアプリケーションに変換する
  - 最初のモジュールでデプロイした静的ウェブサイトには、このモジュールで構築する API と連携するように設定済みのページが既にある
  - `/ride.html` のページには、ユニコーンに乗ることを要求するシンプルなマップベースのインターフェイスがある
  - `/signin.html` ページを使用した認証後、ユーザーはマップ上のポイントをクリックし、右上隅の [Request Unicorn] ボタンを選択して、乗る場所を選択することができる

<details>
<summary>ここをクリックで手順詳細を開く</summary>

- 新しいREST APIを作成する
  - [API 名] ：「WildRydes」
- オーソライザーを作成する
  - 概要：
    - Amazon API Gateway は、Amazon Cognito ユーザープール (モジュール 2 で作成) から返される JSON ウェブトークン (JWT) を使用して API 呼び出しを認証する
    - このセクションでは、ユーザープールを利用できるように、API のオーソライザーを作成する
  - オーソライザーの [名前] フィールドに「WildRydes」と入力
  - Cognito ユーザープールの [名前] フィールドに「WildRydes」と入力
  - [トークンのソース] に「Authorization」と入力
  - `ride.html` ウェブページからコピーした認証トークンを [承認 (ヘッダー)] フィールドに貼り付け、HTTP ステータスの [レスポンスコード] が 200 であることを確認
- 新しいリソースとメソッドを作成する
  - 概要：
    - このセクションでは、API 内に新しいリソースを作成する
    - その後、そのリソース用の POST メソッドを作成し、このモジュールの最初のステップで作成した RequestUnicorn 関数によってサポートされる Lambda プロキシ統合を使用するようにメソッドを設定する
  - [リソース名] に「ride」と入力すると、リソースパス /ride が自動的に作成される
  - 新しいドロップダウンから [POST] を選択
  - 統合タイプとして [Lambda 関数] を選択
  - [Lambda 関数] に「RequestUnicorn」と入力
- APIをデプロイする
  - 概要：Amazon API Gateway コンソールから API をデプロイする
  - [ステージ名] に「prod」と入力
  - [呼び出し URL] をコピー
- ウェブサイトの設定を更新する
  - 概要：
    - ウェブサイトのデプロイで /js/config.js ファイルを更新し、作成したステージの呼び出し URL を含める
    - 呼び出し URL は、Amazon API Gateway コンソールのステージエディタページの上部から直接コピーし、サイトの config.js ファイルの invokeUrl キーに貼り付ける
    - 設定ファイルには、前のモジュールで行った Amazon Cognito userPoolID、userPoolClientID、およびリージョンの更新が引き続き含まれている
  - ローカルマシンで js フォルダに移動し、任意のテキストエディタで config.js ファイルを開く
  - Amazon API Gateway コンソールからコピーした呼び出し URL を config.js ファイルの invokeUrl 値に貼り付ける
  - 更新した config.js ファイルを追加、コミット（`new_configuration`）、Git リポジトリにプッシュすると、自動的に Amplify コンソールにデプロイされる
- 実装を検証する
  - ride.html ファイルの ArcGIS JS バージョンを 4.3 から 4.6 に次のように更新
  ```
    <script src="https://js.arcgis.com/4.6/"></script>
    <link rel="stylesheet" href="https://js.arcgis.com/4.6/esri/css/main.css">
  ```

    
  - 変更したファイルを保存。 ファイルを追加、コミット（`update_ArcGIS`）、Git リポジトリに git プッシュ
  - `/ride.html` にアクセス
  - マップがロードされたら、マップ上の任意の場所をクリックして、ピックアップ場所を設定
  - [ユニコーンをリクエスト] を選択。右のサイドバーの通知で、ユニコーンが向かっていることが表示され、続いてピックアップ場所にユニコーンのアイコンが移動。
</details>

</details>

## エラー対応
### 発生場所：S3 から静的ファイルをコピー

<details>
<summary>ここをクリックで開く</summary>

#### エラー内容
チュートリアルにある以下のコードで、S3バケット（このチュートリアルに関連付けられた一般がアクセスできる既存のもの）がコピーされない
`aws s3 cp s3://wildrydes-ap-northeast-1/WebApplication/1_StaticWebHosting/website ./ --recursive`
エラー文：`fatal error: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied`

#### 原因
チュートリアル当時用意されていたS3バケットが現在はなくなっていた

#### 解決法
[aws-samples公式リポジトリのIssueの最新のIssue](https://github.com/aws-samples/aws-serverless-workshops/issues/292#issuecomment-1942911448)にて、「the problem is that the s3 bucketv isn't publicly accessible anymore. in order to find it check it here`aws s3 cp s3://ttt-wildrydes/wildrydes-site ./ --recursive`」とあり、このコードを使用したらS3をcloneできた


#### 試したこと
- 検索して出てきた[AWS re:POST](https://repost.aws/questions/QUKwoXr7h-RdO9uKdYYzfwjw/build-a-serverless-web-application-errors-on-copying-s3-file-wildrydes)を参照して試す
  - IAMユーザーが使用しているIAMポリシーでS3への「s3:ListBucket」が許可されているかどうかを確認してください。
  - 対象のS3バケットのバケットポリシーでも「s3:ListBucket」が許可されているかどうかを確認してください。
  - これを参考に、IAMポリシーを以下の記事を参考に設定
    - [ポリシーの例: &Amazon S3](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_examples.html#policy_library_S3)の中の[Amazon S3 バケットオブジェクトへの読み書きアクセスを付与する](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_examples_s3_rw-bucket.html)
    - [S3のポリシーに定義した「Resource」を正しく理解できていますか？](https://dev.classmethod.jp/articles/how-to-write-resource-of-s3-bucket-policy-and-iam-policy/)
  - 作成したIAMポリシーは以下の通り
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
  - しかしこれではエラー解決せず。<br>
- 次にGitHubのaws-samples公式リポジトリのIssue「[S3からWildrydesファイルをコピーしようとするとアクセスが拒否され失敗する](https://github.com/aws-samples/aws-serverless-workshops/issues/292)」によると、IAMユーザーにAmazonS3ReadOnlyAccessポリシーを付与することで解決したようなので試したが、私のIAMではすでにそのポリシーは存在した。
- [aws-samples公式リポジトリのIssueの最新のIssue](https://github.com/aws-samples/aws-serverless-workshops/issues/292#issuecomment-1942911448)にて、「the problem is that the s3 bucketv isn't publicly accessible anymore. in order to find it check it here`aws s3 cp s3://ttt-wildrydes/wildrydes-site ./ --recursive`」とあり、これで解決。

</details>

### 発生場所：Lambda関数のコード（Node.jsバージョン非互換）

<details>
<summary>ここをクリックで開く</summary>

#### エラー内容
Lambda関数にチュートリアルのコードをコピーしたところ、エラーが発生した

#### 原因
AWSのランタイムバージョンが更新され続けているため、チュートリアル作成時に指定されていたNode.js 16.xが選択できなくなっていた。<br>
選択可能な最も古いバージョンであるNode.js 21.xを選択したが、チュートリアルのコードはNode.js 21.xに対応していなかった。

#### 解決法
AIを使用してNode.js 21.x対応のコードに書き換えることで解決した

#### 修正内容
- aws-sdk はNode.js 18以降のLambdaでは削除されたので代わりに @aws-sdk/client-dynamodb を使う
- ファイルが .mjs（ESモジュール）として認識されてしまう=requireが使えないためimportを使う
- Node.js 24を使っているためcallbackが使えなくなっているためasync/awaitに書き換える




  <details>
  <summary>コード詳細を開く</summary>

  ```
  import { randomBytes } from 'crypto';
    import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
    import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';

    const client = new DynamoDBClient();
    const ddb = DynamoDBDocumentClient.from(client);

    const fleet = [
        { Name: 'Angel', Color: 'White', Gender: 'Female' },
        { Name: 'Gil', Color: 'White', Gender: 'Male' },
        { Name: 'Rocinante', Color: 'Yellow', Gender: 'Female' },
    ];

    export const handler = async (event) => {
        if (!event.requestContext.authorizer) {
            return errorResponse('Authorization not configured');
        }

        const rideId = toUrlString(randomBytes(16));
        console.log('Received event (', rideId, '): ', event);

        const username = event.requestContext.authorizer.claims['cognito:username'];
        const requestBody = JSON.parse(event.body);
        const pickupLocation = requestBody.PickupLocation;
        const unicorn = findUnicorn(pickupLocation);

        try {
            await recordRide(rideId, username, unicorn);
            return {
                statusCode: 201,
                body: JSON.stringify({
                    RideId: rideId,
                    Unicorn: unicorn,
                    Eta: '30 seconds',
                    Rider: username,
                }),
                headers: { 'Access-Control-Allow-Origin': '*' },
            };
        } catch (err) {
            console.error(err);
            return errorResponse(err.message);
        }
    };

    function findUnicorn(pickupLocation) {
        console.log('Finding unicorn for ', pickupLocation.Latitude, ', ', pickupLocation.Longitude);
        return fleet[Math.floor(Math.random() * fleet.length)];
    }

    function recordRide(rideId, username, unicorn) {
        return ddb.send(new PutCommand({
            TableName: 'Rides',
            Item: {
                RideId: rideId,
                User: username,
                Unicorn: unicorn,
                RequestTime: new Date().toISOString(),
            },
        }));
    }

    function toUrlString(buffer) {
        return buffer.toString('base64')
            .replace(/\+/g, '-')
            .replace(/\//g, '_')
            .replace(/=/g, '');
    }

    function errorResponse(errorMessage) {
        return {
            statusCode: 500,
            body: JSON.stringify({
                Error: errorMessage,
            }),
            headers: { 'Access-Control-Allow-Origin': '*' },
        };
    }
  ```
  

  </details>


</details>
