---
title: "Storybook 8.3 で導入された Vitest 対応を React と Next.js で試す"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [storybook, vitest, react, nextjs]
published: true
publication_name: yumemi_inc
---

## Storybook 8.3 のリリースつについて

先日 Storybook 8.3 がリリースされました。
このリリースでの目玉機能は、なんといっても、待望の Vitest 対応ではないでしょうか。

以下は、7月末に一部公開されていたスクリーンキャスト。

https://x.com/yannbf/status/1816150365286314333

とはいえ、何故か大々的に告知されていなかったり、Changelog には以下のようにあるのですが、

> ⚡️ First-class Vitest integration to run stories as component tests
> 🔼 Next.js-Vite framework for Vitest compatibility and better DX
> 🗜️ Further reduced bundle size for a smaller install footprint
> 🌐 Experimental Story globals to standardize stories for themes, viewports, and locales
> 💯 Hundreds more improvements

上記の内、Vitest に関する以下2つについては、実はまだ Experimental だったりと...。

> ⚡️ First-class Vitest integration to run stories as component tests
> 🔼 Next.js-Vite framework for Vitest compatibility and better DX

不思議な？（雑な？）リリースがされております。

とはいえ、Storybook の Story が手軽に Vitest で実行できることになったのは大変喜ばしいことなので、この記事ではドキュメントを参照しつつ、React と Next.js におけるセットアップから、テストの実行までを紹介したいと思います。

### 追記: 2024/09/25

後日 (2024/09/24) に、公式の X と Blog にて以下の告知と、記事が公開されました。

https://x.com/storybookjs/status/1838628784720826411

https://storybook.js.org/blog/storybook-8-3/

### 追記: 2024/10/21

現状、`@vitest/browser` v2.1.2 以降と組み合わせて利用すると、`jsdom` や `happy-dom` を利用した Non-Browser Mode で実行すると、以下の issue に記載のエラーが発生し、テストの実行が失敗します。

https://github.com/storybookjs/storybook/issues/29404

そのため、`jsdom` や `happy-dom` でテストを実行したい場合は、 利用するプロジェクトの `package.json` の `dependencies` で以下を指定する必要があります。

```
"@vitest/browser": "2.1.1",
"vitest": "2.1.1"
```

## Storybook Vitest Plugin について

Vitest で Storybook の Story を実行するための Plugin になります。
パッケージとしては `@storybook/experimental-addon-test` と、addon として配布されています。

- [ドキュメント](https://storybook.js.org/docs/writing-tests/vitest-plugin)
- [コード](https://github.com/storybookjs/storybook/tree/next/code/addons/test)

## React で試す

以降で紹介するセットアップを適用し、作成したコードは以下になります。
https://github.com/strozw/demo_storybook-react-vitest-auto

以下、順序を追ってその内容を見ていきたいと思います。

### 検証用プロジェクトの用意

以下のように、適当なディレクトリを作成し、`storybook@latest init` コマンドを実行し、
※ 以降、`pnpm` をご利用の場合は `npx` を `pnpx` に、`npm` を `pnpm` に置き換えてお試しください。

```
mkdir demo_storybook-react-vitest
cd demo_storybook-react-vitest
npx storybook@latest init
```

`storybook init` を空のディレクトリで実行すると、雛形に基づいたコードを生成してくれるので、聞かれた質問には以下の様に回答してください。

- `Choose a project template` -> ` React + Vite (TS)`

これで、vite で bundle を行う React App の Storybook を用いたプロジェクトの雛形が完成します。

### Storybook Vitest Plugin のセットアップ

#### addon 経由のセットアップ

以下のコマンドを実行する事で、`@storybook/experimental-addon-test` 経由で、必要な package のインストールと、必要なファイルのセットアップを自動で行う事ができます。

```
npx storybook add @storybook/experimental-addon-test
```

#### セットアップコマンドで作成されたファイルの内容

上記のセットアップが環境すると、以下のファイルの変更と追加が行われます。

##### 変更されたファイル

###### `package.json` に以下のモジュールが追加される

- `vite`
- `@vitest/browser`
- `@storybook/experimental-addon-test`

内容から、Storybook Vite Plugin は Vite の Browser Mode に対応しており、利用することを推奨していることが分かります。
また、ドキュメントにもその様に記載があります。

##### 追加されたファイル

###### `vitest.workspace.ts`

このファイルは、 [Vitest の workspace 機能](https://vitest.dev/guide/workspace)を利用するたもえのもので、1つのプロジェクト内で複数の vitest の設定を定義し、使い分けることを可能にするものです。

```ts: vitest.workspace.ts
import { defineWorkspace } from 'vitest/config';
import { storybookTest } from '@storybook/experimental-addon-test/vitest-plugin';

// More info at: https://storybook.js.org/docs/writing-tests/vitest-plugin
export default defineWorkspace([
  'vite.config.ts',
  {
    extends: 'vite.config.ts',
    plugins: [
      // See options at: https://storybook.js.org/docs/writing-tests/vitest-plugin#storybooktest
      storybookTest(),
    ],
    test: {
      name: 'storybook',
      browser: {
        enabled: true,
        headless: true,
        name: 'chromium',
        provider: 'playwright',
      },
      // Make sure to adjust this pattern to match your stories files.
      include: ['**/*.stories.?(m)[jt]s?(x)'],
      setupFiles: ['./.storybook/vitest.setup.ts'],
    },
  },
]);
```

`defineWorkspace` には、配列で「Vitest の config オブジェクト」あるいは、「Vitest の config のパス」を指定でき、ここでは、プロジェクトルートの `vite.config.ts` を Story 以外の Unit テストの config として見立て、それとは別に `test.name` で `storybook` と命名した 「`*.stories.*` ファイルのみをテストする」 config を定義しているようです。これにより、 `npm exec vitest --project=storybook` で、`stories` のみを対象とした Vitest を実行できるようになります。

また、config の内容からも Browser Mode で実行するように設定されていることが分かります。

###### `.storybook/vitest.setup.ts`

vitest 実行時 Storybook の `preview.ts` と同じ環境をつくるための設定

```ts: .storybook/vitest.test.ts
import { beforeAll } from "vitest";
import { setProjectAnnotations } from "@storybook/react";
import * as projectAnnotations from "./preview";

// This is an important step to apply the right configuration when testing your stories.
// More info at: https://storybook.js.org/docs/api/portable-stories/portable-stories-vitest#setprojectannotations
const project = setProjectAnnotations([projectAnnotations]);

beforeAll(project.beforeAll);
```

### Vitest の実行を検証

#### Browser Mode で headeless

セットアップ完了後、以下のコマンドで vitest を実行すると、 Storybook の Story が Vitest の Browser Mode で実行されます。

```sh
npm exec vitest
```

:::message
明示的に、Stoorybook の Story のテストのみを実行したい場合は、`test.name` に指定された `storybook` を `--project` に指定して実行します。

```sh
npm exec vitest --project=storybook
```

:::

結果は、以下のように Vitest の watch mode での実行結果が表示され、各種 Vitest のキー操作ができます。

![Test が全て実行されているスクリーンショット](/images/storybook-8-3-vitest/2024-09-14-react-storybook-vitest-success.png)

#### Browser Mode を headless にせず実行する

headless を無効化すると、`borwser.name` で指定しているブラウザで `http://localhost:5173/#/` を開いた状態で起動し、以下のような画面が表示され、ブラウザ上で実行結果を確認できます。

![Vitest の Browser Mode で起動するGUIのスクリーンショット](/images/storybook-8-3-vitest/2024-09-14-vitest-browser-mode-gui.png)

#### `vitest.workspace.ts` の修正

Vitest の Browser Mode を `headless: false` に修正する。

:::details 修正後の `vitest.workspace.ts`

```ts: vitest.workspace.ts
import { storybookTest } from "@storybook/experimental-addon-test/vitest-plugin";
import { defineWorkspace } from "vitest/config";

// More info at: https://storybook.js.org/docs/writing-tests/vitest-plugin
export default defineWorkspace([
  "vite.config.ts",
  {
    extends: "vite.config.ts",
    plugins: [
      // See options at: https://storybook.js.org/docs/writing-tests/vitest-plugin#storybooktest
      storybookTest(),
    ],
    test: {
      name: "storybook",
      browser: {
        enabled: true,
        headless: false,
        name: "chromium",
        provider: "playwright",
      },
      // Make sure to adjust this pattern to match your stories files.
      include: ["**/*.stories.?(m)[jt]s?(x)"],
      setupFiles: ["./.storybook/vitest.setup.ts"],
    },
  },
]);
```

:::

#### Brower Mode を利用せず実行する

Browser Mode を使った方法については、先の結果でわかったと思います。
しかし、以下の理由で `.stories.*` ファイルをそのまま、vitest で `jsdom` や `happy-dom` を使って簡単にテストの実行を行いたいという考えの方も多いはず。

弊社でも以下の記事の手法を用いてコンポーネントテストを実行するケースがいくつかあります。
https://zenn.dev/yumemi_inc/articles/run-all-stories-as-test-with-vitest-jsdom

というわけで、ここでは `vitest.workspace.ts` の設定を変更し、Browser Mode の実行を無効化した上で `happy-dom` を利用した方法を試してみます。

##### 必要なパッケージの追加

以下のコマンドで利用する `happy-dom` を追加します。

```sh
npm i -D happy-dom
```

##### `vitest.workspace.ts` の変更

`test.browser.enabled` に `false` を指定あるいは、`test.browser` を削除し、 `test.environment` に `happy-dom` を指定します。

:::details 修正後の `vitest.workspace.ts`

```ts: vitest.workspace.ts
import { defineWorkspace } from 'vitest/config';
import { storybookTest } from '@storybook/experimental-addon-test/vitest-plugin';

// More info at: https://storybook.js.org/docs/writing-tests/vitest-plugin
export default defineWorkspace([
  'vite.config.ts',
  {
    extends: 'vite.config.ts',
    plugins: [
      // See options at: https://storybook.js.org/docs/writing-tests/vitest-plugin#storybooktest
      storybookTest(),
    ],
    test: {
      name: 'storybook',
      browser: {
        enabled: false,
        headless: true,
        name: "chromium",
        provider: "playwright",
      }
      environment: 'happy-dom',
      include: ['**/*.stories.?(m)[jt]s?(x)'],
      setupFiles: ['./.storybook/vitest.setup.ts'],
    },
  },
]);
```

:::

##### テストの実行

```sh
npm exec vitest
```

結果

![happy-dom を利用した vitest 実行結果のスクリーンショット](/images/storybook-8-3-vitest/2024-09-13-react-storybook-vitest-happy-dom-success.png)
Browser Mode 利用時に表示されていた `Browser runner started by playwright at ...` が表示されていない事がわかります。

:::message
※ 実行速度があまり変わらないものになっていますが、これは筆者の環境が M2 MacBookPro なためかと（コア数が多いためかと）思われます。
:::

## Next.js で試す

今Next.js で試すNext.js で試す回、新たに [@storybook/experimental-nextjs-vite](https://github.com/storybookjs/storybook/tree/next/code/frameworks/experimental-nextjs-vite)（と、内部で利用されている [vite-plugin-storybook-nextjs](https://github.com/storybookjs/vite-plugin-storybook-nextjs) ）がリリースされており、これらを利用することで、Next.js 向けに実装されたコンポーネントの Story についても、Vitest で実行できるようになっています。

:::message alert
とはいえ、Storybook の Next.js RSC & Server Action 対応は、Server 側 (Node.js) で実行するのではなく、Client 側 (Browser) で実行をエミュレートしたもののため、完全に再現されたものではないことをご承知おきください。

そのため、Node.js 依存モジュールを利用している場合などは、適切なモック等を行う必要があります。
:::

以降で紹介するセットアップを適用し、作成したコードは以下になります。
https://github.com/strozw/demo_storybook-next-vitest

### 検証用のプロジェクトを用意

#### Next.js のプロジェクトを作成

```sh
npx create-next-app@latest
```

質問に対しては、以下の様に回答します。

- `What is your project named?` => `demo_storybook-next-vitest`
- `Would you like to use TypeScript?` => `Yes`
- `Would you like to use ESLint?` => `No`
- `Would you like to use Tailwind CSS?` => `Yes`
- `Would you like to use src/ directory?`=>`Yes`
- `Would you like to use App Router? (recommended)` => `Yes`
- `Would you like to customize the default import alias (@/*)?` => `Yes`
- `What import alias would you like configured?` => `@/*`

#### Storybook を追加

以下のコマンドで、storybook を追加し `@storybook/nextjs` を利用した環境をつくります。

```sh
cd demo_storybook-next-vitest
npx storybook@latest init
```

### Storybook Vitest Plugin のセットアップ

以下のコマンドで、先に紹介した `@storybook/experimental-nextjs-vite` と `vite-plugin-storybook-nextjs` を追加した状態を作ります。

```sh
npx storybook@latest add @storybook/experimental-addon-test
```

#### 変更されたファイル

##### `package.json`

以下のパッケージが追加されます。

- `vite`
- `vitest`
- `playwright`
- `@vitest/browser`
- `@storybook/experimental-nextjs-vite`
- `@storybook/experimental-addon-test`

#### 追加されたファイル

##### `vitest.config.ts`

```ts: vitest.config.ts
import { defineConfig } from 'vitest/config';
import { storybookTest } from '@storybook/experimental-addon-test/vitest-plugin';
import { storybookNextJsPlugin } from '@storybook/experimental-nextjs-vite/vite-plugin';

// More info at: https://storybook.js.org/docs/writing-tests/vitest-plugin
export default defineConfig({
  plugins: [
    // See options at: https://storybook.js.org/docs/writing-tests/vitest-plugin#storybooktest
    storybookTest(),
    // More info at: https://github.com/storybookjs/vite-plugin-storybook-nextjs
    storybookNextJsPlugin(),
  ],
  test: {
    name: 'storybook',
    browser: {
      enabled: true,
      headless: true,
      name: 'chromium',
      provider: 'playwright',
    },
    // Make sure to adjust this pattern to match your stories files.
    include: ['**/*.stories.?(m)[jt]s?(x)'],
    setupFiles: ['./.storybook/vitest.setup.ts'],
  },
});
```

##### `.storybook/vitest.setup.ts`

```ts: .storybook/vitest.setup.ts
import { beforeAll } from 'vitest';
import { setProjectAnnotations } from '@storybook/nextjs';
import * as projectAnnotations from './preview';

// This is an important step to apply the right configuration when testing your stories.
// More info at: https://storybook.js.org/docs/api/portable-stories/portable-stories-vitest#setprojectannotations
const project = setProjectAnnotations([projectAnnotations]);

beforeAll(project.beforeAll);
```

### Vitest の実行を検証

```sh
npm exec vitest
```

実行すると、以下のように、 `sytled-jsx/style` の import が解決できないと怒られます。

```sh
[vite] Internal server error: Failed to resolve import "styled-jsx/style" from "src/stories/Button.tsx". Does the file exist?
```

storybook 追加時に生成された Component が `styled-jsx` に依存したものになっているからですね。

##### `styled-jsx` を追加して際実行

まず、`styled-jsx` を追加します

```sh
npm i -S styled-jsx
```

その後、以下のコマンドで再度 Vitest を実行します。

```sh
npm exec vitest
```

結果

![vitest の Browser Mode で Storybook の Story を実行した結果](/images/storybook-8-3-vitest/2024-09-14 17-storybook-nextjs-vitest-success.png)

ちゃんと全て成功している事が分かります。

#### Next.js の API を利用したコンポーネントの Story を検証する

しかし、`storybook init` で追加された Component はどれも、Client Component かつ、`next/headers` 等に依存していないため、次は、Next.js の `cookies` や RSC を利用しているコンポーネントの Story の実行を検証してみます。

手っ取り早く、検証に使えそうな `vite-plugin-storybook-nextjs` の `example` から `next/headers` と `use server` を利用しているコード `app/components/Header` 以下にを持ってきて配置します。

https://github.com/storybookjs/vite-plugin-storybook-nextjs/tree/main/example/src/app/components/Header

上記のコードを配置した後、再度 `npm exec vitest` を実行すると、以下のエラーメッセージが表示され、`Header.stories.tsx` のみ失敗します。

```
async/await is not yet supported in Client Components, only Server Components. This error is often caused by accidentally adding `'use client'` to a module that was originally written for the server.
```

#### Storybook で RSC に関する設定をして実行

上記の問題に対処する方法は、いまのところドキュメントには記載がないため、`vitest-plugin-storybook-nextjs` の `example/.storybook` 内の設定を確認します。

すると、以下の `@storybook/react` から `rsc` を有効にしていると思われる `preview` の import がみつかります。
https://github.com/storybookjs/vite-plugin-storybook-nextjs/blob/main/example/.storybook/storybook.setup.ts#L7

こちらのファイルの内容を見てみると、以下のようになっており、

https://github.com/storybookjs/storybook/blob/next/code/renderers/react/src/entry-preview-rsc.tsx#L1-L5

`.storybook/vitest.setup.ts` の setProjectAnnotations に `{ parameters: { react: { rsc: true } } }` を追加すればよいことがわかります。

:::details 修正後の `.storybook/vitest.setup.ts`

```ts: .storybook/vitest.setup.ts
import { setProjectAnnotations } from "@storybook/nextjs";
import { beforeAll } from "vitest";
import * as projectAnnotations from "./preview";

// This is an important step to apply the right configuration when testing your stories.
// More info at: https://storybook.js.org/docs/api/portable-stories/portable-stories-vitest#setprojectannotations
const project = setProjectAnnotations([
  projectAnnotations,
  {
    parameters: {
      react: {
        rsc: true,
      },
    },
  },
]);

beforeAll(project.beforeAll);
```

:::

修正した上で、再度実行した結果が以下です。

![`.storybook/vitest.setup.ts` を修正した後に実行した結果のスクリーンショット](/images/storybook-8-3-vitest/2024-09-14-storybook-nextjs-rsc-vitest-success.png)

#### Browser Mode を利用しない形で実行

こちらも、Browser Mode を利用せずに実行できるの確認したいと思います。
`vitest-plugin-storybook-nextjs` の README をみると、以下のように、RSC を利用したい場合は、`jsdom` を追加し、設定する必要があると記載があります。

> When testing components that rely on Next.js Server Actions, you need to ensure that your story files are set up to use the jsdom environment in Vitest. This can be done in two ways:When testing components that rely on Next.js Server Actions, you need to ensure that your story files are set up to use the jsdom environment in Vitest. This can be done in two ways:
> https://github.com/storybookjs/vite-plugin-storybook-nextjs#nextjs-server-actions

試しに、`happy-dom` で試したところ、以下のエラーが発生し、Server Action を利用している Story のテストが失敗しました。

> Error: Cannot find package 'jsdom' imported from ...

というわけで、以下のコマンドで `jsdom` を追加し、

```sh
npm i -D jsdom
```

`vitest.config.ts` を修正した上で、

:::details 修正後の `vitest.config.ts`

```ts
import { storybookTest } from "@storybook/experimental-addon-test/vitest-plugin";
import { storybookNextJsPlugin } from "@storybook/experimental-nextjs-vite/vite-plugin";
import { defineConfig } from "vitest/config";

// More info at: https://storybook.js.org/docs/writing-tests/vitest-plugin
export default defineConfig({
  plugins: [
    // See options at: https://storybook.js.org/docs/writing-tests/vitest-plugin#storybooktest
    storybookTest(),
    // More info at: https://github.com/storybookjs/vite-plugin-storybook-nextjs
    storybookNextJsPlugin(),
  ],
  test: {
    name: "storybook",
    browser: {
      // Browser Mode を無効化
      enabled: false,
      headless: true,
      name: "chromium",
      provider: "playwright",
    },
    // jsdom を利用するように指定
    environment: "jsdom",
    // Make sure to adjust this pattern to match your stories files.
    include: ["**/*.stories.?(m)[jt]s?(x)"],
    setupFiles: ["./.storybook/vitest.setup.ts"],
  },
});
```

:::

再度実行してみます。

```sh
npm exec vitest
```

結果は、以下のように playwright を起動せずに実行されます。

![vitest と jsdom を利用して next.js の story を実行した結果](/images/storybook-8-3-vitest/2024-09-14-storybook-nextjs-vitest-jsdom.png)

細かい部分は触れませんが、`next/navigation` などの module についても mock した上でテストを実行できるはずです。

### Storybook での Next.js 関連ファイルのビルドも Vite で行うようにする

ここまでで、作成した内容で`npm run storybook` を実行すると、以下のように webpack で実行されている事がわかります。

```
@storybook/core v8.3.0

info => Starting manager..
info => Starting preview..
info Addon-docs: using MDX3
info => Using implicit CSS loaders
info => Using SWC as compiler
info => Using default Webpack5 setup
```

Next.js のコードを Vitest で実行できるようになったので、Storybook についても同じ様に Vitest 経由で実行するようにしてみましょう。

#### `@storybook/experimental-nextjs-vite` を利用する

Storybook の Next.js 向けのドキュメントをみると、`With Vite` という Vite を利用するためのドキュメントが追加されています。

https://storybook.js.org/docs/get-started/frameworks/nextjs#with-vite

上記にあるように、`.storybook/main.ts` の `framework` を `@storybook/experimental-nextjs-vite` に修正します。

:::details 修正後の `.storybook/main.ts`

```ts: .storybook/main.ts
import type { StorybookConfig } from "@storybook/nextjs";

const config: StorybookConfig = {
  stories: ["../src/**/*.mdx", "../src/**/*.stories.@(js|jsx|mjs|ts|tsx)"],
  addons: [
    "@storybook/addon-onboarding",
    "@storybook/addon-links",
    "@storybook/addon-essentials",
    "@chromatic-com/storybook",
    "@storybook/addon-interactions",
    "@storybook/experimental-addon-test",
  ],
  framework: {
    // vite で build を行うようにする
    name: "@storybook/experimental-nextjs-vite",
    options: {},
  },
};
export default config;
```

:::

これで、`npm run storybook` や `npm run build-storybook` も Vite で実行され、これまでよりも高速に build できます。

## 感想

React については、ドキュメントの [Auto Setup](https://storybook.js.org/docs/writing-tests/vitest-plugin#automatic-setup) を参考にセットアップすれば、わりと素直に実行できることがわかりました。
（※ 今回は試していませんが、おそらく vue 等も同じ様に設定できるのではないでしょうか。）

また、Next.js については、まだドキュメントの整備ができておらず、[vite-plugin-storybook-nextjs の README](https://github.com/storybookjs/vite-plugin-storybook-nextjs) や example を参照しながら設定する必要があるため、注意が必要そうです。（※ vite-plugin-storybook-nextjs の example のコードを確認してみると、8.3 に対応できていなかったりと、こちらも参考にする際、注意が必要です。）

今回は、触れられていませんが、Storybook Vitest Plugin と Next.js の以下の FAQ の内容についても目を通しておく必要がありそうです。

- https://storybook.js.org/docs/writing-tests/vitest-plugin#faq
- https://storybook.js.org/docs/get-started/frameworks/nextjs#faq

以上、導入の参考になれば幸いです。
