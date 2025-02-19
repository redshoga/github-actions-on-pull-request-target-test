# github-actions-on-pull-request-target-test

GitHub Actions上での `pull_request_target` と `pull_request` の挙動を調べるリポジトリ。

## 背景

dependabotやOSSコントリビューターは `pull_request_target` でないとシークレットを扱ったCIを実行できない。
かといって `pull_request` と `pull_request_target` を2つ設定すると内部のコントリビューターがPRを作成したときに2度ワークフローが実行されてしまったりする。([例](https://github.com/redshoga/github-actions-on-pull-request-target-test/pull/12))

`pull_request`と`pull_request_target`の違いは[この記事](https://madogiwa0124.hatenablog.com/entry/2022/05/29/110148)がわかりやすい。

## ２度実行させないように

[この記事](https://engineering.mobalab.net/2021/04/30/make-secrets-available-in-pull-request-opened-by-dependabot/)の通り。
if文で以下のどちらかのパターンのみ許可する。("pull_requestで発火 && ユーザーはdependabot"のような例を無視する)
- pull_request_targetで発火 && ユーザーはdependabot
- pull_requestで発火 && ユーザーはdependabot以外

記事上だとメインステップに条件を付与していた。ステップが多い場合は以下のようにjobを区切ってしまってneedsで連結させると良いかもしれない。

注意点としては steps を空にすると望みの動作をしない。([このPR](https://github.com/redshoga/github-actions-on-pull-request-target-test/pull/4)で実験)

```yml
jobs:
  check_user:
    name: Check User
    runs-on: ubuntu-22.04
    # dependabotならpull_request_target以外を無視
    # dependabot以外ならpull_request_targetを無視
    if: |
      (github.event_name == 'pull_request_target' && github.actor == 'dependabot[bot]') ||
      (github.event_name != 'pull_request_target' && github.actor != 'dependabot[bot]')
    steps:
      - run: echo "completed Check User"
  example:
    needs: check_user
    name: Example
```

## チェックアウトの設定

`pull_request_target`トリガー時は明示的にrefを指定しないと変更内容に対する CI を実行できない。

`pull_request`トリガー時は明示的に記載してもしなくても挙動は変わらなさそうなので、`pull_request_target`時を考慮し、すべてrefを指定する。

```yml
- name: Checkout
  uses: actions/checkout@v4
  # pull_request_target 駆動の場合は HEAD コミットを明示的に指定する必要があるため
  with:
    ref: ${{ github.event.pull_request.head.sha }}
```

[この記事](https://engineering.mobalab.net/2021/04/30/make-secrets-available-in-pull-request-opened-by-dependabot/)ではCheckoutのステップを2つ書いている(ifで分岐して1つだけ実行するようにしている)。上記の例のほうがスッキリするが、細かい挙動設定する場合はステップをわけてもよさそう。

## その他

- 外部ユーザーでもdependabotとそうで無い場合で微妙に違いがある。
  - dependabotはpull_requestでも動作する。(外部コントリビュータは実行自体にapproveが必要になる)
    - 動作するが、シークレットは例によって参照できない。
- `pull_request_target`で実行している + チェックアウト先を明示的にしても、外部ユーザーはワークフローファイル自体の変更はPR上で確認できない。(例: ワークフローファイルに`- run: echo "hello world"`を追記してPRを出しても、その変更がある前の状態のワークフローとして実行される)
  - ワークフローファイル以外は変更後のものが用いられる。
    - このリポジトリでいう`sample.txt`の変更はワークフロー上で反映される。
    - このリポジトリでいう`.github/workflows/on_pull_request.yml`の変更はワークフロー上で反映されない。

## 参考

- [Events that trigger workflows - GitHub Docs](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#pull_request_target)
- [[GitHub Actions] Secrets や書き込み権限が必要な Workflow を Dependabot からも使えるようにする – もばらぶエンジニアブログ](https://engineering.mobalab.net/2021/04/30/make-secrets-available-in-pull-request-opened-by-dependabot/)
- [GitHub Actionの`on: pull_request`と`on: pull_request_target`の違いMEMO - Madogiwa Blog](https://madogiwa0124.hatenablog.com/entry/2022/05/29/110148)