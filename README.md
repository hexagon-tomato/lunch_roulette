
-----

#  ランチマッチングアプリ (Lunch Roulette)

**〜 ランダムマッチングで、部署を越えた交流を楽しく文化に！ 〜**

社内コミュニケーションの活性化を目的として開発された、自動ランチマッチングアプリケーションです。ユーザーはWeb画面からランチに参加したい日を登録するだけで、システムが毎日決まった時間に最適なペアを自動でマッチングし、結果をSlackで通知します。

このプロジェクトは、フロントエンドとサーバーレスバックエンドを組み合わせた、モダンなクラウドネイティブアーキテクチャで構築されています。

-----

##  主な機能

#### ユーザー向け機能

  * ログイン機能
  * カレンダーUIによるランチ希望日の登録・編集
  * 自身のマッチング履歴の閲覧

#### 管理者向け機能

  * 全社員の登録状況や希望日を一覧できるダッシュボード
  * 任意の日付での手動マッチング実行

#### システム機能

  * **自動マッチング**
      * 毎日午前9時に、その日の参加希望者を対象に自動でマッチングを実行します。
  * **賢いスコアリングロジック**
      * 単なるランダムではなく、以下の点を考慮して最適なペアを探索します。
          * 同じ**拠点**のメンバー: `+3点`
          * 違う**部署**のメンバー: `+2点`
          * 過去**28日以内**にマッチしたペア: `-5点`
  * **Slack通知**
      * マッチング完了後、結果を自動で指定のSlackチャンネルに通知します。

####  セキュリティ機能

  * **IPアドレス制限**: `login.php`へのアクセスを特定のIPアドレスに制限します。
  * **ベーシック認証**: Webサイト全体をパスワードで保護します。
  * **WAF (Web Application Firewall)**: SQLインジェクションなどの一般的なWeb攻撃をブロックします。
  * **プライベートネットワーク**: データベースを外部から直接アクセスできないVPC内のプライベートサブネットに配置します。

-----

##  技術スタック

  * **バックエンド**: PHP 8.3
  * **データベース**: MySQL 5.7
  * **フロントエンド**: HTML, CSS, JavaScript (Bootstrap 5, flatpickr.js)
  * **クラウド (AWS)**
      * **コンピューティング**: Elastic Beanstalk (EC2), Lambda
      * **データベース**: RDS for MySQL
      * **ネットワーキング**: VPC, Security Groups, Subnets, VPC Endpoint
      * **アプリケーション連携**: SNS (Simple Notification Service)
      * **自動化・トリガー**: EventBridge (CloudWatch Events)
      * **セキュリティ**: WAF, Certificate Manager (ACM), IAM
      * **ドメイン管理**: Route 53
      * **設定管理**: SSM Parameter Store
  * **DevOps / ツール**
      * Serverless Framework (v3)
      * Composer
      * MySQL Workbench

-----

##  セットアップとデプロイ手順

このアプリケーションをゼロから構築するための手順です。

### 1\.  ローカル環境の準備

  * MAMP (または同等のPHP/MySQL環境)
  * MySQL Workbench
  * Node.js, npm
  * Composer for PHP
  * Serverless Framework v3
    ```bash
    npm install -g serverless@3
    ```

### 2\.  AWSインフラの準備

1.  **VPC**: デフォルトVPCを利用します。
2.  **セキュリティグループ**:
      * `lambda-sg`: Lambda関数用のセキュリティグループを作成します。
      * `db-sg`: RDSデータベース用のセキュリティグループを作成します。
3.  **RDSデータベース**:
      * MySQLエンジンでプライベートなRDSインスタンスを「パブリックアクセス: なし」で作成します。
      * セキュリティグループとして`db-sg`を割り当てます。
      * 初期データベース名として`lunch_roulette`を指定します。
4.  **SNSトピック**:
      * `lunch-match-topic-v2`という名前でスタンダードトピックを手動で作成します。
5.  **VPCエンドポイント**:
      * SNS用のVPCエンドポイントを作成し、Lambdaと同じサブネットに配置して`lambda-sg`を割り当てます。
      * **プライベートDNS名を有効化**してください。
6.  **IAM**:
      * デプロイ用のIAMユーザーを作成し、アクセスキーを発行後、MFAを有効化します。
      * 必要な権限 (`AdministratorAccess`など) をアタッチします。
7.  **SSM Parameter Store**:
      * データベース関連のパラメータ（`DB_HOST`, `DB_USER`など5つ）と`SLACK_WEBHOOK_URL`を登録します。

### 3\.  データベースの初期化

1.  RDSのセキュリティグループ`db-sg`に、一時的に自身のグローバルIPアドレスからのMySQL接続を許可するインバウンドルールを追加します。
2.  MySQL Workbenchを使い、RDSインスタンスに接続します。
3.  `lunch_roulette.sql`を実行し、テーブル構造と基本データ (`employees`, `users`) を作成します。
4.  提供された**デモデータ用のSQLスクリプト**を実行し、マッチングデータを投入します。
5.  セキュリティのため、手順1で追加したインバウンドルールを`db-sg`から削除します。

### 4\.  サーバーレスバックエンドのデプロイ (`lambda/`)

1.  `lambda`ディレクトリに移動し、必要なライブラリをインストールします。
    ```bash
    composer require bref/bref aws/aws-sdk-php
    ```
2.  `serverless.yml`を開き、自身のAWS環境の値 (VPC ID, Subnet ID, 各種ARNなど) を設定します。
3.  ターミナルで以下のコマンドを実行し、デプロイします。
    ```bash
    serverless deploy
    ```

### 5\.  Webアプリケーションのデプロイ (`web/`)

1.  セッション管理を中央集権化するため、`session_handler.php`を作成し、`db.php`などの関連ファイルを修正します。
2.  `web`フォルダの中身を`app.zip`のようなZIPファイルに圧縮します。
3.  AWS Elastic Beanstalkで新しいPHP環境を「ロードバランス済み」で作成し、`app.zip`をアップロードします。
4.  Elastic Beanstalkの[設定] \> [環境プロパティ]で、データベース接続情報 (`DB_HOST`など) とベーシック認証用の`BASIC_AUTH_USER`/`PASS`を設定します。
5.  Elastic Beanstalkが自動作成したEC2インスタンスのセキュリティグループIDを調べ、RDSのセキュリティグループ`db-sg`の受信ルールに追加します (EC2からRDSへの接続を許可)。
6.  Route 53でドメインを取得し、ACMでSSL証明書を発行します。
7.  Elastic BeanstalkのロードバランサーにHTTPSリスナーを追加し、発行したACM証明書をアタッチします。
8.  Route 53で、ドメインからElastic Beanstalk環境へのエイリアスレコードを作成します。
9.  AWS WAFでWeb ACLを作成し、`AWSManagedRulesCommonRuleSet`を有効にしてロードバランサーに関連付けます。