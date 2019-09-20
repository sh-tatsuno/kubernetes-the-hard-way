# Provisioning a CA and Generating TLS Certificates

この章ではCloudFlare's PKI toolkit、[cfssl](https://github.com/cloudflare/cfssl)を使用して[PKI基盤](https://en.wikipedia.org/wiki/Public_key_infrastructure)、認証局(Certificate Authority, CA)や、kubernetesに必要な下記コンポーネントのTLS証明書を作成します。

 etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy.

## Certificate Authority

この項ではTLS証明書を作成するための認証局(CA)を作成します。

CAの設定ファイル、証明書、秘密鍵を生成します。

```
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

下記が生成されます。

```
ca-key.pem
ca.pem
```

## Client and Server Certificates

この項では、Kubernetesの各コンポーネントのクライアント及びサーバ証明書、加えて `admin` ユーザのクライアント証明書を作成します。

### The Admin Client Certificate

`admin` ユーザのクライアント証明書と秘密鍵を作成します。

```
{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

下記が生成されます。

```
admin-key.pem
admin.pem
```

### The Kubelet Client Certificates

Kubernetesは[Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/)と呼ばれる特別な権限があり、特に[Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet)からのAPIリクエストを管理します。Node Authorizerを利用するためには、KubeletはGroup名を`system:nodes`、User名を`system:node:<nodeName>`とする必要があります。この項ではNode Authorizerを利用できるKubernetesのWorkerノードの証明書を作成します。

Kubernetesの証明書と秘密鍵を作成します。

```
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

下記が生成されます。

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### The Controller Manager Client Certificate

`kube-controller-manager`の証明書と秘密鍵を作成します。

```
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

下記が生成されます。

```
kube-controller-manager-key.pem
kube-controller-manager.pem
```


### The Kube Proxy Client Certificate

`kube-proxy`のクライアント証明書と秘密鍵を生成します。

```
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

下記が生成されます。

```
kube-proxy-key.pem
kube-proxy.pem
```

### The Scheduler Client Certificate

`kube-scheduler`の証明書と秘密鍵を生成します。

```
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

下記が生成されます。

```
kube-scheduler-key.pem
kube-scheduler.pem
```


### The Kubernetes API Server Certificate

The `kubernetes-the-hard-way` static IP address will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

`kubernetes-the-hard-way`の静的IPはKubernetesAPIサーバ証明書のSubject Alternative Name(SAN)に含まれます。

> Subject Alternative Name(SAN)は1枚の証明書で複数のウェブサイトの暗号化通信を実現する仕組みです。SSL証明書でCommon Name(CN)と呼ばれる通常のホスト名などを記載する領域の他にSANという領域を持ち、そこに追加で証明したいFQDNを記載します。

KubernetesAPIサーバの証明書と秘密鍵を発行します。

```
{

KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

> Kubernetes APIサーバは`kubernetes`という内部DNSが自動的に張られ、最初のIP (今回の場合は `10.32.0.1`)に自動で紐付けられます。詳細は後半の演習 [control plane bootstrapping](08-bootstrapping-kubernetes-controllers.) で実施します。

下記が生成されます。

```
kubernetes-key.pem
kubernetes.pem
```

## The Service Account Key Pair

Kubernetesのコントローラマネージャは[managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/)に書かれているようにサービスアカウントトークンを作成して署名します。

`service-account`の証明書と秘密鍵を作成します。

```
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

下記が生成されます。

```
service-account-key.pem
service-account.pem
```


## Distribute the Client and Server Certificates

作成した証明書と秘密鍵をそれぞれのWorkerインスタンスにコピーします。

```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```

作成した証明書と秘密鍵をそれぞれのControllerインスタンスにコピーします。

```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

> `kube-proxy`、`kube-controller-manager`、`kube-scheduler`、`kubelet` のクライアント証明書はクライアント認証の設定ファイルを作成するのに使われます。次の章で詳しく実施していきます。

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
