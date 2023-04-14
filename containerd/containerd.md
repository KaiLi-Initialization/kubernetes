
# RunC 是什么？
runc是一个 CLI 工具，用于根据 OCI 规范在 Linux 上生成和运行容器。
参考文档：https://github.com/opencontainers/runc
平台需要开启对seccomp的支持，所以平台必须安装libseccomp。

Docker、Google、CoreOS 和其他供应商创建了开放容器计划 (OCI)，目前主要有两个标准文档：容器运行时标准 （runtime spec）和 容器镜像标准（image spec）。
https://pic1.zhimg.com/80/v2-06a5a9fb7bc906ab3233208d9ae18134_1440w.webp


RunC 是一个轻量级的工具，它是用来运行容器的，只用来做这一件事，并且这一件事要做好。我们可以认为它就是个命令行小工具，可以不用通过 docker 引擎，直接运行容器。
