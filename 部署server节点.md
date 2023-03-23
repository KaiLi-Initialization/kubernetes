# 部署 kubectl



## 生成 ssl 证书

`kubectl` 作为 `kube-apiserver` 的客户端工具，需要访问 `kube-apiserver` 的服务，所以需要 `kube-apiserver` 的 `ca`
机构为其签发客户端 `client` 证书。

证书颁发已在**`3.创建证书`**中生成，目录为：`/etc/kubernetes/pki/apiserver`中的`apiserver-kubectl-client.pem`和`apiserver-kubectl-client-key.pem`。

## 创建 kubeconfig

[官方参考文档](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

> `kubectl` 是使用 `kubeconfig` 跟 `kube-apiserver` 进行通信的。
> 在 `kubeconfig` 配置文件中会包含了 `kubectl` 的客户端 `client` 证书信息以及身份信息。
> 需要在每台服务器都创建该请求文件。
> 以下操作在每台 `master` 服务器创建，`ip` 地址设置为本地的 `kube-apiserver` 的服务地址 `ip`。
> 文件存放目录为：`/etc/kubernetes/kubeconfig/`。

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/kubernetes-ca.pem \
--embed-certs=true \
--server=https://192.168.10.201:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.conf
```

设置客户端认证参数

```shell
kubectl config set-credentials kubernetes-admin \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-kubectl-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-kubectl-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.conf
```

设置上下文参数

```shell
kubectl config set-context kubernetes-admin@kubernetes \
--cluster=kubernetes \
--user=kubernetes-admin \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.conf
```

设置当前上下文

```shell
kubectl config use-context kubernetes-admin@kubernetes \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.conf
```

复制到 `k8s` 默认的引用目录位置

*一般是 `/root/.kube/` 目录*

```shell
cp /etc/kubernetes/kubeconfig/admin.conf /root/.kube/config
```

## 验证

> [!NOTE]
> 每台 `master` 服务器配置 `admin.kubeconfig` 都验证一遍。

随便执行 `kubectl` 命令查看是否可以正常访问 `kube-apiserver`

查看集群信息

```shell
kubectl cluster-info
```

*显示*

```text
Kubernetes control plane is running at https://172.16.222.122:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

查看所有服务

```shell
kubectl get all --all-namespacesCOPY
```

*显示*

```text
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.68.0.1    <none>        443/TCP   120m
```



# 部署apiserver

## apiserver简介

API 服务器的静态 Pod 清单会受到用户提供的以下参数的影响:

- 要绑定的 `apiserver-advertise-address` 和 `apiserver-bind-port`； 如果未提供，则这些值默认为机器上默认网络接口的 IP 地址和 6443 端口。
- `service-cluster-ip-range` 给 service 使用

- 如果指定了外部 etcd 服务器，则应指定 `etcd-servers` 地址和相关的 TLS 设置 （`etcd-cafile`、`etcd-certfile`、`etcd-keyfile`）； 如果未提供外部 etcd 服务器，则将使用本地 etcd（通过主机网络）
- 如果指定了云提供商，则配置相应的 `--cloud-provider`，如果该路径存在，则配置 `--cloud-config` （这是实验性的，是 Alpha 版本，将在以后的版本中删除）

无条件设置的其他 API 服务器标志有：

- `--insecure-port=0` 禁止到 API 服务器不安全的连接
- `--enable-bootstrap-token-auth=true` 启用 `BootstrapTokenAuthenticator` 身份验证模块。 更多细节请参见 [TLS 引导](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)。
- `--allow-privileged` 设为 `true`（诸如 kube-proxy 这些组件有此要求）
- `--requestheader-client-ca-file` 设为 `front-proxy-ca.crt`

- `--enable-admission-plugins` 设为：

  - [`NamespaceLifecycle`](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#namespacelifecycle) 例如，避免删除系统保留的名字空间
  - [`LimitRanger`](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#limitranger) 和 [`ResourceQuota`](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#resourcequota) 对名字空间实施限制
  - [`ServiceAccount`](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#serviceaccount) 实施服务账户自动化

  - [`PersistentVolumeLabel`](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#persistentvolumelabel) 将区域（Region）或区（Zone）标签附加到由云提供商定义的 PersistentVolumes （此准入控制器已被弃用并将在以后的版本中删除）。 如果未明确选择使用 `gce` 或 `aws` 作为云提供商，则默认情况下，v1.9 以后的版本 kubeadm 都不会部署。

  - [`DefaultStorageClass`](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) 在 `PersistentVolumeClaim` 对象上强制使用默认存储类型
  - [`DefaultTolerationSeconds`](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#defaulttolerationseconds)
  - [`NodeRestriction`](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#noderestriction) 限制 kubelet 可以修改的内容（例如，仅此节点上的 Pod）

- `--kubelet-preferred-address-types` 设为 `InternalIP,ExternalIP,Hostname;` 这使得在节点的主机名无法解析的环境中，`kubectl log` 和 API 服务器与 kubelet 的其他通信可以工作

- 使用在前面步骤中生成的证书的标志：
  - `--client-ca-file` 设为 `ca.crt`
  - `--tls-cert-file` 设为 `apiserver.crt`
  - `--tls-private-key-file` 设为 `apiserver.key`
  - `--kubelet-client-certificate` 设为 `apiserver-kubelet-client.crt`
  - `--kubelet-client-key` 设为 `apiserver-kubelet-client.key`
  - `--service-account-key-file` 设为 `sa.pub`
  - `--requestheader-client-ca-file` 设为 `front-proxy-ca.crt`
  - `--proxy-client-cert-file` 设为 `front-proxy-client.crt`
  - `--proxy-client-key-file` 设为 `front-proxy-client.key`

- 其他用于保护前端代理（ [API 聚合层](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)） 通信的标志:
  - `--requestheader-username-headers=X-Remote-User`
  - `--requestheader-group-headers=X-Remote-Group`
  - `--requestheader-extra-headers-prefix=X-Remote-Extra-`
  - `--requestheader-allowed-names=front-proxy-client`

## 创建目录

> 每台 `master` 服务器都要创建。

```bash
# 创建证书目录
mkdir -p /etc/kubernetes/pki/{apiserver/,kubelet/,aggregator/,service-account/,sign/}

# 创建配置文件存放目录以及 kubeconfig 存放目录和初始化集群所需配置文件目录
mkdir -p /etc/kubernetes/{config/,kubeconfig/,init_k8s_config/}

# 创建 kubectl 使用 config 的默认目录
mkdir -p /root/.kube/

# 创建日志存放目录
mkdir -p /var/log/kubernetes/{apiserver/,controller/,scheduler/}COPY
```



## 安装server端

下载服务端

```shell
[root@k8s-master01 ~]# wget https://dl.k8s.io/v1.26.2/kubernetes-server-linux-amd64.tar.gz
```

解压缩

```shell
[root@k8s-master01 ~]# tar -xf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}

# 获得一下进程
 kube-apiserver  kube-controller-manager  kubectl  kubelet  kube-proxy  kube-scheduler
```

分发到其他server节点

```shell
scp /usr/local/bin/kube* master02:/usr/local/bin
scp /usr/local/bin/kube* master03:/usr/local/bin
```



## 部署 kube-apiserver

### 5-1.创建服务启动配置文件

> [!NOTE]
> 以下操作需要登录到对应的 `master` 服务器操作。
> 只需要修改配置参数中的 `ip` 地址即可。
>
> 注意：以下配置中，日志等级设置为：`6` 。产生的日志的速度会非常快。学习部署后可以修改为：`2`。

#### 5-1-1.创建 master-01 服务器的启动配置文件

[参数配置参考](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-apiserver/)

```shell
cat > /etc/kubernetes/apiserver.conf <<EOF
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota

--anonymous-auth=false

--bind-address=192.168.10.201

--secure-port=6443

--advertise-address=192.168.10.201

--insecure-port=0

--authorization-mode=Node,RBAC

--runtime-config=api/all=true

--service-cluster-ip-range=10.68.0.1/16

--service-node-port-range=30000-39999

--service-account-key-file=/etc/kubernetes/pki/apiserver/kubernetes-ca.pem

--tls-cert-file=/etc/kubernetes/pki/apiserver/apiserver-server.pem

--tls-private-key-file=/etc/kubernetes/pki/apiserver/apiserver-server-key.pem

--client-ca-file=/etc/kubernetes/pki/apiserver/kubernetes-ca.pem

--service-account-signing-key-file=/etc/kubernetes/pki/apiserver/kubernetes-ca-key.pem

--service-account-issuer=https://kubernetes.default.svc.cluster.local

--api-audiences=https://kubernetes.default.svc

--etcd-cafile=/etc/kubernetes/pki/etcd/etcd-ca.pem

--etcd-certfile=/etc/kubernetes/pki/etcd/etcd-apiserver-client.pem

--etcd-keyfile=/etc/kubernetes/pki/etcd/etcd-apiserver-client-key.pem

--etcd-servers=https://192.168.10.201:2379,https://192.168.10.202:2379,https://192.168.10.203:2379

--feature-gates=RemoveSelfLink=false

--enable-swagger-ui=true

--allow-privileged=true

--apiserver-count=3

--enable-aggregator-routing=true

--audit-log-maxage=30

--audit-log-maxbackup=3

--audit-log-maxsize=100

--audit-log-path=/var/log/kubernetes/apiserver/apiserver-audit.log

--event-ttl=1h

--alsologtostderr=true

--logtostderr=false

--log-dir=/var/log/kubernetes/apiserver/

--v=6"
EOF
```





### 5-2.创建服务启动文件

> [!NOTE]
> 使用 `cat` 命令创建文件时，环境变量参数会丢失。需要在开头的 `EOF` 加上单引号即可。
> 在每台 `master` 服务器都要创建。每个启动文件都一样。

```shell
cat > /usr/lib/systemd/system/kube-apiserver.service <<'EOF'
[Unit]
Description=Kubernetes API Server Service
Documentation=https://github.com/kubernetes/kubernetes
#Requires=etcd.service
#After=etcd.service
Before=kube-controller-manager.service kube-scheduler.service

[Service]
EnvironmentFile=-/etc/kubernetes/apiserver.conf
ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF



[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --advertise-address=192.168.10.201 \
      --service-cluster-ip-range=192.168.0.0/16  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.10.201:2379,https://192.168.10.202:2379,https://192.168.10.203:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/kubernetes-ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
	  --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

```

### 5-3.启动服务

启动服务

```shell
systemctl daemon-reload && \
systemctl start kube-apiserver
```

### 5-4.设置开机启动

没有错误后，设置开启启动

```shell
systemctl enable kube-apiserver
```

# 部署Controller manager

## 控制器管理器简介

控制器管理器的静态 Pod 清单受用户提供的以下参数的影响:

- 如果调用 kubeadm 时指定了 `--pod-network-cidr` 参数， 则可以通过以下方式启用某些 CNI 网络插件所需的子网管理器功能：
  - 设置 `--allocate-node-cidrs=true`
  - 根据给定 CIDR 设置 `--cluster-cidr` 和 `--node-cidr-mask-size` 标志

- 如果指定了云提供商，则指定相应的 `--cloud-provider`，如果存在这样的配置文件， 则指定 `--cloud-config` 路径（此为试验性功能，是 Alpha 版本，将在以后的版本中删除）。

其他无条件设置的标志包括：

- `--controllers` 为 TLS 引导程序启用所有默认控制器以及 `BootstrapSigner` 和 `TokenCleaner` 控制器。详细信息请参阅 [TLS 引导](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)
- `--use-service-account-credentials` 设为 `true`

- 使用先前步骤中生成的证书的标志：
  - `--root-ca-file` 设为 `ca.crt`
  - 如果禁用了 External CA 模式，则 `--cluster-signing-cert-file` 设为 `ca.crt`，否则设为 `""`
  - 如果禁用了 External CA 模式，则 `--cluster-signing-key-file` 设为 `ca.key`，否则设为 `""`
  - `--service-account-private-key-file` 设为 `sa.key`

# 部署Scheduler

## 调度器简介
