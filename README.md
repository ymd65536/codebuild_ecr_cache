# 【AWS】検証！Amazon ECRをCodeBuildのリモートキャッシュとして利用する

## はじめに

この記事では「この前リリースされた機能って実際に動かすとどんな感じなんだろう」とか
「もしかしたら内容次第では使えるかも？？」などAWSサービスの中でも特定の機能にフォーカスして検証していく記事です。

主な内容としては実践したときのメモを中心に書きます。（忘れやすいことなど）
誤りなどがあれば書き直していく予定です。

今回はAmazon ECRをコンテナイメージのキャッシュとして利用するCodeBuildの機能を検証してみます。

## 前提知識：コンテナイメージをキャッシュするとはどういうことか

結論から説明するとコンテナイメージをキャッシュするということは以下のような意味合いがあります。

- 本当に必要なときだけイメージをプルすることで、無駄なリソース消費を防げる
  - ネットワーク帯域の節約になる
  - ビルドコストの削減につながる可能性がある
- コンテナイメージのビルド時間を短縮できる
- 本当に変更が必要な部分のみにフォーカスできる
  - ビルドの再現性が向上する

上記のメリットを理解できる人は次のセクションに進んでください。`コンテナイメージのキャッシュについて理解する`はスキップしても問題ありません。

### コンテナイメージのキャッシュについて理解する

まずはじめに今回説明する機能がどれほどのものかを理解するため、コンテナイメージのキャッシュについて簡単に説明します。

コンテナイメージは複数のレイヤーで構成されており、各レイヤーはDockerfileの各命令に対応しています。
例えば以下のようなDockerfileがあったとします。

```Dockerfile
FROM python:3.12.1-slim
COPY . /app
RUN pip install -r /app/requirements.txt
CMD ["python", "/app/app.py"]
```

このDockerfileは以下のようなレイヤーで構成されます。

1. `FROM python:3.12.1-slim` - ベースイメージのレイヤー
2. `COPY . /app` - アプリケーションコードをコピーするレイヤー
3. `RUN pip install -r /app/requirements.txt` - 依存関係をインストールするレイヤー
4. `CMD ["python", "/app/app.py"]` - コンテナ起動コマンドのレイヤー

コンテナイメージのビルド時に、Dockerは各レイヤーをキャッシュとして保存します。
レイヤーをキャッシュとして保存することで、同じレイヤーを再度ビルドする必要がなくなり、ビルド時間を短縮できます。

これはDockerあるいはコンテナ技術の大きなメリットの一つでもあり、キャッシュを利用することはDockerにおいてはベストプラクティスとされています。
キャッシュを利用するベストプラクティスの一例としてはマルステージビルドを使った特定のレイヤーのコピーがあります。

例えば、ビルドに必要なツールやライブラリをインストールするレイヤーを別のステージで作成し、最終的なイメージには必要なファイルだけをコピーする方法です。

具体的には以下のようなDockerfileになります。

```Dockerfile
# ビルドステージ
FROM python:3.12.1-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt
# 最終ステージ
FROM python:3.12.1-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
CMD ["python", "/app/app.py"]
```

この方法では、ビルドに必要な依存関係をインストールするレイヤーをビルドステージで作成し、最終的なイメージには必要なファイルだけをコピーしています。

これにより、最終的なコンテナイメージのサイズを小さく保ちつつ、ビルド時間を短縮できます。
また、セキュリティ面からは余計なパッケージやファイルを含めないことで、攻撃対象領域を減らすことができます。

## CodeBuildでキャッシュが利用できることの凄さ

CodeBuildは`ビルドに必要なコンピューティングを一時的に借りる`という仕組みであるために原則としてストレージを持たず
予め何か保持することはありません。
予め指定しておいたイメージをもとにコンピューティング環境を整えて、buildspecの内容に従い、ビルドを実行することを基本とします。

つまり、コンテナイメージはビルドのたびにレジストリからプルする必要があり、そのために毎回ビルド時間を確保する必要があります。
CodBuildでビルド時間を確保するということはより多くの課金を必要とするということになるため
キャッシュが利用できないということは料金に影響してくるということです。

今回のアップデートはこの問題をサクッと解決するためのアップデートと言えます。

## ハンズオン

それでは実際にAmazon ECRをCodeBuildのリモートキャッシュとして利用する方法をハンズオンで確認していきます。

### リポジトリをクローンする

セットアップするために、リポジトリをクローンします。

```bash
git clone https://github.com/ymd65536/CodebuildEcrCache.git
```

このリポジトリに今回のセットアップに必要なファイルが含まれています。
ディレクトリを変更します。

```bash
cd CodebuildEcrCache
```

## CodeBuildを構築してイメージをビルド

ではここから、コンテナイメージをCodeBuildでビルドしてECRにプッシュする手順を紹介します。
今回はCloudFormationテンプレートを使用してCodeBuildのプロジェクトを作成し、ビルドを実行します。

以下のコマンドでCloudFormationスタックをデプロイします。

```bash
aws cloudformation deploy --template-file image_build.yml --stack-name image_build --capabilities CAPABILITY_NAMED_IAM
```

スタックのデプロイが完了したら、以下のコマンドでビルドを開始します。`BUILD_ID`という変数の後ろから最後までのコマンドはビルドの進捗をポーリングしてステータスを表示するためのものです。

```bash
aws codebuild start-build --project-name image-build-stack-build-project --region ap-northeast-1 && BUILD_ID="image-build-stack-build-project:e8aeb764-5c76-480c-87e5-3a1183944c69" && while true; do STATUS=$(aws codebuild batch-get-builds --ids "$BUILD_ID" --region ap-northeast-1 --query 'builds[0].buildStatus' --output text); echo "Build status: $STATUS"; if [ "$STATUS" != "IN_PROGRESS" ]; then break; fi; sleep 10; done && echo "Final status: $STATUS"
```

※2回ほどビルドを試してみてください。

### ビルドログを確認する

では、実際にCodeBuildのビルドログを確認していきます。
最初の実行ではエラーが発生していますが、これはキャッシュが存在しないために発生しています。

![codebuildcache-1.png](./images/codebuildcache-1.png)

2回目以降のビルドではキャッシュが存在するため、ECRからキャッシュがプルされ、ビルド時間が短縮されていることがわかります。

![codebuildcache-2.png](./images/codebuildcache-2.png)

### 実行結果

今回のハンズオンでは以下のようなビルド時間となりました。
ビルドイメージが小さいため、あまり大きな差は出ていませんが、リモートキャッシュを利用することでビルド時間が短縮されていることがわかります。

|回数|ビルド時間（秒）|
|---|---|
|1回目|51|
|2回目|48|
|3回目|47|

## 検証してみての感想

ハンズオンは以上です。ここからは検証してみての感想を書いていきます。

### よいところ： リモートキャッシュの導入が非常に簡単

以前まではビルド時間を短縮するためにさまざまな方法があったと思います。具体的には以下のとおりです。

- ベースイメージを別のECRリポジトリに保存してdockerfileから直接参照
- cache-fromオプションを利用してレジストリをキャッシュとして利用

上記の工夫をするためにマルチステージビルドを利用したり、そもそものイメージサイズの削減を検討したりなどDockerfileの設計に工夫が必要でした。
※マルチステージビルドを活用しない場合はdockerfileを分割するなどの対応が必要になる場合もあります。
※イメージサイズの削減： パッケージキャッシュのクリーンや不要パッケージの削除など

今回紹介したリモートキャッシュは少しだけ設定を変えてあげるだけで利用できるので非常に導入しやすいです。

### よくないところ: CodeBuildがECRのキャッシュを利用しているかどうかの確認方法

CodeBuildがリモートキャッシュを正しく利用しているかどうかを確認するには、ビルドログを詳細に確認する必要があります。
ぱっと見ではキャッシュが利用されているかどうかはわかりにくいように感じました。

たしかに、ECRには`pull counts`というメトリクスが用意されており、どれくらいアクセスされているかは確認できます。
しかし、そのプルが実際にキャッシュを利用しているかどうかはビルドログを確認しないことにはわかりません。

### 要検討: 導入するかどうかの判断基準

リモートキャッシュを導入するかどうかの判断基準としては、実装コストと削減効果のトレードオフを考慮する必要があります。

特にこのリモートキャッシュを利用する上で、buildxの導入をCodeBuildの環境で毎回セットしておく必要があり、このセットにはおよそ8.4秒程度の時間がかかります。
※筆者の環境での検証結果

また、ビルドのたびにキャッシュの更新が発生するため、ストレージコストとそして、短いながらも追加のビルド時間を考慮する必要があります。
※キャッシュの更新にはexport/importの時間がかかり、筆者の環境ではexportに3.9秒、importに2.3秒かかりました。

なお、キャッシュを実装したことによって発生した追加のビルド時間ですが
筆者の環境では合計で14.6秒の追加時間が発生しました。
※`public.ecr.aws/amazonlinux/amazonlinux:2023.9.20251117.1`だけのdockerfileの場合、8.4+3.9+2.3=14.6秒の追加時間が発生しました。

### 要検討: リモートキャッシュ(ECR)に何を保存するか

CodeBuildのリモートキャッシュとしてECRを利用する場合、キャッシュ用のイメージをどのように管理するかが重要です。
変更がほとんどいらないようなベースイメージを管理することがこの機能の主なユースケースになると思います。

たとえば、頻繁に更新されるアプリケーションコードや依存関係を含むレイヤーはキャッシュの恩恵を受けにくいです。
一方で、ベースイメージや共通のライブラリを含むレイヤーはキャッシュとして保存することで、ビルド時間の短縮やリソースの節約につながります。

ですので、この機能を最大限に利用できる人はDockerfileの設計に工夫を凝らし、キャッシュとして保存するレイヤーを明確に分離することが重要です。
また、Dockerfileやコンテナイメージの知見はもちろんのこと、CodeBuildのビルドプロセスに関する理解も必要になると感じました。

### 要検討: リモートキャッシュをどう作成して管理するか

リモートキャッシュをECRに保存する場合、専用のリポジトリを作成して管理することが推奨されます。
専用のリポジトリをどのアカウントにどれくらい保存するのか、またライフサイクルポリシーをどう設定するかなど運用面での検討が必要です。

今回はシングルアカウントでの検証でしたが、マルチアカウント環境での利用を考慮すると
共有アカウントにキャッシュ用のECRリポジトリを作成し、各開発アカウントからアクセスできるようにする方法が考えられます。

本番環境がある場合は、開発環境と本番環境でキャッシュ用のECRリポジトリを分けることも検討すべきです。

## まとめ

今回はECRをCodeBuildのキャッシュに利用してみました。
今回の検証では小さなイメージを利用していたため、あまり凄さを体感できなかったと思いますが
大きなコンテナイメージを利用している場合は特に効果が大きいと思います。

大きなコンテナイメージというとたとえば、Node.jsのnode_moduleなどを固めたレイヤーなどが考えられ
細かいファイル群を固めて持ってくるということができる、そういったケースでキャッシュの恩恵を受けられかなと思います。

キャッシュを必要とする例に最近ではよく遭遇するため、積極的に使っていきたいと思います。

## 参考

- [Reduce Docker image build time on AWS CodeBuild using Amazon ECR as a remote cache](https://aws.amazon.com/jp/blogs/devops/reduce-docker-image-build-time-on-aws-codebuild-using-amazon-ecr-as-a-remote-cache/)

## AWS CLI インストールと SSO ログイン手順 (Linux環境)

このガイドでは、Linux環境でAWS CLIをインストールし、AWS SSOを使用してログインするまでの手順を説明します。

## 前提条件

- Linux環境（Ubuntu、CentOS、Amazon Linux等）
- インターネット接続
- 管理者権限（sudoが使用可能）
- AWS SSO が組織で設定済み
- Python 3.12.1

## AWS CLI のインストール

### 公式インストーラーを使用（推奨）

最新版のAWS CLI v2を公式インストーラーでインストールします。

```bash
# 1. インストーラーをダウンロード
curl "https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip" -o "awscliv2.zip"

# 2. unzipがインストールされていない場合はインストール
sudo apt update && sudo apt install unzip -y  # Ubuntu/Debian系
# または
sudo yum install unzip -y                     # CentOS/RHEL系

# 3. ダウンロードしたファイルを展開
unzip awscliv2.zip

# 4. インストール実行
sudo ./aws/install

# 5. インストール確認
aws --version

# ダウンロードしたzipファイルと展開したディレクトリを削除してクリーンアップします。
rm  "awscliv2.zip"

# 解凍したディレクトリを削除
rm -rf aws
```

## AWS SSO の設定とログイン

### 1. AWS SSO の設定

AWS SSOを使用するための初期設定を行います。

```bash
aws configure sso
```

設定時に以下の情報の入力が求められます：

- **SSO start URL**: 組織のSSO開始URL（例：`https://my-company.awsapps.com/start`）
- **SSO Region**: SSOが設定されているリージョン（例：`us-east-1`）
- **アカウント選択**: 利用可能なAWSアカウントから選択
- **ロール選択**: 選択したアカウントで利用可能なロールから選択
- **CLI default client Region**: デフォルトのAWSリージョン（例：`ap-northeast-1`）
- **CLI default output format**: 出力形式（`json`、`text`、`table`のいずれか）
- **CLI profile name**: プロファイル名（`default`とします。）

### 2. AWS SSO ログイン

設定完了後、以下のコマンドでログインを実行します。

```bash
aws sso login
```

ログイン時の流れ：
1. コマンド実行後、ブラウザが自動的に開きます
2. AWS SSOのログインページが表示されます
3. 組織のIDプロバイダー（例：Active Directory、Okta等）でログイン
4. 認証が成功すると、ターミナルに成功メッセージが表示されます

### 3. ログイン状態の確認

認証情報を確認します。

```bash
aws sts get-caller-identity
```

正常にログインできている場合、以下のような情報が表示されます：

```json
{
    "UserId": "AROAXXXXXXXXXXXXXX:username@company.com",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/RoleName/username@company.com"
}
```

## トラブルシューティング

### よくある問題と解決方法

#### 1. ブラウザが開かない場合

```bash
# 手動でブラウザを開く場合のURL確認
aws sso login --no-browser
```

表示されたURLを手動でブラウザで開いてください。

#### 2. セッションが期限切れの場合

```bash
# 再ログイン
aws sso login
```

#### 4. プロキシ環境での設定

プロキシ環境の場合、以下の環境変数を設定してください：

```bash
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
export NO_PROXY=localhost,127.0.0.1,.company.com
```

## セキュリティのベストプラクティス

1. **定期的な認証情報の更新**: SSOセッションには有効期限があります。定期的に再ログインを行ってください。

2. **最小権限の原則**: 必要最小限の権限を持つロールを使用してください。

3. **プロファイルの分離**: 本番環境と開発環境で異なるプロファイルを使用してください。

4. **ログアウト**: 作業終了時は適切にログアウトしてください：
   ```bash
   aws sso logout --profile <プロファイル名>
   ```

## 参考リンク

- [AWS CLI ユーザーガイド](https://docs.aws.amazon.com/cli/latest/userguide/)
- [AWS SSO ユーザーガイド](https://docs.aws.amazon.com/singlesignon/latest/userguide/)
- [AWS CLI インストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
