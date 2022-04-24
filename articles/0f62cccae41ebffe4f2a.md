---
title: "macOS Catalina zsh環境でpyenvを使ってPython 3.9.0をインストール"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [mac, python, pyenv]
published: true
---

# Pythonの環境構築
お仕事で若者に「このPythonスクリプトちょっと動かしてみてくれや」と言われたのですが、新しいMacBookにPython 3.x系が入っていなかったため環境構築しました。

Python何もわからんマンのおじさんですが「バージョン管理は必須だよな！」ってことくらい知っているので[pyenv](https://github.com/pyenv/pyenv)を使います（ｷﾘｯ


:::message
追記：macOS Monterey（Apple M1）環境での解説記事も書きました。
:::
https://zenn.dev/unsoluble_sugar/articles/283ebf698c307c
# Mac標準インストールのPythonのバージョン確認
ちなみにmacOS Catalina 10.15.6で標準インストールされているPythonバージョンは2.7.16でした。

```
$ python --version
Python 2.7.16
```

2020年モデルのMacBook Proなのですが…そろそろ3.x系を入れてくれても良いのよ？

# pyenvのインストール
複数バージョン切り替えに対応するため、まずはpyenv入れます。ここではHomebrewを使ってインストールします。

[The Missing Package Manager for macOS (or Linux) — Homebrew](https://brew.sh/)

Homebrew未インストールの場合は、公式に記載のコマンドを叩いて入れましょう。
```shell
% /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

すでにHomebrewインストール済みなら、アップデートをかけた上でpyenvをインストール。
```shell
% brew update
% brew install pyenv
```
ちょっと時間かかると思いますが、気長に待ちましょう。


## pyenv にパスを通す

macOS Catalinaでは、シェルがbashではなくzshになっているので、パスを通すため`.zshrc`ファイルに以下の設定を追記します。

```shell:.zshrc
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
```

[Which shell startup file do I put pyenv config in? · pyenv/pyenv Wiki](https://github.com/pyenv/pyenv/wiki#which-shell-startup-file-do-i-put-pyenv-config-in)

## pyenvのバージョン確認
ターミナルを再起動して、pyenvコマンドでバージョン確認します。

```shell
% pyenv -v
pyenv 1.2.21
```

pyenvコマンドが正常に実行できていればOKです。

# バージョン一覧確認
インストール可能なPythonのバージョン一覧を見てます。

```shell
$ pyenv install --list
Available versions:
  2.1.3
  2.2.3
  2.3.7
  ...
  3.9.0
  3.9-dev
  3.10-dev
  activepython-2.7.14
  activepython-3.5.4
  activepython-3.6.0
  anaconda-1.4.0
  anaconda-1.5.0
  anaconda-1.5.1
  ...
  stackless-3.4.7
  stackless-3.5.4
  stackless-3.7.5
```

ズラズラーっと出てきますね。

# Python 3.9.0をインストール
今回は最新の安定版である`3.9.0`入れてみました。

```
% pyenv install 3.9.0
```

## バージョン確認
インストールが終わったらpyenvで切替可能なバージョン一覧を確認します。

```
$ pyenv versions
* system (set by /Users/unsoluble_sugar/.pyenv/version)
  3.9.0
```
システム標準のPythonバージョンに加えて、3.9.0が追加されました。
この時点では、システム標準で入ってるPython 2.7.16がカレントバージョンとなっています。

# カレントバージョンの切り替え
以下コマンドで3.9.0をカレントにします。ここでは`global`指定していますが、ディレクトリ単位で切り替える場合は`global`ではなく`local`を使いましょう。

```
% pyenv global 3.9.0
```

## バージョン確認

バージョンが切り変わったか確認します。

```
% pyenv versions
  system
* 3.9.0 (set by /Users/unsoluble_sugar/.pyenv/version)
```
これでPython 3.9.0が使えるようになりました。

簡単ですね。