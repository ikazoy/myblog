---
title: KintoneでWebpack + Babel + Vueを使ってモダンに爆速開発していくサンプル(1/2)
date: 2017-08-01T07:42:10.707Z
categories:
  - development
author: me
layout: blog
---

# 背景
Kintoneのカスタマイズを始めたのですが、開発を進めるにあたって「1つのアプリに対して、JSファイルやCSSファイルをどんどんアップロードしていく仕組みだと開発スピードが落ちる」ということを強く感じました。その理由としては、以下の通りです。

- 使いたいライブラリが出て来るたびにいちいちアップロードするのが面倒
- JSファイルの構成を変えたときもアップロードし直さないといけない
- 複数開発者が1つのアプリに対して作業するときにコンフリクトしやすい

なお、webpack + Babelの導入だけであればKintone devcamp 2017のセッションの[スライド](https://www.slideshare.net/kintone-papers/webpackjs)がステップバイステップで説明してあり、かなりわかりやすいです。

# Webpackを導入するぞ！

上記の問題を受けて、現代のフロントエンド開発では欠かせない[Webpack](https://webpack.github.io/)を導入することにしました。

Webpackとは、複数ファイル（Javascript、css、テンプレートなどなど）を1つのファイルにいい感じにまとめて（Bundleと呼びます）くれるツールのことです。（他にわかりやすい記事がたくさんあるので詳しく知りたい方は検索しましょう）

例：
顧客が一覧になっている画面に、メール送信用のボタンをそれぞれに追加する状況を考えましょう。メール送信用ボタンを押すと宛先がすでに入力された状態でポップアップが立ち上がり、メールのタイトル、本文が編集可能になる、とします。

<img src="/image/kintone-sample-screen.png" alt="サンプル画面" title="サンプルアプリ" width="800">


通常であれば、1つのJavascriptファイルに、
- メール送信ボタンを配置するためのDOMを検索する処理
- ボタンのHTMLとそれを配置する処理
- ボタンが押されたときに表示されるポップアップのHTMLテンプレート
- etc...

という風にゴリゴリ処理を書き、それをJavascriptファイルとしてKintoneにアップロードすることでしょう。どうでしょう、辛い未来が見えませんか？そこで登場するのがWebpackです。

更に今回はWebpackのメリットをより実感するために[Vuejs](https://jp.vuejs.org/index.html)や[KeenUI](https://josephuspaye.github.io/Keen-UI/#/ui-alert)もついでに入れてみたいと思います。

# まずはシンプルにボタンを置く

Kintoneのサンプルによくあるこんな感じでKintoneに読み込ませて、顧客詳細情報を表示します。
「メール送信」ボタンが1つページ上部に表示されるはずです。

`popup-email-simple.js`

```javascript
(function () {
  'use strict';
  var events = ['app.record.detail.show'];
  kintone.events.on(events, function (event) {
    var se = kintone.app.record.getHeaderMenuSpaceElement();
    var btn = document.createElement("BUTTON");
    var t = document.createTextNode("メール送信");
    btn.appendChild(t);
    se.appendChild(btn);
  });
})();
```

# そろそろwebpackでも入れようか、、、

ボタンは無事表示できましたが、このまま拡張していくのは早々に諦め、Webpackを使う方向に転換します。

## package.json

```javascript
{
  "name": "kintone-webpack-sample",
  "version": "1.0.0",
  "scripts": {
    "webpack": "webpack",
    "serve": "webpack-dev-server --https --inline --hot --port=8081"
  },
  "author": "Yoshiki Ozaki",
  "dependencies": {
    "glob": "^7.1.2",
    "keen-ui": "^1.0.0",
    "vue": "^2.3.4"
  },
  "devDependencies": {
    "babel": "^6.23.0",
    "babel-core": "^6.24.1",
    "babel-loader": "^7.0.0",
    "babel-polyfill": "^6.23.0",
    "babel-preset-es2015": "^6.24.1",
    "babel-register": "^6.24.1",
    "style-loader": "^0.18.2",
    "stylus": "^0.54.5",
    "stylus-loader": "^3.0.1",
    "vue-loader": "^12.2.1",
    "vue-template-compiler": "^2.3.4",
    "webpack": "^2.6.1",
    "webpack-dev-server": "^2.4.5"
  }
}
```

大事な部分は、webpackとbabel*の部分です。また、ついでにvue,keen-ui、その他必要そうなloaderを追加しておきます。


## .babelrc

```
{
  "presets": ["es2015"]
}
```

## webpack.config.babel.js

webpackの設定ファイルもes2015で書きたい！ということで、`webpack.config.bable.js`というファイル名にしておきます。
`entry`でwebpackに渡したいファイルを指定、`output`でwebpackがバンドルしたファイルの置き場を指定します。

`loaders`セクションで、拡張子ごとにどのように処理していくか（どのloaderに処理させるか）を指定します。

```javascript
const glob = require('glob');
export default {
  // entry: glob.sync('./src/*.js'),
  entry: './popup-email-simple.js',
  output: {
    path: `${__dirname}/dist`,
    filename: 'bundle.js'
  },
  devtool: 'source-map',
  module: {
    loaders: [
      {
        test: /\.js?$/,
        exclude: /node_modules/,
        loader: 'babel-loader'
      },
      {
        test: /\.vue$/,
        loader: 'vue-loader'
      },
      {
        test: /\.css$/,
        loader: 'style-loader!css-loader'
      }
    ]
  },
  resolve: {
    alias: { vue: 'vue/dist/vue.js' }
  },
};
```

# ちなみにBabelとは
開発者：「新しいJavascriptの書き方を使いたい！」「でもまだサポートしてないブラウザもあるしなー」

Babel：「わいがトランスパイルしてどのブラウザでも動くようにしといたるやでー」

という存在です。

# とうとうwebpack実行

2. `$ npm run webpack`
	- ずらっとファイル名が出力され、コンパイルされます。
3. `cat dist/bundle.js`
	- コンパイルされて、新しくbundle.jsが出来ていることを確認します。
4. できあがった`bundle.js`をKintoneにアップロードして動くことを確認

# それでもKintoneに毎回アップロードするの面倒じゃない？
webpackでバンドルしたとしても、javascirptファイルをアップロードすることには変わりありません。そこで便利なのが、`webpack-dev-server`コマンドです。

上記package.jsonにもありますが、このように設定しておくと、webpackでバンドルしたファイルをhttps経由（※KintoneにURLでJavascriptを読み込ませるにはSSLが必須のため）で配信してくれます。

`package.json`

```javascript
{
.......
  "scripts": {
    "webpack": "webpack",
    "serve": "webpack-dev-server --https --inline --hot --port=8081"
  },
.........
  "devDependencies": {
    "webpack-dev-server": "^2.4.5"
  }
}
```
`npm run serve`というコマンドを叩くことで、https://localhost:8081/index.jsというアドレスでバンドルされたファイルが取得できます。

hotオプションのおかげで、webpackが処理しているファイルを更新すると、すぐに反映されますので、開発中のファイル更新→画面で確認、というサイクルが爆速でで回せるようになります。これはかなり便利ですので、ぜひお試しください。

※注意：ssl証明書などを準備していないため、chromeなどのブラウザでは接続を拒否されてしまい、Kintoneからうまく読み込めないことがあります。その際は一度localhodtのURLを直接入力して、警告を無視する設定にしておくと、Kintoneでもうまく動作します。警告を出さずにやりたい方は、`--key`オプションや`--cert`オプションを併用されるとよいでしょう。

長くなってきたので、Vueの導入は次回後編で。