# Markdownの拡張機能

VitePressはMarkdownの拡張機能を内蔵しています。

## ヘッダーアンカー

ヘッダーには自動的にアンカーリンクが適用されます。アンカーのレンダリングはmarkdown.anchorオプションで設定することができます。

## リンク

内部リンク、外部リンクともに特別な扱いを受けています。

### 内部リンク

内部リンクはSPAのナビゲーションのためにルーターリンクに変換されます。また、各サブディレクトリに含まれるindex.mdは自動的にindex.htmlに変換され、対応するURLは / となります。

例えば、以下のようなディレクトリ構造であった場合。

```
.
├─ index.md
├─ foo
│  ├─ index.md
│  ├─ one.md
│  └─ two.md
└─ bar
   ├─ index.md
   ├─ three.md
   └─ four.md
```

そしてfoo/one.mdにあることを提供します。

```md
[Home](/) <!-- ルートの index.md にリンクしています。 -->
[foo](/foo/) <!-- foo ディレクトリの index.html にリンクしています。 -->
[foo heading](./#heading) <!-- foo インデックスファイル内の見出しにリンクしています。 -->
[bar - three](../bar/three) <!-- 拡張子を省略することができます。 -->
[bar - three](../bar/three.md) <!-- .mdを追加することができます。 -->
[bar - four](../bar/four.html) <!-- または.html を追加することができます。 -->
```

### ページサフィックス

ページと内部リンクの接尾辞は、デフォルトで.htmlで生成されます。

### 外部リンク

外部リンクは自動的にtarget="_blank" rel="noreferrer"を取得します。

- vuejs.org
- VitePress on GitHub

## フロントマター

YAMLフロントマターがそのままサポートされます。

```yaml
---
title: Blogging Like a Hacker
lang: en-US
---
```

このデータはすべてのカスタムコンポーネントとテーマコンポーネントとともに、ページの他の部分でも利用できるようになります。

詳しくは、Frontmatter を参照してください。

## GitHub-Style Tables

Input

```md
| Tables        |      Are      |  Cool |
| ------------- | :-----------: | ----: |
| col 3 is      | right-aligned | $1600 |
| col 2 is      |   centered    |   $12 |
| zebra stripes |   are neat    |    $1 |
```

Output

| Tables        |      Are      |  Cool |
| ------------- | :-----------: | ----: |
| col 3 is      | right-aligned | $1600 |
| col 2 is      |   centered    |   $12 |
| zebra stripes |   are neat    |    $1 |

## Emoji 🎉 #

Input

```md
:tada: :100:
```

Output

🎉 💯

全絵文字の一覧を見ることができます。

## 目次

Input

```md
[[toc]]
```

Output

- Header Anchors
- Links
  - Internal Links
  - Page Suffix
  - External Links
- Frontmatter
- GitHub-Style Tables
- Emoji 🎉
- Table of Contents
- Custom Containers
  - Default Title
  - Custom Title
  - raw
- Syntax Highlighting in Code Blocks
- Line Highlighting in Code Blocks
- Focus in Code Blocks
- Colored diffs in Code Blocks
- Errors and warnings
- Line Numbers
- Import Code Snippets
- Markdown File Inclusion
- Advanced Configuration
- Rendering of the TOC can be configured using the markdown.toc option.

## カスタムコンテナ

カスタムコンテナは、タイプ、タイトル、コンテンツによって定義することができます。

### デフォルトのタイトル

Input

```md
::: info
This is an info box.
:::

::: tip
This is a tip.
:::

::: warning
This is a warning.
:::

::: danger
This is a dangerous warning.
:::

::: details
This is a details block.
:::
```

Output

> INFO
> This is an info box.

> TIP
> This is a tip.

> WARNING
> This is a warning.

> DANGER
> This is a dangerous warning.

> Details
> This is a details block.

### カスタムタイトル

コンテナの "type "の直後にテキストを追加することで、カスタムタイトルを設定することができます。

Input

```md
::: danger STOP
Danger zone, do not proceed
:::

::: details Click me to view the code
```js
console.log('Hello, VitePress!')
```
:::
```

Output

> STOP
> Danger zone, do not proceed

> Click me to view the code
> `console.log('Hello, VitePress!')`

### raw

これは、VitePressのスタイルとルータの競合を防ぐために使用できる特別なコンテナです。これは、コンポーネント・ライブラリのドキュメントを作成する際に特に便利です。また、より良い分離のためにwhyframeをチェックアウトしたいかもしれません。

### Syntax

```md
::: raw
Wraps in a <div class="vp-raw">
:::
```

vp-rawクラスは、エレメントにも直接使用することができます。スタイル分離は現在オプトインです。

#### 詳細

- お好みのパッケージマネージャで必要なdepsをインストールします。

```sh
$ yarn add -D postcss postcss-prefix-selector
```

- docs/.postcssrc.cjs という名前のファイルを作成し、これを追加します。

```js
module.exports = {
  plugins: {
    'postcss-prefix-selector': {
      prefix: ':not(:where(.vp-raw *))',
      includeFiles: [/vp-doc\.css/],
      transform(prefix, _selector) {
        const [selector, pseudo = ''] = _selector.split(/(:\S*)$/)
        return selector + prefix + pseudo
      }
    }
  }
}
```

## コードブロックのシンタックスハイライト

VitePressは、Markdownコードブロック内の言語構文をカラーテキストでハイライトするためにShikiを使用しています。Shikiは様々なプログラミング言語をサポートしています。必要なことは、コードブロックの最初のバックティックに有効な言語エイリアスを追加することだけです。

Input

    ```js
    export default {
    name: 'MyComponent',
    // ...
    }
    ```

    ```html
    <ul>
    <li v-for="todo in todos" :key="todo.id">
        {{ todo.text }}
    </li>
    </ul>
    ```

Output

```js
export default {
  name: 'MyComponent'
  // ...
}
```

```html
<ul>
  <li v-for="todo in todos" :key="todo.id">
    {{ todo.text }}
  </li>
</ul>
```

有効な言語の一覧は、Shikiのリポジトリで公開されています。

また、アプリの設定でシンタックスハイライトのテーマをカスタマイズすることができます。詳しくはマークダウンオプションをご覧ください。

## コードブロックのラインハイライト

Input

    ```js{4}
    export default {
    data () {
        return {
        msg: 'Highlighted!'
        }
    }
    }
    ```

Output

```js
export default {
  data () {
    return {
      msg: 'Highlighted!'
    }
  }
}
```

1行だけでなく、複数の1行、範囲、またはその両方を指定することも可能です。

- 行の範囲：例えば{5-8}、{3-10}、{10-17}など。
- 複数の単線：例えば{4,7,9}。
- 行範囲と単一行：例えば {4,7-13,16,23-27,40} など。

Input

    ```js{1,4,6-8}
    export default { // Highlighted
    data () {
        return {
        msg: `Highlighted!
        This line isn't highlighted,
        but this and the next 2 are.`,
        motd: 'VitePress is awesome',
        lorem: 'ipsum'
        }
    }
    }
    ```

Output

```js
export default { // Highlighted
  data () {
    return {
      msg: `Highlighted!
      This line isn't highlighted,
      but this and the next 2 are.`,
      motd: 'VitePress is awesome',
      lorem: 'ipsum',
    }
  }
}
```

あるいは、 // [!code hl] というコメントを使うことで、行内で直接ハイライトすることも可能です。

Input

    ```js
    export default {
    data () {
        return {
        msg: 'Highlighted!' // [!codeㅤ hl]
        }
    }
    }
    ```

Output

```js
export default {
  data () {
    return {
      msg: 'Highlighted!' 
    }
  }
}
```

## コードブロックにフォーカス

行に // [!code focus] コメントを追加すると、その行にフォーカスが当たり、コードの他の部分がぼやけます。

さらに、フォーカスを当てる行数を // [!code focus:\<lines\>] で定義することができます。

Input

    ```js
    export default {
    data () {
        return {
        msg: 'Focused!' // [!codeㅤ focus]
        }
    }
    }
    ```

Output

```js
export default {
  data () {
    return {
      msg: 'Focused!' 
    }
  }
}
```

## コードブロックの色付き差分

ある行に // [!code --] または // [!code ++] コメントを追加すると、コードブロックの色を維持したままその行の diff が作成されます。

Input

    ```js
    export default {
    data () {
        return {
        msg: 'Removed' // [!codeㅤ --]
        msg: 'Added' // [!codeㅤ ++]
        }
    }
    }
    ```

Output

```js
export default {
  data () {
    return {
      msg: 'Removed' 
      msg: 'Added' 
    }
  }
}
```

## エラーと警告

行に // [!code warning] または // [!code error] のコメントを追加すると、それに応じて色がつきます。

Input

    ```js
    export default {
    data () {
        return {
        msg: 'Error', // [!codeㅤ error]
        msg: 'Warning' // [!codeㅤ warning]
        }
    }
    }
    ```

Output

```js
export default {
  data () {
    return {
      msg: 'Error', 
      msg: 'Warning' 
    }
  }
}
```

## 行番号

設定により、各コードブロックの行番号を有効にすることができます。

```js
export default {
  markdown: {
    lineNumbers: true
  }
}
```

詳しくはマークダウンオプションをご覧ください。

## コードスニペットのインポート

以下の構文で、既存のファイルからコード・スニペットをインポートすることができます。

```md
<<< @/filepath
```

また、ラインハイライトにも対応しています。

```md
<<< @/filepath{highlightLines}
```

Input

```md
<<< @/snippets/snippet.js{2}
```

Code file

```js
export default function () {
  // ..
}
```

Output

```js
export default function () {
  // ..
}
```

> ヒント: @の値は、ソースルートに対応します。srcDirが設定されていない限り、デフォルトではVitePressのプロジェクトルートです。

また、VS Codeリージョンを使用して、コードファイルの対応する部分のみを含めることができます。ファイルパスに続く#の後に、カスタムリージョン名を指定することができます。

Input

```md
<<< @/snippets/snippet-with-region.js#snippet{1}
```

Code file

```js
// #region snippet
function foo() {
  // ..
}
// #endregion snippet

export default foo
```

Output

```js
function foo() {
  // ..
}
```

また、次のように中括弧（{}）の中に言語を指定することもできます。

```md
<<< @/snippets/snippet.cs{c#}

<!-- with line highlighting: -->
<<< @/snippets/snippet.cs{1,2,4-6 c#}
```

これは、ファイルの拡張子からソース言語が推測できない場合に便利です。

## Markdownファイルのインクルージョン

マークダウン・ファイルを別のマークダウン・ファイルにインクルードすることができます。

Input

```md
# Docs

## Basics

<!--@include: ./parts/basics.md-->
```

Part file (parts/basics.md)

```md
Some getting started stuff.

### Configuration

Can be created using `.foorc.json`.
```

Equivalent code

```md
# Docs

## Basics

Some getting started stuff.

### Configuration

Can be created using `.foorc.json`.
```

> 警告: この機能は、ファイルが存在しない場合エラーをスローしないことに注意してください。したがって、この機能を使用する場合はコンテンツが期待どおりにレンダリングされることを確認してください。

# アドバンスト・コンフィグレーション

VitePressはMarkdownのレンダラとしてmarkdown-itを使用しています。上記の拡張機能の多くは、カスタムプラグインによって実装されています。.vitepress/config.jsのmarkdownオプションを使って、markdown-itのインスタンスをさらにカスタマイズすることができます。

```js
const anchor = require('markdown-it-anchor')

module.exports = {
  markdown: {
    // options for markdown-it-anchor
    // https://github.com/valeriangalliat/markdown-it-anchor#usage
    anchor: {
      permalink: anchor.permalink.headerLink()
    },

    // options for @mdit-vue/plugin-toc
    // https://github.com/mdit-vue/mdit-vue/tree/main/packages/plugin-toc#options
    toc: { level: [1, 2] },

    config: (md) => {
      // use more markdown-it plugins!
      md.use(require('markdown-it-xxx'))
    }
  }
}
```

設定可能なプロパティの一覧はConfigsをご覧ください。アプリの設定(Config)