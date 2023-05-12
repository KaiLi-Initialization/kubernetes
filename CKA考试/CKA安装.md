## containerd

参考文档：https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#containerd-systemd

**系统环境配置**

1. 配置允许 iptables 转发 ipv4桥接流量

   ```shell
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   overlay
   br_netfilter
   EOF
   
   sudo modprobe overlay
   sudo modprobe br_netfilter
   
   # 设置所需的 sysctl 参数，参数在重新启动后保持不变
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward                 = 1
   EOF
   
   # 应用 sysctl 参数而不重新启动
   sudo sysctl --system
   ```

   

2. 确认 `br_netfilter` 和 `overlay` 模块被加载：

   ```shell
   lsmod | grep br_netfilter
   lsmod | grep overlay
   ```

   

3. 确认 `net.bridge.bridge-nf-call-iptables`、`net.bridge.bridge-nf-call-ip6tables` 和 `net.ipv4.ip_forward` 系统变量在你的 `sysctl` 配置中被设置为 1：

   ```shell
   sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
   ```

   

**containerd安装**

```shell
apt-get update
apt-get install containerd
```



**containerd配置**

创建containerd配置文件`config.toml` 配置文件，路径为：`/etc/containerd/config.toml`.

1. 生成containerd配置文件：`config.toml`

   ```shell
   $ mkdir /etc/containerd
   $ containerd config default | tee /etc/containerd/config.toml
   ```

   

2. **配置 `systemd` cgroup 驱动**

   ```shell
   # 将：SystemdCgroup = false，修改为SystemdCgroup = true 即可
   sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
   ```

   

3. **设置pause镜像**

   ```shell
   #将：sandbox_image = "registry.k8s.io/pause:3.6" 设置为符合当前k8s所需版本的pause镜像仓库（建议国内，国外有时无法访问）
   
   sed -i 's#registry.k8s.io/pause#registry.aliyuncs.com/google_containers/pause#g' /etc/containerd/config.toml
   ```

   

**启动containerd服务**

```shell
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl restart containerd

sudo systemctl status containerd


#  通过命令验证
ctr plugin ls
```



## kubernetes组件安装

**环境配置**

1. 关闭swap

   ```shell
   #关闭swap分区
   swapoff -a
   
   # 注释掉：/swap.img 这一行,行首加字符‘#’
   vim /etc/fstab
   ```



**安装kubernetes**

更新apt源，并安装kubernetes apt仓库所

1. 添加apt-key

   ```shell
   sudo curl -s http://mirrors.aliyun.com/kubernetes/yum/doc/apt-key.gpg | sudo apt-key add -
   ```

   

2. 添加kubernetes apt仓库

   ```shell
   echo "deb http://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" >/etc/apt/sources.list.d/kubernetes.list
   ```

   

3. 更新 `apt` 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：

   ```shell
   sudo apt-get update
   sudo apt-get install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

   

4. 下载初始化所需镜像

   ```shell
   #查看当前版本所需镜像
   kubeadm config images list
   
   #下载镜像（使用参数：--kubernetes-version指定版本号）
   kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers --kubernetes-version 1.24.0      
   ```



## 集群初始化

方式一：通过`kubeadm.yaml`初始化文件

```shell
kubeadm config print init-defaults > kubeadm.yaml
```



方式二：通过`kubeadm init`命令

```shell
# 格式
kubeadm init --apiserver-advertise-address k8s-Master的IP地址 --image-repository 仓库地址 --cri-socket "运行时接口"

#例如：
kubeadm init --apiserver-advertise-address 192.168.10.131 --image-repository registry.aliyuncs.com/google_containers --cri-socket "unix:///var/run/containerd/containerd.sock"
```

## 安装CNI

master节点执行

```shell
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

