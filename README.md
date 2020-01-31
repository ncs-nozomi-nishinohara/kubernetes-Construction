# [kubernetes-construction](https://github.com/ncs-nozomi-nishinohara/kubernetes-Construction)

`$`マークはコピペしやすい様にあえて付けていません。

## Kubernetes + Nvidia Docker の構築

## 検証済み環境

| OS           | ARCH    | docker Ver.                              | ESXi           |
| :----------- | :------ | :--------------------------------------- | :------------- |
| Ubuntu 18.04 | amd64   | Docker version 19.03.5, build 633a0ea838 | 6.7 Custom ISO |
| Ubuntu 18.04 | ppc64le | Docker version 18.06.1-ce, build e68fc7a | None           |

ppc64le は[IBM Power System AC922 POWER9](https://www.ibm.com/jp-ja/marketplace/power-systems-ac922)で検証

コンテナイメージはそれぞれのアーキテクチャに合わせる必要があります。

## docker のインストール(docker-ce)

### 既存のアンインストール

`sudo apt-get remove docker docker-engine docker.io containerd runc`

### docker-ce のインストール

```bash:bash
# パッケージの更新
sudo apt-get update
# 必要モジュールのインストール
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

# キーの追加
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88

# リポジトリの追加
# amd64
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

# ppc64le
sudo add-apt-repository \
   "deb [arch=ppc64el] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

# パッケージの更新
sudo apt-get update

# dockerのインストール
sudo apt-get install -y docker-ce

# ユーザーの追加
sudo usermod -aG docker $USER

# dockerサービスの自動起動
sudo systemctl enable docker

# dockerサービスの起動
sudo systemctl start docker


```

補足情報

```bash:bash
# インストール可能なリスト
apt-cache madison docker-ce

# バージョンを指定してインストール
sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
```

## kubernetes インストール(Master/Worker)

特段何も記載されていない場合はMaster/Worker両方の作業です。

### リポジトリにキーの登録

```bash:bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

### リポジトリの更新

```bash:bash
sudo apt update
sudo apt install -y kubeadm
```

### スワップ OFF

```bash:bash
## 都度適応？
## /etc/fstabを削除すると泣くことになる -> なぜかdockerが立ち上がらなくなる
sudo swapoff -a
```

/etc/fstabを以下のようにコメントアウトを行うことで一旦は対応できました。

`sudo nano /etc/fstab`

```diff:/etc/fstab
- /swap.img      none    swap    sw      0       0
+ #/swap.img      none    swap    sw      0       0
```


### kubeadm でセットアップ(Worker Node)

```bash:bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

### kubernetes config の追加(Master)

```bash:bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### iptables の編集

`sudo sysctl net.bridge.bridge-nf-call-iptables=1`

これをしないと`core-dns`の pod が起動しない

### [flannel.yaml](https://raw.githubusercontent.com/ncs-nozomi-nishinohara/kubernetes-Construction/master/flannel.yaml) の追加(Master)

`kubectl apply -f flannel.yml`

### LoadBranser の構築(Master)

[metallb.yaml](https://raw.githubusercontent.com/ncs-nozomi-nishinohara/kubernetes-Construction/master/metallb.yaml)を適用する

`kubectl apply -f metallb.yaml`

[LAN 内の IP に割り振る yaml を apply する](https://raw.githubusercontent.com/ncs-nozomi-nishinohara/kubernetes-Construction/master/metallb-config.yaml)

```yaml:metallb-config.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - xxx.xxx.xxx.xxx-xxx.xxx.xxx.xxx # この部分をLAN内のIPに変更する
```

### Master Nodes にもデプロイする場合(シングルマスタノード構成の場合とか)

`kubectl taint nodes 名前空間 node-role.kubernetes.io/master:NoSchedule-`

`名前空間`部分は`kubectl get node`で取得できる`Name`を設定(Master の Name)

```bash:bash
kubectl describe node node名

# ~~ 中略 ~~
Taints:             <none>
# ~~ 中略 ~~
# 上記になっていることを確認
```

### Master へ Node を追加する(Worker)

```bash:bash
sudo kubeadm join xxx.xxx.xxx.xxx:6443 --token xxxxxx.xxxxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxx
```

Master を構築した時に最後に表示される`--token` `--discovery-token-ca-cert-hash`を設定する

### Token が不明になった場合(Master)

```bash:bash
# tokenが不明になった場合は再度発行すれば良い
kubeadm token create --print-join-command
```

### Node が追加されたか確認(Master)

```bash:bash
kubectl get nodes
NAME           STATUS   ROLES    AGE     VERSION
Master         Ready    master   3m11s   v1.17.1
Node1          Ready    <none>   2m10s   v1.17.1
```

### DashBorad(v2.0.0-rc2) のデプロイ(以下、Masterでの作業)

kubernetes リポジトリから直接デプロイする場合はクラスタ内部からしかアクセス出来ないため、
NodePort を設定した`recommended.yaml`をデプロイする。
ただし、リポジトリから直接デプロイし、NodePort を設定しても
[issue](https://github.com/kubernetes/dashboard/issues/3804) にある様なエラーが発生するためここではすでに用意している[dashboard/recommended.yaml](https://raw.githubusercontent.com/ncs-nozomi-nishinohara/kubernetes-Construction/master/dashboard/recommended.yaml)を使用する
また、その際に証明書の設定をする必要があるため、下記を実施すること

### 自己証明書の発行

```bash:bash
mkdir certs
openssl req -nodes -newkey rsa:2048 -keyout certs/dashboard.key -out certs/dashboard.csr -subj "/C=JP/ST=kubernetes/L=kubernetes/O=kubernetes/OU=kubernetes/CN=kubernetes-dashboard"
openssl x509 -req -sha256 -days 365 -in certs/dashboard.csr -signkey certs/dashboard.key -out certs/dashboard.crt

```

### すでに定義されている分を削除(直接デプロイした場合)

```bash:bash
# 削除
kubectl -n kubernetes-dashboard delete secret kubernetes-dashboard-certs
```

### 証明書を pod から使用できる様にする

- certs/dashboard.crt
- certs/dashboard.csr
- certs/dashboard.key

上記の証明書ファイルを Secret オブジェクトへ設定できる形にする(改行コードは含めず1行で設定)

```bash:bash
# base64形式で設定する必要があるのでbase64コマンドで設定する
# それぞれのファイルで実施する
# certs/dashboard.crt
# certs/dashboard.csr
# certs/dashboard.key
cat certs/dashboard.crt | base64
cat certs/dashboard.csr | base64
cat certs/dashboard.key | base64
```

[recommended.yaml](https://raw.githubusercontent.com/ncs-nozomi-nishinohara/kubernetes-Construction/master/dashboard/recommended.yaml) の編集

```yaml:recommended.yaml
---
apiVersion: v1
data:
  dashboard.crt: base64に変換された.crtファイルを記述
  dashboard.csr: base64に変換された.crsファイルを記述
  dashboard.key: base64に変換された.keyファイルを記述
kind: Secret
metadata:
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
type: Opaque
```

apply する！

`kubectl apply -f dashboard/recommended.yaml`

### ログイン用のトークンを設定

ログイン画面には token or kubeconfig が求められます。
今回は Token を用いてログインする方法を記載します。

- [token を発行するユーザーの apply](https://raw.githubusercontent.com/ncs-nozomi-nishinohara/kubernetes-Construction/master/dashboard/admin-user.yaml)

```yaml:admin-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

```bash:bash
kubectl apply -f admin-user.yaml
kubectl get secret -n kubernetes-dashboard | grep admin
  admin-user-token-xxxx             kubernetes.io/service-account-token   3
kubectl describe secret -n kubernetes-dashboard admin-user-token-xxxxx
```

表示された`token`をコピーし、

`トークン`を選択し、`トークンを入力`にペーストし、サインインを行う。

![dashboard](https://raw.githubusercontent.com/ncs-nozomi-nishinohara/kubernetes-Construction/master/dashboard/dashboard.png)

## GPUへの対応(以下、Workerでの作業)

ESXi上のUbuntuへ対応する場合は下記を設定する(シャットダウンをした状態で)

### GPUパススルーの設定

ホスト > 管理 > ハードウェア > 該当のGPUを選択し、`パススルーの切り替え`を押下 > 再起動

### PCIデバイスの追加

設定の編集からPCIデバイスの追加
GPU以外にパススルーされてるものも一緒に追加する

### ゲストOS上からGPUが認識出来るように設定

仮想マシンのオプション > 詳細 > 構成の編集に下記の設定を追加

| キー | 値 |
| :-- | :--|
| hypervisor.cpuid.v0| FALSE |

### グラボの確認

`lspci | grep -i nvidia`

正しく表示されなかった場合は情報をアップデートする

`sudo update-pciids`

## CUDAのインストール(docker:v19.03は不要)

以下のサイトから自身のGPUに適応出来るCUDAをインストールする

[CUDA toolkit](https://developer.nvidia.com/cuda-toolkit)

```bash:参考
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-ubuntu1804.pin
sudo mv cuda-ubuntu1804.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/7fa2af80.pub
sudo add-apt-repository "deb http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/ /"
sudo apt-get update
sudo apt-get -y install cuda
```
`.bashrc`にパスを追加

```bash
export PATH="/usr/local/cuda/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda/lib64:$LD_LIBRARY_PATH"
```

再起動を行う

`sudo shutdown -r now`

### NVIDIA Graphics Driver をインストールする

```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
```

推奨ドライバーをインストール

```bash
sudo apt -y install ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
```

### NVIDIA Container Toolkitのインストール

`NVIDIA Container Toolkit`をインストールする為にリポジトリに`apt`に追加

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```

`NVIDIA Container Toolkit`のインストールと`Docker`の再起動

```bash
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

再起動を行う

`sudo shutdown -r now`

インストール後の確認
ドライバーとCUDAのバージョンが確認出来る

```bash
nvidia-container-cli info

NVRM version:   440.48.02
CUDA version:   10.2

Device Index:   0
Device Minor:   0
Model:          GeForce RTX 2070
Brand:          GeForce
GPU UUID:       GPU-2318f1ba-cc45-3c5b-a000-798ad9ba8390
Bus Location:   00000000:0b:00.0
Architecture:   7.5
```

Dockerから使用可能かどうか確認

```bash
docker run --rm --gpus all -it ubuntu nvidia-smi

+-----------------------------------------------------------------------------+
| NVIDIA-SMI 440.48.02    Driver Version: 440.48.02    CUDA Version: N/A      |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce RTX 2070    Off  | 00000000:0B:00.0 Off |                  N/A |
| 29%   33C    P8    21W / 175W |      0MiB /  7982MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

### KubernetesにGPUを認識させる

`nvidia-container-runtime`をインストール(docker:v19.03で必要)

`sudo apt-get install nvidia-container-runtime`

kubernetesから使用出来るように`Docker`のデフォルトランタイムをGPUに変更

```json:/etc/docker/daemon.json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

gpu-pluginを適用(Master)

`kubectl apply -f https://raw.githubusercontent.com/ncs-nozomi-nishinohara/kubernetes-Construction/master/nvidia-device-plugin.yml`

