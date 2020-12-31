# NoerPageManager
nomotoeriko.github.io の管理用アプリです。

## bookshelf
[本棚](https://nomotoeriko.github.io/book.html)、[映画](https://nomotoeriko.github.io/film.html)、[アニメ](https://nomotoeriko.github.io/anime.html)の視聴履歴ページに作品を追加するためのslack連携アプリ

### 使い方
bookshelf インストール済みの Slack ワークスペースで下記のようにスラッシュコマンドを打つ。

    /addtonoerpage film タイトル 監督 視聴のきっかけ 感想

コマンドが成功すると以下のように登録情報と作成されたブランチへのリンクが返ってくる。

![success-add-to-noer-page-responce](images/success-add-to-noer-page-responce.png)

フォーマットが正しくない場合は以下のようにメッセージが返ってくる。

![fail-add-to-noer-page-responce](images/fail-add-to-noer-page-responce.png)

### 構成
登場人物は以下

* Slack API
* AWS Elastic Beanstalk
* Ruby + Sinatra

#### Slack API
Slackアプリの受け口。今回は[スラッシュコマンド](https://api.slack.com/interactivity/slash-commands)のみ利用。他にも、botにメンションを送るなどの[イベントをトリガ](https://api.slack.com/events-api)とすることもできる。

#### AWS Elastic Beanstalk
スラッシュコマンドが叩かれると Slack API は設定された URL に HTTP POST を送る（[参考](https://api.slack.com/interactivity/slash-commands#app_command_handling)）。ローカル環境でこのポストを受け取れるようにするのはちょっと面倒だったので AWS を利用することにした。

ちょうど友人と遊びでアプリ開発をしていて、その時に [Elastic Beanstalk](https://aws.amazon.com/jp/elasticbeanstalk/) を使ってみてたところだったので、今回も同じく Elastic Beanstalk を利用することにした。

AWS 上で Web アプリが動く環境を整えようとすると、EC2、S3、ELBなどなど、さまざまな設定を行う必要がある。Elastic Beanstalk はその辺りの各種設定をいい感じに行ってくれて、利用者はアプリ本体の開発にのみ専念できる。最低限動くソースが提供されていたので、今回のアプリ開発はそれに手を加えるだけでよくてお手軽だった。

#### アプリ本体
Slack API から HTTP POST を受け取って処理を実行する部分。今回は作品情報を受け取ってファイルを書き換えるだけの簡単な処理なので、手軽にWebアプリケーションを作成できる[Sinatra](http://sinatrarb.com/intro-ja.html)を利用。

処理の流れはざっと以下
1. 受け取った作品情報のフォーマットを検証
1. 検証結果に基づきレスポンスを返却。フォーマットが正しい場合はファイル書き換えに必要な情報をインスタンス変数に保持しておく。
1. レスポンス返却後、インスタンス変数に格納した情報を元にhtmlファイルを書き換えてコミットを積む

先にレスポンスを返してからファイル編集の処理を行っているのはスラッシュコマンドのタイムアウト除けのため。3秒以内にレスポンスを返せなければタイムアウトとして処理されてしまうから。

### デプロイ手順
備忘録として、簡単にデプロイ手順を記しておく。

* bookshelf アプリのソースを zip に固める

      cd bookshelf
      zip ../bookshelf.zip -r * .[^.gD]*

* AWS Elastic Beanstalk でアプリケーションを作成
  * プラットフォームは Ruby 2.7（2020/12/31現在での最新版）を選択
  * ソースはアップロードを選択し、bookshelf.zipをアップロードする
  * 作成したアプリの [設定] からソフトウェア設定の編集画面に行き、以下の環境変数を登録する
    * GIT_REPOSITORY_URI: ホームページのリポジトリのURI。認証を通すために、`https://USER_NAME:PASSWORD@github.com/USER_NAME/REPOSITORY_NAME.git` というフォーマットにしておく。
    * GIT_USER_NAME: コミットを積むユーザー名。多分なんでもいいので、適当に bookshelf とかにしておけばいいと思う。
    * GIT_USER_EMAIL: コミットを積むユーザーのメールアドレス。多分なんでもいいので、適当に自分のメアドとかを入れておけばいいと思う。
  * デプロイ完了後、Elastic Beanstalkのアプリケーション管理画面に表示されているアプリケーションのURLを踏んで画面に OK と表示されていればデプロイは成功
* Slack API で bookshelf アプリを作成し、Slask Commands タブからスラッシュコマンドを新規作成
  * Request URL には Elastic Beanstalk で作成したアプリケーションのURL の末尾に `/add` を加えたものを設定
* bookshelf アプリを Slack にインストール
  * Slack API の bookshelf アプリ管理画面の Install App タブから、好きなワークスペースにアプリをインストールする
