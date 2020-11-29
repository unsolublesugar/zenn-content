---
title: "Electron完全に理解した ～黎明編～"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [electron, Windows, Mac, helloworld]
published: false
---

# Electronとは

[Electron](https://www.electronjs.org/)とは、GitHubが開発したオープンソースのフレームワークです。macOS、Windows、Linuxといったクロスプラットフォームに対応したデスクトップアプリを開発することができます。
![](https://storage.googleapis.com/zenn-user-upload/8ffofe346xj2f3csn3v8delp3ymd)

ChromiumとNode.jsを使用しているため、HTML、CSS、JavaScriptといったWeb技術を駆使して、デスクトップアプリをつくることができるのが大きな特徴のひとつです。

エンジニアにはお馴染みのVSCodeやSlackをはじめ、FigmaやTwich、Microsoft TeamsなどのデスクトップアプリにもElectronが採用されています。
![](https://storage.googleapis.com/zenn-user-upload/ekd0gvi30n5h0dngw6vt5j2jcvmp)

# WindowsでHello Worldしてみる
そんなElectronを完全に理解するために、まずはお約束のHello World入門です。本記事では、Windowsでの環境構築とアプリのインストーラー作成までの流れを書き残しておきます。

## 環境
- OS：Windows 10
- CLI：[PowerShell](https://docs.microsoft.com/ja-jp/powershell/)
- エディタ：[VSCode（Visual Studio Code）](https://azure.microsoft.com/ja-jp/products/visual-studio-code/)

Macは最初からgitも入ってますしNode.jsの導入も比較的容易なため、環境構築まわりでハマることは少ないと思いますので割愛します。

:::message
本記事の内容は2020年12月時点での情報となります。今後のアップデート等で環境構築等の手順が変わる可能性がありますので、バージョンが異なる場合は最新の公式情報をご参照ください。
:::

## はじめに（主にプログラミング初学者向け）
Qiitaの適当な記事を参考にHello Worldするのはなるべく避けましょう。日本語情報でググって上位に出てくるのが大体Qiitaの記事なのですが…公開時期やバージョン違いによる情報が混在しており、不安定な環境構築をしてしまう恐れがあります。

記事のソースがQiita記事の芋づる式で「誰も公式を見ていないのである！」といった状況が見受けられ、あまりにも地獄だったので注意喚起させていただきます。

公式の手順を踏まずに環境構築した場合、他のエンジニアからのサポートをうまく受けられないことがありますので、まずは公式情報を見にいく癖をつけましょう。

本記事内のリンクは、各ツールの公式情報やWindows開発者向けのMicrosoftドキュメントとなっていますので、不明点は参照先のサイトから辿っていくと解決も早いです。

# gitのインストール

Electronのプロジェクト作成まわりで内部的に[git](https://git-scm.com/)が使われるため、事前にインストールしておきます。Windowsでは標準でgitはインストールされていませんので、これまでgitを使ったことが無い方はこの機会に入れておきましょう。

:::message
※すでにgitがインストール済みであれば読み飛ばしてください。
:::

[gitのダウンロードページ](https://git-scm.com/downloads)からWindows版のインストーラーをダウンロードします。本記事執筆時点の最新バージョンは`2.29.2`となっていました。
![](https://storage.googleapis.com/zenn-user-upload/p8zikusrui31nvlj9ie0r2m0w9fi)
インストーラを起動して`Next`ボタンをポチポチしていきます。基本的には設定そのままで、一部選択変更が必要でしょうか。gitを使うデフォルトエディタでVSCodeを選択したり
![](https://storage.googleapis.com/zenn-user-upload/1rfi0vj5y41o136wm65dsbqyzji9)
シンボリックリンクの有効にチェックを付けておくくらいですかね。バージョンによって画面の選択肢等が変わるので、設定内容の詳細は各自ご確認ください。
![](https://storage.googleapis.com/zenn-user-upload/acr3o9cf1r3s839ujzsz8jha6mbo)

ポチポチ押していってインストールが完了したら、gitコマンドが使えるか確認します。PowerShellを開いて`git --version`コマンドを叩いて、インストールしたgitバージョンが出力されればOK。

```shell
> git --version
git version 2.29.2.windows.2
```

これでgitのインストールは完了です。

# Node.jsのインストール
Node.jsはバージョンアップが早いため、`nvm（バージョンマネージャー）`の使用をおすすめします。Microsoftのドキュメントが丁寧でしたので、こちらを参考にインストールしてみました。

[NodeJS をネイティブ Windows 上に設定する | Microsoft Docs](https://docs.microsoft.com/ja-jp/windows/nodejs/setup-on-windows)

:::message
※すでにNode.jsがインストール済みであれば読み飛ばしてください。
:::

## nvmのインストール
[nvm-windows](https://github.com/coreybutler/nvm-windows#node-version-manager-nvm-for-windows)リポジトリから最新のインストーラーをダウンロードします。
![](https://storage.googleapis.com/zenn-user-upload/8s0w223dty7me35cxcegakdqkos1)

zipファイルを展開してインストーラーを起動。nvmをインストールします。
![](https://storage.googleapis.com/zenn-user-upload/nlixmf2m1bdrd9ef43p86gg50zt4)

`nvm ls`コマンドが叩けるか確認します。

```shell
> nvm ls
No installations recognized.
```

Node.jsの最新バージョンをインストールします。
```shell
> nvm install latest
```
LTS（長期の保守運用が約束されたバージョン）の確認。
```shell
> nvm list available

|   CURRENT    |     LTS      |  OLD STABLE  | OLD UNSTABLE |
|--------------|--------------|--------------|--------------|
|    15.3.0    |   14.15.1    |   0.12.18    |   0.11.16    |
|    15.2.1    |   14.15.0    |   0.12.17    |   0.11.15    |
|    15.2.0    |   12.20.0    |   0.12.16    |   0.11.14    |
|    15.1.0    |   12.19.1    |   0.12.15    |   0.11.13    |
|    15.0.1    |   12.19.0    |   0.12.14    |   0.11.12    |
|    15.0.0    |   12.18.4    |   0.12.13    |   0.11.11    |
|   14.14.0    |   12.18.3    |   0.12.12    |   0.11.10    |
|   14.13.1    |   12.18.2    |   0.12.11    |    0.11.9    |
|   14.13.0    |   12.18.1    |   0.12.10    |    0.11.8    |
|   14.12.0    |   12.18.0    |    0.12.9    |    0.11.7    |
|   14.11.0    |   12.17.0    |    0.12.8    |    0.11.6    |
|   14.10.1    |   12.16.3    |    0.12.7    |    0.11.5    |
|   14.10.0    |   12.16.2    |    0.12.6    |    0.11.4    |
|    14.9.0    |   12.16.1    |    0.12.5    |    0.11.3    |
|    14.8.0    |   12.16.0    |    0.12.4    |    0.11.2    |
|    14.7.0    |   12.15.0    |    0.12.3    |    0.11.1    |
|    14.6.0    |   12.14.1    |    0.12.2    |    0.11.0    |
|    14.5.0    |   12.14.0    |    0.12.1    |    0.9.12    |
|    14.4.0    |   12.13.1    |    0.12.0    |    0.9.11    |
|    14.3.0    |   12.13.0    |   0.10.48    |    0.9.10    |

This is a partial list. For a complete list, visit https://nodejs.org/download/release
```

最新の安定バージョンもインストールしておきます。
```shell
> nvm install 14.15.1
```

使用するバージョンを選択。複数バージョンインストール時は`nvm use`コマンドで切り替えられます。
```shell
> nvm use 15.3.0
```

インストール済みバージョン一覧でカレントバージョンを確認します。
```shell
> nvm ls
  * 15.3.0 (Currently using 64-bit executable)
    14.15.1
```

`npm（Node Package Manager）`のバージョンも確認しておきましょう。
```shell
> npm --version
7.0.14
```
以上でNode.jsのインストールは完了です。
# Electronのインストール

`npm`コマンドを使ってElectronのインストールを行います。
[Installation | Electron](https://www.electronjs.org/docs/tutorial/installation)

```shell
> npm install electron --save-dev
```

バージョン確認。
```shell
> npx electron -v
v11.0.3
```

デフォルトのElectronアプリを起動してみます。
```shell
> npx electron
```
こんなアプリが立ち上がればOKです。
![](https://storage.googleapis.com/zenn-user-upload/hq9y6aiuk3fkuiclxxzeersczreu)

# ElectronでHello World

Hello World用に新規プロジェクトを作成します。
[Getting Started - Electron Forge](https://www.electronforge.io/)
```shell
> npx create-electron-app my-electron-app
```

プロジェクトフォルダに移動してアプリを実行。
```shell
> cd my-electron-app
> npm start
```
Hello World!
![](https://storage.googleapis.com/zenn-user-upload/klwvrhpywzn5sssvt1602fittr7s)

あ、あっさり過ぎる…

初見では環境構築までに少々手間取るかもしれませんが、Hello Worldするのに1行もコードを書いていません。これで良いのかElectron。

## Hello Worldのソースコード

ちなみにHello Worldアプリのプロジェクト構造はこんな感じになっています。Node.jsなので`node_modules`や`package.json`が含まれています。
![](https://storage.googleapis.com/zenn-user-upload/071g91k472708webem2oq3h1t2un)

画面を構成しているソースコードは、`html`、`JavaScript`、`CSS`の3つ。普通にWebサイトで見かける組み合わせですね。
```html:index.html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Hello World!</title>
    <link rel="stylesheet" href="index.css">
  </head>
  <body>
    <h1>💖 Hello World!</h1>
    <p>Welcome to your Electron application.</p>
  </body>
</html>

```

```js:index.js
const { app, BrowserWindow } = require('electron');
const path = require('path');

// Handle creating/removing shortcuts on Windows when installing/uninstalling.
if (require('electron-squirrel-startup')) { // eslint-disable-line global-require
  app.quit();
}

const createWindow = () => {
  // Create the browser window.
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
  });

  // and load the index.html of the app.
  mainWindow.loadFile(path.join(__dirname, 'index.html'));

  // Open the DevTools.
  mainWindow.webContents.openDevTools();
};

// This method will be called when Electron has finished
// initialization and is ready to create browser windows.
// Some APIs can only be used after this event occurs.
app.on('ready', createWindow);

// Quit when all windows are closed, except on macOS. There, it's common
// for applications and their menu bar to stay active until the user quits
// explicitly with Cmd + Q.
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

app.on('activate', () => {
  // On OS X it's common to re-create a window in the app when the
  // dock icon is clicked and there are no other windows open.
  if (BrowserWindow.getAllWindows().length === 0) {
    createWindow();
  }
});

// In this file you can include the rest of your app's specific main process
// code. You can also put them in separate files and import them here.
```

```css:index.css
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  margin: auto;
  max-width: 38rem;
  padding: 2rem;
}
```

モバイルアプリ開発で言えば、Titanium mobileやFirefox OSで見た構成です。懐かしさで目頭が熱くなりますね…（遠い目

というわけで、Web系のスキルがあれば何やかんやでサクッとデスクトップアプリが作れてしまうやつです。いやはやすごい時代になりましたね。
# Windowsインストーラー作成
有識者であればこの時点で完全に理解したと言えるでしょうが、せっかくのデスクトップアプリということでWindowsのインストーラー作成までやってみました。
