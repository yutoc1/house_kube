# house_kube

## ToDO

- TODO Prometheusインストール
- TODO ArgoCDインストール

## 前提

- MasterNode1台
- WorkerNode2台
- VirtualBox上でUbuntuを動かす
- ランタイムはCRI-Oを使う
- CNIはFlannelを利用する
- 設定後はcode-server上で作業する｡

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

SSH鍵の作成で止まった場合は､VirtualBox上で対象のサーバを選択すると処理が進む｡

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

マニフェストをApplyする｡

```bash
kubectl apply -f kube-flannel.yml
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

### PV用のディレクトリ作成

```bash
sudo mkdir -p /mnt/disks/pv{01..03}
sudo chmod 777 /mnt/disks/pv{01..03}
sudo mkdir -p /mnt/disks/code
sudo chmod 777 /mnt/disks/code
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
cd ~/house_kube/manifest
kubectl apply -f storage -R
```

## LB設定

MetalLBインストール

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb --set crds.create=true
```

MetalLB設定

```bash
cd ~/house_kube/manifest
kubectl apply -f metallb/ipaddress_pool.yaml
kubectl apply -f metallb/l2_advertisement.yaml
```

## Ingress設定

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
kubectl get pod -n default
```

## Grafanaインストール

```bash
cd ~/house_kube/manifest
kubectl create namespace grafana
kubectl apply -f grafana/grafana.yaml --namespace=grafana
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

## Code-Serverインストール

証明書作成

```bash
mkdir certs
cd certs
# 自己署名証明書の作成
## 秘密鍵の作成
openssl ecparam -out ca.key -name prime256v1 -genkey
## CSRの作成
openssl req -new -sha256 -key ca.key -out ca.csr -subj "/C=JP/ST=Chiba/O=myorg/CN=code-ca"
## ルート証明書の作成
openssl x509 -req -sha256 -days 36500 -in ca.csr -signkey ca.key -out ca.crt

# code-server用の証明書の作成
## 秘密鍵の作成
openssl ecparam -out codeserver.key -name prime256v1 -genkey
## CSRの作成
openssl req -new -sha256 -key codeserver.key -out codeserver.csr -subj "/C=JP/ST=Chiba/O=myorg/CN=code-server"
## SANの作成
cat <<EOF > subjectnames.txt
subjectAltName=DNS:pub.code.loc
EOF
## 証明書の作成
openssl x509 -req -sha256 -days 36500 -in codeserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out codeserver.crt -extfile subjectnames.txt
## 証明書の表示
openssl x509 -in codeserver.crt -text -noout

# DERファイル変換
openssl x509 -outform der -in ca.crt -out ca.der
openssl x509 -outform der -in codeserver.crt -out codeserver.der
```

ホストへCA証明書とサーバ証明書を送信する

```powershell
cd Vagrant
vagrant ssh-config | Out-File -Encoding utf8 ssh.config
scp -F ssh.config vagrant@cp01:/home/vagrant/certs/ca.der ./
scp -F ssh.config vagrant@cp01:/home/vagrant/certs/codeserver.der ./
```

Windowsに証明書をインストールする

```
codeserver.der → 「信頼された発行元」
ca.der → 「信頼されたルート証明機関」
```

Secretに証明書を登録する

```bash
CERT_PATH=${HOME}/certs/codeserver.crt
KEY_PATH=${HOME}/certs/codeserver.key
kubectl create secret tls code-server-tls --cert=${CERT_PATH} --key=${KEY_PATH} -n default
```

helm valuesの修正

```bash
CODE_PASS=<Password>
git clone https://github.com/coder/code-server
cd code-server
sed -i '$apasswod: '${CODE_PASS} ci/helm-chart/values.yaml
```

helm valuesの修正｡Ingress用｡

```bash
vim ci/helm-chart/values.yaml

ingress:
  enabled: true
  annotations:
    kubernetes.io/tls-acme: "true"
  hosts:
    - host: pub.code.loc
      paths:
        - /
  ingressClassName: "nginx"
  tls:
    - secretName: code-server-tls
      hosts:
        - pub.code.loc
```

helm chartの適用

```bash
helm upgrade --install code-server ci/helm-chart
```

パスワードを確認する

```bash
echo $(kubectl get secret --namespace default code-server -o jsonpath="{.data.password}" | base64 --decode)
```

code-serverへログインする｡

http://<EXTERNAL-IP>:8080

Terminalを開く

kubectlインストール

```bash
sudo apt update -y
sudo apt upgrade -y
sudo apt install -y software-properties-common curl apt-transport-https

export KUBERNETES_VERSION=v1.31
export CRIO_VERSION=v1.30
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
sudo echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt update -y
sudo apt upgrade -y
sudo apt install -y kubectl helm
```

ホストから資源のコピー

```bash
# ホスト側で操作する
cd ${HOME}
CODE_SERVER=$(kubectl get pod -l app.kubernetes.io/name=code-server -n default -o jsonpath="{.items[*].metadata.name}")
HOMEDIR="/home/coder"
kubectl cp .gitconfig ${CODE_SERVER}:${HOMEDIR}/.gitconfig
kubectl cp .ssh/github ${CODE_SERVER}:${HOMEDIR}/.ssh/github
kubectl cp .ssh/github.pub ${CODE_SERVER}:${HOMEDIR}/.ssh/github.pub
kubectl cp .ssh/config ${CODE_SERVER}:${HOMEDIR}/.ssh/config
kubectl cp .kube/config ${CODE_SERVER}:${HOMEDIR}/.kube/config
```

bashrc作成

```bash
cd ${HOME}
touch .bashrc
cat << EOF > .bashrc
alias kc='kubectl'
EOF
source .bashrc
```
