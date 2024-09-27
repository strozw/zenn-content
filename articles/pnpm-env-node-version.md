---
title: "pnpm の Node.js 管理可能について"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [pnpm, node]
published: true
publication_name: yumemi_inc
---

## 概要

[pnpm 9.6.0](https://github.com/pnpm/pnpm/releases/tag/v9.6.0) からパッケージ内での npm-scripts の実行に利用する Node.js のバージョン指定が出来るようになっており、以下について調べたので紹介したいと思います。

- `package.json` で指定可能なになった `pnpm.executionEnv.nodeVersion` について
- `pnpm env` を利用した pnpm 内で利用する Node.js の管理方法について

##  `pnpm.executionEnv.nodeVersion` について

名前のとおり、実行環境における Node.js のバージョンを以下のように `package.json` で指定できます。

```json
{
  "pnpm": {
    "executionEnv": {
      "nodeVersion": "22.0.0"
    }
  } 
}
```

### 指定可能な値

指定可能な値は、以下のように完全なバージョンを指定する必要があります。

- ⭕ OK
  - 完全なバージョン
    - `20.17.0`
- ❌ NG
  - 完全じゃない
    - `20`
  - バージョン表記にない
    - `lts`

:::message
指定する値については、後述する `pnpm env list --remote` を利用して調べると良さそうです。
:::


### 実行に利用されるタイミング

以下の様になっています。

|コマンド|内容|
|---|---|
|`pnpm run <script>`| `npm-scripts` (`packages.json` の `scripts`) の実行時 |
|`pnpm node <command>`| `pnpm.executionEnv.nodeVersion` に指定されている Node.js を利用する |
|`pnpm exec <bin>`| プロジェクトスコープ内でのシェル実行時。（依存するパッケージによって利用可能になったコマンドの実行時など） |

実行時に当該バージョンの Node.js ランタイムが pnpm によってインストールされていれば、それを用いて実行し、インストールされていなければ、インストールした後、これを用いて実行する様になっています。

:::message alert
`pnpm` を介さない `node` は システムのもの（`$PATH` のもの）が利用されるため、`pnpm.executionEnv.nodeVersion` で指定したバージョンの Node.js を利用する場合は、プロジェクト内での Node.js の実行には、必ず上記コマンドを経由する必要があります。

また、`pnpx` についてもシステムのものが利用されるため、ここも注意が必要です。
:::

### Node.js のインストールディレクトリ

pnpm によってインストールされる Node.js は、以下の様にバージョン別で環境変数 `PNPM_HOME` の配下にインストールされます。

```txt
$PNPM_HOME/
  nodejs/
    22.0.0/
      bin/
      include/
      lib/
      share/
      ...
    20.0.0/
      ...
```

## `pnpm env` を利用した Node.js ランタイムの管理

インストールされた、Node.js ランタイムの管理は `pnpm env` コマンドによって行え、以下の事が可能になっています。

- グローバルに利用する Node.js を指定
    - `pnpm env use <nodejs_version>`
- 指定したバージョンの Node.js をインストール
    - `pnpm env add <nodejs_version>`
- 指定したバージョンの Node.js をアンインストール
    - `pnpm env remove <nodejs_version>`
- インストールした Node.js の一覧を出力
    - `pnpm env list`
- インストール可能な Node.js の一覧を出力
    - `pnpm env list --remote <nodejs_version>`

ここで指定可能な `<nodejs_version>` は、`package.json` の `pnpm.executionEnv.nodeVersion` とは異なり、以下のように曖昧なものが許可されています。

|分類|値|
|---|---|
| メジャーバージョンのみ| `20` |
|キャレット表記 `^` | `^20.10`|
| LTS バージョン| `lts`|
| コードネーム| `argon` (v4 のコードネーム) |
| RC バージョン| `rc` (最新のRCバージョン)|
| | `rc/18` (v18 の RC バージョン)|

### グローバルに利用する Node.js を指定

```sh
pnpm env use --global <nodejs_version>`
```

`$PNPM_HOME/nodejs` 配下に指定したバージョンのランタイムのインストールを行い、そのランタイムの `bin/node` を `$PNPM_HOME/node` としてシムリンクを作成します。

```sh
pnpm env use --global 20
pnpm env use --global lts
pnpm env use --global argon
pnpm env use --global latest
pnpm env use --global rc/20
```

:::message
`pnpm env use` によって指定した `bin/node` を利用するには `$PNPM_HOME` が `$PATH` に登録されている必要があります。
:::

### 指定したバージョンの Node.js をインストール

```sh
pnpm env add --global <nodejs_version>`
```

`$PNPM_HOME/nodejs` 配下に指定したバージョンのランタイムのインストールのみを行います。

```sh
pnpm env add --global 20
pnpm env add --global lts
pnpm env add --global argon
pnpm env add --global latest
pnpm env add --global rc/20
```

### 指定したバージョンの Node.js をアンインストール

```sh
pnpm env remove --global <nodejs_version>`
```

`$PNPM_HOME/nodejs` 配下にインストールされた Node.js ランタイムをアンインストールします。
※  `remove` は `rm` に省略可能となっています。

```sh
pnpm env remove --global 20
pnpm env remove --global lts
pnpm env remove --global argon
pnpm env remove --global latest
pnpm env remove --global rc/20
```

### インストールした Node.js の一覧を出力

```sh
pnpm env list`
```

pnpm によってインストールされたの Node.js の一覧を出力します。
※  `list` は `ls` に省略可能となっています。

`$PNPM_HOME/nodejs` 配下にインストールされた Node.js ランタイムをアンインストールします。

### インストール可能な Node.js の一覧を出力

```sh
pnpm env list --remote <nodejs_version>`
```

インストール可能な Node.js ランタイムバージョンを出力します。

```sh
pnpm env list --remote 20
pnpm env list --remote lts
pnpm env list --remote argon
pnpm env list --remote latest
pnpm env list --remote rc/20
```

## 感想

[pnpm 9.8 で `packageManager` にも対応した](https://pnpm.io/ja/package_json#pnpmexecutionenvnodeversion)ので、pnpm が入っていれば、pnpm と Node.js の両方のバージョンを管理できるため、以下のようなケースで有用なのではないかと思いました。

- パッケージマネージャーを pnpm に限定しているプロジェクトや、業態
- Docker Image の作成
- CI のステップ簡略化
  - `$PNPM_HOME` 以下をキャッシュすると良さそう


## 参照

- https://pnpm.io/ja/package_json#pnpmexecutionenvnodeversion
- https://pnpm.io/ja/cli/env


