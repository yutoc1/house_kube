# house_kube

## WSL2 Setting

### Windowsの機能

Windowsの機能の有効化で以下を有効化する。

- Hyper-V
- Linux用Windowsサブシステム

### WSLのリソースを設定する

ユーザフォルダ配下に`.wslconfig`を作成する。

内容を以下の通りとする。

```
[wsl2]
memory=16GB
swap=0
```

### WSL2のインストール

```
wsl --set-default-version 2
```

```
wsl --update
```

### Ubuntuのインストール

Linuxディストリビューションを一覧表示する。

```
wsl --list --online
```

Ubuntuをインストールする。

```
wsl --install -d ubuntu
```

