# 高可用组件安装

安装HAProxy和KeepAlived

```shell
yum install keepalived haproxy -y

```

所有Master节点配置HAProxy（详细配置参考HAProxy文档，所有Master节点的HAProxy配置相同）：

```shell
mkdir /etc/haproxy
vim /etc/haproxy/haproxy.cfg 
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

frontend monitor-in
  bind *:33305
  mode http
  option httplog
  monitor-uri /monitor

frontend k8s-master
  bind 0.0.0.0:16443
  bind 127.0.0.1:16443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master

backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-master01	192.168.1.19:6443  check    # 所有master节点IP地址都需要写入 
  server k8s-master02	192.168.1.18:6443  check
  server k8s-master03	192.168.1.20:6443  check

```

启动HAProxy

```shell
systemctl start haproxy && systemctl enable haproxy
```

查看HAProxyz状态及端口：`16443`是否被监听

```shell
systemctl status haproxy
netstat -lntp
```



keepalived配置文件（每台master节点配置不同，需要修改）

```shell
mkdir /etc/keepalived
vim /etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 2
    weight -5
    fall 3  
    rise 2
}
vrrp_instance VI_1 {
    state MASTER
    interface ens160               # 对应本机网卡名称
    mcast_src_ip 192.168.1.19      # master节点本地地址
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.1.100           # VIP地址，需要和主机的IP地址处于同一个网段 	
    }
#    track_script {
#       chk_apiserver
#    }
}

```

所有节点配置keepalived监控检查

```shell
 cat /etc/keepalived/check_apiserver.sh 
#!/bin/bash

err=0
for k in $(seq 1 5)
do
    check_code=$(pgrep kube-apiserver)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 5
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi

# 给予执行权限
chmod +x /etc/keepalived/check_apiserver.sh 
```

启动KeepAlived

```shell
$ systemctl start keepalived
$ systemctl enable keepalived
```

测试VIP

所有master节点是否可以访问vip地址及端口

```shell
ping vip地址
telnet vip地址:16443
```



# 安装Kubernetes组件

```powershell
# 1、由于kubernetes的镜像在国外，速度比较慢，这里切换成国内的镜像源
# 2、编辑/etc/yum.repos.d/kubernetes.repo,添加下面的配置
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgchech=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
			http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 3、安装kubeadm、kubelet和kubectl
指定版本
[root@master ~]# yum install --setopt=obsoletes=0 kubeadm-1.17.4-0 kubelet-1.17.4-0 kubectl-1.17.4-0 -y

下载最新版本：
[root@master ~]# yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# 4、配置kubelet的cgroup
#编辑/etc/sysconfig/kubelet, 添加下面的配置
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"

# 5、设置kubelet开机自启
[root@master ~]# systemctl enable kubelet
```

###  准备集群镜像

```powershell
# 在安装kubernetes集群之前，必须要提前准备好集群需要的镜像，所需镜像可以通过下面命令查看
[root@master ~]# kubeadm config images list

kubeadm config images list --kubernetes-version v1.25.4

registry.k8s.io/kube-apiserver:v1.26.0
registry.k8s.io/kube-controller-manager:v1.26.0
registry.k8s.io/kube-scheduler:v1.26.0
registry.k8s.io/kube-proxy:v1.26.0
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.6-0
registry.k8s.io/coredns/coredns:v1.9.3

#查看国内镜像
kubeadm config images list --kubernetes-version v1.25.4 --image-repository registry.aliyuncs.com/google_containers

registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.0
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.26.0
registry.aliyuncs.com/google_containers/kube-scheduler:v1.26.0
registry.aliyuncs.com/google_containers/kube-proxy:v1.26.0
registry.aliyuncs.com/google_containers/pause:3.9
registry.aliyuncs.com/google_containers/etcd:3.5.6-0
registry.aliyuncs.com/google_containers/coredns:v1.9.3

推荐使用以下方式（此处是kubenetes v1.25.4版本为例）
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.0
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.0  registry.k8s.io/kube-apiserver:v1.26.0
docker rmi registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.0

docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.26.0
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.26.0  registry.k8s.io/kube-controller-manager:v1.26.0
docker rmi registry.aliyuncs.com/google_containers/kube-controller-manager:v1.26.0

docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.26.0
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.26.0 registry.k8s.io/kube-scheduler:v1.26.0
docker rmi registry.aliyuncs.com/google_containers/kube-scheduler:v1.26.0

docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.26.0
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.26.0 registry.k8s.io/kube-proxy:v1.26.0
docker rmi registry.aliyuncs.com/google_containers/kube-proxy:v1.26.0

docker pull registry.aliyuncs.com/google_containers/pause:3.9
docker tag registry.aliyuncs.com/google_containers/pause:3.9 registry.k8s.io/pause:3.9
docker rmi registry.aliyuncs.com/google_containers/pause:3.9

docker pull registry.aliyuncs.com/google_containers/etcd:3.5.6-0
docker tag registry.aliyuncs.com/google_containers/etcd:3.5.6-0 registry.k8s.io/etcd:3.5.6-0
docker rmi registry.aliyuncs.com/google_containers/etcd:3.5.6-0

docker pull registry.aliyuncs.com/google_containers/coredns:v1.9.3
docker tag registry.aliyuncs.com/google_containers/coredns:v1.9.3 registry.k8s.io/coredns/coredns:v1.9.3
docker rmi registry.aliyuncs.com/google_containers/coredns:v1.9.3

```

### 集群初始化

>下面的操作只需要在master节点上执行即可

```powershell
# 创建集群
[root@master ~]# kubeadm init \
	--apiserver-advertise-address=192.168.10.130 \
	--image-repository registry.aliyuncs.com/google_containers \
	--kubernetes-version=v1.25.4 \
	--service-cidr=10.96.0.0/12 \
	--pod-network-cidr=10.244.0.0/16 \
	--cri-socket unix://var/run/cri-dockerd.sock \
	--ignore-preflight-errors=Numcpu

```

显示以下内容为安装完成

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
#  master节点创建必要文件
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:
#  master节点运行
  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:
#  node节点运行
kubeadm join 192.168.10.130:6443 --token oat3ii.jimyx3sob5cpi15v \
	--discovery-token-ca-cert-hash sha256:fa442bdfc24302c2e355a2e2462a5972db5492f73312c77169d11805941b88df 

```

重置再初始化

```shell
kubeadm reset

rm -fr ~/.kube/  /etc/kubernetes/* var/lib/etcd/*

[root@master ~]# kubeadm init \
	--apiserver-advertise-address=192.168.10.130 \
	--image-repository registry.aliyuncs.com/google_containers \
	--kubernetes-version=v1.25.4 \
	--service-cidr=10.96.0.0/12 \
	--pod-network-cidr=10.244.0.0/16 \
	--cri-socket unix://var/run/cri-dockerd.sock \
	--ignore-preflight-errors=Numcpu
```



> 下面的操作只需要在node节点上执行即可（node节点加入集群）

```powershell
# 根据提示把worker节点加进master节点，复制你们各自在日志里的提示，然后分别粘贴在2个worker节点上，最后回车即可（注意要在后面加上--cri-socket unix:///var/run/cri-dockerd.sock这一参数，不然可能会失败）

以node01节点为例：
[root@node01 ~]# kubeadm join 192.168.10.130:6443 --token oat3ii.jimyx3sob5cpi15v --discovery-token-ca-cert-hash sha256:fa442bdfc24302c2e355a2e2462a5972db5492f73312c77169d11805941b88df --cri-socket unix:///var/run/cri-dockerd.sock
[preflight] Running pre-flight checks
	[WARNING FileExisting-tc]: tc not found in system path
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```

在master上查看节点信息

```powershell
[root@master ~]# kubectl get nodes
NAME    STATUS   ROLES     AGE   VERSION
master  NotReady  master   6m    v1.17.4
node1   NotReady   <none>  22s   v1.17.4
node2   NotReady   <none>  19s   v1.17.4
```

### 安装网络插件，只在master节点操作即可

```shell
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

等待它安装完毕稍等片刻， 发现集群的状态已经是Ready

![img](C:/Users/李凯/Desktop/kubernetes入门笔记(2)/kubernetes入门笔记/Kubenetes.assets/安装-03)

### kubeadm中的命令

```powershell
# 生成 新的token
[root@master ~]# kubeadm token create --print-join-command
```

### k8s-service代理模式IPVS

kubeadm方式修改ipvs模式：

```shell
[root@master ~]# kubectl edit configmap kube-proxy -n kube-system

    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    mode: ""             #  此处修改为 mode: “ipvs“
    nodePortAddresses: null

# 删除已经在运行的kube-pory代理，系统会自动生成新的代理

[root@master ~]# kubectl get pods -n kube-system -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
coredns-c676cc86f-b8cqv          1/1     Running   0          20h   10.244.0.2     master   <none>           <none>
coredns-c676cc86f-lwhs4          1/1     Running   0          20h   10.244.0.3     master   <none>           <none>
etcd-master                      1/1     Running   0          20h   172.31.0.35    master   <none>           <none>
kube-apiserver-master            1/1     Running   0          20h   172.31.0.35    master   <none>           <none>
kube-controller-manager-master   1/1     Running   0          20h   172.31.0.35    master   <none>           <none>
kube-proxy-4glqf                 1/1     Running   0          20h   172.31.0.245   node02   <none>           <none>
kube-proxy-qbz5b                 1/1     Running   0          20h   172.31.0.94    node01   <none>           <none>
kube-proxy-vlxpl                 1/1     Running   0          20h   172.31.0.35    master   <none>           <none>
kube-scheduler-master            1/1     Running   0          20h   172.31.0.35    master   <none>           <none>

[root@master ~]# kubectl delete pod kube-proxy-4glqf -n kube-system
[root@master ~]# kubectl delete pod kube-proxy-qbz5b -n kube-system
[root@master ~]# kubectl delete pod kube-proxy-vlxpl -n kube-system

# 再次查看发现有新的kube-pory代理生成
[root@master ~]# kubectl get pods -n kube-system -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
coredns-c676cc86f-b8cqv          1/1     Running   0          20h   10.244.0.2     master   <none>           <none>
coredns-c676cc86f-lwhs4          1/1     Running   0          20h   10.244.0.3     master   <none>           <none>
etcd-master                      1/1     Running   0          20h   172.31.0.35    master   <none>           <none>
kube-apiserver-master            1/1     Running   0          20h   172.31.0.35    master   <none>           <none>
kube-controller-manager-master   1/1     Running   0          20h   172.31.0.35    master   <none>           <none>
kube-proxy-2pnw9                 1/1     Running   0          11s   172.31.0.94    node01   <none>           <none>
kube-proxy-8jff8                 1/1     Running   0          3s    172.31.0.35    master   <none>           <none>
kube-proxy-xhzfh                 1/1     Running   0          18s   172.31.0.245   node02   <none>           <none>
kube-scheduler-master            1/1     Running   0          20h   172.31.0.35    master   <none>           <none>

```

在master节点和node节点验证IPVS模式是否生效

```shell
[root@master ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 172.31.0.35:6443             Masq    1      0          0         
TCP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0         
  -> 10.244.0.3:53                Masq    1      0          0         
TCP  10.96.0.10:9153 rr
  -> 10.244.0.2:9153              Masq    1      0          0         
  -> 10.244.0.3:9153              Masq    1      0          0         
UDP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0         
  -> 10.244.0.3:53                Masq    1      0          0         
[root@master ~]# 

[root@node01 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 172.31.0.35:6443             Masq    1      0          0         
TCP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0         
  -> 10.244.0.3:53                Masq    1      0          0         
TCP  10.96.0.10:9153 rr
  -> 10.244.0.2:9153              Masq    1      0          0         
  -> 10.244.0.3:9153              Masq    1      0          0         
UDP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0         
  -> 10.244.0.3:53                Masq    1      0          0         
[root@node01 ~]# 

```



# 集群初始化

生成kubeadm初始化配置文件

```shell
[root@master ~]# kubeadm config print init-defaults
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4           # 此处修改为master节点的IP地址
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock   # 此处指定使用的container runtime的sock路径
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io      # 此处为下载镜像的仓库地址
kind: ClusterConfiguration
kubernetesVersion: 1.26.0       # 修改为所需要的K8S版本号
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}

# 或者生成配置文件存储在文件：kubeadm.yaml中
[root@master ~]#  kubeadm config print init-defaults > kubeadm.yaml
```

下载集群所需组件镜像

```shell
# 查看所需要的镜像
[root@master ~]# kubeadm config images list
registry.k8s.io/kube-apiserver:v1.26.3
registry.k8s.io/kube-controller-manager:v1.26.3
registry.k8s.io/kube-scheduler:v1.26.3
registry.k8s.io/kube-proxy:v1.26.3
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.6-0
registry.k8s.io/coredns/coredns:v1.9.3


[root@master ~]# kubeadm config images pull --config /root/kubeadm.yaml
```

初始化集群

```shell
[root@master ~]# kubeadm init --config /root/kubeadm.yaml  --upload-certs
```



报错日志查看

一般错误会以E开头

```shell
tail -f /var/log/messages
```

初始化报错

```shell
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
```

