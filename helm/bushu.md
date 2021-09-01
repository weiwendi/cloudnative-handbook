# 部署 Kubernetes 和 Helm 环境

Helm 能为我们提供很多帮助，但在使用 Helm 前，我们要有 Kubernetes 环境。搭建高可用的 Kubernetes 集群需要的主机资源比较多，搭建过程也比较复杂，所幸使用 Minikube 可以快速搭建单机版的 Kubernetes 集群，又不失真实性。 

> 如果你对安装高可用的 Kubernetes 集群感兴趣，或者想要学习 Kubernetes 相关知识，可以阅读我的 《Kubernetes 最佳实践》。

## 使用 Minikube 部署 Kubernetes

没有 Kubernetes 集群，Helm 全无用武之地。因此，我们先使用 Minikube 搭建单机版的 Kubernetes 集群。

Minikube 是社区驱动的工具，使用它可以很方便的部署一个单节点的 Kubernetes 集群。我们可以创建一台虚拟机，用来作为部署 Kubernetes 的主机。

### 安装 Minikube

Minikube 支持 Windows、MacOS 和 Linux 操作系统，我们可以在 GitHub 页面上下载对应的最新版本的二进制包。本书的示例都是在 Rcoky Linux 上进行的，所以我下载安装 Linux 版本的 Minikube。

1. 下载地址：https://github.com/kubernetes/minikube/releases/。在该页面 “Latest release” 的 “Assets” 部分，查看并下载对应版本。如图：

   <img src="https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/2.1versions.png" alt="2.1versions" style="zoom:80%;" />

   图2.1 - GitHub 页面中的 Minikube 二进制文件，该图仅展示了 “Assets” 的部分内容

   下载与操作系统对应版本的二进制文件：

   ```shell
   [root@aiops0113 ~]# wget https://github.com/kubernetes/minikube/releases/download/v1.20.0/minikube-linux-amd64
   ```

2. 下载完成后，将二进制文件重命名为 minikube。例如，我的是 Linux 系统，需要执行以下命令：

   ```shell
   [root@aiops0113 ~]# mv minikube-linux-amd64 minikube
   ```

3. 要执行 minikube，Linux 和 macOS 用户需要为该文件添加可执行权限：

   ```shell
   [root@aiops0113 ~]# chmod a+x
   ```

4. 将 minikube 放置 PATH 变量管理的路径中，以便在命令行执。PATH 变量的内容取决于操作系统。Linux 和 macOS 用户，可以在终端执行以下命令查看路径信息：

   ```shell
   [root@aiops0113 ~]# echo $PATH
   ```

   Windows 用户可以在命令提示符或者 PowerShell 中执行以下命令：

   ```powershell
   env: PATH
   ```

5. 将 minikube 移动到 /usr/local/bin/ 目录：

   ```shell
   [root@aiops0113 ~]# mv minikube /usr/local/bin/
   ```

6. 验证命令是否能正常使用：

   ```shell
   [root@aiops0113 ~]# minikube version
   # 返回类似以下内容
   minikube version: v1.20.0
   commit: c61663e942ec43b20e8e70839dcca52e44cd85ae
   ```

Minikube 已经安装完成，我们还需要 hypervisor 工具管理本地 Kubernetes 集群，推荐使用 VirtualBox。

### Minikube 驱动

Minikube 依赖驱动安装 Kubernetes 集群。操作系统不同，Minikube 支持的驱动也不同。下图是 Minikube 在不同操作系统上支持的驱动：

<img src="https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/2.2minikubeDrivers.jpg" alt="2.2minikubeDrivers" style="zoom:100%;" />

针对不同的操作系统，Minikube 都有推荐的驱动器。在本书示例中，我使用 VirtualBox 驱动，你可以尝试其它驱动，作为技术拓展。

### 安装 VirtualBox

Minikube 依赖 hypervisors (虚拟机管理器) 将 Kubernetes 集群安装到 VM (虚拟机)上。在本书我们使用 VirtualBox，因为它同时支持 Windows、macOS 和 Linux 操作系统。

* Windows 上安装 VirtualBox：

  ```powershell
  > choco install virtualbox
  ```

* macOS 上安装 VirtualBox：

  ```shell
  $ brew cask install virtualbox
  ```

* Debian Linux 上安装 VirtualBox：

  ```shell
  $ apt-get install virtualbox
  ```

* 基于 RHEL 的 Linux 上安装 VirtualBox，Rocky Linux 和 CentOS 都是基于 RHEL 的操作系统：

  ```shell
  [root@aiops0113 ~]# dnf config-manager --add-repo=https://download.virtualbox.org/virtualbox/rpm/el/virtualbox.repo
  ```

  ```shell
  root@aiops0113 ~]# dnf search virtualbox
  VirtualBox-5.2.x86_64 : Oracle VM VirtualBox
  VirtualBox-6.0.x86_64 : Oracle VM VirtualBox
  VirtualBox-6.1.x86_64 : Oracle VM VirtualBox
  ```

  ```shell
  [root@aiops0113 ~]# dnf -y install VirtualBox-6.1
  ```

更多安装方法可以在 VirtualBox 官网查看：https://www.virtualbox.org/wiki/Downloads。

安装 VirtualBox 后，须将 Minikube 的默认 hypervisor 设置为 VirtualBox。

### 设置 VirtualBox 为默认 hypervisor

设置 minikube 的 vm-driver 为 VirtualBox：

```shell
[root@aiops0113 ~]# minikube config set vm-driver virtualbox
```

执行此命令，可能会有打印出以下警告：

`! These changes will take effect upon a minikube delete and then a minikube start`

如果你还没有通过 Minikube 创建集群，可以忽略此消息。这句话的意思是：在执行此命令前创建的 Kubernetes 集群都不会以 VirtualBox 作为 hypervisor。

查看 vm-driver 的值：

```shell
[root@aiops0113 ~]# minikube config get vm-drivervirtualbox
```

如果一切正常，会输出 VirtualBox。

除了配置默认 hypervisor 外，还可以配置分配给 Minikube 集群的资源。

### 设置 Minikube 资源

默认情况下，Minikube 会为虚拟机分配 2 核 CPU 和 2GB 内存的资源。这些资源足以运行本书中大部分示例。如果你的主机资源充足，可以将内存改为 4GB，CPU不变。

执行以下命令，可以将 Minikube 默认分配的内存大小改为 4GB：

```shell
[root@aiops0113 ~]# minikube config set memory 4000
```

运行 **minikube config get memory** 命令验证对内存的设置，类似于前面验证 vm-driver 的更改：

```shell
[root@aiops0113 ~]# minikube config get memory
```

### Minikube 基本用法

Minikube 是一个入门简单的工具，下面是它的三个常用命令：

* **start**
* **stop**
* **delete**

**start** 子命令用于创建单节点的 Kubernetes 集群。执行此命令，会创建一台虚拟机，并在其中安装 Kubernetes 集群，集群安装完成，命令会结束执行。

在执行 **start** 子命令前，我们需要创建一个普通账户，并切换到普通账户执行，因为 VirtualBox 驱动不能以 root 用户，使用 root 用户会报错。如果没有特别说明，以后的示例都会使用该普通账号。

```shell
[root@aiops0113 ~]# useradd aiops
[root@aiops0113 ~]# su - aiops
```

```shell
[aiops@aiops0113 ~]$ minikube start
```

由于网络因素，中国用户在启动 minikube 时，需要设置几个选项：

```shell
[aiops@aiops0113 ~]$ minikube start --image-mirror-country='cn' \
    --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.20.0.iso

# 以下是安装时的正常输出
* minikube v1.20.0 on Rocky 8.3
* Automatically selected the virtualbox driver
* Using image repository registry.cn-hangzhou.aliyuncs.com/google_containers
* Starting control plane node minikube in cluster minikube
* Creating virtualbox VM (CPUs=2, Memory=2200MB, Disk=20000MB) ...
    > kubelet.sha256: 64 B / 64 B [--------------------------] 100.00% ? p/s 0s
    > kubelet: 108.73 MiB / 108.73 MiB [----------] 100.00% 611.48 KiB p/s 3m2s             
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Verifying Kubernetes components...
  - Using image registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-minikube/storage-provisioner:v5 (global image repository)
* Enabled addons: storage-provisioner, default-storageclass
* kubectl not found. If you need it, try: 'minikube kubectl -- get pods -A'
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

> **注意**：
>
> 在执行 **minikube start** 时，可能会遇到以下问题：
>
> 1. *! Startup with virtualbox driver failed, trying with alternate driver ssh: Failed to start host: creating host: create: precreate: We support Virtualbox starting with version 5. Your VirtualBox install is "WARNING: The vboxdrv kernel module is not loaded. Either there is no module\n*
>
>    这是因为 VirtualBox 的内核组件未加载，执行以下步骤可以解决此问题：
>
>    ```shell
>   [root@aiops0113 ~]# dnf -y install gcc make perl kernel-devel elfutils-libelf-devel
    [root@aiops0113 ~]# rcvboxdrv setup
    vboxdrv.sh: Stopping VirtualBox services.
    vboxdrv.sh: Starting VirtualBox services.
    vboxdrv.sh: Building VirtualBox kernel modules.
>    ```
>
> 2. *Failed to start host: creating host: create: precreate: This computer doesn't have VT-X/AMD-v enabled. Enabling it in the BIOS is mandatory*
>
>    需要开启 **VT-X/AMD**。如果你直接在个人电脑的系统上安装的，可以设置 **BIOS** 的 **Intel Virtual Technology**(这个是 Intel CPU的设置) 的值是否为 **Enable**；如果你是在 VMware Workstation 或者 VirtualBox 中嵌套的虚拟机，需要在处理器虚拟化设置中勾选 **VT-X/AMD-v** 。

**stop** 子命令用于关闭集群和虚拟机。集群和虚拟机的运行状态会保存在磁盘上，下次可以通过 **start** 子命令快速启动，而不是重新创建一个新的虚拟机。

```shell
$ minikube stop
```

**delete** 子命令用于删除集群和虚拟机。此命令会删除集群和虚拟机的运行状态，释放分配的磁盘空间。下次执行 **minikube start** 时，将重新创建新集群和虚拟机。

```shell
$ minikube delete
```

Minikube 还有其他子命令，不过在本书中使用甚少，你可以自己研究下。

当 **minikube start** 命令执行完毕，Kubernetes 集群就安装完成了，但安装时打印的内容显示：**kubectl not found**，接下来我们安装 kubectl 命令行工具。

### 安装 kubectl

有了 **kubectl**，用户就无需直接通过 Kubernetes API 端点执行各种操作了(如 创建、查询、删除资源)，大大提高了操作 Kubernetes 集群便捷性。

编写此文档时，最新的 **kubectl** 版本是 xxx，在整本书中，我们就以此版本为例。

可以使用 Minikube 安装 **kubectl**，也可以通过包管理或者直接下载二进制文件的方式安装。

#### 使用 Minikube 安装

Minikube 提供了 **kubectl** 的子命令，执行子命令会下载 kubectl 二进制文件：

```shell
$ minikube kubectl version
```

这个命令会把 **kubectl** 安装到 $HOME/.minikube/cache/linux/v1.20.2/ 目录下，**kubectl** 版本取决于 Minikube 版本，你的路径中的 linux/v1.20.2/ 部分，可能和我的不同。

**minikube kubectl** 的语法如下：

```shell
$ minikube kubectl <subcommand> <flags>
```

示例：

```shell
$ minikube kubectl version client
```

虽然 **minikube kubectl** 实现了 kubectl 的所有功能，但它语法比直接使用 **kubectl** 繁琐。可以把 **kubectl** 可执行文件从 Minikube 缓存目录拷贝到 **$PATH** 管理的路径。在 Linux 服务器上执行：

```shell
$ sudo cp ~/.minikube/cache/linux/v1.20.2/kubectl /usr/local/bin/
```

现在，kubectl 可以作为一个独立的二进制文件使用了，如下所示：

```shell
$ kubectl version client
```

#### 包管理器安装

不同操作系统使用的包管理不同，安装方式也略有差异：

* 在 Windows 上安装 kubectl

  ```shell
  > choco install kubernetes-cli
  ```

* 在 macOS 上安装 kubectl

  ```shell
  $ brew install kubernetes-cli
  ```

* 在基于 Debian 的 Linux 上安装 kubectl：

  ```shell
  $ sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2
  $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
  $ echo 'deb https://apt.kubernetes.io/ kubernetes-xenial main' | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
  $ sudo apt-get update
  $ sudo apt-get install -y kubectl
  ```

* 在基于 RHEL 的 Linux 上安装 kubectl：

  ```shell
  [aiops@aiops0113 ~]$ sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
  EOF

  [aiops@aiops0113 ~]$ sudo dnf -y install kubectl
  ```

#### 二进制安装

这是最常用的方式，和介绍的第一种方式相似，只是以不同方式下载二进制文件。

下载最新稳定版 kubectl 的二进制文件：

```shell
[aiops@aiops0113 ~]$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

添加执行权限

```shell
[aiops@aiops0113 ~]$ chmod +x kubectl
```

移动到 /usr/local/bin/ 目录

```shell
[aiops@aiops0113 ~]$ sudo mv kubectl /usr/local/bin/
```

验证：

```shell
$ kubectl version client
```

## 安装 Helm

Windows 的 Chocolatey 和 macOS 的 Homebrew 包管理器中已经有了 Helm 包，可以直接安装：

* Windows 安装 Helm：

  ```powershell
  > choco install kubernetes-helm
  ```

* macOS 安装 Helm：

  ```shell
  $ brew install helm
  ```

Linux 用户可以通过下载页面，下载 Helm 的二进制文件进行安装：

1. 在 https://github.com/helm/helm/releases 页面下载对应操作系统的二进制文件；

2. 使用 tar 命令，解压压缩包：

   ```shell
   $ tar zxf helm-v3.5.4-linux-amd64.tar.gz
   ```

3. 将解压后的二进制文件移动到 PATH 路径：

   ```shell
   $ sudo mv linux-amd64/helm /usr/local/bin/
   ```

## 配置 Helm

Helm 提供了大量默认配置，实现了开箱即用。但为了使用上更便利，我们可以修改下面几个参数。

### 添加存储库

在“包管理器安装”小节中介绍包管理器安装 kubectl 时，我们为包管理器添加了软件包仓库，用来安装 kubectl。Helm 作为 Kubernetes 的软件包管理器，也可以添加各种可靠的图表存储库，用以在 Kubernetes 集群中安装应用程序。

Helm 提供了 **repo** 子命令管理图表存储库。**repo** 子命令又包含子命令，用于操作指定的存储库。以下是 **repo** 的五个子命令：

* **add**：添加图表存储库
* **list**：列出已添加的图表存储库
* **remove**：删除已添加的图表存储库
* **update**：图表安装后，它的元数据被缓存到本地，**update** 子命令从图表库中更新数据到本地
* **index**：在打包的图表目录下生成索引文件

repo add 子命令示例：

```shell
$ helm repo add $REPO_NAME $REPO_URL
```

repo list 子命令示例：

```shell
$ helm repo list
```

repo update 子命令示例：

```shell
$ helm repo update
```

repo remove 子命令示例：

```shell
$ helm repo remove $REPO_NAME
```

**helm repo index** 命令会在开发图表的时候介绍。

### 添加插件

Helm 插件由其社区维护，用以增强 Helm 的功能。插件列表地址： https://helm.sh/docs/community/related/

Helm 提供了 **plugin** 子命令管理插件，**plugin** 包含的子命令如下表：

| plugin 子命令 | 描述                       | 语法                          |
| ------------- | -------------------------- | ----------------------------- |
| install       | 安装一个或多个 Helm 插件   | helm plugin install $URL      |
| list          | 查询已安装的 插件          | helm plugin list              |
| uninstall     | 卸载一个或多个已安装的插件 | helm plugin uninstall $PLUGIN |
| update        | 更新一个或多个已安装的插件 | helm plugin update $PLUGIN    |

插件增强了 Helm 功能，以下是几个插件的例子：

* **helm diff**：对比已部署的应用程序不同版本的差异
* **helm secrets**：Used to help conceal secrets from Helm charts  加密 helm charts
* **helm monitor**：监控发布过程，并在出现某些事件时回滚
* **helm unittest**：用于在 Helm 图表上执行单元测试

在本书的后面，会有专门一个章节介绍插件。

### 环境变量

Helm 的一些选项可以通过系统的环境变量配置，以下是六个主要环境变量：

* XDG_CACHE_HOME：设置缓存文件的存储目录
* XDG_CONFIG_HOME：设置 helm 的配置文件
* XDG_DATA_HOME：设置数据存储目录
* HELM_DRIVER：设置后端存储驱动程序
* HELM_NO_PLUGINS：禁用插件
* KUBECONFIG：设置 Kubernetes 配置文件

Helm 遵循 XDG 目录规范，该规范定义了一套指向应用程序的环境变量，变量指明了这些应用程序应该存储的基准目录。根据 XDG 规范，Helm 会根据需要，在操作系统上自动创建三个默认目录：

| 操作系统 | 缓存目录                  | 配置目录                       | 数据目录                |
| -------- | ------------------------- | ------------------------------ | ----------------------- |
| Windows  | %TEMP%\helm               | %APPDATA%\helm                 | %APPDATA%\helm          |
| macOS    | $HOME/Library/Caches/helm | $HOME/Library/Preferences/helm | $HOME/Library/helm      |
| Linux    | $HOME/.cache/helm         | $HOME/.config/helm             | %HOME/.local/share/helm |

换成目录缓存 helm 从 Charts 仓库中下载的charts，已安装的 charts 会被缓存到该目录，以便下次能够快速安装。如需更新缓存，可以使用 **helm repo update** 命令，它会下载最新的可用charts，更新本地缓存。

配置目录保存通过 **helm repo add** 命令添加的存储库信息。安装尚未缓存到本地的 Charts 时，Helm 会根据配置目录的路径查询 Charts 存储库的 RUL，通过 URL 找到所需的 Charts。

数据目录用来存储插件，使用 **helm plugin install** 命令安装的插件数据会存储在这个位置。

**HELM_DRIVER** 定义发布状态如何存储在 Kubernetes 中，默认值是 **secret**，也是推荐值，它将状态存储在 Kubernetes Secret中，Secret 是 Base64 编码；该值也可以设置为 **configmap**，它将状态以纯文本的形式存储在 Kubernetes ConfigMap 中；如果该值设置为 **memory**，状态信息将存储在本地内存中，存储在内存中的方式，不适用于生产环境。

**HELM_NO_PLUGINS** 变量用来禁用插件，插件默认为启用状态，其值为 0。如需禁用插件，需要将值设置为 1。

**KUBECONFIG** 变量设置认证 Kubernetes 集群的文件。如果未设置，默认为 *~/.kube/config*。大多数情况下，用户无需修改此值。

### 命令补全

Bash 和 Z Shell 用户可以启用 Tab键 补全命令，既能简化命令输入，又能有效避免输多手误。当按下 Tab键 时，终端会通过输入命令的状态猜测下一个参数，例如：*cd /usr/local/b*，输入 Tab键，会自动补全为 *cd /usr/local/bin*，同样的，执行 *helm upgrade hello-*，按下 Tab键，会自动补全为 *helm upgrade hello-world*。

启用 Tab键 补全功能：

```shell
$ source <(helm completion $SHELL)
# 需要将 $SHELL 替换为 bash or zsh.
```

**$SHELL** 必须是 bash 或 zsh。自动补全仅在执行了此命令的窗口中有效，其他窗口则无效。

### 身份验证

Helm 需要通过 Kubernetes 集群的验证，才能部署和管理应用程序。它们之间通过 **kubeconfig** 文件进行身份验证，该文件定义了一个或多个 Kubernetes 集群如何进行身份验证。

使用 Minikube 安装的 Kubernetes 集群不需要配置身份验证，因为每次创建集群时，Minikube 都会自动生成一个 **kubeconfig** 文件。但那些没有运行 Minikube 的用户需要创建(提供)一个 kubeconfig 文件，这取决于你的 Kubernetes 集群。

通过以下三个 kubectl 命令创建 **kubeconfig** 文件：

* 第一个命令是 set-cluster：

  ```shell
  kubectl config set-cluster
  ```

  **set-cluster** 命令在 **kubeconfig** 文件中定义一个集群条目，确定 Kubernetes 集群的主机名或 IP 地址，以及它的证书颁发机构。

* 第二个命令是 set-credentials：

  ```shell
  kubectl config set-credentials
  ```

  **set-credentials** 命令定义用户名及其身份验证方法和详细信息。该命令可以配置用户名和密码对、客户端证书、承载令牌或身份验证提供者，用户和管理员可以指定不同的身份验证方法。

* 第三个命令是 set-context：

  ```shell
  kubectl config set-context
  ```

  **set-context** 命令将凭证关联到集群。一旦凭证和集群建立了关联，用户就可以使用凭证的身份验证方法对指定的集群进行身份验证。

**kubectl config view** 命令可以查看 **kubeconfig** 文件。

一旦有了 kubeconfig 文件，Kubectl 和 Helm 就可以与 Kubernetes 集群交互了。

在下一节，我们将介绍如何针对 Kubernetes 集群进行授权。

### Authorization/RBAC

身份验证是确认身份的一种方式，而授权定义了允许经过身份验证的用户执行的操作。Kubernetes 使用基于角色访问控制（RBAC）执行授权的。RBAC 是一个设计角色和权限的系统，可以将角色和权限分配给指定的用户或用户组。用户被允许在 Kubernetes 上执行的操作取决于用户被分配的角色。

Kubernetes 提供了许多不同角色，这里列出了三种常见的角色：

* cluster-admin：管理员角色，允许用户对整个集群中的任何资源执行任何操作。
* edit：允许用户读写 namespace 或逻辑分组中的大多数 Kubernetes 的资源。
* view：禁止用户修改资源，仅允许用户读取 namespace 内的资源。

由于 Helm 使用 **kubeconfig** 文件中定义的凭据对 Kubernetes 进行身份验证，因此 Helm 被赋予了与文件中定义的用户相同的访问级别。如果设置的 **edit** 权限，Helm 在大多数情况下拥有安装应用程序的足够权限。如果是 **view** 权限，Helm 将不能安装应用程序，因为这个角色是只读的。

Minikube 默认提供的是 **cluster-admin** 权限，虽然这不是生产环境中的最佳实践，但对于学习和试验来说是可以接受的。使用其他方式安装 Kubernetes 集群的读者，注意至少要分配 **edit** 角色，以便 Helm 能够在大多数情况下部署应用程序：

```shell
$ kubectl create clusterrolebinding  $USER -edit --clusterrole=edit --user=$USER
```

在“Helm 安全”一章，我们会详细讨论 RBAC，以及如何分配合适的角色，以防止错误或恶意意图。
