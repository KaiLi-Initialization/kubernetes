# container runtime

[官方文档](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/)

## 系统必要配置

修改linux的内核采纳数，添加网桥过滤和地址转发功能

以下设置已在文档`1. 初始化系统环境.md`中的`修改linux的内核参数`时已设置。

```shell
# 编辑/etc/sysctl.d/kubernetes.conf文件，添加如下配置：

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

# 重新加载配置
[root@master ~]# sysctl -p
# 加载网桥过滤模块
[root@master ~]# modprobe br_netfilter
[root@master ~]# modprobe overlay
# 查看网桥过滤模块是否加载成功
[root@master ~]# lsmod | grep br_netfilter
[root@master ~]# lsmod | grep overlay
```

通过运行以下指令确认 `net.bridge.bridge-nf-call-iptables`、`net.bridge.bridge-nf-call-ip6tables` 和 `net.ipv4.ip_forward` 系统变量在你的 `sysctl` 配置中被设置为 1：

```shell
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## docker

### 安装docker

1、卸载旧版本

```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

2、查看当前镜像源中支持的docker版本

```shell
yum list docker-ce --showduplicates | sort -r
```

3、设置存储库

```shell
sudo yum install -y yum-utils      
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

4、安装docker-ce

```shell
方式一：安装特定版本的docker-ce
# 必须制定--setopt=obsoletes=0，否则yum会自动安装更高版本
[root@master ~]#  yum install --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7 -y

方式二：安装最新版本docker-ce
[root@master ~]# sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

5、添加一个配置文件

Docker 在默认情况下使用Vgroup Driver为cgroupfs，而Kubernetes推荐使用systemd来替代cgroupfs

```shell
mkdir /etc/docker

cat <<EOF> /etc/docker/daemon.json
{
	"exec-opts": ["native.cgroupdriver=systemd"]      
}
EOF
```

6、启动dokcer

```shell
systemctl restart docker && systemctl enable docker
```

7、查看docker运行状态

```shell
systemctl status docker
```



### cri-dockerd安装

Docker Engine 没有实现 [CRI](https://kubernetes.io/zh-cn/docs/concepts/architecture/cri/)， 而这是容器运行时在 Kubernetes 中工作所需要的。 为此，必须安装一个额外的服务 [cri-dockerd](https://github.com/Mirantis/cri-dockerd)。 cri-dockerd 是一个基于传统的内置 Docker 引擎支持的项目， 它在 1.24 版本从 kubelet 中[移除](https://kubernetes.io/zh-cn/dockershim)。

kubernetes1.24.x以上版本需要安装cri-dockerd作为网络接口

**参考文档：**https://github.com/mirantis/cri-dockerd#build-and-install

#### 1、rpm方式安装cri-dockerd

下载rpm包

```shell
yum install -y wget
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.6/cri-dockerd-0.2.6-3.el8.x86_64.rpm
```

安装cri-dockerd

```shell
yum install -y cri-dockerd-0.2.6-3.el8.x86_64.rpm

systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
```

#### 2、go编译安装cri-dockerd

**安装go环境**

```shell
wget https://storage.googleapis.com/golang/getgo/installer_linux
chmod +x ./installer_linux
./installer_linux
source ~/.bash_profile
```

**克隆cri-dockerd文件**

```shell
git clone https://github.com/Mirantis/cri-dockerd.git
```

**编译安装cri-dockerd**

```shell
cd cri-dockerd
mkdir bin
go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
```

**Kubernetes使用**（安装组件后配置）

1. Kubernetes网络插件默认是cni，需要追加**`--network-plugin=cni`**，通过该配置告诉容器，使用kubernetes的网络接口。
2. pause镜像是一切的 Pod 的基础

```shell
# go编译安装路径
vim /etc/systemd/system/cri-docker.service
# rpm安装路径
vim /usr/lib/systemd/system/cri-docker.service

[root@master ~]# cat /etc/systemd/system/cri-docker.service 
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint fd:// # 此处添加一下内容
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target

```

需要将以上两条1，2的参数追加在ExecStart后面

```shell
ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9
```

**启动cri-dockerd**

```shell
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
systemctl start cri-docker
```

**验证cri-dockerd**

```shell
[root@master ~]# systemctl status cri-docker
● cri-docker.service - CRI Interface for Docker Application Container Engine
     Loaded: loaded (/etc/systemd/system/cri-docker.service; enabled; vendor preset: disabled)
     Active: active (running) since Tue 2022-12-06 23:31:18 CST; 1min 34s ago
TriggeredBy: ● cri-docker.socket
       Docs: https://docs.mirantis.com
   Main PID: 6861 (cri-dockerd)
      Tasks: 6
     Memory: 50.3M
        CPU: 84ms
     CGroup: /system.slice/cri-docker.service
             └─6861 /usr/local/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9

Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Start docker client with request timeout 0s"
Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Hairpin mode is set to none"
Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Loaded network plugin cni"
Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Docker cri networking managed by network plugin cni"
Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Docker Info: &{ID:37LI:R2SF:5ZQM:SFBX:6KGE:KFLT:CWY5:ZF4D:LK2C:GSBR:DBJH:K5SA Contain>
Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Setting cgroupDriver systemd"
Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Docker cri received runtime config &RuntimeConfig{NetworkConfig:&NetworkConfig{PodCid>
Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Starting the GRPC backend for the Docker CRI interface."
Dec 06 23:31:18 master cri-dockerd[6861]: time="2022-12-06T23:31:18+08:00" level=info msg="Start cri-dockerd grpc backend"
Dec 06 23:31:18 master systemd[1]: Started CRI Interface for Docker Application Container Engine.
lines 1-22/22 (END)

```

表示安装完成。

## containerd

### docker方式安装

此方式安装方式参考docker作为runtime安装方式，只不过需要设置containerd作为runtime。



### 二进制安装

[安装文档](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

containerd下载地址https://github.com/containerd/containerd/releases，文件格式为：`containerd-<VERSION>-<OS>-<ARCH>.tar.gz`

#### containerd安装

下载安装文件，解压到目录`/usr/local`:

```shell
$ wget wget https://github.com/containerd/containerd/releases/download/v1.6.20/containerd-1.6.20-linux-amd64.tar.gz
$ tar Cxzvf /usr/local containerd-1.6.20-linux-amd64.tar.gz 
bin/
bin/containerd-shim-runc-v2
bin/containerd-shim
bin/ctr
bin/containerd-shim-runc-v1
bin/containerd
bin/containerd-stress
```

#### 配置systemd启动文件

创建containerd服务启动文件为`containerd.service`，存储目录为：`/usr/lib/systemd/system/containerd.service`。

containerd.service文件配置信息如下：

```shell
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
#uncomment to enable the experimental sbservice (sandboxed) version of containerd/cri integration
#Environment="ENABLE_CRI_SANDBOXES=sandboxed"
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

**安装runc**

从网站https://github.com/opencontainers/runc/releases下载安装包，安装到目录：`/usr/local/sbin/runc`

```shell
$ wget https://github.com/opencontainers/runc/releases/download/v1.1.5/runc.amd64
$ install -m 755 runc.amd64 /usr/local/sbin/runc
```

**安装cni插件**

从网站https://github.com/containernetworking/plugins/releases下载安装包，安装到目录：`/opt/cni/bin`。

```shell
$ wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

$ mkdir -p /opt/cni/bin
$ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz
./
./loopback
./bandwidth
./ptp
./vlan
./host-device
./tuning
./vrf
./sbr
./dhcp
./static
./firewall
./macvlan
./dummy
./bridge
./ipvlan
./portmap
./host-local
```

#### containerd配置

创建containerd配置文件`config.toml` 配置文件，路径为：`/etc/containerd/config.toml`.

生成containerd配置文件：config.toml

```shell
$ mkdir /etc/containerd
$ containerd config default | tee /etc/containerd/config.toml
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
temp = ""
version = 2

[cgroup]
  path = ""

[debug]
  address = ""
  format = ""
  gid = 0
  level = ""
  uid = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
  tcp_address = ""
  tcp_tls_ca = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"

  [plugins."io.containerd.grpc.v1.cri"]
    device_ownership_from_security_context = false
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    enable_selinux = false
    enable_tls_streaming = false
    enable_unprivileged_icmp = false
    enable_unprivileged_ports = false
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "registry.k8s.io/pause:3.6"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
      ip_pref = ""
      max_conf_num = 1

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      ignore_rdt_not_enabled_errors = false
      no_pivot = false
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          cni_conf_dir = ""
          cni_max_conf_num = 0
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_path = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = false

      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]

    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = "node"

    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.internal.v1.tracing"]
    sampling_ratio = 1.0
    service_name = "containerd"

  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"

  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false

  [plugins."io.containerd.runtime.v1.linux"]
    no_shim = false
    runtime = "runc"
    runtime_root = ""
    shim = "containerd-shim"
    shim_debug = false

  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
    sched_core = false

  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]

  [plugins."io.containerd.service.v1.tasks-service"]
    rdt_config_file = ""

  [plugins."io.containerd.snapshotter.v1.aufs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.btrfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.devmapper"]
    async_remove = false
    base_image_size = ""
    discard_blocks = false
    fs_options = ""
    fs_type = ""
    pool_name = ""
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.native"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.overlayfs"]
    root_path = ""
    upperdir_label = false

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

  [plugins."io.containerd.tracing.processor.v1.otlp"]
    endpoint = ""
    insecure = false
    protocol = ""

[proxy_plugins]

[stream_processors]

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar"

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar+gzip"

[timeouts]
  "io.containerd.timeout.bolt.open" = "0s"
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[ttrpc]
  address = ""
  gid = 0
  uid = 0

```

**配置 `systemd` cgroup 驱动**

[参考文档](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)

将containerd的`cgroup`设置为：`systemd`

结合 `runc` 使用 `systemd` cgroup 驱动，在 `/etc/containerd/config.toml` 中设置：

```shell
# 找到此处将：SystemdCgroup = false，修改为SystemdCgroup = true 即可
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = false
```

kubelet cgroup设置为systemd

当使用 kubeadm 时,kubelet配置文件是：`/var/lib/kubelet/config.yaml`

```shell
# 需要将cgroupDriver: "" ，设置为：cgroupDriver: "systemd"
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: "systemd"
```



设置pause镜像



```shell
# 找到此处将：sandbox_image = "registry.k8s.io/pause:3.6" 设置为符合当前k8s所需版本的pause镜像仓库（建议国内，国外有时无法访问）
 [plugins."io.containerd.grpc.v1.cri"]
    device_ownership_from_security_context = false
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    enable_selinux = false
    enable_tls_streaming = false
    enable_unprivileged_icmp = false
    enable_unprivileged_ports = false
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "registry.k8s.io/pause:3.6"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""
```

#### 设置containerd作为runtime

[参考文档](https://github.com/containerd/containerd/blob/main/docs/cri/crictl.md)

在 Linux 上，containerd 的默认 CRI 接口是 `/run/containerd/containerd.sock`。

如果没有安装crictl，需要安装一下和当前正在使用的cri插件版本兼容的crictl

指定crictl客户端连接容器时使用的接口

```shell
$ cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: true
EOF
```

#### 启动containerd服务

```shell
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl restart containerd

sudo systemctl status containerd
```



注意：

如果选择的containerd做为runtime，则需要修改kubelet的配置使用containerd做为runtime

```shell
$ cat > /etc/sysconfig/kubelet <<EOF
KUBELET_KUBEADM_ARGS=“--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock"
EOF
```

