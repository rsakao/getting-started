イメージをビルドしたので、共有しましょう！ Dockerイメージを共有するには、Dockerレジストリを使用する必要があります。デフォルトのレジストリはDocker Hubで、使用しているすべてのイメージはそこから取得されています。

## リポジトリの作成

イメージをプッシュするために、まずDocker Hub上にリポジトリを作成する必要があります。

1. [Docker Hub](https://hub.docker.com) にアクセスし、必要に応じてログインしてください。

2. **Create Repository** ボタンをクリックします。

3. リポジトリ名に `getting-started` を使用してください。Visibilityは `Public` に設定してください。

4. **Create** ボタンをクリックしてください！

ページの右側にある **Docker commands** というセクションを見ると、このリポジトリにプッシュするために実行する必要がある例示コマンドが表示されます。

![Docker command with push example](push-command.png){: style=width:75% }
{: .text-center }

## イメージのプッシュ

1. コマンドラインで、Docker Hubで表示されるプッシュコマンドを実行してみてください。 `docker` ではなく、ご自身の名前空間を使用することに注意してください。

    ```plaintext
    $ docker push docker/getting-started
    The push refers to repository [docker.io/docker/getting-started]
    An image does not exist locally with the tag: docker/getting-started
    ```

    なぜ失敗したのでしょうか？プッシュコマンドは `docker/getting-started` という名前のイメージを探していましたが、それを見つけることができませんでした。`docker image ls` を実行しても、イメージは見つかりません。

    これを修正するには、構築済みの既存のイメージに別の名前を付けるために "tag" を付ける必要があります。

2. Docker Desktopで "Sign In" ボタンをクリックするか、 `docker login -u YOUR-USER-NAME` コマンドを使用してDocker Hubにログインしてください。

3. `getting-started` イメージに新しい名前を付けるために `docker tag` コマンドを使用してください。Docker ID で `YOUR-USER-NAME` を置き換えてください。

    ```bash
    docker tag getting-started YOUR-USER-NAME/getting-started
    ```

4. 今度はプッシュコマンドを再度試してください。Docker Hubから値をコピーしている場合、イメージ名にタグ名を追加しなかったため、それを削除できます。タグを指定しない場合、Dockerは `latest` というタグを使用します。

    ```bash
    docker push YOUR-USER-NAME/getting-started
    ```

## 新しいインスタンスでのイメージの実行

イメージがビルドされてレジストリにプッシュされたので、新しいインスタンスでアプリを実行してみましょう！ これには、Play with Dockerを使用します。

1. ブラウザを[Play with Docker](https://labs.play-with-docker.com/)に開きます。

2. Docker Hubアカウントでログインしてください。

3. ログインしたら、左側のサイドバーにある "+ ADD NEW INSTANCE" リンクをクリックしてください。（表示されない場合は、ブラウザを少し広げてください）数秒後、ブラウザでターミナルウィンドウが開きます。

    ![Play with Docker add new instance](pwd-add-new-instance.png){: style=width:75% }
{: .text-center }

4. ターミナルで、プッシュしたばかりのアプリを起動してください。

    ```bash
    docker run -dp 3000:3000 YOUR-USER-NAME/getting-started
    ```

    イメージがダウンロードされ、最終的に起動されるのを確認できるはずです！

5. badgeの3000をクリックすると、変更されたアプリが表示されます。素晴らしい！

## まとめ

このセクションでは、イメージをレジストリにプッシュして共有する方法を学びました。次に、全く新しいインスタンスでプッシュされたばかりのイメージを実行する方法を紹介しました。これはCIパイプラインで非常に一般的で、パイプラインがイメージを作成してレジストリにプッシュし、本番環境が最新バージョンのイメージを使用できるようにする方法です。

これを解決するために、前のセクションの最後に気づいたことに戻りましょう。リマインダーとして、アプリを再起動すると、すべてのtodoリストアイテムが失われてしまいます。ユーザーエクスペリエンスとしては明らかに良くありませんので、データを再開始時に保持する方法を学びましょう！