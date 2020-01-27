# Nuxt.js (SSRモード) の 一式を Firebase Cloud Functions 上で動かすサンプル  
Nuxt.js 公式 の情報や Firebase 公式の情報だけでは上手く環境を作れなかったので、調べながら作ったサンプルです。
環境構築からデプロイまでの説明となります。

# github  
サンプル一式のリポジトリ  
[firebase-nuxtjs-sample](https://github.com/tfuru/firebase-nuxtjs-sample)

# 環境概要 
利用した各環境についての概要

[Firebase](https://firebase.google.com/?hl=ja) 
- Hosting
- Functions / TypeScript 指定

[Nuxt.js](https://ja.nuxtjs.org/)
- [Nuxt TypeScript](https://typescript.nuxtjs.org/ja/)
- SSR モード (Server Side Rendering)

# Firebase 初期設定
普通に Firebase を初期設定する
`Functions` と `Hosting` と Functions の言語を `TypeScript` にする。

```
mkdir nuxtjs-sample
cd nuxtjs-sample
firebase init
...
 ◉ Functions: Configure and deploy Cloud Functions
❯◉ Hosting: Configure and deploy Firebase Hosting sites
...
> Create a new project 
...
? What language would you like to use to write Cloud Functions? 
❯ TypeScript
```

# Nuxt.js 初期設定
`Nuxt TypeScript` を有効にする場合 `Choose linting tools` については 未選択で Enter キーを押して
進める。`TypeScript` 対応の時に手動で設定する。

`Choose rendering mode Universal (SSR)` は忘れずに選択する。

```
cd nuxtjs-sample
npx create-nuxt-app nuxt-app
...
? Choose the package manager Npm
? Choose UI framework None
? Choose custom server framework Express
? Choose Nuxt.js modules (Press <space> to select, <a> to toggle all, <i> to invert selection)
? Choose linting tools (Press <space> to select, <a> to toggle all, <i> to invert selection)
? Choose test framework None
? Choose rendering mode Universal (SSR)
? Choose development tools jsconfig.json (Recommended for VS Code)

```

# Nuxt TypeScript を有効にする
`Nuxt TypeScript` を有効にするため設定ファイルを調整する
[Nuxt TypeScript 公式 セットアップ](https://typescript.nuxtjs.org/ja/guide/setup.html) を参考に進める。
 
```
cd nuxt-app/
npm install --save-dev @nuxt/typescript-build
npm install @nuxt/typescript-runtime
```

1. nuxt.config.js の修正
```
export default {
  buildModules: ['@nuxt/typescript-build']
}
```

2. tsconfig.json の追加

```
// tsconfig.json
{
  "compilerOptions": {
    "target": "es2018",
    "module": "esnext",
    "moduleResolution": "node",
    "lib": [
      "esnext",
  ...
  },
  "exclude": [
    "node_modules"
  ]
}
```

3. package.json の更新

`scripts` について `nuxt-ts` を利用するように変更

```
"scripts": {
  "dev": "nuxt-ts",
  "build": "nuxt-ts build",
  "generate": "nuxt-ts generate",
  "start": "nuxt-ts start"
},
"dependencies": {
  "@nuxt/typescript-runtime": "latest",
  "nuxt": "latest"
},
"devDependencies": {
  "@nuxt/typescript-build": "latest"
}
```

4. nuxt ビルドしてみる
成功すれば 初期設定は完了している。

```
npm run build
```
# package.json へ functions 対応を追加
Nuxt.js (SSR) server コードを functions に反映させる 為に
必要ファイルを  `/functions` へコピーするように `package.json` に `build-nuxt` を追加する

- `.nuxt` を `/functions/.nuxt` へコピー
- `.nuxt/dist/client/` を `../public/assets` へコピー

```
  "scripts": {
    "dev": "nuxt-ts",
    "build": "nuxt-ts build",
    "build-nuxt": "nuxt-ts build && cp -R .nuxt/ ../functions/.nuxt && rm -rf ../public/* && cp -R .nuxt/dist/client/ ../public/assets",
    "generate": "nuxt-ts generate",
    "start": "nuxt-ts start"
  },

```

ビルドできるか確認する

```
npm run build-nuxt

# コピーできているか確認する
ls ../public/assets/
ls ../functions/.nuxt/
```

# firebase.json を 編集
`rewrites` を追加して `function` `ssrapp` へアクセスするように設定。

```
{
  "functions": {
...
  },
  "hosting": {
...
    "rewrites": [{
      "source": "**",
      "function": "ssrapp"
    }]
  }
}
```

# functions/index.ts を編集

1. functions で Nuxt.js 実行に必要ライブラリを インストールする。

```
npm install --save nuxt express
npm install --save firebase-functions
```

2. Nuxt.js を利用するように `src/index.ts` を変更する

```
import * as functions from 'firebase-functions';
const { Nuxt } = require('nuxt');
const express = require('express');

const app = express();
const config = {
    dev: false,
    srcDir: '.nuxt',
    build: {
      publicPath: '/assets/',
    }
  };
  
const nuxt = new Nuxt(config);
app.use(async (req: any, res: any) => {
  // await nuxt.ready()
  return await nuxt.render(req, res)
})
exports.ssrapp = functions.https.onRequest(app);
```

3. `npm run shell` を実行して function の動作確認  
nuxt-app 等が表示されれば 設定変更は成功している

```
npm run shell
firebase > ssrapp.get('')

...
    <h1 class="title">
      nuxt-app
    </h1> <h2 class="subtitle">
      My world-class Nuxt.js project
    </h2>
...
```
4. `npm run deploy` を実行して function をデプロイしてみる
デプロイ成功すれば 設定変更は成功している

```
npm run deploy

```

# `firebase serve` を実行してローカルで実行できるか確認する  
`http://localhost:5000` にアクセスして `Nuxt.js` の画面がでるか確認する

```
firebase serve
```

# `firebase deploy` を実行
プロジェクト トップ ディレクトリ で deploy を実行してみる

```
firebase deploy
firebase open
```

