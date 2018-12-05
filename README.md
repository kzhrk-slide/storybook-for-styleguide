# StyleguideとしてStorybookを使う

## [Storybook](https://storybook.js.org/)とは

UIコンポーネント開発を目的としてつくられたNode.jsベースのジェネレータ。

React/Vueのコンポーネント管理のために利用され話題になっているが、Angular/Mithril/Marko/Svelte/Riotなどの[複数のUIライブラリもサポート](https://storybook.js.org/basics/slow-start-guide/)している。  
UIライブラリを使わないプレーンなHTMLを扱う[@storybook/html](https://github.com/storybooks/storybook/tree/next/app/html)も存在する。

設定ファイルと、コンポーネントファイルとStorybookを紐付けるstoriesファイルを作成することでコンポーネントの一覧ページが自動で作成される。  
Storybookは[@storybook/coreにwebpackを搭載している](https://github.com/storybooks/storybook/blob/26ae1e005865ba5e15da6455ef24dcd14050e631/lib/core/package.json#L82)ため、storiesファイル内でwebpackのloaderを使用することができる。

@storybook/htmlとpug-loader、style-loader、css-loader、sass-loaderを使ってコーポレートサイトのモックページ管理に使用できないか考えてみる。

## 設定ファイル

### config.js

`.storybook/config.js`に設定を記述する。  
`.storybook`ディレクトリは、StorybookのCLIの設定ディレクトリオプション（`--config-dir` `-c`）のデフォルト値にあたる。

```js
import { configure } from '@storybook/html';

function loadStories () {
  require('../stories');
}

configure(loadStories, module);
```

`configure`の第一引数に渡している`loadStories`関数で呼び出している`../stories`のパスはstoriesファイルが置かれているディレクトリにあたる。  
`configure`の第一引数に渡す関数内で複数のstoriesを読み込みたい場合は、`require`を複数行記述する。

### webpack.config.js

webpackの機能をstoriesファイル内で使う場合は、`.storybook/webpack.config.js`にwebpackの設定を記述する。  
今現在のStorybook 4では、webpack 4の設定ファイルと互換性がある。  
ここで注意したいのが、`.storybook/webpack.config.js`に記述するwebpackの設定は、Storybook実行時に利用される特殊な形式であるということだ。

webpackの設定方法には、[Extend Mode](https://storybook.js.org/configurations/custom-webpack-config/#extend-mode)と[Full Control Mode](https://storybook.js.org/configurations/custom-webpack-config/#full-control-mode)が存在する。  

#### Extend Mode

Extend Modeでは

- entry
- output
- babel-loader

の設定を省いた、下記のように`.storybook/webpack.config.js`を記述することができる。  
ここで記述するwebpackの設定は、Storybookにビルドインされたwebpackの設定にマージされるObjectという理解で問題ない。
　
```js
module.exports = {
  resolve: {
    extensions: ['.js', '.scss', '.pug', '.md']
  },
    
  module: {
    rules: [
      {
        test: /\.pug$/,
        exclude: /node_modules/,
        use: [
            'pug-loader'
        ]
      },
      {
        test: /\.scss$/,
        exclude: /node_modules/,
        use: [
            'style-loader',
            'css-loader',
            'sass-loader'
        ]
      }
    ]
  }
};
```

#### Full Control Mode

Full Control Modeでは`.storybook/webpack.config.js`で関数をexportする。  
第1引数にStorybookのベースになるwebpackの設定オブジェクト、第2引数には`DEVELOPMENT`か`PRODUCTION`の文字列が渡される。  
`build-storybook`コマンドを実行して静的ページをジェネレートする際に第2引数に`PRODUCTION`が渡される。  
Storybook実行時、webpackに設定オブジェクトを生成するための関数という理解で問題ない。

```js
module.exports = (storybookBaseConfig, configType) => {

  storybookBaseConfig.resolve = {
    extensions: ['.js', '.scss', '.pug', '.md']
  };

  storybookBaseConfig.module.rules.push({
    test: /\.scss$/,
    exclude: /node_modules/,
    use: [
      'style-loader',
      'css-loader',
      'sass-loader'
    ]
  });

  storybookBaseConfig.module.rules.push({
    test: /\.pug$/,
    exclude: /node_modules/,
    use: [
      'pug-loader'
    ]
  });

  // Return the altered config
  return storybookBaseConfig;
};
```

## storiesファイル

`stories/index.js`にストーリーを記述する。  
@storybook/htmlの場合は、`add`メソッドの第2引数の関数でHTML文字列かDOMを返却する必要がある。

```js
import { storiesOf } from '@storybook/html';

storiesOf('Headline', module)
  .add('見出し', () => '<h1>見出し</h1>') // return string
  .add('アイコン付き見出し', () => {
    const headline = document.createElement('h1');
    headline.classList.add('icon');

    return headline; // return DOM
  });
```

ここで先ほど`.storybook/webpack.config.js`で設定したloaderを利用すると、外部ファイルで管理されたpugやscssを読み込むことが可能になる。

```js
import { storiesOf } from '@storybook/html';
import priorityTemplate from '../src/pug/components/headline/index.pug';
import iconTemplate from '../src/pug/components/headline/icon.pug';
import '../src/scss/style.scss';

storiesOf('Headline', module)
  .add('見出し', priorityTemplate)
  .add('アイコン付き見出し', iconTemplate)
```

JavaScriptが必要な場合は、関数の中で`require`を実行してJavaScriptファイルを読み込む。

```js
storiesOf('カルーセル', module)
  .add('無限ループ', () => {
    require('../src/js/carousel');
    return carouselTemplate();
  });
```

## Addon

Storybookにはstoriesに機能追加ができるAddon機能が実装されている。

[StorybookのチームがメンテナーをしているAddon](https://storybook.js.org/addons/addon-gallery/)には、storiesの下部に表示されるタブにNotesタブを追加できる[Notes](https://github.com/storybooks/storybook/tree/master/addons/notes)というAddonがある。  
コンポーネントの説明欄としてNotesをAddonに追加する場合、下記のような設定を行う必要がある。

### Install

```
# npm
npm i -D @storybook/addon-notes

# yarn
yarn add -D @storybook/addon-notes
```

### 設定ファイル

@storybook/htmlから`addDecorator`を呼び出し、Notesを引数として渡す。

```js
import { configure, addDecorator } from '@storybook/html';
import { withNotes } from '@storybook/addon-notes';

function loadStories () {
  require('../stories');
}

addDecorator(withNotes);
configure(loadStories, module);
```

### storiesファイル

`add`メソッドの第3引数にnotesのkeyを持つObjectを渡す。  
notesのvalueはStringがObjectを設定することができる。

Stringの場合は文字列がそのままNotesタブに表示される。  

Objectの場合は、markdownのkeyを定義し、valueにMarkdownの文字列を渡すとNotesタブにHTMLが表示される。

```js
import { storiesOf } from '@storybook/html';
import iconTemplate from '../../../src/pug/components/headline/icon';

// string
storiesOf('Headline', module)
  .add('アイコン付き見出し', iconTemplate, {
    notes: 'description'
  })

// makrdown
storiesOf('Headline', module)
  .add('アイコン付き見出し', iconTemplate, {
    notes: {
      markdown: `
- .c-headline__icon--circle
- .c-headline__icon--square
`
    }
  })
```

[RWD向けのAddon](https://github.com/storybooks/storybook/tree/master/addons/viewport)などもあるので用途によってAddonを追加するとUI管理が便利になる。  
また、Addon開発を自前で開発するのもよいかもしれない。

以上がStorybookの設定方法になる。  
このあとは、HTMLのStyleguideとしてStorybookを使う場合、どのようなアプローチがあるか考えてみる。

## ディレクトリ構造

StyleguideとしてStorybookを使う際のディレクトリ構造としては、下記のような構造が考えられる。  
`src/pug/components`にコンポーネント、`src/pug/pages`にモックページを作成する。

CSS設計として[FLOCSS](https://github.com/hiloki/flocss)を採用した場合、`src/pug/components`配下はFLOCSSのComponents確認、`src/pug/pages`配下にはFLOCSSのProject確認として利用できるだろう。

```
.
├── src
│   ├── webpack
│   ├── pug
│   │   ├── component
│   │   │   ├── button
│   │   │   │   ├── icon.pug
│   │   │   │   └── text.pug
│   │   │   ├── ...
│   │   │   └── index.pug
│   │   ├── page
│   │   │   ├── top
│   │   │   │   ├ index.pug
│   │   │   │   └ no_login.pug
│   │   │   └── ...
│   │   └── config.pug
│   └── scss 
│       ├── foundation
│       ├── layout
│       └── object
│           ├── component
│           │   ├── _button.scss
│           │   └── ...
│           ├── project
│           └── utility
└── stories
    ├── page
    |   ├── index.js
    |   ├── user.js
    |   └── ...
    └── component
        ├── button.js
        └── ...
```

scssファイルはsrcファイル内のものをそのままビルドすればプロダクトに乗せることができるが、pugファイルはページ作成時にモジュールリストから必要なpugをコピペする必要があり、モックとプロダクトの間に乖離が生まれる運用リスクがある。  
この運用リスクを回避し、モックの再利用性をより高めるために[mixin](https://pugjs.org/language/mixins.html)を利用してpugを構築する方法がある。

## pugのmixin

pugには再利用性を高める機能として[Mixins](https://pugjs.org/language/mixins.html)が実装されている。  
`mixin {name}(arguments)`で定義ブロックを作成し、`+{name}(arguments)`で呼び出すことができる。

listコンポーネントのためにlist mixinを定義して実行した場合、下記のようになる。

```pug
mixin list(type, items)
  case type
    when 'col2'
      ul(class='c-col2List')
        each item in items
          li(class='c-col2List__item')
            dl
              dt= item.title
              dd= item.description

    when 'col3'
      ul(class='c-col3List')
        each item in items
          li(class='c-col3List__item')
            dl
              dt= item.title
              dd= item.description

+list('col2', [{
  title: 'タイトル1',
  description: '説明文1'
},{
  title: 'タイトル2',
  description: '説明文2'
}])
```

pugのmixinを使いつつ、コンポーネント管理をすると`src/pug`の下のディレクトリ構成は下記のようなものが考えられる。  

```
.
├── src
│   ├── pug
│   │   ├── components
│   │   │   ├── button
│   │   │   │   ├── icon.pug
│   │   │   │   ├── index.pug
│   │   │   │   └── mixin.pug
│   │   │   └── list
│   │   │       ├── index.pug
│   │   │       └── mixin.pug
│   │   ├── pages
│   │   │   └── ...
│   │   └── config.pug
```

`src/pug/components`の各カテゴリのディレクトリ配下にmixin.pugを用意し、ここで該当カテゴリに含まれるコンポーネントの全パターンを定義する。  
mixin.pug以外のpugファイルで、Storybookに出力する各コンポーネントの出力例を定義する。

```pug
- var config = require('../../config.js')

mixin headline(type, data)
  case type
    when config.headline.icon.circle
      h1.c-headline__icon--circle= data.text
    when config.headline.icon.square
      h1.c-headline__icon--square= data.text
```

```pug
- var config = require('../../config.js')
include ./mixin.pug

+headline(config.headline.icon.square, {
  text: '四角アイコン付き見出し'
})

hr

+headline(config.headline.icon.circle, {
  text: '丸アイコン付き見出し'
})
```

以上のpugの構成であれば、Storybook用に出力するためにUI確認用のサンプルファイルとコンポーネントを出力するためのmixinファイルがきれいに分離できる。  
プロダクトページを実装する際に各mixin.pugをincludeしてpugファイルを作成することでStorybookのUIコンポーネント一覧とプロダクトページのHTMLの間で差異が生まれなくなる。  
また、UIコンポーネントのアップデート確認をStorybook上で行うことで影響範囲の把握を容易に行うことができ、運用によるUIの破損リスク低下が期待できる。
