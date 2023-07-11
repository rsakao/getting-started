空のToDoリストがない場合に表示される "empty text" を変更するように製品チームから機能追加の要望がありました。以下のように変更することを希望しています。

> You have no todo items yet! Add one above!

それでは、この変更を行ってみましょう。

## ソースコードの更新

1. `src/static/js/app.js` ファイル内の 56 行目を、新しい空のテキストを使用するように更新してください。

    ```diff
    -                <p className="text-center">No items yet! Add one above!</p>
    +                <p className="text-center">You have no todo items yet! Add one above!</p>
    ```

1. 前回と同じコマンドを使用して、更新したイメージをビルドしてみましょう。

    ```bash
    docker build -t getting-started .
    ```

1. 更新されたコードを使用して新しいコンテナを起動しましょう。

    ```bash
    docker run -dp 3000:3000 getting-started
    ```

**あれ？** おそらくこのようなエラーが表示されたはずです（ID は異なります）：

```bash
docker: Error response from daemon: driver failed programming external connectivity on endpoint laughing_burnell 
(bb242b2ca4d67eba76e79474fb36bb5125708ebdabd7f45c8eaf16caaabde9dd): Bind for 0.0.0.0:3000 failed: port is already allocated.
```

どうしてこれが起こったのでしょうか？古いコンテナがまだ実行されているため、新しいコンテナを開始できません。
この問題が発生する理由は、そのコンテナがホストのポート3000を使用しているためで、マシン上の1つのプロセス（コンテナを含む）しか特定のポートを聞くことができません。この問題を解決するには、古いコンテナを削除する必要があります。

## 古いコンテナの置換

コンテナを削除するには、まず停止する必要があります。停止したら、コンテナを削除できます。古いコンテナを削除する方法は２つあります。自分が最も使い慣れている方法を選択してください。

### CLIを使用してコンテナを削除する

1. `docker ps` コマンドを使用してコンテナの ID を取得します。

    ```bash
    docker ps
    ```

1. `docker stop` コマンドを使用してコンテナを停止します。

    ```bash
    # <the-container-id> の部分を docker ps に表示された ID で置き換えてください。
    docker stop <the-container-id>
    ```

1. 一度コンテナが停止したら、`docker rm` コマンドを使用して削除できます。

    ```bash
    docker rm <the-container-id>
    ```

!!! info "プロのヒント"
    `-f`フラグを `docker rm` コマンドに加えることで、コンテナを１つのコマンドで停止 & 削除できます。例： `docker rm -f <the-container-id>`

### Dockerダッシュボードを使用してコンテナを削除する

Dockerダッシュボードを開けば、２クリックでコンテナを削除できます！コンテナのIDを調べて削除するよりも簡単です。

1. ダッシュボードを開いた状態で、アプリのコンテナにカーソルを合わせると、右側にアクションボタンの集合が表示されます。

1. ゴミ箱のアイコンをクリックしてコンテナを削除します。

1. 削除を確認したら、完了です！

![Dockerダッシュボード - コンテナの削除](dashboard-removing-container.png)


### 更新されたアプリコンテナの開始

1. 更新されたアプリを開始します。

    ```bash
    docker run -dp 3000:3000 getting-started
    ```

1. [http://localhost:3000](http://localhost:3000) でブラウザを更新して、更新されたヘルプテキストが表示されることを確認してください。

![Updated application with updated empty text](todo-list-updated-empty-text.png){: style="width:55%" }
{: .text-center }


## まとめ

アップデートができたものの、２つ問題点がありました：

- ToDoリストにあるすべての項目が消えてなくなってしまいました！あまり良いアプリではありません。 これについてはすぐに説明します。
- この小さな変更に対して多くの手順が必要でした。次のセクションでは、変更ごとにイメージを再ビルドして新しいコンテナを起動する必要がない方法を見ていきます。

永続性について話す前に、これらのイメージを他の人と共有する方法についてすぐに見てみましょう。