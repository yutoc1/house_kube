# house_kube

## 前提

- MasterNode1台
- WorkerNode2台
- VirtualBox上でUbuntuを動かす
- ランタイムはCRI-Oを使う

## 仮想サーバ概要

- Host
  - HostOnlyAdapter: 192.168.56.1/24
  - DHCP: 192.168.56.240/24 - 192.168.56.254/24

- ControlPlane:
  - box: bento/ubuntu-24.04
  - Hostname: cp01
  - private: 192.168.56.200/24
  - memory: 4096
  - cpus: 2
    
- WorkerNode1:
  - box: bento/ubuntu-24.04
  - Hostname: wk01
  - private: 192.168.56.201/24
  - memory: 8192
  - cpus: 2
 
- WorkerNode2:
  - box: bento/ubuntu-24.04
  - Hostname: wk02
  - private: 192.168.56.202/24
  - memory: 8192
  - cpus: 2

- IPaddress-Pool
  - addresses: 192.168.56.210/24 - 192.168.56.239/24

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

### Boxにログインする｡

```
vagrant ssh cp01
```

### サービスの稼働状態を確認する｡

```bash
sudo systemctl status crio
sudo systemctl status kubelet
```

### CLIのインストール状況を確認する｡

```bash
kubectl version --client
kubeadm version
```

### kubeadmの初期セットアップ

```bash
sudo kubeadm init \
--pod-network-cidr=10.244.0.0/16 \
--control-plane-endpoint=$(hostname -i | xargs -n1 | grep ^192.) \
--apiserver-advertise-address=$(hostname -i | xargs -n1 | grep ^192.)
```

### 設定ファイルをホームディレクトリにコピーする｡

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get all -A
```

### CNIのセットアップ

flannelのセットアップ

```bash
curl https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml -O
```

マニフェストの以下を修正する｡

```bash
        command:
        - /opt/bin/flanneld
        args:
        - --iface=eth1 #ifaceを設定
        - --ip-masq
        - --kube-subnet-mgr
```

~CNI(Calico)のセットアップ~

[公式手順](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml -O
sed -i 's,192.168.0.0/16,172.16.0.0/16,g' calico.yaml
kubectl apply -f calico.yaml
kubectl get pod -A
```

Calicoのルーティングを変更する｡

```bash
kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=interface=eth1
```

### GitHubとの連携

SSHの鍵を作成する｡

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/github
cat << EOF > .ssh/config
Host github.com
    IdentityFile ~/.ssh/github
    User git
EOF
cat .ssh/github.pub
```

GitHubに鍵を登録する｡

`Settings > SSH and GPG keys`

Git clone

```bash
git clone git@github.com:yutoc1/house_kube.git
```

### WorkerNode Join用の設定を確認する｡

```
sudo kubeadm token create --print-join-command
```

### ~InternalIPを変更する｡~

以下の対応は不要｡

```bash
sudo vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# 変更前
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# 変更後
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml --node-ip=192.168.56.200"
# サービスを再起動
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## WorkerNodeの初期設定

### boxにログインする｡

```
vagrant ssh wkxx
```

### サービスの稼働状態を確認する｡

```bash
sudo systemctl status crio
sudo systemctl status kubelet
```

### CLIのインストール状況を確認する｡

```bash
kubectl version --client
kubeadm version
```

### kubeadmの初期セットアップ

```bash
sudo kubeadm join 10.x.x.x:6443 --token xxxx --desicovery-token-ca-cert-hash sha256:xxxx
```

### ~InternalIPを変更する｡~

```
sudo vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# 変更前
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# 変更後
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml --node-ip=192.168.56.201"
# サービスを再起動
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### PV用のディレクトリ作成

```bash
sudo mkdir -p /mnt/disks/pv01
sudo chmod 777 /mnt/disks/pv01
```

## クラスタの初期化

Contol Planeの初期化｡

```bash
sudo kubeadm reset
sudo iptables -F
sudo iptables -F -t nat
sudo iptables -F -t mangle
sudo iptables -X
sudo rm -r /etc/cni/net.d
```

Worker Nodeの初期化｡
Control Planeで以下を実行する｡

```bash
kubectl delete node <ワーカーノード名>
```

## 動作確認

```bash
kubectl run nginx --image=nginx --port=80
kubectl get pod -o wide
kubectl exec nginx -- nginx -v
kubectl expose pod nginx --type=NodePort
IP=`kubectl get node wk01 -o=jsonpath='{.status.addresses[?(@.type == "InternalIP")].address}'`
PORT=`kubectl get svc nginx -o yaml -o=jsonpath='{.spec.ports[0].nodePort}'`
curl -I http://$IP:$PORT
```

## Helmインストール

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt install apt-transport-https
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update -y
sudo apt install helm -y
```

## Storage設定

```bash
cd /house_kube/manifest
kubectl apply -f storage/local-storage.yaml
kubectl apply -f storage/local-pv-wk01-pv01.yaml
kubectl apply -f storage/local-pv-wk02-pv01.yaml
```

## Ingress設定

MetalLBインストール

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb --set crds.create=true
```

MetalLB設定

```bash
kubectl apply -f metallb/ipaddress_pool.yaml
kubectl apply -f metallb/l2_advertisement.yaml
```

## Grafanaインストール

```bash
kubectl create namespace grafana
kubectl apply -f grafana.yaml --namespace=grafana
```

デプロイ状況を確認する｡

```bash
kubectl get pvc --namespace=grafana -o wide
kubectl get deployments --namespace=grafana -o wide
kubectl get svc --namespace=grafana -o wide
```

Webコンソールへログインする｡

`https://<EXTERNAL-IP>:3000`

admin:adminでログイン可能｡初期ログイン時にパスワード変更が必要｡
