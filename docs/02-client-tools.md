# Client toolsのインストール

この章では、kubernetesを利用する際に使う[cfssl](https://github.com/cloudflare/cfssl)、[cfssljson](https://github.com/cloudflare/cfssl)、[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)をインストールしていきます。

## Install CFSSL

The `cfssl` and `cfssljson` command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

`cfssl`及び`cfssljson`は[PKI基盤](https://en.wikipedia.org/wiki/Public_key_infrastructure)を準備し、TLS証明書を発行します。
PKI基盤についての日本語wikiの説明は[こちら](https://ja.wikipedia.org/wiki/%E5%85%AC%E9%96%8B%E9%8D%B5%E5%9F%BA%E7%9B%A4)。
お互いのやりとりをするための鍵情報のセットを一括発行するものという認識で捉えてもらえればと思います。

以下で`cfssl` 及び `cfssljson`をインストールしていきます。

### OS X

```
curl -o cfssl https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/darwin/cfssl
curl -o cfssljson https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/darwin/cfssljson
```

```
chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

OS Xのバージョンによってはうまくいかない場合があるので、その場合は[Homebrew](https://brew.sh)を試してみてください。

```
brew install cfssl
```

### Linux

```
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssljson
```

```
chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

### インストールの確認

version 1.3.4以上の `cfssl` ならびに `cfssljson` がインストールされているかを確認します。

```
cfssl version
```

> output

```
Version: 1.3.4
Revision: dev
Runtime: go1.13
```

```
cfssljson --version
```
```
Version: 1.3.4
Revision: dev
Runtime: go1.13
```

## Install kubectl

`kubectl` はコマンドライン上でKubernetesのAPIサーバーとやり取りをするためのツールです。`kubectl`を公式ページからダウンロード並びにインストールしていきましょう。

### OS X

```
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/darwin/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Linux

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### インストールの確認

version 1.15.3以上の`kubectl` がインストールされているかを確認します。

```
kubectl version --client
```

> output

```
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning Compute Resources](03-compute-resources.md)
