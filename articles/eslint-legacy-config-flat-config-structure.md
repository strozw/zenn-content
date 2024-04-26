---
title: "ESLint の Legacy Config と Flat Config における Plugin 構造の違いと両対応 Plugin の構造"
emoji: "🩺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eslint", "npm", "javascript", "typescript"]
published: true
publication_name: yumemi_inc
---

## 概要

ESLint v9 への対応を進めていると、いくつか Flat Config に対応していない Plugin や Config に遭遇することがあります。その際に、Legacy Config のみ対応している Plugin と、Flat Config も対応している Plugin の両方の構造の違いを知っておくと、Flat Config に向けた対応をすすめる上で便利だったため、その違いを俯瞰するための資料としてまとめました。

## 前提

- 以下で例示する Plugin は、パッケージ名を `eslint-plugin-example` として、いくつかのカスタムルールと、カスタム Processor を持つものを想定しています。
- Plugin の実装方法や API の解説はしない。
  - [ESLint v9 では、v8 で廃止予定となった API が削除されているため](https://eslint.org/blog/2023/09/preparing-custom-rules-eslint-v9/#from-context-to-sourcecode)、Plugin を作成したり、ESLint v9 の環境で利用する際には、この点で注意が必要です。ESLint v8 の環境においては問題ないと思われます。

## Legacy Config Plugin の構造

参照したドキュメント
https://eslint.org/docs/v8.x/extend/plugins

### Plugin のコード例

```js
const plugin = {
  meta: {
    name: 'eslint-plugin-example',
    version: '1.0.0'
  },
  configs: {
    recommended: {
      // この plugin を利用する設定。パブリッシュ時のパッケージ名に依存している
      plugins: ['example'], 
      rules: {
        "example/my-rule": "error" 
      }
    }
  },
  rules: {
    "my-rule": myRule
  },
  processors: {
    markdown: myMarkdownProcessor
  }
}

module.exports = plugin
```

###  Plugin の利用例

Legacy Config の場合は、 `plugin` や `extends` で指定する名称は、[命名規則](https://eslint.org/docs/v8.x/use/configure/plugins#naming-convention)に基づいて、`node_modules` から解決されるようになっています。

####  Plugin は利用するが、Shareable Config は利用しない場合

```js
modules.exports = {
  plugins: ["example"],

  // Plugin で利用可能になった Custom Rule の利用
  rules: {
    // plugin の名称にあわせて、custom rule の prefix が予め決まっている
    // 例) `eslint-plugin-sample` の場合は、`sample/` になる
    "example/my-rule": "warn",
  },

  // Plugin で利用可能になった Processor の利用
  processor: "example/my-processor",
}
```

#### Shareable Config を利用する場合

パッケージ名によって、`extends` で指定できる名称の省略方法が異なる。

##### パッケージ名が `eslint-plugin-` から始まる場合

```js
modules.exports = {
  extends: ["plugins:example/recommended"],
}
```

##### パッケージ名が `eslint-config-` から始まる場合

```js
modules.exports = {
  extends: ["example/recommended"],
}
```

## Flat Config Plugin の構造

参照したドキュメント
https://eslint.org/docs/latest/extend/plugins

### Plugin のコード例

```js
const plugin = {
  // デバッグや、キャッシュに利用される
  meta: {
    name: 'eslint-plugin-example',
    version: '1.0.0'
  },

  // Shareable Config の登録先
  configs: {},

  // Custom Rule
  rules: {
    "my-rule": myRule
  },
  processors: {
    hoge: myMarkdownProcessor
  }
};

// plugins フィールドで plugin object を参照する必要があるため、
// 以下のようにして Config を登録
// plugin.configs に登録せずに、別モジュールとして提供してもよい。
Object.assign(plugin.configs, {
  recommended: [
    {
      plugins: { example: plugin }
      rules: {
        "example/my-rule": "error" 
      }
    },
  ]
});

module.exports = plugin;
```

### Plugin の利用例

Flat Config の **[プラグインを自分で(ESLing 側が)解決しない](https://zenn.dev/babel/articles/eslint-flat-config-for-babel#%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3%E3%82%92%E8%87%AA%E5%88%86%E3%81%A7%E8%A7%A3%E6%B1%BA%E3%81%97%E3%81%AA%E3%81%84)** という特徴どおり、`eslint.config.js` にて、利用する Plugin を読み込んで利用する必要があります。個人的には、Legacy Config に比べると明示的で分かりやすくなったと感じています。

#### Plugin のみを利用

```js
const examplePlugin = require("eslint-plugin-example");

module.exports = [
  {
    // Plugin の利用
    plugins: {
      example: examplePlugin,
    },

    // Plugin で利用可能になった Custom Rule の利用
    // `example/` の部分は、`plugins.example` と対応しており、
    // `<prefix>/<rule>` の形式で指定する
    // 例) `plugins.sample` としたなら、`sample/` となる
    rules: {
      "example/my-rule": "warn",
    },

    // Plugin で利用可能になった Processor の利用
    // `rules` と同じ規則の prefix を `<prefix>/<processor>` の形式で指定する
    processor: "example/my-processor",
  }
];
```

#### Shareable Config を利用

```js
const examplePlugin = require("eslint-plugin-example");

module.exports = [
  examplePlugin.configs.recommended, 
  {
    // 追加の設定
  }
];
```

## 両者の構造見ての雑感

### `configs` 以外は一緒

Flat Config では、`processor` のキーに拡張子 (`.md` など)が利用できないという仕様の点での違いはありつつも、構造としては `configs` 以外は同じであることが分かります。
実際、 `@types/eslint` の `Plugin` をみると `configs` の型が `ConfigData` と `FlatConfig` の Union 型の `Record` になっています。

https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/eslint/index.d.ts#L1375-L1380


### ものによっては、[`FlatCompat`](https://eslint.org/docs/latest/use/configure/migration-guide#using-eslintrc-configs-in-flat-config) を使わずに Flat Config に移行できそう

[Flat Config も `rules` と `processor` の構文は同じため](https://eslint.org/docs/latest/use/configure/migration-guide#things-that-havent-changed-between-configuration-file-formats)、例えば、[eslint-plugin-react-hooks](https://github.com/facebook/react/tree/main/packages/eslint-plugin-react-hooks)のように、Sharable Config が `plugins` と `rules` のみの単純なものの場合は、
https://github.com/facebook/react/blob/main/packages/eslint-plugin-react-hooks/src/index.js#L13-L26

以下のように、plugin の指定と、config の指定を分割して行うだけで済む事がわかります。

```js
const pluginReactHooks = require('eslint-plugin-react-hooks')

module.exports = [
  {
    plugins: {
      // configs.recommended.rules で指定されている rule の
      // prefix が `react-hooks` になっているため
      'react-hooks': pluginReactHooks
    },
  },
  {
    rules: {
      ...pluginReactHooks.configs.recommended.rules
    }
  }
]
```

:::message
#### 移行に際しての注意点

- 利用している Plugin の Shareable Config で `globals` や、`parserOptions` などが利用されているケース
  - `FlatCompat` を頼る方がよいでしょう。
- [eslint-plugin-react の issue](https://github.com/jsx-eslint/eslint-plugin-react/issues/3699) のように、[ESLint v9 で削除された API](https://eslint.org/blog/2023/09/preparing-custom-rules-eslint-v9/#from-context-to-sourcecode) を利用しているケース
  - ESLint v8 のまま Flat Config へ移行するか、対応していない rule を個別に無効化する必要があります。
- [next lint](https://github.com/vercel/next.js/blob/59ad831a13a121ef815f0e6ea322ae1e66c364ff/packages/next/src/lib/eslint/runLintCheck.ts#L305-L318) と [eslint-config-next](https://www.npmjs.com/package/eslint-config-next) (内部で利用している [@rushstack/eslint-patch](https://www.npmjs.com/package/@rushstack/eslint-patch) が対応していない) の様に、`eslint.config.js` に対応していないケース 
  - Flat Config への対応を待つのが無難だと思います。
:::


## オマケ

### 両対応 Plugin の構造

`configs` 以外は基本的に同じ構造をしているため、公開されている Plugin では以下の2パターンをみかけます。

- `configs` に Legacy と Flat の両方の Shareable Config を入れているパターン
  - [eslint-config-n](https://github.com/eslint-community/eslint-plugin-n/blob/master/lib/index.js#L70-L82) や [eslint-plugin-playwright]()
- main では、Legacy Config の構造をとり、別ファイルで Flat Config の Shareable Config を export しているパターン
  - [eslint-plugin-react](https://github.com/jsx-eslint/eslint-plugin-react/tree/master/configs)

以下は、1つめの「`configs` に Legacy と Flat の両方の Shareable Config を入れているパターン」を例示したものになります。


```js
const plugin = {
  // デバッグや、キャッシュに利用される
  meta: {
    name: 'eslint-plugin-example',
    version: '1.0.0'
  },
  
  // Custom Rule
  rules: {
    "my-rule": myRule
  },
  
  processors: {
    hoge: myMarkdownProcessor
  }
};

const legacyConfigs = {
  recommended: {
    plugins: ['example']
      rules: {
        "example/my-rule": "error" 
      }
    }
  }
}

const flatConfigs = {
  'flat/recommended': {
    plugins: {
      example: plugin
    },
    rules: plugin.configs.recommended.rules
  }
}

module.exports = {
  ...plugin,
  configs: {
    ...legacyConfigs,
    ...flatConfigs,
  }
};
```

### 資料

- [ESLint v9 の Flat Config サポート状況](https://github.com/eslint/eslint/issues/18093)
- [Flat Config への Migration ガイド](https://eslint.org/docs/latest/use/configure/migration-guide)
- [Legacy Config 向けの Plugin 作成ガイド](https://eslint.org/docs/v8.x/extend/plugins)
- [Flat Config 向けの Plugin 作成ガイド](https://eslint.org/docs/latest/extend/plugins)




