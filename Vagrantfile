# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    sudo apt update -y
    sudo apt upgrade -y
    sudo swapoff -a
  SHELL

  config.vm.provision "shell", inline: <<-SHELL
    sudo swapoff -a
    sudo modprobe br_netfilter
    sudo sysctl -w net.ipv4.ip_forward=1

    sudo cat <<-'EOF' > /etc/modules-load.d/k8s.conf
      overlay
      br_netfilter
EOF
    sudo modprobe overlay
    sudo modprobe br_netfilter

    sudo cat <<-'EOF' > /etc/sysctl.d/k8s.conf
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward = 1
EOF
    sudo sysctl --system

    sudo apt update -y
    sudo apt upgrade -y
    sudo apt install -y software-properties-common curl

    export KUBERNETES_VERSION=v1.31
    export CRIO_VERSION=v1.30
    sudo apt install -y software-properties-common curl
    curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    sudo echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
    sudo echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

    sudo apt update -y
    sudo apt upgrade -y
    sudo apt install -y cri-o
    sudo apt install -y containernetworking-plugins

    sudo systemctl daemon-reload
    sudo systemctl enable crio
    sudo systemctl start crio

    sudo apt install -y kubelet kubeadm kubectl
    sudo systemctl enable --now kubelet

    echo $(hostname -I | xargs -n1 | grep ^192.) $(hostname) | sudo tee -a /etc/hosts
  SHELL

  config.vm.define "cp01" do |server|
    server.vm.box = "bento/ubuntu-24.04"
    server.vm.hostname = "cp01"
    server.vm.network "private_network", ip: "192.168.56.200"   
    server.vm.synced_folder ".", "/vagrant", disabled: true
    server.vm.provider "virtualbox" do |vb|
      vb.customize [
        "modifyvm", :id,
        "--memory", "4096",
        "--cpus", "2"
      ]
    end
  end

  config.vm.define "wk01" do |server|
    server.vm.box = "bento/ubuntu-24.04"
    server.vm.hostname = "wk01"
    server.vm.network "private_network", ip: "192.168.56.201"    
    server.vm.synced_folder ".", "/vagrant", disabled: true
    server.vm.provider "virtualbox" do |vb|
      vb.customize [
        "modifyvm", :id,
        "--memory", "8192",
        "--cpus", "2"
      ]
    end
  end

  config.vm.define "wk02" do |server|
    server.vm.box = "bento/ubuntu-24.04"
    server.vm.hostname = "wk02"
    server.vm.network "private_network", ip: "192.168.56.202"    
    server.vm.synced_folder ".", "/vagrant", disabled: true
    server.vm.provider "virtualbox" do |vb|
      vb.customize [
        "modifyvm", :id,
        "--memory", "8192",
        "--cpus", "2"
      ]
    end
  end
end
