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
  - enp0s3: 10.0.2.15
  - enp0s8: 192.168.56.200
  - memory: 4096
  - cpus: 2
    
- WorkerNode1:
  - box: bento/ubuntu-24.04
  - Hostname: wk01
  - enp0s3: 10.0.2.16
  - enp0s8: 192.168.56.201
  - memory: 8192
  - cpus: 2
 
- WorkerNode2:
  - box: bento/ubuntu-24.04
  - Hostname: wk02
  - enp0s3: 10.0.2.17
  - enp0s8: 192.168.56.202
  - memory: 8192
  - cpus: 2

## 事前インストール

- VirtualBox
- Vagrant

## Vagrantによる仮想サーバ構築

```
mkdir Vagrant
cd Vagrant
```

以下のVagrantfileを作成する。

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("3") do |config|
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
    server.vm.provision "shell", run: "always", inline: <<-SHELL
        apt get update
        apt get upgrade
        apt get 
    SHELL
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
    server.vm.provision "shell", run: "always", inline: <<-SHELL
sudo ip route del default via 10.0.2.2
sudo ip route add default via 192.168.56.1
    SHELL
  end
end
```

