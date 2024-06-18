

## Docker私有仓库创建

创建和配置一个 Docker 私有镜像仓库可以帮助您在内部网络中存储和分发 Docker 镜像。以下是具体步骤：

### 步骤 1：安装 Docker

如果尚未安装 Docker，请首先安装 Docker。

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

### 步骤 2：运行 Docker Registry

Docker 提供了一个官方的镜像仓库镜像 `registry`，可以直接运行这个镜像来创建私有镜像仓库。

```
sudo docker run -d -p 5000:5000 --name registry --restart=always registry:2
```

这将启动一个 Docker Registry 并监听在 `5000` 端口。

### 步骤 3：推送镜像到私有仓库

1. **标记镜像：** 将需要推送到私有仓库的镜像标记为私有仓库的格式。例如，假设镜像是 `nginx:latest`，私有仓库地址是 `localhost:5000`：

   ```
   docker tag nginx:latest 101.132.173.221:5000/nginx:latest
   ```
   
2. **推送镜像：** 使用 `docker push` 命令将镜像推送到私有仓库：

   ```
   docker push 101.132.173.221:5000/nginx:latest
   ```

### 步骤 4：从私有仓库拉取镜像

您可以使用 `docker pull` 命令从私有仓库拉取镜像：

```
docker pull 101.132.173.221:5000/nginx:latest
```

### 安全配置（可选）

为了确保您的私有仓库的安全，您可以添加认证和 TLS。

#### 配置基本认证

1. **安装 `htpasswd` 工具：**

   ```
   bash
   复制代码
   sudo yum install -y httpd-tools
   ```

2. **创建认证文件：**

   ```
   bash复制代码mkdir -p /etc/docker/registry
   htpasswd -Bc /etc/docker/registry/htpasswd myuser
   ```

3. **运行带基本认证的 Docker Registry：**

   ```
   bash复制代码sudo docker run -d -p 5000:5000 --name registry --restart=always \
     -v /etc/docker/registry:/auth \
     -e "REGISTRY_AUTH=htpasswd" \
     -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
     -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
     registry:2
   ```

#### 配置 TLS

1. **生成自签名证书：**

   ```
   bash复制代码mkdir -p /etc/docker/certs
   openssl req -newkey rsa:4096 -nodes -sha256 -keyout /etc/docker/certs/domain.key -x509 -days 365 -out /etc/docker/certs/domain.crt
   ```

2. **运行带 TLS 的 Docker Registry：**

   ```
   bash复制代码sudo docker run -d -p 5000:5000 --name registry --restart=always \
     -v /etc/docker/certs:/certs \
     -e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt" \
     -e "REGISTRY_HTTP_TLS_KEY=/certs/domain.key" \
     registry:2
   ```

3. **配置 Docker 客户端信任自签名证书：**

   将证书复制到 Docker 客户端的证书目录：

   ```
   bash复制代码sudo mkdir -p /etc/docker/certs.d/localhost:5000
   sudo cp /etc/docker/certs/domain.crt /etc/docker/certs.d/localhost:5000/ca.crt
   ```

通过以上步骤，您可以创建和配置一个安全的 Docker 私有镜像仓库。如果需要在生产环境中使用，建议使用有效的证书并配置防火墙和访问控制。

## Docker Hub镜像同步

为了在本地或在私有 Docker 仓库中同步 Docker Hub 上的镜像，您可以使用 `docker pull` 命令将镜像拉取到本地，然后将镜像推送到您的私有仓库。以下是具体步骤：

### 步骤 1：从 Docker Hub 拉取镜像

使用 `docker pull` 命令从 Docker Hub 拉取镜像。例如，拉取官方的 `nginx` 镜像：

```
docker pull nginx:latest
```

### 步骤 2：标记镜像以推送到私有仓库

将镜像标记（tag）为私有仓库的格式。假设您的私有仓库地址是 `myregistrydomain.com`，并且您希望使用同样的镜像名称 `nginx`：

```
docker tag nginx:latest myregistrydomain.com/nginx:latest
```

### 步骤 3：登录到私有仓库

使用 `docker login` 命令登录到您的私有仓库：

```
docker login myregistrydomain.com
```

系统会提示您输入用户名和密码。

### 步骤 4：推送镜像到私有仓库

使用 `docker push` 命令将镜像推送到私有仓库：

```
docker push myregistrydomain.com/nginx:latest
```

### 示例完整流程

假设我们要同步 `nginx:latest` 镜像到私有仓库 `myregistrydomain.com`：

1. **从 Docker Hub 拉取镜像**：

   ```
   docker pull nginx:latest
   ```

2. **标记镜像**：

   ```
   docker tag nginx:latest myregistrydomain.com/nginx:latest
   ```

3. **登录私有仓库**：

   ```
   docker login myregistrydomain.com
   ```

4. **推送镜像**：

   ```
   docker push myregistrydomain.com/nginx:latest
   ```

### 自动化同步

为了定期同步 Docker Hub 上的镜像到私有仓库，您可以编写一个脚本并使用 cron 定时任务进行自动化。

示例脚本 `sync_docker_images.sh`：

```
#!/bin/bash

# 设置变量
IMAGE_NAME="nginx:latest"
PRIVATE_REGISTRY="myregistrydomain.com"

# 从 Docker Hub 拉取最新镜像
docker pull $IMAGE_NAME

# 标记镜像
docker tag $IMAGE_NAME $PRIVATE_REGISTRY/$IMAGE_NAME

# 登录私有仓库
docker login $PRIVATE_REGISTRY -u your_username -p your_password

# 推送镜像到私有仓库
docker push $PRIVATE_REGISTRY/$IMAGE_NAME
```

将脚本设置为可执行：

```
chmod +x sync_docker_images.sh
```

然后在 cron 中添加定时任务。例如，每天凌晨2点同步镜像：

```
crontab -e
```

添加以下行：

```
0 2 * * * /path/to/sync_docker_images.sh
```

通过以上步骤，您可以将 Docker Hub 上的镜像同步到本地或私有仓库，并可实现自动化定期同步。