[Docker Compose](https://docs.docker.com/compose/)は、マルチコンテナーアプリケーションの定義と共有を支援するために開発されたツールです。Composeを使用するとYAMLファイルを作成してサービスを定義し、単一のコマンドですべてを起動またはシャットダウンすることができます。

Composeを使用する大きな利点は、アプリケーションスタックをファイルで定義し、プロジェクトリポジトリのルートに保持し（バージョン管理されるようになり）、他の人がプロジェクトに貢献できるようにすることができることです。誰かがあなたのリポジトリをクローンしてComposeアプリを起動するだけで、その人があなたのプロジェクトに貢献することができます。実際、GitHub / GitLab でこれを実行しているプロジェクトがかなりあります。

それでは、どうやって始めればいいでしょうか？

## Docker Composeのインストール

Windows、Mac、Linux用にDocker Desktopをインストールした場合、すでにDocker Composeがインストールされています！Play-with-DockerインスタンスにもDocker Composeがインストールされています。他のシステムを使用している場合は、[こちらの手順](https://docs.docker.com/compose/install/)に従ってDocker Composeをインストールできます。


## Composeファイルの作成

1. アプリのフォルダー内に、`Dockerfile`と`package.json`ファイルの横に`docker-compose.yml`という名前のファイルを作成します。

1. composeファイルでは、アプリケーションの一部として実行するサービス（またはコンテナ）のリストを定義することから始めます。

    ```yaml
    services:
    ```

それでは、composeファイルにサービスをひとつずつ移行していきましょう。


## アプリケーションサービスの定義

以下は私たちがアプリのコンテナを定義するために使用していたコマンドを思い出すためのものです。

```bash
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

1. まず、サービスエントリとコンテナのイメージを定義することから始めましょう。サービスに任意の名前を選択できます。
   名前は自動的にネットワークエイリアスになり、MySQLサービスを定義するときに役立ちます。

    ```yaml hl_lines="2 3"
    services:
      app:
        image: node:18-alpine
    ```

1. 通常、コマンドは `image` の定義に近い位置にありますが、順序には制限はありません。
   そこで、コマンドもファイルに追加してください。

    ```yaml hl_lines="4"
    services:
      app:
        image: node:18-alpine
        command: sh -c "yarn install && yarn run dev"
    ```


1. 次に、コマンドラインの `-p 3000:3000` 部分を移行して、サービスのポートを定義します [short syntax](https://docs.docker.com/compose/compose-file/#short-syntax-2) を使用しますが、 [long syntax](https://docs.docker.com/compose/compose-file/#long-syntax-2) もあります。

    ```yaml hl_lines="5 6"
    services:
      app:
        image: node:18-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
          - 3000:3000
    ```

1. 次に、ボリュームマッピング(`-w /app`)と(`-v "$(pwd):/app"`)の両方を `working_dir` と `volumes` 定義を使って移行してください。ボリュームには、相対パスを現在のディレクトリから使用できる多くのオプションがあります[Docker Compose volumes definitions](https://docs.docker.com/compose/compose-file/#volumes-top-level-element)。

    ```yaml hl_lines="7 8 9"
    services:
      app:
        image: node:18-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
          - 3000:3000
        working_dir: /app
        volumes:
          - ./:/app
    ```

1. 最後に、環境変数定義を `environment` キーを使用して移行します。

    ```yaml hl_lines="10 11 12 13 14"
    services:
      app:
        image: node:18-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
          - 3000:3000
        working_dir: /app
        volumes:
          - ./:/app
        environment:
          MYSQL_HOST: mysql
          MYSQL_USER: root
          MYSQL_PASSWORD: secret
          MYSQL_DB: todos
    ```
  
### MySQLサービスの定義

さあ、MySQLサービスを定義する時間です。そのコンテナに使用したコマンドは以下の通りです。

```bash
docker run -d \
  --network todo-app --network-alias mysql \
  -v todo-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  mysql:8.0
```

1. 新しいサービスを定義して、自動的にネットワークエイリアスが付与されるように `mysql` と名前を付けます。次に、使用するイメージを指定します。

    ```yaml hl_lines="4 5"
    services:
      app:
        # The app service definition
      mysql:
        image: mysql:8.0
    ```

1. 次に、ボリュームマッピングを定義します。 'docker run'でコンテナを実行すると、名前付きボリュームが自動的に作成されます。ただし、Composeを使用して実行する場合は、ボリュームをトップレベルの `volumes:` セクションで定義し、サービス設定でマウントポイントを指定する必要があります。ボリューム名のみを指定するだけで、デフォルトオプションが使用されます。

    ```yaml hl_lines="6 7 8 9 10"
    services:
      app:
        # The app service definition
      mysql:
        image: mysql:8.0
        volumes:
          - todo-mysql-data:/var/lib/mysql
    
    volumes:
      todo-mysql-data:
    ```

1. 最後に、環境変数を指定するだけです。

    ```yaml hl_lines="8 9 10"
    services:
      app:
        # The app service definition
      mysql:
        image: mysql:8.0
        volumes:
          - todo-mysql-data:/var/lib/mysql
        environment: 
          MYSQL_ROOT_PASSWORD: secret
          MYSQL_DATABASE: todos
    
    volumes:
      todo-mysql-data:
    ```

ここまでの `docker-compose.yml` ファイル全体は次のようになります:

```yaml
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment: 
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

## アプリケーションスタックの実行

docker-compose.ymlファイルがあるので、それを起動できます！

1. 他にアプリ/DBコピーを実行していないことを確認します（`docker ps` および `docker rm -f <ids>`）。

1. アプリケーションスタックを `docker compose up` コマンドで起動します。バックグラウンドですべてを実行するために `-d` フラグを追加します。

    ```bash
    docker compose up -d
    ```

    実行時に、次のような出力が表示されるはずです。

    ```plaintext
    [+] Running 3/3
    ⠿ Network app_default    Created                                0.0s
    ⠿ Container app-mysql-1  Started                                0.4s
    ⠿ Container app-app-1    Started                                0.4s
    ```

    ボリュームとネットワークが作成されたことに注意してください！ Docker Composeは、アプリケーションスタック専用にネットワークを自動的に作成するように設定されています（そのため、Composeファイルでネットワークを定義する必要はありませんでした）。

1. `docker compose logs -f`コマンドを使用してログを見てみましょう。各サービスのログが結合されて1つのストリームになっているのがわかります。これは、タイミング関連の問題を監視したい場合に非常に便利です。フラグ`-f`はログを「フォロー」するため、生成されるたびにライブ出力を提供します。

    既に持っていない場合は、次のような出力が表示されます...

    ```plaintext
    mysql_1  | 2022-11-23T04:01:20.185015Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.31'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
    app_1    | Connected to mysql db at host mysql
    app_1    | Listening on port 3000
    ```

    サービス名が行の先頭に表示され（しばしば色付けされている）ため、メッセージを区別するのに役立ちます。特定のサービスのログを表示したい場合は、ログにサービス名を追加できます（例：`docker compose logs -f app`）。

    !!! info "追加情報：アプリを起動する前にDBが準備できるまで待機する"
        アプリが起動すると、MySQLがアップして使用可能になるまで待機します。Dockerには、他のコンテナが完全にアップして実行され、準備が整うのを待つためのビルトインサポートはありません。 Nodeベースのプロジェクトでは、[wait-port](https://github.com/dwmkerr/wait-port)依存関係を使用できます。他の言語/フレームワーク向けにも同様のプロジェクトがあります。

1. この時点で、アプリを開いて実行しているのを確認できるはずです。何と！単一のコマンドに縮小しました！

## Docker Dashboardでのアプリケーションスタックの表示

Docker Dashboardを見ると、**app** という名前のグループがあることがわかります。これはDocker Composeからの「プロジェクト名」で、コンテナをグループ化するために使用されます。デフォルトでは、プロジェクト名は `docker-compose.yml` が存在するディレクトリの名前です。

![Docker Dashboard with app project](dashboard-app-project-collapsed.png)

appを展開すると、コンポーズファイルで定義した2つのコンテナが表示されます。名称も少し説明的であり、`<プロジェクト名>_<サービス名>_<レプリカ番号>` のパターンに従っています。そのため、私たちのアプリのコンテナとmysqlデータベースのコンテナをすばやく見分けることが非常に簡単です。

![Docker Dashboard with app project expanded](dashboard-app-project-expanded.png)


## スタックのシャットダウン

すべてを解体する準備ができたら、単に `docker compose down` を実行するか、Docker Dashboardでアプリ全体のごみ箱をクリックしてください。コンテナーが停止し、ネットワークが削除されます。

!!! warning "ボリュームの削除"
    デフォルトでは、Composeファイル内の名前付きボリュームは、`docker compose down` を実行したときには削除されません。ボリュームを削除したい場合は、`--volumes`フラグを追加する必要があります。

    全体のアプリを削除すると、Docker Dashboardはボリュームを削除しません。

一度取り壊した後、他のプロジェクトに切り替えて、`docker compose up`を実行し、そのプロジェクトに貢献する準備ができます！それ以上にシンプルなことはありません！


## まとめ

このセクションでは、Docker Composeについて学び、複数のサービスアプリケーションの定義と共有を大幅に簡素化する方法を学びました。コマンドを適切なCompose形式に変換して、Composeファイルを作成しました。

この時点で、チュートリアルのまとめに入っています。しかし、私たちが使用しているDockerfileに大きな問題があるため、イメージのビルドに関するいくつかのベストプラクティスをカバーしたいと思います。それでは、見てみましょう！