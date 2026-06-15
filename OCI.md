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
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
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
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10250 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --match multiport --dports 30000:32767 -j ACCEPT

# 以下は削除する
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
---

sudo iptables-restore < /etc/iptables/rules.v4
```

## Containerdのインストール

```bash
apt-get update && apt-get install -y curl apt-transport-https ca-certificates software-properties-common gnupg

## Docker公式のGPG鍵を追加
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

## Dockerのaptリポジトリの追加
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

## containerdのインストール
apt-get update && apt-get install -y containerd.io

# containerdの設定
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```

## kubeadm, kubelete, kubectlのインストール

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
