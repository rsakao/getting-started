前の章では、**名前付きボリューム** を使用してデータを永続化する方法について話しました。
名前付きボリュームは、単にデータを保存したい場合に優れています。なぜなら、データがどこに保存されているかを心配する必要がないからです。

**バインドマウント** では、ホスト上のマウントポイントを正確に制御できます。これを使用してデータを永続化することもできますが、しばしばコンテナに追加のデータを提供するために使用されます。アプリケーションを開発する場合、バインドマウントを使用して、ソースコードをコンテナにマウントして、変更に応答し、変更をすぐに確認できるようにします。

Node.jsベースのアプリケーションの場合、[nodemon](https://npmjs.com/package/nodemon)はファイルの変更を監視してアプリケーションを再起動するための素晴らしいツールです。他の言語やフレームワークでも同様のツールがあります。

## ボリュームの種類を比較する

バインドマウントと名前付きボリュームは、Dockerエンジンに付属する2つの主要なボリュームのタイプです。ただし、その他のユースケースをサポートするために、追加のボリュームドライバが利用可能です（[SFTP](https://github.com/vieux/docker-volume-sshfs)、[Ceph](https://ceph.com/geen-categorie/getting-started-with-the-docker-rbd-volume-plugin/)、[NetApp](https://netappdvp.readthedocs.io/en/stable/)、[S3](https://github.com/elementar/docker-s3-volume)など）。

|   | 名前付きボリューム | バインドマウント |
| - | ------------- | ----------- |
| ホスト上の位置 | Dockerが選択 | あなたが制御 |
| ボリュームを新しいコンテナ内のコンテンツで埋める | はい | いいえ |
| ボリュームドライバをサポートする | はい | いいえ |


## 開発モードのコンテナを開始する

開発ワークフローをサポートするためにコンテナを実行するには、次の操作を実行します。

- ソースコードをコンテナにマウントする
- 「dev」依存関係を含むすべての依存関係をインストールする
- ファイルシステムの変更を監視するためにnodemonを開始する

それではやってみましょう！

1. 自分自身の`getting-started`コンテナが実行されていないことを確認してください（チュートリアル自体だけが実行されている必要があります）。

1. `app`のソースコードディレクトリ内にいることを確認してください。例：`/path/to/getting-started/app`。そうでない場合は、次のように`cd`してください。

    ```bash
    cd /path/to/getting-started/app
    ```

1. 次に、以下のコマンドを実行します。後で何が起こっているか説明します。

    ```bash
    docker run -dp 3000:3000 \
        -w /app -v "$(pwd):/app" \
        node:18-alpine \
        sh -c "yarn install && yarn run dev"
    ```

    PowerShellを使用している場合は、このコマンドを使用してください。

    ```powershell
    docker run -dp 3000:3000 `
        -w /app -v "$(pwd):/app" `
        node:18-alpine `
        sh -c "yarn install && yarn run dev"
    ```

    - `-dp 3000:3000` - 前回と同じです。デタッチド（バックグラウンド）モードで実行し、ポートマッピングを作成します。
    - `-w /app` - コマンドが実行されるコンテナのカレントディレクトリを設定します。
    - `-v "$(pwd):/app"` - ホストの現在の`getting-started/app`ディレクトリを、コンテナの`/app`ディレクトリにバインドマウント（リンク）します。注意：Dockerはバインドマウントに絶対パスを必要とするため、この例では、作業ディレクトリ、つまり`app`ディレクトリの絶対パスを印刷するために`pwd`を使用します。
    - `node:18-alpine` - 使用するイメージです。これは、Dockerfileからアプリのベースイメージであることに注意してください。
    - `sh -c "yarn install && yarn run dev"` - コマンドです。`sh`（alpineには`bash`がないため）を使用してシェルを起動し、_すべて_の依存関係をインストールするために`yarn install`を実行し、その後`yarn run dev`を実行します。 `package.json`を見ると、`dev`スクリプトが`nodemon`を開始していることがわかります。

1. `docker logs -f <container-id>`を使用してログをウォッチできます。次のように表示されると、準備完了です...

    ```bash
    docker logs -f <container-id>
    $ nodemon src/index.js
    [nodemon] 2.0.20
    [nodemon] to restart at any time, enter `rs`
    [nodemon] watching path(s): *.*
    [nodemon] watching extensions: js,mjs,json
    [nodemon] starting `node src/index.js`
    Using sqlite database at /etc/todos/todo.db
    Listening on port 3000
    ```

    ログを見ているときに終了するには、`Ctrl`+`C`を押して終了してください。

1. 今度はアプリケーションを変更してみましょう。 `src/static/js/app.js`ファイルで、「Add Item」ボタンを「Add」に変更します。この変更は109行目にあります - ファイルを保存することを忘れないでください。

    ```diff
    -                         {submitting ? 'Adding...' : 'Add Item'}
    +                         {submitting ? 'Adding...' : 'Add'}
    ```

1. 単純にページを更新するか、開くだけで、ブラウザで変更がほぼ即座に反映されるはずです。 Nodeサーバーが再起動するまで数秒かかる場合があるため、エラーが表示された場合は数秒後にもう一度更新してください。

    ![Updated label for Add button のスクリーンショット](updated-add-button.png){: style="width:75%;"}
    {: .text-center }

1. 他にも変更したいことがあれば、自由に変更してください。終わったら、コンテナを停止し、新しいイメージを`docker build -t getting-started .`を使用してビルドしてください。

バインドマウントは、ローカル開発セットアップでは非常に一般的です。利点は、開発マシンにビルドツールと環境をすべてインストールする必要がないことです。1つの`docker run`コマンドで、dev環境がプルされて準備完了になります。Docker Composeについては、将来のステップで説明します。これにより、フラグが多くなっています。

## まとめ

この時点で、データベースを永続化し、投資家や創業者の要望に素早く対応することができます。やったね！ただし、素晴らしいニュースが届きました！

**あなたのプロジェクトが将来開発対象に選ばれました！**

生産準備を整えるために、SQLiteで動作していたデータベースを、少しスケーラブルなものに切り替える必要があります。単純化のために、リレーショナルデータベースを使用したまま、アプリケーションをMySQLに切り替えます。しかし、MySQLをどのように実行すればよいのでしょうか？コンテナ同士が通信する方法は？次の章で説明します！