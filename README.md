# OCI Kubernetes

## 前提

- MasterNode 1台
- WorkerNode 2台
- OCIのFreeAlways枠を活用する
  - 無料アカウントではインスタンスが起動できないため、「Pay As You Go」プランにアップグレードする
- 「請求とコスト管理」→「予算」から1USDの予算、アラートを設定しておく

## OCIの事前設定

### VCN

- Name: vcn-k8s
- IPv4 CIDR: 10.0.0.0/16
- DNSリゾルバ: vcn-k8s

### Subnet

- Name: subnet-k8s-public
- IPv4 CIDR: 10.0.0.0/24
- サブネットタイプ: リージョナル
- パブリック・サブネット

### インターネット・ゲートウェイ

- Name: internetgateway-k8s
- ルート表関連付けはしない

### ルーティング

- Default Routeにインターネット・ゲートウェイを紐づける
- 0.0.0.0/0 -> internetgateway-k8s

### セキュリティリスト

- Defaultのセキュリティリストに以下を設定する

|方向|ソースタイプ|ソース|IPプロトコル|ソースポート範囲|宛先ポート範囲|
|:--|:---------|:----|:----------|:------------|:------|
|イングレス|CIDR|10.0.0.0/16|TCP|All|6443|
|イングレス|CIDR|10.0.0.0/16|TCP|All|2379-2380|
|イングレス|CIDR|10.0.0.0/16|TCP|All|10250|
|イングレス|CIDR|10.0.0.0/16|TCP|All|10259|
|イングレス|CIDR|10.0.0.0/16|TCP|All|10257|
|イングレス|CIDR|0.0.0.0/0|TCP|All|30000-32767|

### Instance

|インスタンス名|Private IP|CPU|メモリ|ブートボリューム|イメージ|Shape|用途|
|:-----------|:---------|:----|:----|:--------------|:----|:----|:----|
|k8s-master01|10.0.0.100|2|12GB|100GB|Ubuntu24.04|Ampere(ARM)|MasterNode/操作用|
|k8s-worker01|10.0.0.101|1|6GB|50GB|Ubuntu24.04|Ampere(ARM)|WorkerNode|
|k8s-worker02|10.0.0.102|1|6GB|50GB|Ubuntu24.04|Ampere(ARM)|WorkerNode|

- Instance作成時のSSH秘密鍵は保存する

## Nodeの初期設定

- CloudShellを起動しSSH秘密鍵をアップロードしておく
- SSH鍵は`chmod 600 <SSH鍵ファイル>`で権限を適切に設定する
- AliasにSSH接続のコマンドを設定して効率化しておく

各Nodeで以下の初期設定をする

```bash
# システムアップデート
sudo apt update && sudo apt upgrade -y

# スワップの無効化
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 必要なカーネルモジュールのロード
cat <<EOF | sudo tee /etc/modules-load.d/99-kubernetes-cri.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# ブリッジネットワークを通過するパケットをiptablesの処理対象にする。
# IP フォワーディングを有効にする。
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

OCIのUbuntuではufwが非推奨のため停止し、必要なポートを解放する

**MasterNode**

```bash
# ufwの無効化
sudo systemctl disable --now ufw

# iptablesのルールを編集
sudo vim /etc/iptables/rules.v4

# ポート22の下に、以下を追加
---
-A INPUT -p tcp -m state --state NEW -m tcp --dport 6443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2379 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2380 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10250 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10257 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10259 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --match multiport --dports 30000:32767 -j ACCEPT

# 以下は削除する
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
---

sudo iptables-restore < /etc/iptables/rules.v4
```

**WorkerNode**

```bash
# ufwの無効化
sudo systemctl disable --now ufw

# iptablesのルールを編集
sudo vim /etc/iptables/rules.v4

# ポート22の下に、以下を追加
---
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10250 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --match multiport --dports 30000:32767 -j ACCEPT

# 以下は削除する
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
---

sudo iptables-restore < /etc/iptables/rules.v4
```

## Containerdのインストール

全Nodeに対して実行する

```bash
apt update && apt install -y apt-transport-https ca-certificates software-properties-common curl gpg

## Docker公式のGPG鍵を追加
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

## Dockerのaptリポジトリの追加
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

## containerdのインストール
apt update && apt install -y containerd.io

# containerdの設定
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```

## kubeadm, kubelete, kubectlのインストール

全Nodeに対して実行する

```bash
# Kubernetesのバージョンを環境変数に指定
export KUBERNETES_VERSION=v1.36

# Kubernetesの公式GPGキーを追加
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/${KUBERNETES_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Kubernetesのリポジトリを追加
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBERNETES_VERSION}/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# kubeadm、kubelet、kubectlのインストール
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# kubeletの起動
systemctl daemon-reload
systemctl restart kubelet
```

## ControlPlaneのセットアップ

### kubeadmの初期セットアップ

kubeadm initを実行する

```bash
sudo kubeadm init \
--pod-network-cidr=10.244.0.0/16 \
--control-plane-endpoint=10.0.0.100 \
--apiserver-advertise-address=10.0.0.100
```

### 設定ファイルをホームディレクトリにコピーする

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

### Join用のコマンド確認

```
sudo kubeadm token create --print-join-command
```

### NFSサーバのインストール

```bash
sudo apt install -y nfs-kernel-server
NFS_DIR=/export/nfs
sudo mkdir -p ${NFS_DIR}
sudo chown nobody:nogroup ${NFS_DIR}
sudo chmod 777 ${NFS_DIR}
sudo sed -i '$a/export 10.0.0.0/24(rw,fsid=0,insecure,no_subtree_check_async)' /etc/exports
sudo systemctl restart nfs-server
sudo exportfs -v
```

## WorkerNodeのセットアップ

### kubeadmでJoinする

```bash
sudo kubeadm join 192.168.56.200:6443 --token xxxx --desicovery-token-ca-cert-hash sha256:xxxx
```

### NFSクライアントのインストール

```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/nfs
sudo mount -v 10.0.0.100:/export/nfs /mnt/nfs
```

## 稼働確認

### Node/Podの状態確認

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

### Nginxのデプロイ確認

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
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=192.168.56.200 \
--set nfs.path=/export/nfs
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
subjectAltName=DNS:code.kube.loc
EOF
## 証明書の作成
openssl x509 -req -sha256 -days 36500 -in codeserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out codeserver.crt -extfile subjectnames.txt
## 証明書の表示
openssl x509 -in codeserver.crt -text -noout

# DERファイル変換
openssl x509 -outform der -in ca.crt -out ca.der
```

ホストへCA証明書を送信する

```powershell
cd Vagrant
vagrant ssh-config | Out-File -Encoding utf8 ssh.config
scp -F ssh.config vagrant@cp01:/home/vagrant/certs/ca.der ./
```

Windowsに証明書をインストールする

```
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

helm valuesの修正｡Storage用｡

```bash
volumePermissions:
  enabled: false # trueからfalseへ変更する
```

helm valuesの修正｡Ingress用｡

```bash
vim ci/helm-chart/values.yaml

ingress:
  enabled: true
  annotations:
    kubernetes.io/tls-acme: "true"
  hosts:
    - host: code.kube.loc
      paths:
        - /
  ingressClassName: "nginx"
  tls:
    - secretName: code-server-tls
      hosts:
        - code.kube.loc
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

http://code.kube.loc:8080

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
