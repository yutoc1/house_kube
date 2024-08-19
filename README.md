# house_kube

## 前提

- MasterNode1台
- WorkerNode2台
- VirtualBox上でUbuntuを動かす
- 低レイヤーランタイムはCRI-Oを使う

## 仮想サーバ概要

- ControlPlane:
  - box: bento/ubuntu-24.04
  - Hostname: cp01
  - private: 192.168.56.200
  - memory: 4096
  - cpus: 2
    
- WorkerNode1:
  - box: bento/ubuntu-24.04
  - Hostname: wk01
  - private: 192.168.56.201
  - memory: 8192
  - cpus: 2
 
- WorkerNode2:
  - box: bento/ubuntu-24.04
  - Hostname: wk02
  - private: 192.168.56.202
  - memory: 8192
  - cpus: 2

## 事前インストール

- VirtualBox
- Vagrant

## Vagrantによる仮想サーバ構築

WSL2のインストールでVirtualMachinePlatformがEnableの場合にBoxのSSH鍵作成で停止する｡
回避方法は以下の通り､VirtualMachinePlatformをDisableに変更するか､virtualBoxで対象のVMをクリックする｡

```
Get-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform

# Enableの場合Disableに変更する
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform
Get-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform 
```

```
mkdir Vagrant
cd Vagrant
curl https://github.com/yutoc1/house_kube/blob/main/Vagrantfile
vagrant up
```

Boxの構築に失敗した場合は一度初期化して再構築する｡

```
vagrant halt
vagrant destroy <対象Box>
vagrant up
```

## ControlPlaneの初期設定

Boxにログインする｡

```
vagrant ssh cp01
```

サービスの稼働状態を確認する｡

```bash
sudo systemctl status crio
sudo systemctl status kubelet
```

CLIのインストール状況を確認する｡

```bash
kubectl version --client
kubeadm version
```

kubeadmの初期セットアップ

```bash
sudo kubeadm init --pod-network-cidr 172.16.0.0/16 --apiserver-advertise-address 192.168.56.200
```

設定ファイルをホームディレクトリにコピーする｡

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get all -A
```

CNIのセットアップ

[公式手順](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml -O
vim calico.yaml

# CALICO_IPV4POOL_CIDRのValueを172.16.0.0/16に変更する｡
# - name: CALICO_IPV4POOL_CIDR
#   value: "192.168.0.0/16"

kubectl apply -f calico.yaml
kubectl get pod -A
```

WorkerNode Join用の設定を確認する｡

```
sudo kubeadm token create --print-join-command
```

## WorkerNodeの初期設定

boxにログインする｡

```
vagrant ssh wkxx
```

サービスの稼働状態を確認する｡

```bash
sudo systemctl status crio
sudo systemctl status kubelet
```

CLIのインストール状況を確認する｡

```bash
kubectl version --client
kubeadm version
```

kubeadmの初期セットアップ

```bash
sudo kubeadm join 10.x.x.x:6443 --token xxxx --desicovery-token-ca-cert-hash sha256:xxxx
```

## クラスタの初期化

Contol Planeの初期化｡

```
sudo kubeadm reset
sudo iptables -F
sudo iptables -F -t nat
sudo iptables -F -t mangle
sudo iptables -X
sudo rm -r /etc/cni/net.d
```

Worker Nodeの初期化｡
Control Planeで以下を実行する｡

```
kubectl delete node <ワーカーノード名>
```
