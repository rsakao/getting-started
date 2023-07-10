マルチコンテナーアプリケーションをすでに扱ってきましたが、MySQL をアプリケーションスタックに追加することを考えています。以下のような問題がしばしば発生します。「MySQL はどこで実行するのですか？同じコンテナにインストールするのか、別々に実行するのか？」一般的には、「それぞれのコンテナは1つのことをしてそれに徹するべきです。」いくつかの理由があります。

- API やフロントエンドをデータベースよりも別々にスケールする必要がある可能性が高い。
- 別々のコンテナでは、バージョンを分離して更新することができます。
- ローカルでデータベース用のコンテナを使用するかもしれませんが、本番環境ではデータベースのマネージドサービスを使用したい場合もあるでしょう。データベースエンジンをアプリと一緒に出荷するわけにはいきません。
- 複数のプロセスを実行するにはプロセスマネージャーが必要です（コンテナは1つのプロセスしか開始しないため）、これはコンテナの起動/シャットダウンに複雑さを追加します。

そして、もっと理由があります。そのため、次のようにアプリケーションを更新します。

![Todo App connected to MySQL container](multi-app-architecture.png)
{: .text-center }


## コンテナネットワーキング

コンテナはデフォルトで分離されており、同じマシン上の他のプロセスまたはコンテナについて何も知りません。では、1つのコンテナが別のコンテナと話す方法は何でしょうか？答えは **ネットワーキング** です。 ネットワークエンジニアである必要はありません（やった！）。単に次のルールを覚えてください...

> 同じネットワークにある2つのコンテナは、互いに通信できます。そうでない場合、通信できません。


## MySQL の起動

コンテナをネットワークに追加する方法は2つあります。 1) スタート時に割り当てるか 2) 既存のコンテナに接続するか。今回は、最初にネットワークを作成し、MySQL コンテナを起動時にアタッチすることにします。

1. ネットワークを作成します。

    ```bash
    docker network create todo-app
    ```

1. MySQL コンテナを開始し、ネットワークにアタッチします。 また、データベースを初期化するために使用するいくつかの環境変数を定義します（ [MySQL Docker Hub リスト](https://hub.docker.com/_/mysql/) の「環境変数」セクションを参照）。

    ```bash
    docker run -d \
        --network todo-app --network-alias mysql \
        -v todo-mysql-data:/var/lib/mysql \
        -e MYSQL_ROOT_PASSWORD=secret \
        -e MYSQL_DATABASE=todos \
        mysql:8.0
    ```

    PowerShell を使用している場合は、次のコマンドを使用してください。

    ```powershell
    docker run -d `
        --network todo-app --network-alias mysql `
        -v todo-mysql-data:/var/lib/mysql `
        -e MYSQL_ROOT_PASSWORD=secret `
        -e MYSQL_DATABASE=todos `
        mysql:8.0
    ```

    `--network-alias` フラグを指定したことにも注目してください。すぐに戻ってきます。

    !!! info "プロのアドバイス"
        ここで `todo-mysql-data` という名前のボリュームを使用して、データの保存場所である `/var/lib/mysql` にマウントしていることに気付くかもしれません。しかし、私たちは `docker volume create` コマンドを実行していません。Docker は、名前付きボリュームを使用することを望んでいると認識して自動的にそれを作成します。

1. データベースが起動していることを確認するために、データベースに接続して接続されていることを確認します。

    ```bash
    docker exec -it <mysql-container-id> mysql -p
    ```

    パスワードプロンプトが表示されたら、**secret**と入力します。 MySQL シェルで、データベースをリストし、`todos` データベースが表示されることを確認します。

    ```cli
    mysql> SHOW DATABASES;
    ```

    以下のような出力が表示されるはずです。

    ```plaintext
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    | todos              |
    +--------------------+
    5 rows in set (0.00 sec)
    ```

    準備が整いました！`todos`データベースを使用できるようになりました！

    sql ターミナルから抜けるには、ターミナルに `exit` を入力してください。


## MySQL に接続する

MySQL が起動していることがわかったので、使ってみましょう！でも、問題は...どうやって？同じネットワーク上で別のコンテナを実行した場合、各コンテナには独自の IP アドレスがあることを覚えているでしょうか？

それを解決するために、非常に役立つトラブルシューティングやデバッグに役立つ多くのツールを備えた [nicolaka/netshoot](https://github.com/nicolaka/netshoot) コンテナを利用します。

1. nicolaka/netshoot イメージを使用して新しいコンテナを起動します。同じネットワークに接続することを忘れないでください。

    ```bash
    docker run -it --network todo-app nicolaka/netshoot
    ```

1. コンテナ内で、DNS ツールとして役立つ `dig` コマンドを使用します。`mysql` のホスト名の IP アドレスを検索します。

    ```bash
    dig mysql
    ```

    次のような出力が表示されます。

    ```text
    ; <<>> DiG 9.18.8 <<>> mysql
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32162
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

    ;; QUESTION SECTION:
    ;mysql.				IN	A

    ;; ANSWER SECTION:
    mysql.			600	IN	A	172.23.0.2

    ;; Query time: 0 msec
    ;; SERVER: 127.0.0.11#53(127.0.0.11)
    ;; WHEN: Tue Oct 01 23:47:24 UTC 2019
    ;; MSG SIZE  rcvd: 44
    `` 
 ```
「ANSWER SECTION」には、`mysql` の `A` レコードが表示され、そのホスト名に対する IP アドレスが `172.23.0.2` であることが分かります（あなたの IP アドレスは大抵異なる値になるはずです）。通常、`mysql` は有効なホスト名ではありませんが、Docker は、このネットワークエイリアスを持つコンテナの IP アドレスに解決できました（先に使った `--network-alias` フラグを覚えていますか？）。

つまり、私たちのアプリケーションは、`mysql` という名前のホストに接続するだけで、データベースと通信できるようになりました！もう少し簡単な方法はありません！

完了したら、 `exit` を実行してコンテナを終了してください。


## MySQL を使用したアプリケーションの実行

todoアプリは、MySQL 接続設定を指定するためのいくつかの環境変数をサポートしています。 それらは以下のとおりです。

- `MYSQL_HOST` - 実行中の MySQL サーバーのホスト名
- `MYSQL_USER` - 接続に使用するユーザー名
- `MYSQL_PASSWORD` - 接続に使用するパスワード
- `MYSQL_DB` - 接続後に使用するデータベース

!!! warning
    Env vars を使用して接続設定を指定することは、一般的に開発時には問題ありませんが、本番でアプリケーションを実行する場合には **強く推奨されません**。以前 Docker のセキュリティを担当していた Diogo Monica 
    は、[ブログ投稿](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/)で詳しく説明しています。

    より安全な方法は、コンテナオーケストレーションフレームワークによって提供されるシークレット機能を使用することです。ほとんどの場合、これらのシークレットは実行中のコンテナにファイルとしてマウントされます。多くのアプリケーション（MySQL イメージや todo アプリを含む）も、変数を参照する `_FILE` サフィックスの env vars をサポートしています。

    たとえば、`MYSQL_PASSWORD_FILE` 変数を設定すると、アプリは参照されたファイルの内容を接続パスワードとして使用します。 Docker は、これらの env vars をサポートするために何も行いません。アプリが変数を探してファイルの内容を取得する必要があります。


すべての説明を聞いたので、dev-ready コンテナを起動しましょう！

1. 上記のすべての環境変数を指定し、コンテナをアプリのネットワークに接続します。

    ```bash hl_lines="3 4 5 6 7"
    docker run -dp 3000:3000 \
      -w /app -v "$(pwd):/app" \
      --network todo-app \
      -e MYSQL_HOST=mysql \
      -e MYSQL_USER=root \
      -e MYSQL_PASSWORD=secret \
      -e MYSQL_DB=todos \
      node:18-alpine \
      sh -c "yarn install && yarn run dev"
    ```

    PowerShell を利用している場合は、以下のコマンドを使用してください。

    ```powershell hl_lines="3 4 5 6 7"
    docker run -dp 3000:3000 `
      -w /app -v "$(pwd):/app" `
      --network todo-app `
      -e MYSQL_HOST=mysql `
      -e MYSQL_USER=root `
      -e MYSQL_PASSWORD=secret `
      -e MYSQL_DB=todos `
      node:18-alpine `
      sh -c "yarn install && yarn run dev"
    ```

1. コンテナのログ（ `docker logs <container-id>` ）を確認すると、MySQL データベースを使用していることが示されるメッセージが表示されます。

    ```plaintext hl_lines="7"
    # Previous log messages omitted
    $ nodemon src/index.js
    [nodemon] 2.0.20
    [nodemon] to restart at any time, enter `rs`
    [nodemon] watching path(s): *.*
    [nodemon] watching extensions: js,mjs,json
    [nodemon] starting `node src/index.js`
    Connected to mysql db at host mysql
    Listening on port 3000
    ```

1. ブラウザでアプリを開き、タスクを追加します。

1. データベースに書き込まれていることを確認するために、MySQL データベースに接続してください。パスワードは **secret** です。

    ```bash
    docker exec -it <mysql-container-id> mysql -p todos
    ```

    MySQL シェルで、次のコマンドを実行します。

    ```plaintext
    mysql> select * from todo_items;
    +--------------------------------------+--------------------+-----------+
    | id                                   | name               | completed |
    +--------------------------------------+--------------------+-----------+
    | c906ff08-60e6-44e6-8f49-ed56a0853e85 | Do amazing things! |         0 |
    | 2912a79e-8486-4bc3-a4c5-460793a575ab | Be awesome!        |         0 |
    +--------------------------------------+--------------------+-----------+
    ```

    明らかに、あなたのテーブルは異なる見た目をしています。しかし、そこに格納されているタスクを確認できるはずです。

Docker Dashboard をチェックすると、2つのアプリケーションコンテナが実行されていることが分かります。しかし、それらが1つのアプリケーションにグループ化されているという明確な指示はありません。次のセクションでは、Docker Compose について説明します。Docker Compose を使用することで、他の人に簡単なコマンドでアプリケーションスタックを共有することができます！


![グループ化されていない2つのアプリコンテナが表示される Docker Dashboard](dashboard-multi-container-app.png)


## まとめ

この時点で、外部のデータベースを使用するアプリケーションを持っています。ネットワークについて少し学び、DNS を使ったサービスディスカバリーの方法を見ました。

しかし、起動するために行うことがいろいろあって、少し圧倒されてしまったかもしれません。ネットワークを作成したり、コンテナを開始したり、すべての環境変数を指定したり、ポートを公開したり、その他色々な手順を実行する必要があります！それは覚えるのに多くの時間が必要で、他の人に渡すのも難しくなります。

次のセクションでは、Docker Composeについて説明します。Docker Composeを使えば、もっと簡単な方法でアプリケーションスタックを共有し、1つの（そしてシンプルな）コマンドで他の人に起動させることができます！