## セキュリティスキャン

!!! warning "docker scan コマンドは削除されました"
    イメージの脆弱性やその他多くの機能について引き続き学習するには、新しい docker scout コマンドを使用してください。docker scout --help を実行してください。 https://docs.docker.com/engine/reference/commandline/scout/

エラー: docker scan は削除されました

Dockerイメージを構築したあと、`docker scan`コマンドを使ってセキュリティ脆弱性をスキャンすることが良い習慣です。
Dockerは[Snyk](http://snyk.io)と提携して、脆弱性スキャンサービスを提供しています。

例えば、チュートリアルの前に作成した`getting-started`イメージをスキャンするには、単に以下のように入力するだけです。

```bash
docker scan getting-started
```

このスキャンでは常に更新される脆弱性データベースが使用されます。したがって見る出力は新しい脆弱性が発見されるたびに異なりますが、以下のように表示される場合があります。

```plaintext
✗ Low severity vulnerability found in freetype/freetype
  Description: CVE-2020-15999
  Info: https://snyk.io/vuln/SNYK-ALPINE310-FREETYPE-1019641
  Introduced through: freetype/freetype@2.10.0-r0, gd/libgd@2.2.5-r2
  From: freetype/freetype@2.10.0-r0
  From: gd/libgd@2.2.5-r2 > freetype/freetype@2.10.0-r0
  Fixed in: 2.10.0-r1

✗ Medium severity vulnerability found in libxml2/libxml2
  Description: Out-of-bounds Read
  Info: https://snyk.io/vuln/SNYK-ALPINE310-LIBXML2-674791
  Introduced through: libxml2/libxml2@2.9.9-r3, libxslt/libxslt@1.1.33-r3, nginx-module-xslt/nginx-module-xslt@1.17.9-r1
  From: libxml2/libxml2@2.9.9-r3
  From: libxslt/libxslt@1.1.33-r3 > libxml2/libxml2@2.9.9-r3
  From: nginx-module-xslt/nginx-module-xslt@1.17.9-r1 > libxml2/libxml2@2.9.9-r3
  Fixed in: 2.9.9-r4
```

出力には脆弱性の種類、詳細を調べるためのURL、そして重要なことに脆弱性を修正するための各ライブラリのバージョンが示されています。

`docker scan`には他にもいくつかのオプションがあり、[docker scanドキュメント](https://docs.docker.com/engine/scan/)で詳しく読むことができます。

ビルドした直後にコマンドラインで新しいイメージをスキャンする以外に、[Docker Hubを設定](https://docs.docker.com/docker-hub/vulnerability-scanning/)して、自動的に新しくプッシュされたすべてのイメージをスキャンし、結果をDocker HubおよびDocker Desktopで確認することもできます。

![Hub vulnerability scanning](hvs.png){: style=width:75% }
{: .text-center }

## イメージのレイヤー化

イメージがどのように構成されているかを見ることができるということは、多くの情報を得ることができます。
これには、`docker image history`コマンドを使用して、イメージ内の各レイヤーを作成するために使用されたコマンドを示すことが含まれます。

1. `getting-started`イメージ内のレイヤーを見るために`docker image history`コマンドを使用します。

    ```bash
    docker image history getting-started
    ```

    次のような出力が得られます (日付やIDは異なる場合があります)。

    ```plaintext
    IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
    05bd8640b718   53 minutes ago   CMD ["node" "src/index.js"]                     0B        buildkit.dockerfile.v0
    <missing>      53 minutes ago   RUN /bin/sh -c yarn install --production # b…   83.3MB    buildkit.dockerfile.v0
    <missing>      53 minutes ago   COPY . . # buildkit                             4.59MB    buildkit.dockerfile.v0
    <missing>      55 minutes ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
    <missing>      10 days ago      /bin/sh -c #(nop)  CMD ["node"]                 0B        
    <missing>      10 days ago      /bin/sh -c #(nop)  ENTRYPOINT ["docker-entry…   0B        
    <missing>      10 days ago      /bin/sh -c #(nop) COPY file:4d192565a7220e13…   388B      
    <missing>      10 days ago      /bin/sh -c apk add --no-cache --virtual .bui…   7.85MB    
    <missing>      10 days ago      /bin/sh -c #(nop)  ENV YARN_VERSION=1.22.19     0B        
    <missing>      10 days ago      /bin/sh -c addgroup -g 1000 node     && addu…   152MB     
    <missing>      10 days ago      /bin/sh -c #(nop)  ENV NODE_VERSION=18.12.1     0B        
    <missing>      11 days ago      /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B        
    <missing>      11 days ago      /bin/sh -c #(nop) ADD file:57d621536158358b1…   5.29MB 
    ```

    各行はイメージ内の1つのレイヤーを表しています。ここでの表示は、下部にベースがあり、一番新しいレイヤーが上部にあることを示しています。これにより、各レイヤーのサイズを簡単に確認できるため、大きなイメージの診断に役立ちます。

1. 行が切り捨てられていることに気づくでしょう。`--no-trunc`フラグを追加すると、完全な出力を取得することができます（はい...切り捨てられたフラグを使用して、切り捨てられていない出力を取得するのは面白いですね？）

    ```bash
    docker image history --no-trunc getting-started
    ```
    

## レイヤーキャッシュ

レイヤーが変更されると、下流のすべてのレイヤーも再作成する必要があります。

私たちのDockerfileを再度見てみましょう...

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
```

イメージの履歴出力に戻ると、Dockerfileの各コマンドがイメージ内の新しいレイヤーになることがわかります。
イメージに変更を加えたとき、yarnの依存関係を再インストールする必要があったことを覚えているかもしれません。これを修正する方法はありますか？同じ依存関係をビルドするたびにインストールするのはあまり意味がないですよね？

これを修正するためには、依存関係をキャッシュすることをサポートするためにDockerfileを再構成する必要があります。
Nodeベースのアプリケーションの場合、これらの依存関係は`package.json`ファイルに定義されています。したがって、最初に`package.json`ファイルのみをコピーして依存関係をインストールし、その後で他のすべてをコピーすることから始めましょう。その後に、`package.json`に変更があった場合にのみyarnの依存関係を再作成します。分かりますか？

1. Dockerfileを更新し、最初に`package.json`をコピーして依存関係をインストールし、その後で他のすべてをコピーするようにします。

    ```dockerfile hl_lines="3 4 5"
    FROM node:18-alpine
    WORKDIR /app
    COPY package.json yarn.lock ./
    RUN yarn install --production
    COPY . .
    CMD ["node", "src/index.js"]
    ```

1. Dockerfileと同じフォルダに`.dockerignore`ファイルを作成し、次の内容を記述します。

    ```ignore
    node_modules
    ```

    `.dockerignore`ファイルは、選択的にイメージに関連するファイルのみをコピーする簡単な方法です。
    [ここ](https://docs.docker.com/engine/reference/builder/#dockerignore-file)で詳しく読むことができます。
    この場合、2番目の`COPY`ステップでは`node_modules`フォルダのファイルを省略する必要があります。さもなければ、`RUN`ステップで作成されたファイルが上書きされる可能性があるためです。
    Node.jsアプリケーションにおいてこれがなぜ推奨されるのか、およびそれ以外のベストプラクティスについては、彼らの
    [Dockerizing a Node.js web app](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/)のガイドを見てください。

1. `docker build`を使用して新しいイメージを構築します。

    ```bash
    docker build -t getting-started .
    ```

    次のような出力が表示されます。

    ```plaintext
    [+] Building 16.1s (10/10) FINISHED
    => [internal] load build definition from Dockerfile                                               0.0s
    => => transferring dockerfile: 175B                                                               0.0s
    => [internal] load .dockerignore                                                                  0.0s
    => => transferring context: 2B                                                                    0.0s
    => [internal] load metadata for docker.io/library/node:18-alpine                                  0.0s
    => [internal] load build context                                                                  0.8s
    => => transferring context: 53.37MB                                                               0.8s
    => [1/5] FROM docker.io/library/node:18-alpine                                                    0.0s
    => CACHED [2/5] WORKDIR /app                                                                      0.0s
    => [3/5] COPY package.json yarn.lock ./                                                           0.2s
    => [4/5] RUN yarn install --production                                                           14.0s
    => [5/5] COPY . .                                                                                 0.5s 
    => exporting to image                                                                             0.6s 
    => => exporting layers                                                                            0.6s 
    => => writing image sha256:d6f819013566c54c50124ed94d5e66c452325327217f4f04399b45f94e37d25        0.0s 
    => => naming to docker.io/library/getting-started                                                 0.0s
    ```

    全レイヤーが再構築されたことがわかります。Dockerfileをかなり変更したので完全に正常です。

1. 今度は、`src/static/index.html`ファイルを変更します（タイトルを「The Awesome Todo App」と言うように変更します）。

1. `docker build -t getting-started .`を使って、今度はDockerイメージを構築します。今回は、出力が少し異なるはずです。

    ```plaintext hl_lines="10 11 12"
    [+] Building 1.2s (10/10) FINISHED
    => [internal] load build definition from Dockerfile                                               0.0s
    => => transferring dockerfile: 37B                                                                0.0s
    => [internal] load .dockerignore                                                                  0.0s
    => => transferring context: 2B                                                                    0.0s
    => [internal] load metadata for docker.io/library/node:18-alpine                                  0.0s
    => [internal] load build context                                                                  0.2s
    => => transferring context: 450.43kB                                                              0.2s
    => [1/5] FROM docker.io/library/node:18-alpine                                                    0.0s
    => CACHED [2/5] WORKDIR /app                                                                      0.0s
    => CACHED [3/5] COPY package.json yarn.lock ./                                                    0.0s
    => CACHED [4/5] RUN yarn install --production                                                     0.0s
    => [5/5] COPY . .                                                                                 0.5s
    => exporting to image                                                                             0.3s
    => => exporting layers                                                                            0.3s
    => => writing image sha256:91790c87bcb096a83c2bd4eb512bc8b134c757cda0bdee4038187f98148e2eda       0.0s
    => => naming to docker.io/library/getting-started                                                 0.0s
    ```

    まず、ビルドがとても高速であることに気づくはずです！いくつかのステップが以前にキャッシュされたレイヤーを使用していることがわかります。万歳！

## マルチステージビルド

このチュートリアルでは深く掘り下げることはありませんが、マルチステージビルドは複数のステージを使用してイメージを作成する強力なツールであり、次のような多数のメリットがあります。

- ビルド時の依存関係とランタイム時の依存関係を分離する
- アプリが実行するために必要なものだけを提供することにより、全体的なイメージサイズを縮小する

### Maven/Tomcatの例

Javaベースのアプリケーションを構築する場合、JavaソースコードをJavaバイトコードにコンパイルするためにJDKが必要です。ただし、本番環境ではJDKは必要ありません。また、MavenやGradleといったツールを使用してアプリを構築する場合もありますが、これらも最終的なイメージでは必要ありません。マルチステージビルドを使用すると助けられます。

```dockerfile
FROM maven AS build
WORKDIR /app
COPY . .
RUN mvn package

FROM tomcat
COPY --from=build /app/target/file.war /usr/local/tomcat/webapps 
```

この例では、Javaビルドを実際に実行するための1つのステージ（`build`と呼ばれる）を使用しています。2番目のステージ（`FROM tomcat`以降）では、`build`ステージからファイルをコピーしています。最終的なイメージは、`--target`フラグを使用してオーバーライドできる最後のステージのみが作成されます。


### Reactの例

Reactアプリケーションを構築する場合、JSコード（通常はJSX）、SASSスタイルシートなどを静的なHTML、JS、およびCSSにコンパイルするにはNode環境が必要です。ただし、サーバで実行する場合は、Node環境が必要ありません。また、Reactアプリでは通常、コンパイルされた静的資産だけが必要です。

以下は、マルチステージビルドによるReactアプリの例です。

```dockerfile
FROM node:14-alpine as build
WORKDIR /app
COPY package.json .
RUN yarn install --production
COPY . .
RUN yarn build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```

この例では、最初のステージ（`build`と呼ばれる）は、Node.jsを使用してReactアプリをビルドします。2番目のステージでは、Nginxを使用してサーバーを提供し、最初のステージからビルドされた静的資産を提供します。依存関係は分離されます。

## おわりに

[Docker documentation](https://docs.docker.com/) には、Dockerの基本的な概念、構文、およびコマンドについて詳しく説明されています。これらのリソースは、Dockerによるコンテナ化の学習に役立ちます。