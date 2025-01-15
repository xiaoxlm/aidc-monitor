# 说明
该文档是创建 k8s 集群的手册。同时安装 nvidia-gpu-operator 和 nvidia-network-operator。


# 创建 k8s 集群
## 一、 环境准备

1. 修改所有节点主机名本地域名解析
```shell
在每台主机修改/etc/hosts，把集群里所有主机名和IP地址映射关系放进去。
每台节点的 /etc/hosts 文件内容如下:
10.10.1.84 node1
10.10.1.85 node2
10.10.1.87 node3
10.10.1.96 aidc
```
2.  所有节点主机时钟同步
```shell
$ apt -y install chrony
$ systemctl enable chrony && systemctl start chrony
$ chronyc sources -v
```
3. 所有节点关闭swap分区
```shell
sudo swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
```
4. 所有节点禁用防火墙
```shell
ufw disable
ufw status
```
5. 所有节点启主机IP转发
```shell
$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

$ sudo modprobe overlay
$ sudo modprobe br_netfilte

$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system 

# 通过运行以下指令确认 br_netfilter 和 overlay 模块被加载
$ lsmod | grep br_netfilter
$ lsmod | grep overlay

# 运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1
$ sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

# 出现：
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```
6. 开启 ipvs
```shell
$ apt install -y ipset ipvsadm

# 内核加载 ipvs
$ cat <<EOF | sudo tee /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

$ sudo modprobe ip_vs
$ sudo modprobe ip_vs_rr
$ sudo modprobe ip_vs_wrr
$ sudo modprobe ip_vs_sh
$ sudo modprobe nf_conntrack

# 确认ipvs模块加载
$ lsmod |grep -e ip_vs -e nf_conntrack

# 出现
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 176128  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          172032  1 ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  5 nf_conntrack,btrfs,nf_tables,raid456,ip_vs
```

> 如果有安装selinux，记得关闭SELinux。
> 
>$ setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
reboot
> 
> $ rebot


## 二、 安装 docker 和 容器运行时
### 安装docker
```shell
$ apt install docker.io -y
$ systemctl enable docker
$ systemctl start docker --now
```

### 设置 nvidia-docker-runtime
   
将 nvidia-docker-runtime 配置到 /etc/docker/daemon.json
```json
{
  "runtimes": {
    "nvidia": {
      "args": [],
      "path": "/usr/local/nvidia/toolkit/nvidia-container-runtime"
    },
    "nvidia-cdi": {
      "args": [],
      "path": "/usr/local/nvidia/toolkit/nvidia-container-runtime.cdi"
    },
    "nvidia-legacy": {
      "args": [],
      "path": "/usr/local/nvidia/toolkit/nvidia-container-runtime.legacy"
    }
  }
}
```

### 安装 cri-dockerd
安装:
```shell
$ wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.14/cri-dockerd-0.3.14.amd64.tgz
$ tar -xvf cri-dockerd-0.3.14.amd64.tgz
$ cd cri-dockerd/
$ mkdir -p /usr/local/bin
$ install -o root -g root -m 0755 ./cri-dockerd /usr/local/bin/cri-dockerd
```

设置 cri-dockerd 开机启动:
```shell
$ cat > /etc/systemd/system/cri-dockerd.socket <<-EOF
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-dockerd.service    #systemd cri-dockerd.servics 文件名
 
[Socket]
ListenStream=/var/run/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker
 
[Install]
WantedBy=sockets.target
EOF


$ cat > /etc/systemd/system/cri-dockerd.service<<-EOF
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-dockerd.socket     #system cri-dockerd.socket  文件名
 
[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9
 --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --container-runtime-endpoint=unix:///var/run/cri-dockerd.sock --cri-dockerd-root-directory=/var/lib/dockershim --docker-endpoint=unix:///var/run/docker.sock --cri-dockerd-root-directory=/var/lib/docker
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
EOF

$ systemctl daemon-reload
$ systemctl enable cri-dockerd.service
$ systemctl restart cri-dockerd.service

# 禁用/移除服务
$ sudo systemctl stop cri-dockerd 
$ sudo systemctl disable cri-dockerd
```

## 三、 安装 kubectl kubeadm kubelet
1. 使用清华镜像站：
```shell
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

2. 添加apt下载源并更新:
```shell
$ echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

$ apt update
```
3. 所有节点安装 kubeadm 和 kubelet；主节点需要安装 kubectl
```shell
$ sudo apt-get install -y kubelet kubeadm kubectl

# 锁定版本，避免被系统的自动更新机制升级
$ sudo apt-mark hold kubelet kubeadm kubectl
```

## 四、 部署集群
### 查看 k8s 组件需要用到的镜像
查看版本：
```shell
$ sudo kubeadm config images list
```

### 安装 control-plane(master)
在主节点运行如下命令:
```shell
$ kubeadm init --apiserver-advertise-address=主节点ip --control-plane-endpoint=需要修改的hostname --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock --upload-certs
```
> 默认 k8s.io 源拉取会比较慢，我们在kubeadm init 时，可以指定使用阿里的镜像源: -image-repository registry.aliyuncs.com/google_containers

如果只有一个 master 节点，可以使用简化命令:
```shell
$ kubeadm init --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock --upload-certs
```

在主节点部署成功之后，按照成功界面的输出建议执行后面的步骤：
```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 安装 worker 节点
首先复制上节 kubeadm init 成功后返回的 kubeadm join 命令。 然后在其他节点用执行。
> 如果加入的是控制平面节点，记得加 --control-plane 参数
> 
> 忘了token 可以在主节点使用如下命令: kubeadm token create --print-join-comman

节点加入完毕，可以在主节点上用命令行查看集群节点状态：
```shell
$ kubectl get nodes
```
因为我们还没有安装 CNI，所以这个时候 nodes 是 NotReady 状态。

### 可能会踩的坑
1. 可能会使用默认的 containerd 作为运行时，join 的时候记得加 --cri-socket unix:///var/run/cri-dockerd.sock
2. pod 中遇到如下问题:
```shell
Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 223.5.5.5 8.8.8.8 114.114.114.114
```
修改 `/var/lib/kubelet/config.yaml` 目录的`resolvConf` 字段:
```yaml
resolvConf: /etc/resolv.conf
```
参考: https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/#known-issues

### 安装失败会退
在要回退的节点上运行:
```shell
kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock
```
卸载完成后，需要手动清理以下两项：
1. CNI文件夹：rmdir /etc/cni/net.d
2. Iptables 与 ipvs：iptables –flush && ipvsadm --clear 

### 安装 CNI
下载 Calico CNI yaml: 
```shell
$ wget https://github.com/projectcalico/calico/blob/master/manifests/calico.yaml
```
将文件中的两个环境变量，修改如下:
```yaml
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
- name: IP_AUTODETECTION_METHOD
  value: "interface=ens*"
```
部署：
```shell
kubectl apply -f calico.yaml
```
再次查看节点状态，所有节点都是 Ready 状态

# 安装 nvidia GPU-Operator 与 Network-Operator
为了可以使用 k8s 进行分布式训练。我们需要安装 nvidia gpu-operator 与 network-operator

## 将 down 状态的从驱动去掉
当network-operator只要识别到IB卡，不会区分它们的状态。所以会随机分配的时候，分到down网口。
这里我们要将 down 的IB网卡从mlx驱动里去掉。
1. 查看 ib 卡的状态
```shell
$ ibdev2netdev
mlx5_0 port 1 ==> ens3np0 (Down)
mlx5_1 port 1 ==> enp77s0np0 (Down)
mlx5_2 port 1 ==> ens9f0np0 (Up)
mlx5_3 port 1 ==> ens9f1np1 (Up)
```
2. 查看网卡对应的驱动信息
```shell
$ ethtool -i enp77s0np0

driver: mlx5_core
version: 23.10-3.2.2
firmware-version: 28.39.3560 (MT_0000000838)
expansion-rom-version:
bus-info: 0000:4d:00.0 # 这就是驱动信息
supports-statistics: yes
supports-test: yes
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: yes
```

3. 删除mlx驱动
```shell
echo 0000:4d:00.0 > /sys/bus/pci/drivers/mlx5_core/unbind
```
> 要添加回来也很简单:
> echo 0000:4d:00.0 > /sys/bus/pci/drivers/mlx5_core/bind

## 安装 Network-Operator
1. 添加 helm repository 并更新:
```shell
$ helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
$ helm repo update
```
2. 生成配置文件:
```shell
$ helm show values nvidia/network-operator --version v24.1.0 > values.yaml
```
然后按需修改 values.yaml 字段:
```yaml
nfd:
  enabled: true
sriovNetworkOperator:
  enabled: false
# NicClusterPolicy CR values:
deployCR: true
ofedDriver:
  deploy: false
 
rdmaSharedDevicePlugin:
  deploy: false
 
sriovDevicePlugin:
  deploy: true
  resources:
    - name: hostdev
      vendors: [15b3]
secondaryNetwork:
  deploy: true
  multus:
    deploy: true
  cniPlugins:
    deploy: true
  ipamPlugin:
    deploy: true
```
3. 安装:
```shell
$ helm install network-operator nvidia/network-operator \
  -n nvidia-network-operator \
  --create-namespace \
  --version v24.1.0 \
  -f ./values.yaml \
  --wait
```
安装成功后，下面的 pod 都会是 running 状态:
```shell
$ kubectl get pods -n nvidia-network-operator
NAME                                                              READY   STATUS    RESTARTS       AGE
cni-plugins-ds-fhbph                                              1/1     Running   1 (19h ago)    24h
cni-plugins-ds-wvlxg                                              1/1     Running   2 (138m ago)   24h
kube-multus-ds-gm669                                              1/1     Running   2 (138m ago)   24h
kube-multus-ds-hgg6b                                              1/1     Running   1 (19h ago)    24h
network-operator-24-1736835010-79bf89cf87-tg2qq                   1/1     Running   2 (138m ago)   24h
network-operator-24-1736835010-node-feature-discovery-gc-6thx5m   1/1     Running   1 (19h ago)    24h
network-operator-24-1736835010-node-feature-discovery-mast8cqv6   1/1     Running   2 (138m ago)   24h
network-operator-24-1736835010-node-feature-discovery-work5gntx   1/1     Running   7 (134m ago)   24h
network-operator-24-1736835010-node-feature-discovery-workw6kg8   1/1     Running   2 (138m ago)   24h
nv-ipam-controller-854585f594-bx7sb                               1/1     Running   2 (138m ago)   24h
nv-ipam-controller-854585f594-splcb                               1/1     Running   2 (19h ago)    24h
nv-ipam-node-2bdz7                                                1/1     Running   1 (19h ago)    24h
nv-ipam-node-nqz8s                                                1/1     Running   2 (138m ago)   24h
sriov-device-plugin-2fdnn                                         1/1     Running   1 (19h ago)    24h
sriov-device-plugin-dcl7m                                         1/1     Running   2 (138m ago)   24h
whereabouts-7mhdf                                                 1/1     Running   2 (138m ago)   24h
whereabouts-hctm8                                                 1/1     Running   1 (19h ago)    24h
```
4. 创建 HostDeviceNetwork
```shell
$ kubectl apply -f - <<EOF
apiVersion: mellanox.com/v1alpha1
kind: HostDeviceNetwork
metadata:
  name: dlrover-hostdevice-network
spec:
  networkNamespace: "dlrover"
  resourceName: "nvidia.com/hostdev"
  ipam: |
    {
      "type": "whereabouts",
      "range": "192.168.0.0/24"
    }
EOF
```

## 安装 nvidia-gpu-operator
使用 helm 安装:
```shell
$ helm install --wait --generate-name \
     -n gpu-operator --create-namespace \
     nvidia/gpu-operator
```
安装成功后，下面的 pod 都会是 running 状态:
```shell
$ kubectl get pods -n gpu-operator
NAME                                                              READY   STATUS      RESTARTS        AGE
gpu-feature-discovery-k7nxc                                       1/1     Running     2 (19h ago)     45h
gpu-feature-discovery-zb8jg                                       1/1     Running     3 (151m ago)    45h
gpu-operator-866659cb9b-b7jnv                                     1/1     Running     3 (151m ago)    45h
gpu-operator-v24-1736759645-node-feature-discovery-gc-68c5l67dv   1/1     Running     2 (19h ago)     45h
gpu-operator-v24-1736759645-node-feature-discovery-master-jlmxh   1/1     Running     3 (151m ago)    45h
gpu-operator-v24-1736759645-node-feature-discovery-worker-d9c2r   1/1     Running     3 (151m ago)    45h
gpu-operator-v24-1736759645-node-feature-discovery-worker-kckxs   1/1     Running     10 (147m ago)   45h
nvidia-container-toolkit-daemonset-k26zd                          1/1     Running     3 (151m ago)    45h
nvidia-container-toolkit-daemonset-r6l6z                          1/1     Running     2 (19h ago)     45h
nvidia-cuda-validator-877v9                                       0/1     Completed   0               144m
nvidia-cuda-validator-qfjr9                                       0/1     Completed   0               18h
nvidia-device-plugin-daemonset-n6qfb                              1/1     Running     3 (151m ago)    45h
nvidia-device-plugin-daemonset-nh4mb                              1/1     Running     2 (19h ago)     45h
nvidia-mig-manager-7lwls                                          1/1     Running     3 (151m ago)    45h
nvidia-mig-manager-9mf97                                          1/1     Running     2 (19h ago)     45h
nvidia-operator-validator-7rcm7                                   1/1     Running     2 (19h ago)     45h
nvidia-operator-validator-qbskl                                   1/1     Running     3 (151m ago)    45h
```

然后我们再describe 一个 node 会返现如下信息:
```shell
$ kubectl describe node node1

...
...
Capacity:
  cpu:                        224
  ephemeral-storage:          3687573632Ki
  hugepages-1Gi:              0
  hugepages-2Mi:              0
  memory:                     2113381416Ki
  nvidia.com/gpu:             8 # 有该信息，说明 gpu-operator 安装成功
  nvidia.com/hostdev:         7 # 有该信息说明 network-operator 安装成功
  pods:                       110
  rdma/rdma_shared_device_a:  0
Allocatable:
  cpu:                        224
  ephemeral-storage:          3398467853625
  hugepages-1Gi:              0
  hugepages-2Mi:              0
  memory:                     2113279016Ki
  nvidia.com/gpu:             8 # 有该信息，说明 gpu-operator 安装成功
  nvidia.com/hostdev:         7 # 有该信息说明 network-operator 安装成功
  pods:                       110
  rdma/rdma_shared_device_a:  0
...
...
```

## 重装
如果遇到问题，先使用 helm list -n namespace，查看已经安装好的 chart。看到名字后使用 helm uninstall 卸载即可。

然后再按之前的步骤安装 gpu-operator 和 network-operator