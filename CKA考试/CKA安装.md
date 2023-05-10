## containerd

参考文档：https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#containerd-systemd

**containerd安装**

```shell
apt-get update
apt-get install containerd
```



**containerd配置**

创建containerd配置文件`config.toml` 配置文件，路径为：`/etc/containerd/config.toml`.

生成containerd配置文件：config.toml

```shell
$ mkdir /etc/containerd
$ containerd config default | tee /etc/containerd/config.toml
```

**配置 `systemd` cgroup 驱动**

```shell
# 将：SystemdCgroup = false，修改为SystemdCgroup = true 即可
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
```

**设置pause镜像**

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

更新apt源，并安装kubernetes apt仓库所

添加apt-key

```shell
sudo curl -s http://mirrors.aliyun.com/kubernetes/yum/doc/apt-key.gpg | sudo apt-key add -
```

添加kubernetes apt仓库

```shell
echo "deb http://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" >/etc/apt/sources.list.d/kubernetes.list
```

更新 `apt` 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：

```shell
sudo apt-get update
sudo apt-get install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00
sudo apt-mark hold kubelet kubeadm kubectl
```

下载初始化所需镜像

```shell
#查看当前版本所需镜像
kubeadm config images list

#下载镜像（使用参数：--kubernetes-version指定版本号）
kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers --kubernetes-version 1.24.0       
```



生成kubeadm初始化文件

```shell
kubeadm config print init-defaults > kubeadm.yaml
```



