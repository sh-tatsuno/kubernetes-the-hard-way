# Prerequisites

## Google Cloud Platform

このチュートリアルでは[Google Cloud Platform](https://cloud.google.com/)を使って簡単に0からkubernetesのクラスタを構築していきます。[Sign up](https://cloud.google.com/free/)すれば年間$300無料で使えることができます。[calculator](https://cloud.google.com/products/calculator/#id=55663256-c384-449c-9306-e39893e23afb)を使うと、このチュートリアルは1時間$0.23、1日$5.46ほどかかります。

> このチュートリアルはGCPのTrialの枠を超えて利用するのでご注意ください。

## Google Cloud Platform SDK

### Install the Google Cloud SDK

まず最初に[Google Cloud SDK](https://cloud.google.com/sdk/)をインストールし、`gcloud` コマンドを使えるように設定してください。

Google Cloud SDKのバージョンが262.0.0以上であることを確認してください。

```
gcloud version
```

### Set a Default Compute Region and Zone

この章ではデフォルトのリージョンとゾーンを設定します。

もしgcloudコマンドを利用するのがはじめてな場合は`init`を使って初期化してください。

```
gcloud init
```

そしてgcloudコマンドで認証して、GCPにアクセスできることを確認してください。

```
gcloud auth login
```

次にデフォルトのリージョンとゾーンを設定します。

```
gcloud config set compute/region us-west1
```

デフォルトのゾーンをもう一つ設定します。

```
gcloud config set compute/zone us-west1-c
```

> `gcloud compute zones list` のコマンドを使うことで利用可能なリージョンとゾーンのリストを見ることができます。

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) は複数のインスタンスで同時にコマンドを実行することができます。このチュートリアルでは
複数のインスタンスにまたがってコマンドを実行する必要があります。そのため、tmuxを使ってwindowを分割してよりこの辺りの操作を簡単に行う事ができます。

> tmuxの利用は必須ではありません。

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` -> `shift+:`で同期的にパネルを利用することができます。その後に `set synchronize-panes on` と入力してください。同期をオフにするには `set synchronize-panes off` を入力してください。

Next: [Installing the Client Tools](02-client-tools.md)
