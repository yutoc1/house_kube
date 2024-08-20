# house_kube

## 前提

- MasterNode1台
- WorkerNode2台
- VirtualBox上でUbuntuを動かす
- ランタイムはCRI-Oを使う

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
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get all -A
```

CNIのセットアップ

[公式手順](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml -O
sed -i 's,192.168.0.0/16,172.16.0.0/16,g' calico.yaml
kubectl apply -f calico.yaml
kubectl get pod -A
```

WorkerNode Join用の設定を確認する｡

```
sudo kubeadm token create --print-join-command
```

InternalIPを変更する｡

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

Calicoのルーティングを変更する｡

```bash
kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=interface=eth1
```

Helmインストール

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt install apt-transport-https --y
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update -y
sudo apt install helm -y
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

InternalIPを変更する｡

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

## Prometheus, Grafanaインストール

レポジトリの登録｡

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Grafana, Prometheusのインストール

```bash
helm install prometheus-grafana prometheus-community/kube-prometheus-stack
```

インストールの確認｡

```bash
kubectl get pods -n default
kubectl get svc -n default
```

パスワードの確認｡

```bash
kubectl get secret prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

MetalLBインストール

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb --set crds.create=true
```

MetalLB設定

```bash
kc apply -f ipaddress_pool.yaml
kc apply -f l2_advertisement.yaml
```

Nginx-ingressインストール

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx
```

Nginx-ingress設定

```bash
kubectl apply -f grafana-ingress.yml
```

ポートフォワーディング設定｡

```bash
kubectl port-forward svc/prometheus-grafana 3000:80
```
