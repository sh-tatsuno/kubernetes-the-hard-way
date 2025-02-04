# Generating Kubernetes Configuration Files for Authentication

この章では俗にkubeconfigsと呼ばれる[Kubernetesの設定ファイル群](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)を作成します。kubeconfigsを利用することで、KubernetesのクライアントはKubernetesAPIサーバとの認証を行うことができるようになります。

## Client Authentication Configs

この項では`controller manager`, `kubelet`, `kube-proxy`, `scheduler`それぞれのクライアントと`admin`ユーザのkubeconfig filesを作成します。

### Kubernetes Public IP Address

kubeconfig filesの作成には接続用のKubernetesAPIサーバが必要です。高可用性を実現するために、KubernetesAPIサーバが利用する外部ロードバランサに対してIPアドレスが割り当てられます。

`kubernetes-the-hard-way`の静的IPアドレスを検索します。

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

### The kubelet Kubernetes Configuration File

Kubeletsのkubeconfigファイルを作成する際にはKubeletsのnodeに対応するクライアント証明書が必要です。これによって、Kubeletsはkubernetesに認証されます [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/)。

> この後のコマンドは[Generating TLS Certificates](04-certificate-authority.md)の章に作られたSSL証明書と同じディレクトリで実行します。

それぞれのWorkerノードのkubeconfigファイルを作成します。

```
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

Results:

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

`kube-proxy` サービス用にkubeconfigファイルを作成します。

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

下記が生成されます。

```
kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

`kube-controller-manager` サービス用にkubeconfigファイルを作成します。

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

下記が生成されます。

```
kube-controller-manager.kubeconfig
```

### The kube-scheduler Kubernetes Configuration File

`kube-scheduler` サービス用にkubeconfigファイルを作成します。

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

下記が生成されます。

```
kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

`admin` ユーザー用にkubeconfigファイルを作成します。

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

下記が生成されます。

```
admin.kubeconfig
```


## Distribute the Kubernetes Configuration Files

`kubelet` 及び `kube-proxy` のkubeconfigファイルをそれぞれのworkerインスタンスにコピーします。

```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

`kube-controller-manager` 及び `kube-scheduler` のkubeconfigファイルをそれぞれのcontrollerインスタンスにコピーします。

```
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
