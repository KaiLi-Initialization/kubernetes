# 容器之根本：runc

Docker，Podman，CRI-O和Containerd的核心工具：[runc](https://github.com/opencontainers/runc)。

![img](https://fjclub.s3-eu-west-1.amazonaws.com/blog_images/user.png)



正如我们所看到的，无论你使用的是Docker还是Podman或CRI-O，你很可能使用的是runc。为什么是“最有可能”的部分？让我们检查一下：`man runc`

> runc 是一个命令行客户端，用于运行根据开放容器计划 （OCI） 格式打包的应用程序，并且是开放容器计划规范的合规实现

runc运行的依据是“[OCI 捆绑包](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md)”，它基本上是一个根文件系统和一个 [config.json 文件](https://github.com/opencontainers/runtime-spec/blob/master/config.md)。`runc run nginx:latest`

Runc 符合 OCI 规范（具体来说，[运行时规范](https://github.com/opencontainers/runtime-spec)），它可以采用 OCI 捆绑包并从中运行容器。值得重复的是，这些捆绑包不是“容器映像”，它们要简单得多。层、标签、容器注册表和存储库等功能 - 所有这些都不是 OCI 捆绑包的一部分，甚至不是运行时规范的一部分。



### runc构建

参考文档：[实现容器运行时填充程序：runc (iximiuz.com)](https://iximiuz.com/en/posts/implementing-container-runtime-shim/)

参考文档：[挖掘运行时 – runc (quarkslab.com)](https://blog.quarkslab.com/digging-into-runtimes-runc.html)

容器是使用捆绑包配置的。容器的捆绑包是一个目录 这包括一个名为 *config.json* 的规范文件和一个根文件系统。 根文件系统包含容器的内容。

从形式上讲，*runc* 是 *libcontainer* 的客户端包装器。由于 *runc* 遵循容器运行时的 *OCI* 规范，因此需要两条信息：

- **OCI 配置** - 一个类似 JSON 的文件，包含容器进程信息，如命名空间、功能、环境变量等。
- **根文件系统目录** - 容器进程 （chroot） 将使用的根文件系统目录。

![使用低级容器运行时创建容器 - runc。](https://iximiuz.com/implementing-container-runtime-shim/runc-workflow-2000-opt.png)

### 生成 OCI 配置



什么是容器运行时？什么是容器管理器？什么是容器业务流程协调程序和容器引擎？容器业务流程协调程序是否需要容器管理器？容器运行时和容器管理器之间的界限在哪里？

# libcontainer

参考文档：[Runc/libcontainer at Main ·OpenContainers/runc ·GitHub](https://github.com/opencontainers/runc/tree/main/libcontainer)



# runc运行

参考文档：[GitHub - opencontainers/runc：CLI 工具，用于根据 OCI 规范生成和运行容器](https://github.com/opencontainers/runc)



runc安装

## 使用runc

### 创建 OCI 捆绑包

要使用 runc，您必须具有 OCI 捆绑包格式的容器。 如果您安装了 Docker，则可以使用其方法从现有的 Docker 容器中获取根文件系统。`export`

```
# create the top most bundle directory
mkdir /mycontainer
cd /mycontainer

# create the rootfs directory
mkdir rootfs

# export busybox via Docker into the rootfs directory
docker export $(docker create busybox) | tar -C rootfs -xvf -
```

### 生成OCI规范文件

填充根文件系统后，您只需在捆绑包中以文件格式生成规范即可。 提供用于生成基本模板规格的命令，然后您可以编辑该规范。 要查找规范中字段的功能和文档，请参阅[规范](https://github.com/opencontainers/runtime-spec)存储库。`config.json``runc``spec`

```
runc spec
```



## 运行容器

方式一：

```shell
cd /mycontainer
runc run mycontainerid
```







#### 主管

`runc`可与流程监控器和初始化系统一起使用，以确保容器在退出时重新启动。 一个示例 systemd 单元文件如下所示。

```shell
[Unit]
Description=Start My Container

[Service]
Type=forking
ExecStart=/usr/local/sbin/runc run -d --pid-file /run/mycontainerid.pid mycontainerid
ExecStopPost=/usr/local/sbin/runc delete mycontainerid
WorkingDirectory=/mycontainer
PIDFile=/run/mycontainerid.pid

[Install]
WantedBy=multi-user.target
```



