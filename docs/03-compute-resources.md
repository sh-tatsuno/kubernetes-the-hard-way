# Provisioning Compute Resources

Kubernetesはcontrol plane(workerを管理するためのmaster ノード)、及びコンテナが実際に動くworkerノードから構成されます。

この章では実際にGCPのリソースを使ってKubernetesのクラスタを単一の[ゾーン](https://cloud.google.com/compute/docs/regions-zones/regions-zones)に構築していきます。

> デフォルトのゾーンについては前章 [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) を参照してください。

## Networking
Kubernetesの[ネットワークモデル](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)はコンテナとノードが互いにやり取りを行うフラットなネットワーク構造になっています。

もし、無制限に通信を許可したくない場合は[network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)を使って、どれくらいのコンテナが相互に、ないしは外部ネットワークとやりとりができるかを制限することができます。

> network policiesの作成はこのチュートリアルのスコープ外とします。

### Virtual Private Cloud Network

この項ではkubernetesのクラスタをホストするための[Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC)を作成します。

`kubernetes-the-hard-way`というVPC Networkを作成します。

```
gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
```

[サブネット](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) は、Kubernetes内でprivate IPを割り振れるだけのIPアドレスの範囲を指定する必要があります。

`kubernetes`という名前のサブネットを先ほど作ったVPC ネットワーク (`kubernetes-the-hard-way`) 内に作成します。

```
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```

> `10.240.0.0/24` というIP address rangeだと、一般的には全32ビットのうち左側24ビットが固定されて自由に使えるのが8ビット`10.240.0.1/24 - 10.240.0.254/24`なので、2**8-2で最大254個インスタンスをホストすることができます。

> GCPの場合は予約アドレスが4つあるので最大インスタンス数は252個です。
 
### Firewall Rules

内部ネットワーク通信を許可するファイアウォールのルールを設定を行います

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

外部からのSSH、ICMP、HTTPSをそれぞれ許可するファイアウォールのルールを設定します。

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

> [ロードバランサ](https://cloud.google.com/compute/docs/load-balancing/network/) を利用してKubernetesのAPIサーバと操作するリモートクライアントを行います。

`kubernetes-the-hard-way` 内のファイアウォールルールを確認します。

```
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

> output

```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

### Kubernetes Public IP Address

外部のロードバランサとやり取りを行うために、KubernetesのAPIサーバに静的なIPアドレスを配置します。

```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

`kubernetes-the-hard-way` にデフォルトリージョンでstatic IPが作成されていることを確認します。

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## Compute Instances
インスタンスはdockerのコンテナランタイムの[containerd](https://github.com/containerd/containerd)をサポートしている[Ubuntu Server](https://www.ubuntu.com/server) 18.04を利用する。簡単のためにプライベートIPを固定してコンテナを作成します。

### Kubernetes Controllers

Kubernetesのcontrol plane (=master node) として3つのインスタンスを作成します。

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

### Kubernetes Workers

それぞれのworkerインスタンスはKubernetesクラスター内のCIDRブロックからのサブネットの割り当てが必要になります。コンテナのネットワーク接続のためのpodへのサブネット設定は後の章で行います。`pod-cidr` インスタンスのmetadataはサブネットを外部ネットワークに公開し、ランタイム時にインスタンスを計算するのに活用されます。

> KubernetesクラスターのCIDRブロックはkube-controller-managerで`--cluster-cidr`のフラグによって指定することができます。このチュートリアルではクラスターのCIDRブロックを254個のサブネットが利用できる`10.200.0.0/16`に設定します。

Kubernetesのworker nodesとして3つのインスタンスを作成します。

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### Verification

デフォルトのゾーンにおけるインスタンスを表示します。

```
gcloud compute instances list
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

## Configuring SSH Access

SSHはcontrollerインスタンスとworkerインスタンスをそれぞれ設定するために利用します。[connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance)に書かれている通り、はじめてインスタンスに接続する際にSSH用のkeyが作られ、GCPプロジェクト及びインスタンスのmetadataとして保存されます。

試しに `controller-0` インスタンスにSSHアクセスしてみます

```
gcloud compute ssh controller-0
```

はじめての接続の際はSSH用のkeyが作成されます。ターミナル上でパスフレーズを入れて続行します。

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

ここで、SSH用のkeyがアップロードされ、プロジェクトに保存されます。

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

SSHキーが更新された後は`controller-0`にログインできるようになります。

```
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-1042-gcp x86_64)
...

Last login: Sun Sept 14 14:34:27 2019 from XX.XXX.XXX.XX
```

`exit`と打って`controller-0`からログアウトしましょう。

```
$USER@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
