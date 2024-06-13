在 CentOS 8 中，切换 YUM 源通常是为了加快软件包的下载速度或获取某些特定的包。以下是切换 YUM 源的步骤：

### 1. 备份现有的 YUM 源配置文件

为了安全起见，先备份现有的 YUM 源配置文件：

```
sudo cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
```

### 2. 下载新的 YUM 源配置文件

根据你的需求，可以选择使用国内的镜像源，如阿里云、腾讯云等。这里以阿里云的镜像为例：

```
sudo wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo
```

### 3. 清理缓存并生成新的缓存

切换源之后，需要清理缓存并生成新的缓存：

```
sudo yum clean all
sudo yum makecache
```

### 4. 更新系统

切换源之后，建议进行一次系统更新：

```
sudo yum update -y
```

### 其他可选镜像源

除了阿里云，你还可以选择其他的镜像源，以下是一些常见的国内镜像源配置文件的下载地址：

- **腾讯云**：

  ```
  sudo wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.cloud.tencent.com/repo/centos8_base.repo
  ```

- **华为云**：

  ```
  sudo wget -O /etc/yum.repos.d/CentOS-Base.repo https://repo.huaweicloud.com/centos/8.3.2011/BaseOS/x86_64/os/
  ```

- **中科大**：

  ```
  sudo wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.ustc.edu.cn/centos/8.3.2011/BaseOS/x86_64/os/
  ```

根据你所在的地理位置和网络情况，选择合适的镜像源可以显著提高 YUM 操作的速度。