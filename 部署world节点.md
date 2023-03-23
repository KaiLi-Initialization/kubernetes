# 部署 kubelet



kubelet部署在node节点，需要与`kube-apiserver`进行交互。

证书要求：

kubelet证书需要在工作节点生成。

- `kubelet` 需要对外提供服务，所以`kubelet` 需要自身的 `server` 证书，`kubelet` 可以拥有自身的 `ca` 签发机构。
- `kube-apiserver` 需要调用 `kubelet` 以调度 `pod`，所以 `kubelet` 需要为 `kube-apiserber` 签发客户端 `client` 证书。
- `node` 节点上的 `kubelet` 在使用 `kubeconfig` 跟 `kube-apiserver` 建立连接的时候，需要`kube-apiserver`为`kubelet` 颁发客户端 `client` 证书。（此证书部署kube-apiserver时生成）



## 创建目录

```shell

# 创建证书存放目录
mkdir -p /etc/kubernetes/pki/{kubelet/,apiserver/}

# 创建日志存放目录
mkdir -p /var/log/kubernetes/{kubelet/,kube-proxy/}

# 创建存放 kubeconfig 的目录以及配置目录
mkdir -p /etc/kubernetes/{kubeconfig/,config/}

# 创建 kubelet 工作目录以及 kube-proxy 工作目录
mkdir -p /var/lib/{kubelet/,kube-proxy/}
```





## 创建 ssl 证书

> [!NOTE]
> `ca` 根证书采用 `4096` 位加密。

### kubelet根证书

**创建`kubelet` 的 `ca` 签发机构根证书**

创建kubelet根证书**签名请求**文件：kubelet-ca-scr.json

```shell
cat > /ssl/kubelet-ca-scr.json <<EOF
{
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

生成kubelet根证书

```shell
cfssl gencert -initca kubelet-ca-scr.json | \
cfssljson -bare kubelet-ca - && \
ls kubelet-ca* | \
grep kubelet-ca
```



### kubelet server 证书

> [!NOTE]
> 这里部署的 `node` 节点的 `ip` 地址为：`172.16.222.231`。
> 生成的 `server` 证书只针对该服务器生成。

创建kubelet server证书**签名请求**文件：kubelet-server-csr.json

```shell
cat > /ssl/kubelet-server-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "172.16.222.231"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "O": "k8s",
            "OU": "System",
            "ST": "Beijing"
        }
    ]
}
EOF
```

生成kubelet server证书

```shell
cfssl gencert -ca=kubelet-ca.pem \
-ca-key=kubelet-ca-key.pem \
-config=ca-config.json \
-profile=server kubelet-server-csr.json | \
cfssljson -bare kubelet-server && \
ls kubelet-server* | \
grep kubelet-serverCOPY
```



### kubelet client 证书

**创建 kubelet 提供给 kube-apiserver 访问的 client 证书**

该证书由 `kubelet` 的 `ca` 签发机构创建。

客户端 `client` 证书不需要设置 `hosts` 参数。

创建 kubelet client 证书**签名请求**文件：kubelet-apiserver-client-csr.json

```shell
cat > /ssl/kubelet-apiserver-client-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "k8s",
            "OU": "system"
        }
    ]
}
EOF
cd /ssl/ && \

```

生成kubelet client 证书

```shell
cfssl gencert -ca=kubelet-ca.pem \
-ca-key=kubelet-ca-key.pem \
-config=ca-config.json \
-profile=client kubelet-apiserver-client-csr.json | \
cfssljson -bare kubelet-apiserver-client && \
ls kubelet-apiserver-client* | \
grep kubelet-apiserver-clientCOPY
```



## 分发证书

### 分发 kubelet 的 ca 证书

分发到 `master` 节点以及 `kubelet` 节点



### 分发 kubelet 的 server 证书

分发到 `kubelet` 节点



### 分发 kubelet 颁发给 kube-apiserver 的 client 证书

分发到 `master` 节点



### 分发 kube-apiserver 颁发给 kubelet 的 client 证书

分发到 `kubelet` 节点



### 分发 kube-apiserver 的 ca 公钥给 kubelet

分发到 `node` 节点





## 创建 kubeconfig

> [!NOTE]
> `kubelet` 是使用 `kubeconfig` 跟 `kube-apiserver` 建立连接的。
>
> 在 `kubeconfig` 配置文件中会包含 `kubelet` 的客户端 `client` 证书信息以及身份信息。
>
> 由于已经部署了 `master` 高可用，所以设置集群参数的时候指定的参数：`--server` 需要指向 `vip` 地址。
>
> 也就是前面创建的 `172.16.222.110`，并且端口为 `8443`。

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.110:8443 \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfigCOPY
```

设置客户端认证参数

```shell
kubectl config set-credentials kubelet  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-kubelet-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-kubelet-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfigCOPY
```

设置上下文

```shell
kubectl config set-context kubelet \
--cluster=kubernetes \
--user=kubelet \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfigCOPY
```

设置当前上下文参数

```shell
kubectl config use-context kubelet \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfigCOPY
```

## 8.启动服务

### 8-1.设置 kube-apiserver 的客户端证书参数

> [!NOTE]
> `kube-apiserver` 作为访问 `kubelet` 服务的客户端，需要提供 `kubelet` 的 `ca` 机构为其颁发的客户端证书。
> 需要在每台 `master` 服务器上修改 `kube-apiserver` 的启动参数。

### 8-2.修改 kube-apiserver 参数

> [!NOTE]
> 添加 `kubelet` 为 `kube-apiserver` 颁发的客户端 `client` 证书。

```shell
--kubelet-certificate-authority=/etc/kubernetes/pki/kubelet/kubelet-ca.pem

--kubelet-client-certificate=/etc/kubernetes/pki/kubelet/kubelet-apiserver-client.pem

--kubelet-client-key=/etc/kubernetes/pki/kubelet/kubelet-apiserver-client-key.pemCOPY
```

### 8-3.创建 kubelet 配置文件

```shell
cat > /etc/kubernetes/config/kubelet.yaml <<EOF
apiVersion: "kubelet.config.k8s.io/v1beta1"
kind: "KubeletConfiguration"
enableServer: true
address: "172.16.222.231"
tlsCertFile: "/etc/kubernetes/pki/kubelet/kubelet-server.pem"
tlsPrivateKeyFile: "/etc/kubernetes/pki/kubelet/kubelet-server-key.pem"
authentication:
  anonymous:
    enabled: false
  webhook:
     enabled: false
     cacheTTL: "2m"
  x509:
    clientCAFile: "/etc/kubernetes/pki/kubelet/kubelet-ca.pem"
authorization:
   mode: "Webhook"
   webhook:
     cacheAuthorizedTTL: "5m"
     cacheUnauthorizedTTL: "30s"
cgroupDriver: "systemd"
clusterDomain: "cluster.local"
clusterDNS:
- "10.68.0.1"
EOFCOPY
```

### 8-4.创建启动配置文件

```shell
cat > /etc/kubernetes/config/kubelet.conf <<EOF
KUBELET_OPTS="--network-plugin=cni \

--config=/etc/kubernetes/config/kubelet.yaml \

--alsologtostderr=true \

--logtostderr=false \

--log-dir=/var/log/kubernetes/kubelet/ \

--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig \

--pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6 \

--v=2"
EOFCOPY
```

### 8-5.创建启动文件

```shell
cat > /usr/lib/systemd/system/kubelet.service <<'EOF'
[Unit]
Description=Kubernetes Kubelet Service
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet/
EnvironmentFile=-/etc/kubernetes/config/kubelet.conf
ExecStart=/usr/local/bin/kubelet $KUBELET_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOFCOPY
```

## 9.启动服务

启动服务

```shell
systemctl daemon-reload && \
systemctl start kubeletCOPY
```

检查没有错误后，设置开机启动

```shell
systemctl enable kubeletCOPY
```

## 检测

到 `master` 服务器查看节点是否加入

```shell
kubectl get nodes
```

# 部署kube-proxy

## 创建 kubeconfig

> [!NOTE]
> `kube-proxy` 是使用 `kubeconfig` 跟 `kube-apiserver` 进行通信的。
> `kubeconfig` 配置文件中会包含了 `kube-scheduler` 的客户端 `client` 证书信息以及身份信息。
>
> 由于已经部署了 `master` 高可用，所以设置集群参数的时候指定的参数：`--server` 需要指向 `vip` 地址。
>
> 也就是前面创建的 `172.16.222.110`，并且端口为 `8443`。

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.110:8443 \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfigCOPY
```

设置客户端认证参数

```shell
kubectl config set-credentials kube-proxy  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-kube-proxy-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-kube-proxy-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfigCOPY
```

设置上下文

```shell
kubectl config set-context kube-proxy \
--cluster=kubernetes \
--user=kube-proxy \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfigCOPY
```

设置当前上下文参数

```shell
kubectl config use-context kube-proxy \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfigCOPY
```

## 创建 kube-proxy 配置文件

> [!NOTE]
> 在部署 `kube-proxy` 的 `node` 服务器操作。

```shell
cat > /etc/kubernetes/config/kube-proxy.yaml <<EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: '172.16.222.231'
healthzBindAddress: '172.16.222.231:10256'
metricsBindAddress: '127.0.0.1:10249'
bindAddressHardFail: true
clientConnection:
  kubeconfig: /etc/kubernetes/kubeconfig/kube-proxy.kubeconfig
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  qps: 5
clusterCIDR: 10.100.0.0/16
enableProfiling: false
mode: "ipvs"
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: ""
  strictARP: false
  syncPeriod: 30s
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms
winkernel:
  enableDSR: false
  networkName: ""
  sourceVip: ""
EOFCOPY
```

## 创建启动配置文件

```shell
cat > /etc/kubernetes/config/kube-proxy.conf <<EOF
KUBE_PROXY_OPTS="--alsologtostderr=true \

--logtostderr=false \

--config=/etc/kubernetes/config/kube-proxy.yaml \

--log-dir=/var/log/kubernetes/kube-proxy \

--hostname-override=172.16.222.231

--v=2"
EOFCOPY
```

## 创建启动项

```shell
cat > /usr/lib/systemd/system/kube-proxy.service <<'EOF'
[Unit]
Description=Kubernetes Kube Proxy Service
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy/
EnvironmentFile=-/etc/kubernetes/config/kube-proxy.conf
ExecStart=/usr/local/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOFCOPY
```

## 启动服务

启动服务

```shell
systemctl daemon-reload && \
systemctl start kube-proxyCOPY
```

正常启动没有错误，设置开机启动

```shell
systemctl enable kube-proxyCOPY
```

## 验证服务

> [!NOTE]
> 执行查看进程运行情况。没有 `ERROR` 或者 `FAILED` 等错误正常启动就可以。

```shell
systemctl status kube-proxy --no-pager -l
```