---
title: "Label 駆動 CI by Github Action"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

以前、Github の PR への Label 付与のタイミングで CI を実行する事で、業務での余計な CI の実行を減らしコストを抑える事ができたので、そのときの workflow の内容と仕組みをまとめました。

## 構成

```txt
.github/
  workflows/
    pr-labeled-handler.yml
    test.yml
```

## 付与された Label に応じた Workflow を実行するための Workflow

[benc-uk/workflow-dispatch](https://github.com/benc-uk/workflow-dispatch) を利用し、付与された Label に応じた workflow を workflow_dispatch で呼び出す workflow を用意します。

```yaml
name: PR Labeled Handler

on:
  pull_request:
    types:
      - labeled

permissions:
  actions: write
  statuses: write

jobs:
  dispatch_test:
    runs-on: ubuntu-latest
    if: github.event.label.name == 'action:test'
    steps:
      - uses: benc-uk/workflow-dispatch@v1
        with:
          workflow: test.yml
          ref: ${{ github.event.pull_request.head.ref }}
```

`jobs` には、同じ形式のものの `if` で比較するラベル名と、`workflow:` に指定する Workflow File を変更することで、上記の Workflow 内で複数の Workflow の実行を管理できます。

## Label 付与時に実行する Workflow

`on: workflow_dispatch:` では、実行結果を PR の Commit Status に付与しないため、[Status Check によるブランチ保護](https://docs.github.com/ja/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/about-status-checks)を利用したい場合は、Workflow 内で個別に更新する必要があります。

そのため、以下の例では [myrotvorets/set-commit-status-action]() を利用し、以下のタイミングで `Test` という Context で Commit Status を更新しています。

- 実行開始時
  - `pending` (実行中) へ更新
- 実行後
  - `job.status` が `success` or `failure` ならその値を反映
  - `job.status` が上記以外なら、`error`

```yaml
name: PR Test

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
      - name: checkout
        uses: actions/checkout
        with:
          ref: ${{ github.sha }}

      - name: setup
        uses: actions/setup-node

      - name: Commit Status を `pending` (実行中)へ
        uses: myrotvorets/set-commit-status-action@v2
        with:
          status: pending
          context: Test
          sha: ${{ github.sha }}

      # ここがメイン
      - name: Run Test
        run: npm run test

      - name: Commit Status に現在の `job.status` を反映する
        uses: myrotvorets/set-commit-status-action@v2
        if: always()
        with:
          status: ${{ contains(["success", "failure"], job.status) && job.status || "error" }}
          context: Test
          sha: ${{ github.sha }}
```

## まとめ

(Status Check によるブランチ保護を利用する場合は、`Test` という Commit Status を見るようにすれば良いという事になります。)
