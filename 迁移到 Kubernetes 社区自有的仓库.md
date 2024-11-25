## 迁移到 Kubernetes 社区自有的仓库



### 使用 `apt`/`apt-get` 的 Debian、Ubuntu 一起其他操作系统

1. 替换 `apt` 仓库定义，以便 `apt` 指向新仓库而不是托管在 Google 的仓库。 确保将以下命令中的 Kubernetes 次要版本替换为你当前使用的次要版本：

   ```shell
   echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

1. 下载 Kubernetes 仓库的公共签名密钥。所有仓库都使用相同的签名密钥， 因此你可以忽略 URL 中的版本：

   ```shell
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```

1. 更新 `apt` 包索引：

   ```shell
   sudo apt-get update
   ```

### 使用 `rpm`/`dnf` 的 CentOS、Fedora、RHEL 以及其他操作系统

1. 替换 `yum` 仓库定义，使 `yum` 指向新仓库而不是托管在 Google 的仓库。 确保将以下命令中的 Kubernetes 次要版本替换为你当前使用的次要版本：

   ```shell
   cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
   exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
   EOF
   ```