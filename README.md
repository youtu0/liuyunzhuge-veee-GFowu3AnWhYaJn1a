
### 闲聊


考虑了很久，打算写一篇保姆级部署从0\-1构建企业级cicd流水线，把工作上面所用到的技术点分享给大家。从最k8s，harbor，jenkins，gitlab，docker的详细部署到集成。前后端流水线的构建，发布等...如果以下内容有不足的地方，请指出，我会第一时间更正。谢谢大家。


先上一下手绘导图，大致的流程图如下：
大致的部署流程是这样的：开发人员把做好的asp.net core项目代码通过git推送到gitlab，然后Jenkins通过 gitlab webhook （前提是配置好），自动从拉取gitlab上面拉取代码下来，然后进行build，编译、生成镜像、然后把镜像推送到Harbor仓库；然后在部署的时候通过k8s拉取Harbor上面的代码进行创建容器和服务，最终发布完成，然后可以用外网访问。
![](https://img2024.cnblogs.com/blog/2829682/202412/2829682-20241210143535340-815029343.png)


当然啦，上面只是粗略的，请看下图才更加形象。
![](https://img2024.cnblogs.com/blog/2829682/202412/2829682-20241210143623880-980158125.png)


### 一、前言


K8s 集群部署有多种方式，kubeadm 是 K8s 官方提供的集群部署工具，这种方法最为常用，简单快速，适合初学者。本文就使用 kubeadm 搭建集群演示。


### 二、主机准备


本次我们搭建一套 3 个节点的 K8s 集群，操作系统使用Ubuntu 22\.04\.4 LTS，配置2核4G，ip规划如下




| 主机名 | ip地址 | **主机配置** |
| --- | --- | --- |
| master231 | 10\.0\.0\.231 | 2核，4GiB，系统盘 20GiB |
| worker232 | 10\.0\.0\.232 | 2核，4GiB，系统盘 20GiB |
| worker233 | 10\.0\.0\.233 | 2核，4GiB，系统盘 20GiB |


### 三、系统配置


##### 关闭swap分区



```


|  | swapoff -a && sysctl -w vm.swappiness=0  # 临时关闭 |
| --- | --- |
|  | sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab  # 基于配置文件关闭 |


```

##### 确保各个节点MAC地址或product\_uuid唯一



```


|  | ifconfig  ens33  | grep ether | awk '{print $2}' |
| --- | --- |
|  | cat /sys/class/dmi/id/product_uuid |
|  |  |
|  | 温馨提示: |
|  | 一般来讲，硬件设备会拥有唯一的地址，但是有些虚拟机的地址可能会重复。 |
|  | Kubernetes使用这些值来唯一确定集群中的节点。 如果这些值在每个节点上不唯一，可能会导致安装失败。 |


```

##### 检查网络节点是否互通



```


|  | 简而言之，就是检查你的k8s集群各节点是否互通，可以使用ping命令来测试。 |
| --- | --- |
|  | ping www.baidu.com -c 10 |
|  | ping master231 -c 10 |


```

##### 允许iptable检查桥接流量



```


|  | cat <<EOF | tee /etc/modules-load.d/k8s.conf |
| --- | --- |
|  | br_netfilter |
|  | EOF |


```


```


|  | cat <<EOF | tee /etc/sysctl.d/k8s.conf |
| --- | --- |
|  | net.bridge.bridge-nf-call-ip6tables = 1 |
|  | net.bridge.bridge-nf-call-iptables = 1 |
|  | net.ipv4.ip_forward = 1 |
|  | EOF |
|  |  |
|  | sysctl --system |


```

##### 修改cgroup的管理进程



```


|  | 所有节点修改cgroup的管理进程为systemd==乌班图默认不用修改 |
| --- | --- |
|  | [root@master231 ~]# docker info  | grep "Cgroup Driver:" |
|  | Cgroup Driver: systemd |
|  | [root@master231 ~]# |
|  |  |
|  | [root@worker232 ~]# docker info  | grep "Cgroup Driver:" |
|  | Cgroup Driver: systemd |
|  | [root@worker232 ~]# |
|  |  |
|  | [root@worker233 ~]# docker info  | grep "Cgroup Driver:" |
|  | Cgroup Driver: systemd |
|  | [root@worker233 ~]# |
|  | 温馨提示: |
|  | 如果不修改cgroup的管理驱动为systemd，则默认值为cgroupfs，在初始化master节点时会失败 |


```

### 四、安装k8s管理工具


kubeadm：用来初始化K8S集群的工具。
kubelet：在集群中的每个节点上用来启动Pod和容器等。
kubectl：用来与K8S集群通信的命令行工具。


##### **所有节点操作**



```


|  | 1.K8S所有节点配置软件源 |
| --- | --- |
|  | apt-get update && apt-get install -y apt-transport-https |
|  | curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - |
|  | cat <<EOF >/etc/apt/sources.list.d/kubernetes.list |
|  | deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main |
|  | EOF |
|  |  |
|  | 2.获取最新软件包信息 |
|  | apt-get update |
|  |  |
|  | 3.查看一下当前环境支持的k8s版本 |
|  | [root@master231 ~]# apt-cache madison kubeadm |
|  | kubeadm |  1.28.2-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages |
|  | kubeadm |  1.28.1-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages |
|  | kubeadm |  1.28.0-00 | https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages |
|  |  |
|  | 4.安装 kubelet kubeadm kubectl |
|  | apt-get -y install kubelet=1.23.17-00 kubeadm=1.23.17-00 kubectl=1.23.17-00 |
|  |  |
|  |  |
|  | 5.所有节点都要检查各组件版本 |
|  | kubeadm version |
|  | kubectl version |
|  | kubelet --version |


```

##### 安装docker



```


|  | 1.编写docker安装脚本 |
| --- | --- |
|  | [root@master231 docker]# cat install-docker.sh |
|  | #!/bin/bash |
|  | # auther: cherry |
|  |  |
|  | # 加载操作系统的变量，主要是ID变量。 |
|  | . /etc/os-release |
|  |  |
|  | # DOCKER_VERSION=26.1.1 |
|  | DOCKER_VERSION=20.10.24 |
|  | # DOCKER_COMPOSE_VERSION=2.27.0 |
|  | DOCKER_COMPOSE_VERSION=2.23.0 |
|  | FILENAME=docker-${DOCKER_VERSION}.tgz |
|  | DOCKER_COMPOSE_FILE=docker-compose-v${DOCKER_COMPOSE_VERSION} |
|  | URL=https://download.docker.com/linux/static/stable/x86_64 |
|  | DOCKER_COMPOSE_URL=https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-linux-x86_64 |
|  | DOWNLOAD=./download |
|  | BASE_DIR=/softwares |
|  | OS_VERSION=$ID |
|  |  |
|  |  |
|  |  |
|  |  |
|  | # 判断是否下载了docker-compose |
|  | function prepare(){ |
|  | # 判断是否下载docker-compose文件 |
|  | if [ ! -f ${DOWNLOAD}/${DOCKER_COMPOSE_FILE} ]; then |
|  | wget -T 3  -t 2 ${DOCKER_COMPOSE_URL} -O ${DOWNLOAD}/${DOCKER_COMPOSE_FILE} |
|  | fi |
|  |  |
|  | if [ $? != 0 ];then |
|  | rm -f ${DOWNLOAD}/${DOCKER_COMPOSE_FILE} |
|  | echo "不好意思，由于网络波动原因，无法下载${DOCKER_COMPOSE_URL}软件包，程序已退出!请稍后再试......" |
|  | exit 100 |
|  | fi |
|  |  |
|  | # 给脚本添加执行权限 |
|  | chmod +x ${DOWNLOAD}/${DOCKER_COMPOSE_FILE} |
|  | } |
|  |  |
|  |  |
|  | # 定义安装函数 |
|  | function InstallDocker(){ |
|  |  |
|  | if [ $OS_VERSION == "centos" ];then |
|  | [ -f /usr/bin/wget ] || yum -y install wget |
|  | rpm -qa |grep bash-completion || yum -y install bash-completion |
|  | fi |
|  |  |
|  | if [ $OS_VERSION == "ubuntu" ];then |
|  | [ -f /usr/bin/wget ] || apt -y install wget |
|  | fi |
|  |  |
|  | # 判断文件是否存在，若不存在则下载软件包 |
|  | if [ ! -f ${DOWNLOAD}/${FILENAME} ]; then |
|  | wget ${URL}/${FILENAME} -O ${DOWNLOAD}/${FILENAME} |
|  | fi |
|  |  |
|  | # 判断安装路径是否存在 |
|  | if [ ! -d ${BASE_DIR} ]; then |
|  | install -d ${BASE_DIR} |
|  | fi |
|  |  |
|  | # 解压软件包到安装目录 |
|  | tar xf ${DOWNLOAD}/${FILENAME} -C ${BASE_DIR} |
|  |  |
|  | # 安装docker-compose |
|  | prepare |
|  | cp $DOWNLOAD/${DOCKER_COMPOSE_FILE} ${BASE_DIR}/docker/docker-compose |
|  |  |
|  | # 创建软连接 |
|  | ln -svf ${BASE_DIR}/docker/* /usr/bin/ |
|  |  |
|  | # 自动补全功能 |
|  | cp $DOWNLOAD/docker /usr/share/bash-completion/completions/docker |
|  | source /usr/share/bash-completion/completions/docker |
|  |  |
|  | # 配置镜像加速 |
|  | install -d /etc/docker |
|  | cp $DOWNLOAD/daemon.json /etc/docker/daemon.json |
|  |  |
|  | # 开机自启动脚本 |
|  | cp download/docker.service /usr/lib/systemd/system/docker.service |
|  | systemctl daemon-reload |
|  | systemctl enable --now docker |
|  | docker version |
|  | docker-compose version |
|  | tput setaf 3 |
|  | echo "安装成功,欢迎使用cherry二进制docker安装脚本，欢迎下次使用!" |
|  | tput setaf 2 |
|  | } |
|  |  |
|  |  |
|  | # 卸载docker |
|  | function UninstallDocker(){ |
|  | # 停止docker服务 |
|  | systemctl disable --now docker |
|  |  |
|  | # 卸载启动脚本 |
|  | rm -f /usr/lib/systemd/system/docker.service |
|  |  |
|  | # 清空程序目录 |
|  | rm -rf ${BASE_DIR}/docker |
|  |  |
|  | # 清空数据目录 |
|  | rm -rf /var/lib/{docker,containerd} |
|  |  |
|  | # 清除符号链接 |
|  | rm -f /usr/bin/{containerd,containerd-shim,containerd-shim-runc-v2,ctr,docker,dockerd,docker-init,docker-proxy,runc} |
|  |  |
|  | # 使得终端变粉色 |
|  | tput setaf 5 |
|  | echo "卸载成功,欢迎再次使用cherry二进制docker安装脚本哟~" |
|  | tput setaf 7 |
|  | } |
|  |  |
|  |  |
|  | # 程序的入口函数 |
|  | function main(){ |
|  | # 判断传递的参数 |
|  | case $1 in |
|  | install|i) |
|  | InstallDocker |
|  | ;; |
|  | remove|r) |
|  | UninstallDocker |
|  | ;; |
|  | *) |
|  | echo "Invalid parameter, Usage: $0 install|remove" |
|  | ;; |
|  | esac |
|  | } |
|  |  |
|  | # 向入口函数传参 |
|  | main $1 |
|  |  |
|  | [root@master231 docker]# ll |
|  | total 16 |
|  | drwxr-xr-x 3 root root 4096 Dec 10 05:49 ./ |
|  | drwx------ 6 root root 4096 Dec 10 05:49 ../ |
|  | drwxr-xr-x 2 root root 4096 May  9  2024 download/ |
|  | -rwxr-xr-x 1 root root 3497 Dec 10 05:49 install-docker.sh* |
|  |  |
|  | 2.安装docker |
|  | [root@master231 docker]# ./install-docker.sh install |
|  | '/usr/bin/containerd' -> '/softwares/docker/containerd' |
|  | '/usr/bin/containerd-shim' -> '/softwares/docker/containerd-shim' |
|  | '/usr/bin/containerd-shim-runc-v2' -> '/softwares/docker/containerd-shim-runc-v2' |
|  | '/usr/bin/ctr' -> '/softwares/docker/ctr' |
|  | '/usr/bin/docker' -> '/softwares/docker/docker' |
|  | '/usr/bin/docker-compose' -> '/softwares/docker/docker-compose' |
|  | '/usr/bin/dockerd' -> '/softwares/docker/dockerd' |
|  | '/usr/bin/docker-init' -> '/softwares/docker/docker-init' |
|  | '/usr/bin/docker-proxy' -> '/softwares/docker/docker-proxy' |
|  | '/usr/bin/runc' -> '/softwares/docker/runc' |
|  | Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /lib/systemd/system/docker.service. |
|  | Client: |
|  | Version:           20.10.24 |
|  | API version:       1.41 |
|  | Go version:        go1.19.7 |
|  | Git commit:        297e128 |
|  | Built:             Tue Apr  4 18:17:06 2023 |
|  | OS/Arch:           linux/amd64 |
|  | Context:           default |
|  | Experimental:      true |
|  |  |
|  | Server: Docker Engine - Community |
|  | Engine: |
|  | Version:          20.10.24 |
|  | API version:      1.41 (minimum version 1.12) |
|  | Go version:       go1.19.7 |
|  | Git commit:       5d6db84 |
|  | Built:            Tue Apr  4 18:23:02 2023 |
|  | OS/Arch:          linux/amd64 |
|  | Experimental:     false |
|  | containerd: |
|  | Version:          v1.6.20 |
|  | GitCommit:        2806fc1057397dbaeefbea0e4e17bddfbd388f38 |
|  | runc: |
|  | Version:          1.1.5 |
|  | GitCommit:        v1.1.5-0-gf19387a6 |
|  | docker-init: |
|  | Version:          0.19.0 |
|  | GitCommit:        de40ad0 |
|  | Docker Compose version v2.23.0 |
|  | 安装成功,欢迎使用cherry二进制docker安装脚本，欢迎下次使用! |


```

##### 初始化master组件



```


|  | 1.导入镜像 |
| --- | --- |
|  | [root@master231 ~]# docker load -i master-1.23.17.tar.gz |
|  |  |
|  | 2.使用kubeadm初始化master节点 |
|  | [root@master231 ~]# kubeadm init --kubernetes-version=v1.23.17 --image-repository registry.aliyuncs.com/google_containers  --pod-network-cidr=10.100.0.0/16 --service-cidr=10.200.0.0/16  --service-dns-domain=cherry.com |
|  | #这个要记住，添加节点需要用 |
|  | kubeadm join 10.0.0.231:6443 --token lzphw7.kc4iu4k0mswnpy7h \ |
|  | --discovery-token-ca-cert-hash sha256:298393d4dc931d6d13ec2ec1aedd4295bcd143a84e78dfc5a82ec7e53210d511 |
|  | -------------------------------------------------- |
|  |  |
|  | 3.拷贝授权文件，用于管理K8S集群 |
|  | [root@master231 ~]# mkdir -p $HOME/.kube |
|  | [root@master231 ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config |
|  | [root@master231 ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config |
|  |  |
|  | 4.查看集群节点 |
|  | [root@master231 ~]# kubectl get cs |
|  | Warning: v1 ComponentStatus is deprecated in v1.19+ |
|  | NAME                 STATUS    MESSAGE                         ERROR |
|  | scheduler            Healthy   ok |
|  | controller-manager   Healthy   ok |
|  | etcd-0               Healthy   {"health":"true","reason":""} |
|  | [root@master231 ~]# |
|  | [root@master231 ~]# |
|  | [root@master231 ~]# kubectl get nodes |
|  | NAME        STATUS     ROLES                  AGE    VERSION |
|  | master231   NotReady   control-plane,master   117s   v1.23.1 |


```

##### 部署worler组件，添加节点



```


|  | 1.导入镜像 |
| --- | --- |
|  | [root@worker232 ~]# docker load  -i slave-1.23.17.tar.gz |
|  | [root@worker233 ~]# docker load  -i slave-1.23.17.tar.gz |
|  |  |
|  | 2.在worker节点输入刚刚的token |
|  | [root@worker232 ~]# kubeadm join 10.0.0.231:6443 --token lzphw7.kc4iu4k0mswnpy7h \ |
|  | --discovery-token-ca-cert-hash sha256:298393d4dc931d6d13ec2ec1aedd4295bcd143a84e78dfc5a82ec7e53210d511 |
|  |  |
|  | [root@worker232 ~]# kubeadm join 10.0.0.231:6443 --token lzphw7.kc4iu4k0mswnpy7h \ |
|  | --discovery-token-ca-cert-hash sha256:298393d4dc931d6d13ec2ec1aedd4295bcd143a84e78dfc5a82ec7e53210d511 |
|  |  |
|  | 3.master节点检查集群的worker节点列表 |
|  | [root@master231 ~]# kubectl get nodes |
|  | NAME        STATUS     ROLES                  AGE     VERSION |
|  | master231   NotReady   control-plane,master   13m     v1.23.17 |
|  | worker232   NotReady                    3m19s   v1.23.17 |
|  | worker233   NotReady                    2m3s    v1.23.17 |


```

##### 部署CNI插件，打通网络



```


|  | 1.导入镜像 |
| --- | --- |
|  | [root@master231 ~]# docker load  -i flannel-cni-plugin-v1.5.1.tar.gz |
|  | [root@master232 ~]# docker load  -i flannel-cni-plugin-v1.5.1.tar.gz |
|  | [root@master233 ~]# docker load  -i flannel-cni-plugin-v1.5.1.tar.gz |
|  |  |
|  | [root@worker231 ~]# docker load -i flannel.tar.gz |
|  | [root@worker232 ~]# docker load -i flannel.tar.gz |
|  | [root@worker233 ~]# docker load -i flannel.tar.gz |
|  |  |
|  |  |
|  | 2.下载Flannel组件 |
|  | [root@master231 ~]# wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml |
|  |  |
|  | 3.安装Flannel组件 |
|  | [root@master231 ~]# kubectl apply -f kube-flannel.yml |
|  |  |
|  | 4.查看版本，版本要一致不然会启动失败 |
|  | [root@master231 ~]# grep image kube-flannel.yml |
|  |  |
|  | 5.检查falnnel各组件是否安装成功 |
|  | [root@master231 ~]# kubectl get pods -o wide -n kube-flannel |
|  | NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES |
|  | kube-flannel-ds-ckkbk   1/1     Running   0          35s   10.0.0.233   worker233 |
|  | kube-flannel-ds-kst7g   1/1     Running   0          35s   10.0.0.232   worker232 |
|  | kube-flannel-ds-ljktm   1/1     Running   0          35s   10.0.0.231   master231 |
|  |  |
|  | 6.测试各节点组件 |
|  | [root@master231 ~]# kubectl get nodes |
|  | NAME        STATUS   ROLES                  AGE   VERSION |
|  | master231   Ready    control-plane,master   37m   v1.23.17 |
|  | worker232   Ready                     27m   v1.23.17 |
|  | worker233   Ready                     26m   v1.23.17 |
|  |  |


```

### 五、安装kubectl工具自动补全功能



```


|  | 1.临时补全生效 |
| --- | --- |
|  | apt -y install bash-completion |
|  | source /usr/share/bash-completion/bash_completion |
|  | source <(kubectl completion bash) |
|  |  |
|  | 2.永久补全生效需要写入环境变量 |
|  | [root@master231 ~]# vim .bashrc |
|  | ... |
|  | source <(kubectl completion bash) |
|  |  |


```

### 六、修改时区



```


|  | 1.修改时区 |
| --- | --- |
|  | [root@master231 ~]# ln -svf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime |
|  |  |
|  | 2.验证 |
|  | [root@master231 ~]# date -R |


```

### 七、k8s基础巡检



```


|  | 1.检查K8S集群的worker节点列表 |
| --- | --- |
|  | [root@master231 ~]# kubectl get nodes |
|  |  |
|  | 2.检查master组件 |
|  | [root@master231 ~]# kubectl get cs |
|  |  |
|  | 3.检查flannel网卡是否正常 |
|  | [root@master231 ~]# kubectl get pods -o wide -n kube-flannel |
|  |  |
|  | 4.检查各节点网卡 |
|  | ifconfig |
|  |  |
|  | - 如果有节点没有cni0网卡，建议大家手动创建相应的网桥设备，但是注意网段要一致: |
|  | 1.假设 master231的flannel.1是10.100.0.0网段。 |
|  | ip link add cni0 type bridge |
|  | ip link set dev cni0 up |
|  | ip addr add 10.100.0.1/24 dev cni0 |
|  |  |
|  | 2.假设 worker232的flannel.1是10.100.1.0网段。 |
|  | ip link add cni0 type bridge |
|  | ip link set dev cni0 up |
|  | ip addr add 10.100.1.1/24 dev cni0 |
|  |  |
|  | 3.假设 worker233的flannel.1是10.100.2.0网段。 |
|  | ip link add cni0 type bridge |
|  | ip link set dev cni0 up |
|  | ip addr add 10.100.2.1/24 dev cni0 |


```

 本博客参考[PodHub豆荚加速器官方网站](https://rikeduke.com)。转载请注明出处！
