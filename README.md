# ランダムマッチングで部署横断の交流を強制的に楽しく!

-----

# ランチマッチングアプリ (Lunch Roulette)

社内コミュニケーションの活性化を目的として開発された、自動ランチマッチングアプリケーションです。ユーザーはWeb画面からランチに参加したい日を登録でき、システムが平日９時に自動でマッチングを行い、結果をSlackで通知します。

このプロジェクトは、フロントエンドとサーバーレスバックエンドを組み合わせた、モダンなクラウドネイティブアーキテクチャで構築されています。


```

##  主な機能

  * **ユーザー向け機能**
      * ログイン機能
      * カレンダーUIによるランチ希望日の登録・編集
      * 自身のマッチング履歴の閲覧
  * **管理者向け機能**
      * 全社員の登録状況や希望日の集計を閲覧できるダッシュボード
      * 任意の日付で手動マッチングを実行する機能
  * **システム機能**
      * **自動マッチング**: 毎日午前9時に、その日の参加希望者を対象に自動でマッチングを実行。
      * **スコアリングロジック**: 単なるランダムではなく、以下の点を考慮して最適なペアを探索。
          * **同じ拠点**にいるメンバーを優先 (+3点)
          * **違う部署**のメンバーを優先 (+2点)
          * **過去28日以内**にマッチしたことがあるペアを避ける (-5点)
      * **Slack通知**: マッチングが完了すると、結果が自動でSlackチャンネルに通知される。
  * **セキュリティ機能**
      * **IPアドレス制限**: `login.php`へのアクセスを特定のIPに制限。
      * **ベーシック認証**: Webサイト全体にパスワード保護を実装。
      * **WAF**: SQLインジェクションなどの一般的なWeb攻撃をブロック。
      * **プライベートネットワーク**: データベースをVPC内のプライベートサブネットに配置し、外部からの直接アクセスを遮断。

##  技術スタック

  * **バックエンド**: PHP 8.3
  * **データベース**: MySQL 5.7
  * **フロントエンド**: HTML, CSS, JavaScript (Bootstrap 5, flatpickr.js)
  * **クラウド (AWS)**:
      * **コンピューティング**: Elastic Beanstalk (EC2), Lambda
      * **データベース**: RDS for MySQL
      * **ネットワーキング**: VPC, Security Groups, Subnets, VPC Endpoint
      * **アプリケーション連携**: SNS (Simple Notification Service)
      * **自動化・トリガー**: EventBridge (CloudWatch Events)
      * **セキュリティ**: WAF, Certificate Manager (ACM), IAM
      * **ドメイン管理**: Route 53
      * **設定管理**: SSM Parameter Store
  * **DevOps/ツール**:
      * Serverless Framework (v3)
      * Composer
      * MySQL Workbench

##  セットアップとデプロイ手順

このアプリケーションをゼロから構築するための手順です。

### 1\. ローカル環境の準備

  * MAMP (または同等のPHP/MySQL環境)
  * MySQL Workbench
  * Node.js, npm
  * Composer for PHP
  * Serverless Framework v3 (`npm install -g serverless@3`)

### 2\. AWSインフラの準備

1.  **VPC**: デフォルトVPCを利用。
2.  **セキュリティグループ**:
      * `lambda-sg`: Lambda関数用のセキュリティグループを作成。
      * `db-sg`: RDSデータベース用のセキュリティグループを作成。
3.  **RDSデータベース**:
      * MySQLエンジンでプライベートなRDSインスタンスを作成 (`パブリックアクセス: なし`)。
      * セキュリティグループとして `db-sg` を割り当てる。
      * 初期データベース名として `lunch_roulette` を指定。
4.  **SNSトピック**:
      * `lunch-match-topic-v2` という名前でスタンダードトピックを手動作成。
5.  **VPCエンドポイント**:
      * SNS用のVPCエンドポイントを作成。Lambdaと同じサブネットに配置し、`lambda-sg` を割り当てる。
      * **プライベートDNS名を有効化**する。
6.  **IAM**:
      * デプロイ用のIAMユーザーを作成し、アクセスキーを発行。MFAを有効化する。
      * 必要な権限 (`AdministratorAccess`など) をアタッチ。
7.  **SSM Parameter Store**:
      * DB関連の5つのパラメータ (`DB_HOST`, `DB_USER`など) と `SLACK_WEBHOOK_URL` を登録。

### 3\. データベースの初期化

1.  RDSのセキュリティグループ `db-sg` に、一時的に自身のIPアドレスからのMySQL接続を許可するルールを追加。
2.  MySQL WorkbenchでRDSに接続。
3.  **`lunch_roulette.sql`** を実行し、テーブル構造と基本データ (`employees`, `users`) を作成。
4.  提供された**デモデータ用のSQLスクリプト**を実行し、マッチングデータを投入。
5.  `db-sg` から自身のIPアドレスからの接続許可ルールを削除。

### 4\. サーバーレスバックエンドのデプロイ (`lambda/` フォルダ)

1.  `lambda` ディレクトリで `composer require bref/bref aws/aws-sdk-php` を実行。
2.  `serverless.yml` に自身のAWS環境の値 (VPC ID, Subnet ID, ARNなど) を設定。
3.  ターミナルで `serverless deploy` を実行。

### 5\. Webアプリケーションのデプロイ (`web/` フォルダ)

1.  `session_handler.php` を作成し、`db.php` などの関連ファイルを修正して、セッション管理を中央集権化。
2.  `web` フォルダの中身をZIPファイルに圧縮。
3.  AWS Elastic Beanstalkで新しいPHP環境を「ロードバランス済み」で作成し、ZIPファイルをアップロード。
4.  Elastic Beanstalkの環境プロパティに、`DB_HOST` などのデータベース接続情報と、ベーシック認証用の `BASIC_AUTH_USER`/`PASS` を設定。
5.  Elastic Beanstalkが自動作成したEC2インスタンスのセキュリティグループIDを調べ、`db-sg` の受信ルールに追加。
6.  Route 53でドメインを取得し、ACMでSSL証明書を発行。
7.  Elastic BeanstalkのロードバランサーにHTTPSリスナーを追加し、ACM証明書をアタッチ。
8.  Route 53で、ドメインからElastic Beanstalk環境へのエイリアスレコードを作成。
9.  AWS WAFでWeb ACLを作成し、`AWSManagedRulesCommonRuleSet` を有効にしてロードバランサーに関連付ける。