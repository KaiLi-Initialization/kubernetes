官方文档：https://docs.docker.com/go/guides/

### Docker系统

1. 查看Docker系统信息

   docker version

2. 查看Docker系统信息

   docker info

   docker system info

3. 查看docker磁盘使用情况

   docker system df

4. 从服务器获取实时事件(监听docker服务事件)

   docker events

   docker system events

5. Docker系统修剪**（慎用）**

   删除所有未使用的容器、网络、图像（悬空和未使用的）以及可选的卷。

   docker system prune

   







### Docker镜像

Usage:  docker image COMMAND

```shell
Usage:  docker image COMMAND

Manage images

Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Download an image from a registry
  push        Upload an image to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE

Run 'docker image COMMAND --help' for more information on a command.

```

#### 镜像下载

- docker pull

- docker image pull 

#### 镜像上传

- docker push

  格式：docker pull [OPTIONS] NAME[:TAG|@DIGEST]

- docker image push

  格式：docker image pull [OPTIONS] NAME[:TAG|@DIGEST]

#### 镜像压缩

将一个或多个镜像保存到tar文档，默认传输到STDOUT

- docker save

  格式1：docker save [OPTIONS] IMAGE [IMAGE...] > name.tar

  格式2：docker save -o name.tar IMAGE [IMAGE...]

  ```shell
  Options:
    -o, --output string   Write to a file, instead of STDOUT #写入文件，而不是STDOUT
  ```

  **示例：**

  将镜像centos;busybox;nginx保存到image.tar

  ```shell
  
  [root@Viruses ~]# docker images
  REPOSITORY        TAG       IMAGE ID       CREATED         SIZE
  alpine            latest    1d34ffeaf190   2 days ago      7.79MB
  prom/prometheus   latest    ecb74a3b23a9   2 weeks ago     272MB
  nginx             latest    e784f4560448   3 weeks ago     188MB
  busybox           latest    65ad0d468eb1   12 months ago   4.26MB
  centos            latest    5d0da3dc9764   2 years ago     231MB
  
  [root@Viruses ~]# docker save centos busybox nginx > image.tar
  
  
  [root@Viruses ~]# ll -sh image.tar
  415M -rw-r--r-- 1 root root 415M May 25 09:44 image.tar
  
  # docker save -o
  
  [root@Viruses ~]# docker save -o file.tar prom/prometheus
  
  
  [root@Viruses ~]# ll -hs file.tar
  261M -rw------- 1 root root 261M May 25 10:00 file.tar
  
  ```

  

- docker image save

  格式：docker image save [OPTIONS] IMAGE [IMAGE...]

#### 镜像加载

从tar文档或STDIN加载镜像

- docker load

  格式：docker load [OPTIONS] < name.tar

  ```shell
  Options:
    -i, --input string   Read from tar archive file, instead of STDIN
    -q, --quiet          Suppress the load output
  
  ```

  

  **示例**

  ```shell
  [root@Viruses ~]# docker images
  REPOSITORY        TAG       IMAGE ID       CREATED       SIZE
  alpine            latest    1d34ffeaf190   2 days ago    7.79MB
  prom/prometheus   latest    ecb74a3b23a9   2 weeks ago   272MB
  nginx             latest    e784f4560448   3 weeks ago   188MB
  
  
  [root@Viruses ~]# docker load < image.tar
  74ddd0ec08fa: Loading layer [==================================================>]  238.6MB/238.6MB
  Loaded image: centos:latest
  d51af96cf93e: Loading layer [==================================================>]  4.495MB/4.495MB
  Loaded image: busybox:latest
  Loaded image: nginx:latest
  
  # docker load -i
  
  [root@Viruses ~]# docker images
  REPOSITORY        TAG       IMAGE ID       CREATED         SIZE
  alpine            latest    1d34ffeaf190   2 days ago      7.79MB
  prom/prometheus   latest    ecb74a3b23a9   2 weeks ago     272MB
  nginx             latest    e784f4560448   3 weeks ago     188MB
  busybox           latest    65ad0d468eb1   12 months ago   4.26MB
  centos            latest    5d0da3dc9764   2 years ago     231MB
  
  [root@Viruses ~]# docker rmi prom/prometheus
  
  [root@Viruses ~]# docker images
  REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
  alpine       latest    1d34ffeaf190   2 days ago      7.79MB
  nginx        latest    e784f4560448   3 weeks ago     188MB
  busybox      latest    65ad0d468eb1   12 months ago   4.26MB
  centos       latest    5d0da3dc9764   2 years ago     231MB
  
  [root@Viruses ~]# docker load -i file.tar
  1e604deea57d: Loading layer [==================================================>]  1.458MB/1.458MB
  6b83872188a9: Loading layer [==================================================>]  2.455MB/2.455MB
  d6ce738f5c90: Loading layer [==================================================>]  138.4MB/138.4MB
  603ff4c44eba: Loading layer [==================================================>]  130.3MB/130.3MB
  4338e6dae6b8: Loading layer [==================================================>]  3.584kB/3.584kB
  1e80bc7d676a: Loading layer [==================================================>]  13.82kB/13.82kB
  27373cd1de4f: Loading layer [==================================================>]  28.16kB/28.16kB
  d005ffff3451: Loading layer [==================================================>]  13.31kB/13.31kB
  ef3be2c1e398: Loading layer [==================================================>]  5.632kB/5.632kB
  fc14cb076963: Loading layer [==================================================>]  141.3kB/141.3kB
  c8fa07b1bc4b: Loading layer [==================================================>]  1.536kB/1.536kB
  9520e68dff44: Loading layer [==================================================>]  6.144kB/6.144kB
  Loaded image: prom/prometheus:latest
  
  [root@Viruses ~]# docker images
  REPOSITORY        TAG       IMAGE ID       CREATED         SIZE
  alpine            latest    1d34ffeaf190   2 days ago      7.79MB
  prom/prometheus   latest    ecb74a3b23a9   2 weeks ago     272MB
  nginx             latest    e784f4560448   3 weeks ago     188MB
  busybox           latest    65ad0d468eb1   12 months ago   4.26MB
  centos            latest    5d0da3dc9764   2 years ago     231MB
  
  ```

  

- docker image load

#### 镜像列表

- docker image ls

- docker image list

- docker images

  格式：docker images [OPTIONS] [REPOSITORY[:TAG]]

  ```shell
  [root@Viruses ~]# docker images centos
  REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
  centos       latest    5d0da3dc9764   2 years ago   231MB
  ```

  

#### 镜像删除

1. 删除指定一个或多个镜像

   - docker image rm

   - docker image remove

   - docker rmi

     选项

     ```shell
     -f, --force      Force removal of the image        # 强制删除
           --no-prune   Do not delete untagged parents
     ```

     

2. 删除未使用镜像

   - docker image prune

     格式：docker image prune [OPTIONS]

     ```shell
     Options:
       -a, --all             删除未被容器引用的镜像
           --filter filter   Provide filter values (e.g. "until=<timestamp>")
       -f, --force           删除时不提示
     ```

#### 镜像信息

- docker inspect
- docker image inspect

docker history

#### 镜像版本

- docker tag

  格式：docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

- docker image tag

  格式：docker image tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

#### 镜像构建（重点）

1. 从tarball构建镜像

   - docker import 

     格式：docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]

   - docker image import

     格式：docker image import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]

2. docker build

   根据 Dockerfile 和“上下文”构建 Docker镜像。

3. docker buildx

   使用 BuildKit 扩展构建功能

4. docker builder

   管理构建





- docker build

  

  Usage:  docker buildx build [OPTIONS] PATH | URL | -

  Aliases:
    docker buildx build, docker buildx b

#### 过滤器使用（重点）

过滤（--filter）

格式为“key=value”。如果有多个过滤器，则传递多个标志（例如`--filter "foo=bar" --filter "bif=baz"`）

该`label`过滤器接受两种格式。一种是`label=...`(`label=<key>`或`label=<key>=<value>`)，用于删除具有指定标签的图像。另一种格式是`label!=...`(`label!=<key>`或`label!=<key>=<value>`)，用于删除不具有指定标签的图像。

### Docker容器

#### 容器创建（重点）

- docker container create

#### 容器运行

- docker run 

  Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

  参考文档：https://docs.docker.com/reference/cli/docker/container/run/

- docker container run 

  ```shell
  Options:
        --add-host list                    Add a custom host-to-IP mapping (host:ip)
                                         # 添加容器主机名和IP地址映射关系
   -d, --detach                           Run container in background and print                                                 container ID
                                         # 后台运行容器，并打印容器ID
        --dns list                         Set custom DNS servers
                                         # 设置DNS服务地址
        --expose list                      Expose a port or a range of ports
                                         # 暴露一个端口或者某个端口范围
    -h, --hostname string                  Container host name
                                         # 设置容器主机名
    -i, --interactive                      Keep STDIN open even if not attached
                                         # 一般和‘-t’绑定使用
        --ip string                        IPv4 address (e.g., 172.30.100.104)
                                         # 为容器指定IPV4地址
        --ip6 string                       IPv6 address (e.g., 2001:db8::33)
                                         # 为容器指定IPV6地址
    -l, --label list                       Set meta data on a container
                                         # 为容器设置标签
        --mount mount                      Attach a filesystem mount to the container
                                         # 为容器挂在文件系统
        --name string                      Assign a name to the container
                                         # 为容器设置名字
        --network network                  Connect a container to a network
                                         # 为容器指定链接网络
    -p, --publish list                     Publish a container's port(s) to the host
                                         # 小写p,将容器端口映射为指定主机端口
    -P, --publish-all                      Publish all exposed ports to random ports
                                         # 大写P，将容器端口映射为随机主机端口
    --rm                                   Automatically remove the container when it                                                                                    exits
                                         # 退出容器时，删除该容器
    -t, --tty                              Allocate a pseudo-TTY
                                         # 为容器分配一个TTY，一般和‘-i’绑定使用
    -v, --volume list                      Bind mount a volume
                                         # 为容器绑定一个卷
  
  ```

#### 容器进程

- docker top
- docker container top

##### 进程暂停

暂停一个或多个容器中的所有进程	

- docker pause
- docker container pause

##### 取消暂停

取消暂停一个或多个容器内的所有进程

- docker unpause
- ocker container unpause



#### 容器列表

- docker ps
- docker container ps
- docker container ls
- docker container list

#### 容器删除

- docker rm

  Usage:  docker rm [OPTIONS] CONTAINER [CONTAINER...]

- docker container rm

- docker container remove

  参数：

  ```shell
    -f, --force     Force the removal of a running container (uses SIGKILL) 
                   # 强制删除正在运行的容器
    -l, --link      Remove the specified link
    -v, --volumes   Remove anonymous volumes associated with the container
                   # 删除与容器关联的卷
  ```
  
- docker container prune （删除所有未运行容器）

  格式：docker container prune [OPTIONS]

#### 文件复制

在容器和本地文件系统之间复制文件/文件夹

- docker cp

  格式1：docker cp [OPTIONS] SRC_PATH CONTAINER:DEST_PATH

  格式1：docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH

- docker container cp

  格式：docker container cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH
  格式：docker container cp [OPTIONS] SRC_PATH CONTAINER:DEST_PATH

```shell
# 将本地文件复制到容器内
docker cp ./some_file CONTAINER:/work

# 将文件从容器复制到本地路径
docker cp CONTAINER:/var/logs/ /tmp/app_logs

```



#### 容器信息

- docker container inspect

  格式：docker container inspect [OPTIONS] CONTAINER [CONTAINER...]

#### 资源统计

- docker stats

- docker container stats

  显示容器资源使用情况的实时统计信息，要将数据限制到一个或多个特定容器，请指定以空格分隔的容器名称或 ID 列表。

  ```shell
  Options:
    -a, --all             Show all containers (default shows just running)
                        # 显示所有正在运行的容器实时资源信息
        --no-stream       Disable streaming stats and only pull the first result
                        # 结果只刷新一次，如果不带此参数，每秒统计一次
  
  ```

  **示例**

  ```shell
  # 显示容器centos-text；nginx-text的实时资源信息
  [root@Viruses ~]# docker stats centos-text nginx-text
  CONTAINER ID   NAME          CPU %     MEM USAGE / LIMIT    MEM %     NET I/O      BLOCK I/O        PIDS
  50295086c162   centos-text   0.00%     1.5MiB / 1.734GiB    0.08%     936B / 0B    0B / 0B          2
  58785a77da95   nginx-text    0.00%     2.41MiB / 1.734GiB   0.14%     1.3kB / 0B   135kB / 17.9kB   2
  
  # 将输出信息统计到文档stats.txt内
  [root@Viruses ~]# docker stats --no-stream centos-text nginx-text > stats.txt
  - --no-stream 参数：只显示一次统计信息，如果不带此参数，每秒统计一次
  ```

  

  

#### 容器管理

- docker diff
- docker container diff

##### 容器名字

- docker  rename 

  格式：docker rename CONTAINER NEW_NAME

  ```shell
  docker rename my_container my_new_container
  ```

  

- docker container rename 

  格式：docker container rename CONTAINER NEW_NAME

  例如：

  ```shell
  docker container rename my_container my_new_container
  ```

##### 容器启动

- docker start
- docker container start

##### 容器重启

- docker restart
- docker container restart

##### 容器停止

- docker stop
- docker container stop

##### 容器导出

`docker export`命令不会导出与容器关联的卷的内容,如果卷安装在容器中现有目录的顶部，则`docker export`导出底层目录的内容，而不是卷的内容。

- docker export

  ```shell
  docker export red_panda > latest.tar
  ```

  

- docker container export



##### 容器终止

- docker container kill

##### 容器日志

- docker logs
- docker container logs

#### 容器端口

- docker port

  格式：docker port CONTAINER [PRIVATE_PORT[/PROTO]]

- docker container port

#### 容器更新

更新一个或多个容器的资源（内存，CPU等配置）

- docker update
- docker container update



### Docker网络

#### 创建网络

- docker network create 

  格式：docker network create [OPTIONS] NETWORK

  ```shell
  Options:
    -d, --driver string        Driver to manage the Network (default "bridge")
        --gateway strings      IPv4 or IPv6 Gateway for the master subnet
                             # 设置IPV4和IPV6的网关
        --ingress              Create swarm routing-mesh network
                             # 集群内网络访问
        --internal             Restrict external access to the network
                             # 限制外部网络访问此网络
        --ip-range strings     Allocate container ip from a sub-range
                             # 设置容器可使用IP地址
        --ipv6                 Enable IPv6 networking
                             # 启用IPV6功能
        --label list           Set metadata on a network
                             # 为此网络设置标签
        --scope string         Control the network's scope
        --subnet strings       Subnet in CIDR format that represents a network segment
                             # 设置可用IP网段
  
  ```



#### 连接网络

- docker network connect

#### 断开网络

#### 网络列表

#### 网络信息

#### 网络删除

### Docker插件

### Docker服务

### Docker Hub

- docker login

  格式：docker login [OPTIONS] [SERVER]

  参数：

  ```shell
  
  Options:
    -p, --password string   Password   # 指定用户的密码
        --password-stdin    Take the password from stdin
    -u, --username string   Username   # 指定登录用户
  ```

  

- docker logout 

  格式：docker logout [SERVER]



- docker search

  查询Docker Hub内指定镜像版本

  格式：docker search [OPTIONS] TERM



