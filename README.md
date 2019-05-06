# Scorekeep
Scorekeep は Java で実装された RESTful な Web API で、ゲームセッションやユーザー管理のための HTTP のインタフェースを Spring を使って提供しています。このプロジェクトは Scorekeep API とその API を呼び出すフロントエンドのウェブアプリで構成されています。フロントエンドアプリケーションと APIA は同一サーバー・同一ドメインで実行することもできますし、別々、例えば API は Fargate で実行されるコンテナから、フロントエンドは静的コンテンツとして CDN 経由で公開することも可能です。

`fargate` ブランチは Spring, Angular, nginx, [AWS SDK for Java](http://aws.amazon.com/sdkforjava), [Amazon DynamoDB](http://aws.amazon.com/dynamodb), Gradle, [Amazon ECS + AWS Fargate](http://aws.amazon.com/ecs) を利用して次のようなことを行なっていきます:

- 両方のコンポーネントを単一の [Amazon ECS](http://aws.amazon.com/ecs) タスク定義に定義し、[Amazon Application Load Balancer](https://aws.amazon.com/elasticloadbalancing/) の後ろに配置する
- CloudFormation を使って、ハンズオンで必要となる DynamoDB と [Amazon SNS](http://aws.amazon.com/sns) リソースを作成します
- コンテナから出力されるログを [Amazon Cloudwatch Logs](https://aws.amazon.com/cloudwatch) に集めます

その他のブランチでは異なるサービスの組み合わせを使ってアプリケーションを動かすことを学べます。詳細は各ブランチの README.md ファイルをご覧ください。

**その他のブランチの例 (English)**
<details><summary>Click to see details</summary>
<p>

- [`master`](https://github.com/awslabs/eb-java-scorekeep/tree/master) - Original [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) scorekeep application.
- [`cognito`](https://github.com/awslabs/eb-java-scorekeep/tree/cognito) - Support login and store users in an [Amazon Cognito](http://aws.amazon.com/cognito) user pool. Get AWS SDK credentials and make service calls with a Cognito identity pool.
- [`cognito-basic`](https://github.com/awslabs/eb-java-scorekeep/tree/cognito-basic) - Use Cognito for user ID storage. User pool only, no identity pool.
- [`lambda`](https://github.com/awslabs/eb-java-scorekeep/tree/lambda) - Call an [AWS Lambda](http://aws.amazon.com/lambda) function to generate random names.
- [`lambda-worker`](https://github.com/awslabs/eb-java-scorekeep/tree/lambda-worker) - Run a Lambda function periodically to process game records and store the output in Amazon S3.
- [`sql`](https://github.com/awslabs/eb-java-scorekeep/tree/sql) - Use JDBC to store game histories in an attached PostgreSQL database instance.
- [`xray`](https://github.com/awslabs/eb-java-scorekeep/tree/xray) - Use the [AWS X-Ray SDK for Java](http://docs.aws.amazon.com/xray-sdk-for-java/latest/javadoc/) to instrument incoming requests, functions, SDK clients, SQL queries, HTTP clients, startup code, and AWS Lambda functions.
- [`xray-cognito`](https://github.com/awslabs/eb-java-scorekeep/tree/xray-cognito) - Use AWS credentials obtained with Amazon Cognito to upload trace data to X-Ray from the browser.
- [`xray-gettingstarted`](https://github.com/awslabs/eb-java-scorekeep/tree/xray-gettingstarted) ([tutorial](https://docs.aws.amazon.com/xray/latest/devguide/xray-gettingstarted.html)) - Use the AWS X-Ray SDK for Java to instrument incoming requests and SDK clients (no additional configuration required).
- [`xray-worker`](https://github.com/awslabs/eb-java-scorekeep/tree/xray-worker) - Instrumented Python Lambda worker function from the `lambda-worker` branch.

</p>
</details>

続くセクションで Fargate を使ったアプリケーションの実行、ローカル環境でテスト・開発について見ていきましょう。

**セクション一覧**
- [前提条件](#前提条件)
- [リポジトリ構成](#リポジトリ構成)
- [CloudFormation セットアップ](#cloudformation-セットアップ)
- [Java アプリケーションのビルド・構築](#Java-アプリケーションのビルド・構築)
- [アプリケーション構成の解説](#アプリケーション構成の解説)
- [ローカル環境でアプリケーションを実行する](#ローカル環境でアプリケーションを実行する)
- [コントリビューション](#コントリビューション)

# 前提条件
Docker イメージの作成、ECR へのイメージのアップロード、ECS へのタスク定義の登録を行うために以下ツールのインストールが必要です
- Docker
- AWS CLI v1.14.0 以降(最新版が望ましい)

また、少なくとも以下の権限を持つ AWS IAM ユーザーが必要です
- IAM、DynamoDB、SNS、ECS、CloudWatch Logs、ECR へのフルアクセス権限
- CloudFormation でのスタック作成権限

# リポジトリ構成
このリポジトリには2つの独立して動作するアプリケーションが含まれています:

- フロントエンドアプリケーション: HTML + JavaScript(Angular 1.5) で構成され、Nginx で実行される
- バックエンドアプリケーション: Spring を利用して API を公開する Java アプリケーション

バックエンド・フロントエンドの両アプリケーションは `docker` と `make` を利用してビルドされます。作成される Docker イメージは Amazon ECR に格納されます。

| Directory | Contents                                        | Build           | Package         | Publish        | Clean         |
|-----------|-------------------------------------------------|-----------------|-----------------|----------------|---------------|
| `/`       | Java バックエンドアプリケーション (別名 `scorekeep-api`) | `make build`   | `make package`  | `make publish` | `make clean`   |
| `/scorekeep-frontend` | Angular + Nginx のフロントエンドアプリケーション |  N/A            | `make package`  | `make publish`  |  N/A         |
| `/task-definition` | ECS 用タスク定義を生成するためのテンプレート | `./generate-task-definition` | N/A | aws ecs register-task-definition --cli-input-json file://scorekeep-task-definition.json | N/A |
| `/cloudformation` | 依存リソース(i.e. DynamoDB, SNS, CWL, ECR, IAM)を作成するための CloudFormation テンプレート | N/A | N/A | `make stack` | `make clean` |

# CloudFormation セットアップ

このプロジェクトでは、アプリケーションから利用する AWS リソースをセットアップしやすいよう、CloudFormation テンプレートが用意されています。

用意された CloudFormation テンプレートは、テンプレートが格納されたディレクトリで `make publish` コマンドを実行することでリソースのプロビジョンが実行されます。同コマンドは内部的に AWS CLI を実行しますが、その際リポジトリルートに置かれた `aws.env` ファイルの `AWS_REGION` パラメータと、default クレデンシャルを利用します。CloudFormation スタックが正常に作成されるためには、default クレデンシャルが CloudFormation のスタック作成とテンプレート内で指定されているリソースの作成権限を有している必要がある点に留意してください。

# Java アプリケーションのビルド・構築

Java アプリケーションは Gradle Docker コンテナを利用してビルドされるため、ビルドコマンドを実行するローカル環境に Java や Gradle がセットアップされている必要はありません。ビルド成果物は `build/` ディレクトリ以下に出力されます。アプリケーションのビルド後、パッケージングを実行すると、ビルドプロセスにて生成された JAR ファイルが Java ベースイメージにコピーされ、バックエンドアプリケーションの Docker イメージが出来上がります。出来上がった Docker イメージは適切な AWS クレデンシャルを渡して実行することでローカルでも実行できますし、もちろん ECS でも実行できます。

# アプリケーションのデプロイ手順

*依存リソースの作成からデプロイまでの手順*

0. `aws.env` ファイル内のパラメーターを編集します。(AWS アカウント ID は https://console.aws.amazon.com/billing/home?#/account で確認できます)
1. `cloudformation/` ディレクトリに移動し、`make stack` コマンドを実行し、アプリケーションが必要とするリソースを CloudFormation を利用して作成します。
2. リポジトリのルートディレクトリに移動し、`make publish` コマンドを実行します。これにより API コンテナがビルドされ、ステップ 1 の CloudFormation によって作成された ECR リポジトリにコンテナイメージがプッシュされます。(再ビルドを行う際は `make publish` コマンドの実行前に `make clean` コマンドを実行してください)
3. `scorekeep-frontend/` ディレクトリに移動し、`make publish` コマンドを実行します。これによりフロントエンドアプリケーションのコンテナイメージがビルドされ、こちらもステップ 1 の CloudFormation によって作成された ECR リポジトリにコンテナイメージがプッシュされます。
4. `task-definition` ディレクトリに移動し、`generate-task-definition` スクリプトを実行します。これにより、使用している AWS アカウント ID やリージョンなどの情報を含めた ECS 用タスク定義ファイルが生成されます。
5. `aws ecs register-task-definition --cli-input-json file://scorekeep-task-definition.json --region us-west-2` コマンドを実行し、ECS にタスク定義を登録します(注: `--region` オプションは実際に利用しているリージョンに合わせて変更してください)
6. 以上で ECS サービスを実行する準備が完了です！次に進みましょう。

今回はマネジメントコンソールを使って AWS Fargate を利用する ECS サービスを作成します

1. マネジメントコンソールで [ECS](https://console.aws.amazon.com/ecs/home) の画面を開きます (注: 適切なリージョンが選択されているか確認しましょう)
2. **クラスターの作成**をクリック
3. **ネットワーキングのみ**を選択し、**次のステップ**をクリック
4. クラスター名に *${IAMユーザー名}-scorekeep-cluster* と入力し、**作成**をクリック
5. **クラスターの表示**をクリック
6. **サービス**タブが選択されていることを確認し、**作成**をクリック
7. 以下の設定でサービスを作成します。サービスの作成はウィザード形式で進みます。必要に応じて**次のステップ**をクリックし、ウィザードを進めてください。画面上に存在する値で下記で言及されていない項目はデフォルト値のまま進めて構いません。
  - 起動タイプ: **FARGATE**
  - タスク定義 - Family: **${IAMユーザー名}-scorekeep**
  - タスク定義 - Revision: **1 (latest)**
  - プラットフォームのバージョン: **LATEST**
  - クラスター: **${IAMユーザー名}-scorekeep-cluster**
  - サービス名: **${IAMユーザー名}-scorekeep-service**
  - タスクの数: **4**
  - 最小ヘルス率: **50**
  - 最大率: **200**
  - クラスター VPC: **${IAMユーザー名}-scorekeep**
  - サブネット: **${IAMユーザー名}-scorekeep_Private** が2つありますので、両方を選択します
  - セキュリティグループ: **編集**をクリック > **既存のセキュリティグループの選択**を選択 > **default** を選択 > **保存**をクリック
  - パブリック IP の自動割り当て: **ENABLED**
  - ELB type: **Application Load Balancer**
  - ELB 名: **${IAMユーザー名}-scorekeep-lb**
  - コンテナの選択: **scorekeep-frontend:8080:8080** が選ばれていることを確認して **ELB への追加**をクリック
  - リスナーポート: **新規作成**が選ばれていることを確認して **80** と入力
  - ターゲットグループ名: **新規作成** (*ターゲットグループ名 は最大 32 文字を超えることはできません。*というエラーが出ている場合、インプットボックスに自動入力されている名前を適度に変更してください。インプットボックスからフォーカスを外すと再度バリデーションが実行され、32文字以内に収まっている場合はエラーメッセージが消えます)
  - ヘルスチェックパス: /
  - サービスの検出の統合の有効化: チェックを外します
  - Service auto scaling: **サービスの必要数を直接調整しない**
  - 「サービスの確認」画面が表示されるので、**サービスの作成**をクリック
8. **サービスの表示**をクリック
9. マネジメントコンソールで [ELB](https://console.aws.amazon.com/ec2/v2/home#LoadBalancers:) の画面を開きます (注: 適切なリージョンが選択されているか確認しましょう)
10. **${IAMユーザー名}-scorekeep-lb** のチェックボックスを選択
11. **説明**タブが開かれていることを確認し、**DNS 名**の値をコピーします
12. コピーした DNS 名をブラウザで開きます

画面が表示されたら、以下のスクリーンショットのような流れで操作することでデプロイしたアプリケーション群の機能を確認することができます。ブラウザのネットワークコンソールを利用し、画面操作に合わせて API を通して DynamoDB に対してユーザー、セッション、ゲーム、マスの選択が読み書きされていることを確認しましょう。

![Scorekeep flow](/img/scorekeep-flow.png)

# 通知を設定する(任意)
バックエンドアプリケーション(API)は、ゲームの勝敗が決したタイミングで SNS を利用して指定されたEメールアドレスに対して通知を送ることができます。Eメール通知を有効にするには、タスク定義のバックエンドアプリケーション(API)用の環境変数にキー名 **NOTIFICATION_EMAIL**、値を通知先のメールアドレスを追加し、タスク定義を登録、ECS サービスを再デプロイします。

注: 事前に SNS コンソールにて SNS トピックに対して通知先Eメールアドレスをサブスクリプションとして登録しておく必要があります

# アプリケーション構成の解説

## バックエンド
API は /api パス以下でリクエストを待ち受けており、JSON ドキュメントとして DynamoDB に格納されたユーザー、セッション、ゲーム、状態、そしてマスの選択へのアクセスを提供します。
RESTful API として実装されているため、例えば /api/session に対して HTTP POST リクエストを送信することでセッションを作成することができます。同様のリクエストを cURL コマンドで送信する例が [bin/test-api.sh](https://github.com/toricls/eb-java-scorekeep/blob/fargate/bin/test-api.sh) にありますので、合わせてご確認ください。

それぞれのリソース(ユーザー、セッション...)用の DynamoDB テーブルは手順の序盤で利用した CloudFormation テンプレートによって作成されたものです。

## フロントエンド
フロントエンドアプリケーションは Angular 1.5 を利用して作成されたウェブアプリケーションで、`$resource` オブジェクトを利用して API で定義されたリソース群への CRUD オペレーションを実行しています。このアプリケーションのユーザーが最初に開くであろう画面は[メイン画面](https://github.com/toricls/eb-java-scorekeep/blob/fargate/scorekeep-frontend/public/main.html) で、JavaScript で書かれた[コントローラ](https://github.com/toricls/eb-java-scorekeep/blob/fargate/scorekeep-frontend/public/app/mainController.js)が内部で実行されています。その後、ユーザーがリソース(セッションやゲームなど)を作成すると、それらの ID をパス(URL)に含むルートに移動することで、セッションやゲーム画面に進んでいきます。

フロントエンドアプリケーションは Nginx コンテナから静的なコンテンツとして配信されてます。Nginx は scorekeep-frontend ディレクトリに含まれる [nginx.conf](https://github.com/toricls/eb-java-scorekeep/blob/fargate/scorekeep-frontend/nginx.conf) ファイルによって設定され、ルートパスへのアクセスではフロントエンドアプリケーションの HTML ページを、/api で始まるパスへのアクセスについてはポート5000番で待ち受けるバックエンドアプリケーションに対してリクエストをフォワードします。

# ローカル環境でアプリケーションを実行する
フロントエンド、バックエンドアプリケーションは双方ともに Docker コンテナで動作するため、ローカル環境でも実行することができます。

## バックエンドアプリケーション(API)をローカル環境で Docker コンテナとして実行する

*注: バックエンドアプリケーションはデータの格納先として DynamoDB を、Eメール通知を設定している場合は SNS を利用します。そのため、以下の手順は[アプリケーションのデプロイ手順](#アプリケーションのデプロイ手順)セクションの1番、CloudFormation スタック作成が完了していることを前提としています。*

リポジトリのルートディレクトに移動し、`make run-local` コマンドを実行します。実行後、`docker logs` コマンドを利用してアプリケーションのログを確認したり、`docker ps` や `docker stats` コマンドを利用して現在の状態を確認したりすることができます。

本アプリケーションは、ECS 上での実行時にはネットワークモードとして `awsvpc` を利用していますが、ローカル環境での実行時には awsvpc と似た構成である `host` ネットワークモードを利用しています。

バックエンドアプリケーションは DynamoDB や SNS(任意利用)と通信を行うために AWS クレデンシャルを必要とします。ECS での実行時は、アプリケーションはクレデンシャルをタスク定義で設定されたタスクロールを利用して取得しますが、ローカル環境での実行時には ~/.aws/ ディレクトリをコンテナにマウントすることで AWS SDK for Java がクレデンシャルを利用できるようにしています。これ以外にも、例えば環境変数としてクレデンシャルを渡すことも可能です。

環境変数を利用してアプリケーションにアクセスキーを渡す方法については、*AWS SDK for Java 開発者ガイド* も合わせてご覧ください: [開発用の AWS 認証情報のセットアップ](https://docs.aws.amazon.com/ja_jp/sdk-for-java/v1/developer-guide/setup-credentials.html).

(*訳者注: AWS は 2019/3/27 に [Amazon ECS Local Container Endpoints](https://github.com/awslabs/amazon-ecs-local-container-endpoints) をオープンソースとして公開しました。こちらのツールを利用すると、ローカル環境でも ECS タスクロールと同様の方式で AWS クレデンシャルをアプリケーションから利用できます。*)

ローカル環境でバックエンドアプリケーションのコンテナが実行できたら、テストスクリプトを使って API が動作していることを確認してみましょう。

```
~/eb-java-scorekeep$ ./bin/test-api.sh
```

上記のスクリプトは API エンドポイントとして `localhost:5000` を利用します。スクリプトを編集し、API エンドポイントを書き換えることでローカル環境以外にデプロイしたバックエンドアプリケーションに対しても動作確認を実行できます。

## フロントエンドアプリケーションをローカル環境で Docker コンテナとして実行する

`scorekeep-frontend` ディレクトリに移動し、`make run-local` コマンドを実行します。バックエンドアプリケーションのコンテナ同様、ここで実行されるコンテナもネットワークモードとして `host` を利用しています。これにより、ブラウザもしくは curl コマンドなどで [localhost:8080](http://localhost:8080)にアクセスすることでフロントエンドアプリケーションの動作を確認することができます。

# コントリビューション

This sample application could be better with your help!

- Add a new game!
  - Implement game logic in the game class. See [TicTacToe.java](https://github.com/toricls/eb-java-scorekeep/blob/fargate/src/main/java/scorekeep/TicTacToe.java).
  - Add the class to [RulesFactory.java](https://github.com/toricls/eb-java-scorekeep/blob/fargate/src/main/java/scorekeep/RulesFactory.java).
- Create your own client front end!
  - Web frameworks - Angular 2, React, ember, etc.
  - Mobile app
  - Desktop application
- Integrate with other AWS services!
  - CICD with [AWS CodeCommit](http://aws.amazon.com/codecommit), [AWS CodePipeline](http://aws.amazon.com/codepipeline), [AWS CodeBuild](http://aws.amazon.com/codebuild), and [AWS CodeDeploy](http://aws.amazon.com/codedeploy)
  - Analytics with [Amazon Kinesis](https://aws.amazon.com/kinesis), [Amazon Athena](http://aws.amazon.com/athena), [Amazon EMR](http://aws.amazon.com/emr), or [Amazon QuickSight](https://quicksight.aws/)
  - Security with [Amazon VPC](http://aws.amazon.com/vpc)
  - Performance with [Amazon ElastiCache](http://aws.amazon.com/elasticache)
  - Scalability with [Amazon CloudFront](http://aws.amazon.com/cloudfront)
  - Accessibility with [Amazon Polly](https://aws.amazon.com/documentation/polly/)
- Write tests!
  - Unit tests
  - Integration tests
  - Functional tests
  - Load tests
- File an [issue](https://github.com/awslabs/eb-java-scorekeep/issues) to report a bug or request new features.
