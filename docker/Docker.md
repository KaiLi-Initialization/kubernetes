

## Docker 架构

参考文档：https://blog.csdn.net/crazymakercircle/article/details/120747767

Docker 使用客户端-服务器架构。

![Docker架构图](https://docs.docker.com/engine/images/architecture.svg)



## 联合文件系统

## Docker 安装

**官方文档：**https://docs.docker.com/engine/install/centos/

### 卸载旧版本

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

设置 Docker 存储库

```shell
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

### 安装 Docker 

安装*最新版本*的 Docker Engine、containerd 和 Docker Compose

```shell
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

安装特定版本的 Docker 

```shell
# 查看并有序列出Docker 存储库内Docker的版本
[root@Viruses ~]# yum list docker-ce --showduplicates | sort -r
Last metadata expiration check: 2:40:26 ago on Tue 29 Nov 2022 06:09:07 PM CST.
Installed Packages
docker-ce.x86_64               3:20.10.21-3.el9                docker-ce-stable 
docker-ce.x86_64               3:20.10.20-3.el9                docker-ce-stable 
docker-ce.x86_64               3:20.10.19-3.el9                docker-ce-stable 
docker-ce.x86_64               3:20.10.18-3.el9                docker-ce-stable 
docker-ce.x86_64               3:20.10.17-3.el9                docker-ce-stable 
docker-ce.x86_64               3:20.10.16-3.el9                docker-ce-stable 
docker-ce.x86_64               3:20.10.15-3.el9                docker-ce-stable 
Available Packages

# 安装指定版本
# 格式：
sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io docker-compose-plugin

# 例如安装版本docker-ce-3:20.10.21-3.el9：

sudo yum install docker-ce-3:20.10.21-3.el9 docker-ce-cli-3:20.10.21-3.el9 containerd.io docker-compose-plugin
```

启动Docker 

```shell
sudo systemctl start docker    # 启动Docker
sudo systemctl stop docker     # 停止Docker
sudo systemctl enable docker   # 开机启动Docker
sudo systemctl disable docker  # 开机停止Docker
```

### Docker相关目录

Docker配置文件都存储在**/var/lib/docker/**内，包括**images, containers, volumes, and networks**。

```shell
# 查看所安装Docker版本
[root@Viruses ~]# rpm -qa | grep docker
docker-scan-plugin-0.21.0-3.el9.x86_64
docker-ce-cli-20.10.21-3.el9.x86_64
docker-ce-rootless-extras-20.10.21-3.el9.x86_64
docker-ce-20.10.21-3.el9.x86_64

# 查看Docker文件路径
[root@Viruses ~]# rpm -ql docker-ce-20.10.21-3.el9.x86_64
/usr/bin/docker-init
/usr/bin/docker-proxy
/usr/bin/dockerd
/usr/lib/.build-id
/usr/lib/.build-id/5e
/usr/lib/.build-id/5e/0756aa8361953ead9d55132f06d2dd71085b0c
/usr/lib/.build-id/a3
/usr/lib/.build-id/a3/0bf5c03683cfa111b0ee1011adbf557123988b
/usr/lib/.build-id/e1
/usr/lib/.build-id/e1/aab831a48d5ea39d698f48ff76d906ad8c7030
/usr/lib/systemd/system/docker.service
/usr/lib/systemd/system/docker.socket

# Docker配置文件内容
[root@Viruses ~]# cd /var/lib/docker/
[root@Viruses docker]# ls
buildkit  containers  image  network  overlay2  plugins  runtimes  swarm  tmp  trust  volumes

```

### 卸载 Docker

卸载 Docker Engine、CLI、Containerd 和 Docker Compose 包

```shell
sudo yum remove docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

主机上的Images(镜像), containers（容器）, volumes（数据卷）, or customized（自定义配置）不会自动删除，需要手动删除

```shell
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```



![技术分享图片](http://image1.bubuko.com/info/202003/20200327100642605577.png)



## Docker-images

### 镜像查询

**docker search：**

```shell
Usage:  docker search [OPTIONS] TERM

Search the Docker Hub for images

Options:
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print search using a Go template
      --limit int       Max number of search results (default 25)
                      # 最大搜索条目
      --no-trunc        Don't truncate output

```



### 镜像下载

**docker pull：**

### 镜像管理

**docker images：**

```shell
Usage:  docker images [OPTIONS] [REPOSITORY[:TAG]]

List images

Options:
  -a, --all             Show all images (default hides intermediate images)
                      # 列出所有镜像
      --digests         Show digests
                      # 显示镜像摘要
  -f, --filter filter   Filter output based on conditions provided
                      # 根据提供的条件过滤输出
      --format string   Pretty-print images using a Go template
      --no-trunc        Don't truncate output
                      # 不中断输出（显示镜像的完整信息）
  -q, --quiet           Only show image IDs
                      # 仅显示镜像ID

# 列出镜像完整ID
[root@Viruses ~]# docker images --no-trunc
REPOSITORY   TAG       IMAGE ID                                                                  CREATED         SIZE
mysql        5.7       sha256:c20987f18b130f9d144c9828df630417e2a9523148930dc3963e9d0dab302a76   11 months ago   448MB
mysql        latest    sha256:3218b38490cec8d31976a40b92e09d61377359eab878db49f025e5d464367f3b   11 months ago   516MB
ubuntu       latest    sha256:ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1   13 months ago   72.8MB

# 列出镜像摘要：DIGEST
[root@Viruses ~]# docker images --digests
REPOSITORY   TAG       DIGEST                                                                    IMAGE ID       CREATED         SIZE
mysql        5.7       sha256:f2ad209efe9c67104167fc609cca6973c8422939491c9345270175a300419f94   c20987f18b13   11 months ago   448MB
mysql        latest    sha256:e9027fe4d91c0153429607251656806cc784e914937271037f7738bd5b8e7709   3218b38490ce   11 months ago   516MB
ubuntu       latest    sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322   ba6acccedd29   13 months ago   72.8MB

```

**docker tag：** 镜像标签

```shell
Usage:  docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

### 镜像制作

**建议：**看完docker-container后再来学习。

docker commit

### 镜像删除

**docker rmi：**

## Docker-containers

### 运行容器

**docker run：**运行一个容器 （此命令比较重要，且参数较多）

```shell
# Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

# Options:

-d: 后台运行容器，并返回容器ID；

-i: 以交互模式运行容器，通常与 -t 同时使用；

-P: (大写字母)随机端口映射，容器内部端口随机映射到主机的端口

-p: （小写字母）指定端口映射，格式为：主机(宿主)端口:容器端口

-t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；

--name="nginx-lb": 为容器指定一个名称；

--dns 8.8.8.8: 指定容器使用的DNS服务器，默认和宿主一致；

--dns-search example.com: 指定容器DNS搜索域名，默认和宿主一致；

-h "mars": 指定容器的hostname；

-e username="ritchie": 设置环境变量；

--env-file=[]: 从指定文件读入环境变量；

--cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；

-m :设置容器使用内存最大值；

--net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container: 四种类型；

--link=[]: 添加链接到另一个容器；

--expose=[]: 开放一个端口或一组端口；

--volume , -v: 绑定一个卷
```

**实例**

使用docker镜像nginx:latest以后台模式启动一个容器,并将容器命名为mynginx。

```shell
docker run --name mynginx -d nginx:latest
```

使用镜像nginx:latest以后台模式启动一个容器,并将容器的80端口映射到主机随机端口。

```shell
docker run -P -d nginx:latest
```

使用镜像 nginx:latest，以后台模式启动一个容器,将容器的 80 端口映射到主机的 80 端口,主机的目录 /data 映射到容器的 /data。

```shell
docker run -p 80:80 -v /data:/data -d nginx:latest
```

绑定容器的 8080 端口，并将其映射到本地主机 127.0.0.1 的 80 端口上。

```shell
$ docker run -p 127.0.0.1:80:8080/tcp ubuntu bash
```

使用镜像nginx:latest以交互模式启动一个容器,在容器内执行/bin/bash命令。

```shell
runoob@runoob:~$ docker run -it nginx:latest /bin/bash
root@b8573233d675:/# 
```

### 进入容器

**docker exec：**   进入容器并启用一个新的终端进程

```shell
Usage:  docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

Run a command in a running container

Options:
  -d, --detach               Detached mode: run command in the background
                           # 容器后台运行
      --detach-keys string   Override the key sequence for detaching a container
  -e, --env list             Set environment variables
                           # 设置环境变量
      --env-file list        Read in a file of environment variables
                           # 读入环境变量文件
  -i, --interactive          Keep STDIN open even if not attached
                           # 即使没有附加也保持STDIN 打开
      --privileged           Give extended privileges to the command
  -t, --tty                  Allocate a pseudo-TTY
                           # 重新分配一个伪终端
                           # 一般与-i匹配使用：-it 交互模式进入
  -u, --user string          Username or UID (format: <name|uid>[:<group|gid>])
  -w, --workdir string       Working directory inside the container
                           # 进入容器后的工作目录

```



**docker attach：** 进入容器内现有终端进程

### 容器管理

**docker stop：**  停止容器 

**docker start：**  启动容器

**docker restart：** 重启容器

**docker ps：** 查看容器运行信息

```shell
Usage:  docker ps [OPTIONS]

List containers

Options:
  -a, --all             Show all containers (default shows just running)
                      # 显示所有容器（包括已停止容器）
  -f, --filter filter   Filter output based on conditions provided
                      # 根据过滤条件输出  
                      # 过滤格式： -f key=value；
                      # 过滤格式： --filter "foo=bar" --filter "bif=baz"
      --format string   Pretty-print containers using a Go template
  -n, --last int        Show n last created containers (includes all states) (default -1)
                      # 显示n个最后运行的容器
  -l, --latest          Show the latest created container (includes all states)
                      # 显示最近运行的容器
      --no-trunc        Don't truncate output
  -q, --quiet           Only display container IDs
                      # 只显示容器ID
例如：
[root@Viruses ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                 NAMES
809652b7eec9   mysql     "docker-entrypoint.s…"   4 seconds ago    Up 3 seconds    3306/tcp, 33060/tcp   myslq02
00d85321080c   mysql     "docker-entrypoint.s…"   8 seconds ago    Up 7 seconds    3306/tcp, 33060/tcp   myslq01
b81f9e4a0b18   mysql     "docker-entrypoint.s…"   53 seconds ago   Up 52 seconds   3306/tcp, 33060/tcp   thirsty_taussig
[root@Viruses ~]# 
[root@Viruses ~]# docker stop mysql01
Error response from daemon: No such container: mysql01
[root@Viruses ~]# docker stop b81f9e4a0b18
b81f9e4a0b18
[root@Viruses ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                 NAMES
809652b7eec9   mysql     "docker-entrypoint.s…"   44 seconds ago   Up 42 seconds   3306/tcp, 33060/tcp   myslq02
00d85321080c   mysql     "docker-entrypoint.s…"   48 seconds ago   Up 47 seconds   3306/tcp, 33060/tcp   myslq01
[root@Viruses ~]# docker ps -q
809652b7eec9
00d85321080c
[root@Viruses ~]# docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS                     PORTS                 NAMES
809652b7eec9   mysql     "docker-entrypoint.s…"   49 seconds ago       Up 48 seconds              3306/tcp, 33060/tcp   myslq02
00d85321080c   mysql     "docker-entrypoint.s…"   53 seconds ago       Up 52 seconds              3306/tcp, 33060/tcp   myslq01
b81f9e4a0b18   mysql     "docker-entrypoint.s…"   About a minute ago   Exited (0) 8 seconds ago                         thirsty_taussig

#  -f, --filter 条件过滤
[root@Viruses ~]# docker ps -f id=00d85321080c
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                 NAMES
00d85321080c   mysql     "docker-entrypoint.s…"   5 minutes ago   Up 5 minutes   3306/tcp, 33060/tcp   myslq01

[root@Viruses ~]# docker ps -a --filter "name=myslq" --filter "exited=0"
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                     PORTS     NAMES
809652b7eec9   mysql     "docker-entrypoint.s…"   14 minutes ago   Exited (0) 3 minutes ago             myslq02
[root@Viruses ~]# docker ps -a --filter name=myslq --filter exited=0
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                     PORTS     NAMES
809652b7eec9   mysql     "docker-entrypoint.s…"   15 minutes ago   Exited (0) 3 minutes ago             myslq02
```

**docker kill：** 之间杀死一个或多个正在运行的容器

```shell
# docker stop 命令停止的容器：Exited = 0
# docker kill 杀死的容器：Exited = 137
# 例如：
[root@Viruses ~]# docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                        PORTS     NAMES
809652b7eec9   mysql     "docker-entrypoint.s…"   22 minutes ago   Exited (137) 45 seconds ago             myslq02
00d85321080c   mysql     "docker-entrypoint.s…"   22 minutes ago   Exited (0) 3 seconds ago                myslq01
b81f9e4a0b18   mysql     "docker-entrypoint.s…"   23 minutes ago   Exited (137) 45 seconds ago             thirsty_taussig

```

**docker cp：** 在容器和本地文件系统之间复制文件/文件夹

```shell
# 将本地文件复制到容器中
docker cp ./some_file CONTAINER:/work

# 将文件从容器复制到本地路径
docker cp CONTAINER:/var/logs/ /tmp/app_logs
```

**docker top：** 查看容器内进程

**docker pause** :暂停容器中所有的进程。

**docker unpause** :恢复容器中所有的进程

**docker logs：**  查看容器内日志

**docker stats：** 容器的实时资源使用状态

```shell
[root@Viruses ~]# docker stats 00d85321080c 809652b7eec9
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
00d85321080c   myslq01   0.00%     736KiB / 1.733GiB   0.04%     1.01kB / 0B   0B / 0B     1
809652b7eec9   myslq02   0.00%     740KiB / 1.733GiB   0.04%     1.12kB / 0B   0B / 0B     1

```

### 删除容器

**docker rm：**命令

```shell
# Usage:  docker rm [OPTIONS] CONTAINER [CONTAINER...]

# Options:
  -f, --force     Force the removal of a running container (uses SIGKILL) 
                # 强制删除正在运行的容器
  -l, --link      Remove the specified link
                # 删除两个容器之间的网络连接
  -v, --volumes   Remove anonymous volumes associated with the container
                # 删除与容器相关联的卷
```

### 容器管理拓展

**docker container：**

**docker update：** 更新容器配置（硬件以及策略）

**docker commit：**

**docker export:**

## Docker 命令参数

### --filter filter 

```shell
# 过滤标志（-f或--filter）格式为“key=value”。
 -f, --filter filter   Filter output based on conditions provided
                      # 根据过滤条件输出  
                      # 过滤格式： -f key=value；
                      # 过滤格式： --filter "foo=bar" --filter "bif=baz"
```

**例如：**

```shell
[root@Viruses ~]# docker ps -f id=00d85321080c
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                 NAMES
00d85321080c   mysql     "docker-entrypoint.s…"   5 minutes ago   Up 5 minutes   3306/tcp, 33060/tcp   myslq01

[root@Viruses ~]# docker ps -a --filter "name=myslq" --filter "exited=0"
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                     PORTS     NAMES
809652b7eec9   mysql     "docker-entrypoint.s…"   14 minutes ago   Exited (0) 3 minutes ago             myslq02
[root@Viruses ~]# docker ps -a --filter name=myslq --filter exited=0
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                     PORTS     NAMES
809652b7eec9   mysql     "docker-entrypoint.s…"   15 minutes ago   Exited (0) 3 minutes ago             myslq02
```



### --format string   

```shell
# 格式化选项
-f, --format string   Pretty-print images using a Go template
                      # 使用Go模板格式化输出

```

**例如：**

以命令docker images 为例

使用该`--format`选项时，该`image`命令将完全按照模板声明的方式输出数据，或者在使用该 `table`指令时，还将包括列标题。

以下示例使用不带标题的模板， 并为所有镜像输出由冒号` (： ) `分隔的`ID`和`Repository`

```shell
docker images --format "{{.ID}}: {{.Repository}}"

77af4d6b9913: <none>
b6fa739cedf5: committ
78a85c484f71: <none>
30557a29d5ab: docker
5ed6274db6ce: <none>
746b819f315e: postgres
746b819f315e: postgres
746b819f315e: postgres
746b819f315e: postgres
```

要以表格格式列出所有镜像及其存储库和标签，您可以使用：

```shell
docker images --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}"

IMAGE ID            REPOSITORY                TAG
77af4d6b9913        <none>                    <none>
b6fa739cedf5        committ                   latest
78a85c484f71        <none>                    <none>
30557a29d5ab        docker                    latest
5ed6274db6ce        <none>                    <none>
746b819f315e        postgres                  9
746b819f315e        postgres                  9.3
746b819f315e        postgres                  9.3.5
746b819f315e        postgres                  latest
```



## Docker 其他命令

**docker inspect：** 查看某个容器或镜像的详细信息

**docker info：**查看Docker系统的详细信息

```shell
[root@Viruses ~]# docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Docker Buildx (Docker Inc., v0.9.1-docker)
  scan: Docker Scan (Docker Inc., v0.21.0)

Server:
 Containers: 4
  Running: 2
  Paused: 0
  Stopped: 2
 Images: 3
 Server Version: 20.10.21
 Storage Driver: overlay2
  Backing Filesystem: xfs
  Supports d_type: true
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runtime.v1.linux runc io.containerd.runc.v2
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 770bd0108c32f3fb5c73ae1264f7e503fe7b2661
 runc version: v1.1.4-0-g5fd4c4d
 init version: de40ad0
 Security Options:
  seccomp
   Profile: default
  cgroupns
 Kernel Version: 5.14.0-70.22.1.el9_0.x86_64
 Operating System: Rocky Linux 9.0 (Blue Onyx)
 OSType: linux
 Architecture: x86_64
 CPUs: 1
 Total Memory: 1.733GiB
 Name: Viruses
 ID: 37LI:R2SF:5ZQM:SFBX:6KGE:KFLT:CWY5:ZF4D:LK2C:GSBR:DBJH:K5SA
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Registry Mirrors:
  https://j2w5mtnj.mirror.aliyuncs.com/
 Live Restore Enabled: false

WARNING: No cpu cfs quota support
WARNING: No cpu cfs period support
WARNING: No cpu shares support

```



**docker load ：**

例子

```shell
docker image ls

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker load < busybox.tar.gz

Loaded image: busybox:latest
docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              769b9341d937        7 weeks ago         2.489 MB
docker load --input fedora.tar

Loaded image: fedora:rawhide

Loaded image: fedora:20
docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              769b9341d937        7 weeks ago         2.489 MB
fedora              rawhide             0d20aec6529d        7 weeks ago         387 MB
fedora              20                  58394af37342        7 weeks ago         385.5 MB
fedora              heisenbug           58394af37342        7 weeks ago         385.5 MB
fedora              latest              58394af37342        7 weeks ago         385.5 MB
```



**docker import：** 

## Docker 容器连接

## Docker 仓库管理

**docker login：**登陆仓库服务器

```shell
Usage:  docker login [OPTIONS] [SERVER]

Log in to a Docker registry.
If no server is specified, the default is defined by the daemon.

Options:
  -p, --password string   Password   # 登陆仓库的密码
      --password-stdin    Take the password from stdin  # 从stdin自动获取密码
  -u, --username string   Username   # 登陆仓库的用户名
```

**凭证存储**



**docker logout：**退出仓库服务器



**docker push ：**将镜像上传到仓库

```shell
Usage:  docker push [OPTIONS] NAME[:TAG]

Options:
  -a, --all-tags                Push all tagged images in the repository
      --disable-content-trust   Skip image signing (default true)
  -q, --quiet                   Suppress verbose output

```



## Docker-namespace

## Docker-volumes

docker volume

```shell
Usage:  docker volume COMMAND

Manage volumes

Commands:
  create      Create a volume                   # 创建一个volume
  inspect     Display detailed information on one or more volumes   # 查看volume的详细信息
  ls          List volumes                      # 列出所有volume
  prune       Remove all unused local volumes   # 删除所有未使用的volume
  rm          Remove one or more volumes        # 删除一个或多个volume

```

### 匿名挂载

### 具名挂载

## Docker-daemon

## DockerFile

**docker build：**

## Docker-networks

**docker network：**docker 网络管理

```shell
Usage:  docker network COMMAND

Manage networks

Commands:
  connect     Connect a container to a network                     # 将一个容器连接到另外一个网络
  create      Create a network                                     # 创建一个网络
  disconnect  Disconnect a container from a network                # 断开容器和网络的连接
  inspect     Display detailed information on one or more networks # 查看一个网络的详细信息
  ls          List networks                                        # 查看网络列表
  prune       Remove all unused networks                           # 删除未使用的网络
  rm          Remove one or more networks                          # 删除一个或多个网络

```



## docker-compose

**参考文档：**https://blog.csdn.net/weixin_53719271/article/details/125573551



**官方文档：**https://docs.docker.com/network/

## Docker Machine

## Docker Swarm

docker service

docker swarm

