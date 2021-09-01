# Kubernetes 与 Helm 介绍

应用程序对可扩展性、高可用性以及能够承受各种程度负载的要求，是压在技术人员头上的三座大山。在容器化流行之前，我们接触更多的应用程序是 **monolithic**(单体)架构。Docker 的诞生，推动了容器化的发展，**monolithic** 架构也逐渐转向了微服务架构，并以容器化方式部署，直到演变为使用 Kubernetes 之类的工具编排和扩展容器化微服务。

虽然 Kubernetes 是高效的容器编排工具，但它学习曲线陡峭，对于使用 Kubernetes 的团队来说，这是一个不小的挑战。Helm 是一个简化在 Kubernetes 上部署应用程序的工具，用户通过 Helm 可以简单、方便地在 Kubernetes 集群中部署和管理应用程序的生命周期。Helm 抽象了配置应用程序的复杂性，可以有效提高团队在 Kubernetes 平台上的工作效率。

本书会详细介绍 Helm 的各种功能，并探索 Helm 如何使 Kubernetes 上的应用程序部署更加简单。一开始，我们仅作为使用者，使用社区编写好的 Helm Charts 实践程序包管理器。随着内容深入，我们会作为 Helm Charts 开发者角色，并学习如何高效的打出易用的 Kubernetes 应用程序包。最后，介绍 Helm 插件、应用程序管理及安全性这些高级功能。

首先，我们先从了解微服务、容器、Kubernetes，以及使用它们部署应用程序带来的挑战开始。随后，我们将讨论 Helm 的主要功能和优点。

## 从 monolithic 到微服务

你应该听说过摩尔定律，它由英特尔创始人之一的戈登摩尔提出，内如是：“当价格不变时，集成电路上可容纳的元器件数目，约每隔18-24个月便会增加一倍，性能也将提升一倍”。虽然这种趋势近几年稍显平息，但在处理器存在的头 30 年中，这种趋势的持久性对技术的进步至关重要。

编程人员充分利用了这一特性，在开发软件时集成了越来越多的功能和组件。这就导致，一个应用程序可以由多个较小的组件组成，每个组件可以单独开发为独立的服务。这就是 **monolithic** 架构，这种架构的优势之一，就是简化了部署过程。但随着行业趋势发生变化，企业更加关注快速交付的能力。在这种情况下，**monolithic** 架构的应用程序面临了许多挑战：每当对软件进行更改时，都需要验证整个应用程序及其所有基础组件，以确保修改后没有异常。这个过程可能需要多个团队协作完成，大大降低了软件交付的速度及机动性。

企业希望能够更快完成软件交付，尤其是跨组织或部门的快速交付能力，为了实现这一点，很多企业引入了 DevOps 文化。DevOps 提倡以最小的代价帮助企业应用开发进入高效的协作模式和快速交付的迭代过程。为了在这种新模式中保持可持续性，软件架构也从大型 **monolithic** 应用程序逐渐拆分为可以更快速交付的多个较小的应用程序，这些独立的多个小应用被命名为微服务。微服务应用程序固有的特性带来了一些理想的功能，包括并行开发、独立扩展（增加实例数量）服务等。

从 **monolithic** 服务到微服务的架构变化，也改变了软件交付时的打包和部署方式。**monolithic** 服务部署时，整台机器专用于一个或两个应用程序。在微服务时代，单个应用所需的资源整体减少，因此将整台机器专用于一个或两个微服务已不再可行。

幸运的是，容器技术完美贴合了创建微服务运行时所需的功能。容器技术虽然在计算机方面有着悠久历史，但真正“飞入寻常百姓家”，还要从 Docker 的诞生开始，它简单便捷的打包方式、易于分发容器镜像的能力，将容器化推向了大众。

容器和微服务已成了天作之合。结合容器技术，应用程序有了便捷的打包和分发机制、共享计算资源的同时又彼此隔离。然而，随着越来越多的容器化微服务被部署，对容器的管理成为了一个难题。如何确保每个运行中的容器的运行状态？如果一个容器发生了故障怎么办？如果宿主机的计算能力不足，会发生什么？Kubernetes 很好地解决了这些容器编排的需求。

## 什么是 Kubernetes

Kubernetes 常被缩写为 k8s(8 代表 k s 之间的8个字符)，是一个开放源代码的容器编排平台，源自 Google 的内部编排工具 Borg 系统，于 2015 年开源。在 2015 年 7 月 21 日发布 v1.0 之后，Google 和 Linux Foundation 合作成立了 Cloud Native Computing Foundation(CNCF)，是 Kubernete 项目的当前维护者。

Kubernetes 是希腊语，意为“舵手”或“飞行员”。舵手负责操纵船舶，并与船员紧密合作，确保航行安全稳定以及船员的整体安全。Kubernetes 在容器和微服务方面也负有类似的责任。Kubernetes 负责容器的编排和调度，将容器部署到可以处理其工作负载的适当 Worker 节点。Kubernetes 还可以通过提供高可用性和健康检查来确保这些微服务的安全性。

让我们一起看看 Kubernetes 是如何简化管理容器化微服务，以及帮助技术人员卸掉头顶三座大山的：

### 容器编排

容器编排是 Kubernetes 最显著的特性，通过计算资源池，把容器调度到满足其运行的主机上。我用一个简单的示例来说明这一点。下图中的应用程序请求 2Gi 内存和 1 核 CPU，这意味着容器被调度到的 Worker 节点至少拥有满足该应用程序请求的空闲资源。Kubernetes 负责跟踪哪些节点拥有所需的资源，并将容器调度到该节点。没有足够资源的节点，是不会被调度的；如果集群中的所有节点都没有足够的可用资源，该容器不会被部署，直到某个节点具备足够的资源：

<img src="https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/1.1diaodu.jpg" style="zoom:80%;" />

图1.1-Kubernetes 编排和调度

容器编排使我们无需时时关注主机可用资源，Kubernetes 提供了对这些指标的监控功能。因此开发人员只需声明容器期望使用的资源，而无需担心资源是否可用，Kubernetes 会处理好这些工作。

### 高可用

Kubernetes 的另一个显著特性是为应用程序提供了冗余和高可用。高可用是为了防止应用程序停止服务，由负载均衡器在应用程序的多个实例间分配流量。高可用的前提是，如果应用程序的一个实例宕机了，其他实例仍然可以处理请求，这就避免了停服的可能，并且不论其中的一个或多个实例宕机，对客户端来说完全无感知。Kubernetes 提供了 Service 这种网络机制，实现在多个实例间的负载均衡。在 1.3 章节中，我会详细介绍 Service。

### 可扩展

考虑到容器和微服务的轻量级特性，运维人员可以使用 Kubernetes 快速伸缩应用的负载能力，无论是水平还是垂直。

水平扩展是部署更多的容器实例副本数，如果你评估到运行在 Kubernetes 上的某个应用程序最近负载会增大，可以部署更多的副本。这个过程很简单，无需关注应用程序会被部署到哪些节点上，Kubernetes 是一个容器编排工具，它会根据资源需求自动完成这一过程，并将新部署的实例加入到负载均衡池中，保持其高可用。

垂直扩展是为应用程序分配更大的内存和 CPU 资源，运维人员可以在应用程序运行时修改其资源需求。Kubernetes 会重新部署正在运行的实例，并把它们调度到满足新资源需求的节点上。根据不同的配置，Kubernetes 可以在不停服的情况下重新部署每个实例。

### 社区活跃度

Kubernetes 拥有活跃的开源社区，经常收到补丁和新特性。社区还对文档做出了很多贡献，包括官方文档以及专业或业余爱好者的博客网站、公众号。除了文档，社区还通过积极参与计划和参加世界各地的聚会和会议，进行技术分享和创新。

Kubernetes 大型社区还构建了大量的工具来增强 Kubernetes 的功能，Helm 就是其中之一。正如我们将在本章以及整本书中看到的：Helm——Kubernetes 社区成员构建的工具，简化了 Kubernetes 中应用程序部署和生命周期管理，极大改善了开发、运维人员的体验。

了解了 Kubernetes 为技术人员带来的帮助之后，我们来看看如何在 Kubernetes 中部署应用程序。

## Kubernetes 部署应用程序

不论应用程序部署在 Kubernetes 上，还是部署在非 Kubernetes 环境，根本上是相似的，都需要以下要素：

* 网络
* 持久化存储和文件挂载
* 可用性和冗余
* 应用配置
* 安全

可以与 Kubernetes API(Application Programming Interface 应用程序编程接口)交互，在 Kubernetes 集群中配置这些要素。Kubernetes API 是一组端点(Endpoint)，通过这些端点可以查看、修改、删除 Kubernetes 资源，很多 Kubernetes 资源用来配置应用程序的部署细节。

接下来介绍一些基本的 API 端点，以便在 Kubernetes 上部署和配置应用程序。

### Deployment

Deployment 定义了在 Kubernetes 集群中部署应用程序所需的基本细节，如使用哪个镜像。容器镜像可以使用 docker 在本地构建，也可以使用 kaniko 在 Kubernetes 上构建。

除了指定容器镜像外，Deployment 还定义了部署应用程序的副本数。创建 Deployment 时，会生成名为 ReplicaSet 的中间资源。ReplicaSet 部署的应用程序副本数由`replicas`字段定义。应用程序部署在容器中，容器被封装在 Pod 里。Pod 是 Kubernetes 中的最小单元，它包含至少一个容器。

Deployment 还可以配置应用程序的资源限制、运行状况检查和卷挂载。Deployment 创建后的架构如下图：

<img src="https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/1.2dtcreate.jpg" alt="1.2dtcreate" style="zoom:80%;" />

图1.2- Deployment 创建一组 Pods

### Service

Deployment 部署了一组 Pods(一个或多个)，如果我们直接访问这些 Pods，会有很多问题，如不能进行负载均衡、更新 Pods 后地址会改变。这时就需要引入另一个资源 Service 了。Service 的 IP 是固定的，功能类似负载均衡器，把流量分发到后端的 Pods 上，提供负载均衡和高可用的作用。

下图是一个 Service 的示例结构，Service 位于客户端和 Pod 之间，提供负载均衡和高可用的作用：

<img src="https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/1.3Service.jpg" alt="1.3Service" style="zoom:80%;" />

图1.3-Service 负载均衡器

### PersistentVolumeClaim

微服务风格的应用程序生命周期比较短暂，不具备持久化存储。然而在实际环境中，许多应用程序产生的数据，是要在容器生命周期之外，要持久化存储的。Kubernetes 通过一个子系统封装了如何提供存储以及如何使用存储的底层细节，从而解决了这一问题，用户可以通过 **PersistentVolumeClaim** 端点为应用程序分配持久化存储。Kubernetes 运维人员负责静态分配存储(PersistentVolume)、或使用 StorageClass 动态分配存储(它分配PersistentVolume以响应 PersistentVolumeClaim 端点)。**PersistentVolume** 捕获所有必需的存储细节，包括类型(例如NFS、Ceph)、存储的大小。从用户的角度无论在集群中使用哪种 **PersistentVolume**，都不需管理底层存储服务。利用 Kubernetes 的持久化存储功能，可以增加 Kubernetes 平台上可部署的应用程序数量。

下图是 StorageClass 动态分配存储示例：

<img src="https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/1.4-storageclass.jpg" alt="1.4-storageclass" style="zoom:80%;" />

图1.4-PersistentVolume created by PersistentVolumeClaim

除了以上这些，Kubernetes 还有很多资源类型，但这些已满足我们本书的使用了(更详细的 Kubernetes 知识，可以参考我的**Kubernetes 从入门到实战**系列文章)。

下一节，我们来学习如何创建资源。

## 资源管理

在 Kubernetes 集群中部署应用程序时，我们首先要和 Kubernetes API 进行交互。在命令行界面，**kubectl** 是主流的 Kubernetes API 交互工具。

我们来看看 **kubectl** 是如何管理 Kubernetes 资源的。

### 命令式和声明式

**kubectl** 包含很多子命令，以命令式的方式创建和修改资源。下面是几个常用命令：

* create
* describe
* edit
* delete

kubectl 命令格式如下：

*kubectl <verb> <noun> <arguments>*

verb 是 kubectl 的子命令，noun 指特定的 Kubernetes 资源。以下是创建 Deployment 资源的命令示例：

```shell
kubectl create deployment my-deployment --image=busybox
```

这条命令会指示 Kubernetes API 创建一个名为 “my-deployment” 的 Deployment 资源，并使用来自 Docker Hub 的 busybox 镜像。

可以通过 kubectl 的 **describe** 子命令查看 Deployment 的详细信息：

```shell
kubectl describe deployment my-deployment
```

这条命令会格式化的输出 Deployment 的相关信息，我们可以通过这条命令查看 Deployment 的状态。

如果需要对 Deployment 进行修改，可以通过 **edit** 子命令：

```shell
kubectl edit deployment my-deployment
```

这条命令将打开一个文本编辑器，你可以像使用 **vim** 一样修改 Deployment。

需要删除某个资源时，可以使用 **delete** 子命令：

```shell
kubectl delete deployment my-deployment
```

执行此命令，会删除名为 “my-deployment” 的 Deployment 资源。

Kubernetes 资源一旦创建，将以 JSON 文件的格式存在于集群中，可以将其导出为 YAML 文件，以提高可读性。YAML 格式的资源示例如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
        - name: main
          image: busybox
          args:
            - sleep
            - infinity
```

这是一个基础的 yaml 示例，使用来自 Docker Hub 的 busybox 镜像，并无限期地运行 sleep 命令以保持 Pod 运行。

kubectl 是以命令式的方式创建资源的，看上去更简单一些，但在实际使用中，我们更多的是使用 yaml 文件。kubectl 子命令不能配置所有的资源选项，但通过 yaml 文件可以，并且也更灵活。更重要的一点是，yaml 文件是以声明式的方式创建资源的。

在声明式创建资源时，用户首先以 yaml 格式写出要创建的资源，再使用 kubectl 工具将资源应用于 Kubernetes API。在命令式配置中，开发人员使用 kubectl 诸多子命令来管理资源，而声明式配置主要依赖于一个 apply 子命令。

声明式配置通常使用以下命令实现：

```shell
kubectl apply -f my-deployment.yaml
```

使用此命令，Kubernetes 会根据资源是否存在来推断要对资源执行的操作（创建或修改）。

配置声明式应用程序的步骤：

1. 创建名为 my-deployment.yaml 的文件，并将以下 yaml 格式的内容写入该文件：

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: busybox
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: busybox
     template:
       metadata:
         labels:
           app: busybox
       spec:
         containers:
           - name: main
             image: busybox
             args:
               - sleep
               - infinity
   ```

2. 使用以下命令创建 Deployment：

   ```shell
   kubectl apply -f my-deployment.yaml
   ```

   执行此命令后，Kubernetes 将按照 yaml 内容创建 Deployment。

3. 如果要修改 Deployment，例如将副本数修改为 2，可以通过修改 my-deployment.yaml 文件实现：

   ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: busybox
  spec:
    replicas: 2 # 修改此处的数值
    selector:
      matchLabels:
        app: busybox
    template:
      metadata:
        labels:
          app: busybox
      spec:
        containers:
          - name: main
            image: busybox
            args:
              - sleep
              - infinity
   ```

4. 然后执行 kubectl apply 命令应用修改：

   ```shell
   kubectl apply -f my-deployment.yaml
   ```

   执行此命令，Kubernetes 会基于原有部署声明 Deployment，副本数将由 1 扩展为 2。

5. 在删除应用程序时，同样使用 delete 子命令：

   ```shell
   kubectl delete -f my-deployment.yaml
   ```

6. 可以通过 delete -f 标志指定文件，该文件提供了要删除的资源名称，并且该 YAML 文件可以反复使用。

了解了Kubernetes资源是如何创建的，现在我们讨论一下资源配置中涉及的一些挑战。

## 资源配置中的挑战

上一节介绍了 Kubernetes 资源的两种配置方式(命令式和声明式)，接着我们聊聊使用这两种方式会遇到的挑战。

### Kubernetes 资源类型多

Kubernetes有很多很多不同的资源类型，以下是工作中常用的资源类型：

* Deployment
* StatefulSet
* Service
* Ingress
* ConfigMap
* Secret
* StorageClass
* PersistentVolumeClaim
* ServiceAccount
* Role
* RoleBinding
* Namespace

这么多资源类型构成了 Kubernetes 的强大功能，同时也提高了 Kubernetes 的使用复杂度。在 Kubernetes 上部署应用程序时，首先需要确定使用哪些资源，只有深入了解了这些资源，才能进行正确的配置。这需要丰富的 Kubernetes 知识，对使用者来说还是有一定难度的，但这仅仅是更多挑战的开始。

### 保持运行时和本地状态一致

为方便团队间协同工作，可以将配置 Kubernetes 资源的 YAML 文件存储在类似 GitLab 的代码仓库中，然后根据代码仓库中的 YAML 文件部署应用程序。这一过程看似简单，但需要调整资源时，会发生什么呢？正确的做法是修改本地文件，并 **apply** 修改后的文件，以确保本地状态与运行时状态一致。但也会有例外发生，为了方便，有人可能通过 **kubectl patch** 或者 **kubectl edit** 命令跳过本地文件，直接修改运行时。这就会使本地状态和运行时状态不一致，导致后期维护与扩展变得更困难。

### 应用程序生命周期难以管理

姑且将应用程序的安装、升级、回滚定义为应用程序的生命周期吧。在 Kubernetes 中安装应用程序，会创建用于部署和配置应用程序所需的资源。第一次安装应用程序，将创建我们这里所说的该应用程序的第一个版本 **version1**。

因此，升级应用程序，可以被视为是对这些资源的一次或多次编辑。每一次编辑都可以看作是一次升级。运维人员可以修改 Service 资源，版本号就会变为 **version2**。之后，运维人员又修改了 Deployment、ConfigMap 和 Service，版本号将变为 **version3**。

随着新版本持续迭代，跟踪版本号愈发困难。Kubernetes 没有提供固定保存版本变化历史的方法，这不利于版本回滚。假设运维人员之前对某个资源做了不正确的更改，团队如何知道回滚到哪个版本？“n - 1” 的情况比较容易确定回滚到的版本，但最近的正确版本如果是最近五个之前的版本呢？团队就只能把生产当测试，在生产环境匆忙处理故障，因为无法识别哪个是最近的正确版本，从而通过版本回滚来快速解决生产环境问题。

### 资源文件是静态的

把它也列为挑战之一，是因为它影响 YAML 声明式配置的应用。在使用声明式时，Kubernetes 资源文件本身不支持参数化。有些资源文件可能很长，包含许多可定制字段，而且完整地编写、配置 YAML 文件非常麻烦。

可以把静态文件做成模板。模板就是框架，在不同但相似的上下文中保持一致的文本或代码，我们只需填入自己需要的内容。运维人员管理不同的应用程序，需要维护不同的 Deployment、Service 等资源，这个工作量会很大，并随着应用程序的数量成倍增加，因为一个应用程序，需要引入多个 Kubernetes 资源。但我们比较不同的应用程序资源文件时，可以发现它们之间有大量类似的 YAML 配置。

下图是两个资源的 YAML 文件对比示例，它们之间有大量的相同配置，这些相同的部分就是模板。蓝色文本表示模板行，红色文本表示可定制的行：
<img src="https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/1.5muban.jpg" alt="1.5muban" style="zoom:80%;" />

图1.5 - 两个有模板的资源配置

在这个示例中，两个文件几乎完全相同。对于管理声明式应用程序的团队来说，模板文件会成为一个让人头痛的问题。

## Helm 脚踏七彩祥云而来

Kubernetes 社区也意识到了通过创建、维护 Kubernetes 资源部署应用程序的复杂度，为了帮助使用者客服这些挑战，一个简单、强大、开源、能够在 Kubernetes 上部署应用程序的工具诞生了。这个工具就是 Helm，用于在 Kubernetes 上打包和部署应用程序。因为它与操作系统上的包管理器有很多相似之处，它也被称为 Kubernetes 的包管理器。Helm 在 Kubernetes 社区被广泛使用，是 CNCF 的毕业项目。

鉴于 Helm 与操作系统包管理器的相似之处，并且操作系统包管理器又是我们或多或少使用过的。为了便于大家理解，我们先以操作系统包管理器为切入点，作为学习 Helm 的开始。

### 包管理器

包管理器用于简化操作系统中应用程序的安装、升级、恢复和删除过程。这些应用程序以包为单元定义，包包含目标软件及其依赖项的元数据。

包管理器的工作过程很简单。首先用户将需要安装的软件包名称作为参数传递，包管理器在包存储库中查找，如果找到，包管理器将包定义的应用程序及依赖项安装到操作系统的指定位置。

包管理器让管理软件变得非常容易，举个例子，我们在 Rocky Linux 8 系统上安装 **htop** 包(htop 命令可以查看主机资源使用情况)，只需执行以下命令：

```shell
dnf -y install htop
```

执行此命令，dnf 会在 Rocky Linux 包存储库中查找并安装 htop，同时也会安装 htop 的依赖包(如果有依赖的话)，所以你无需担心依赖项。

如果出现了 htop 的新版本，可以通过 upgrade 子命令安装更新：

```shell
dnf upgrade htop -y
```

这条命令会升级 htop 及其依赖包到最新的版本。

包管理器还支持将包回滚到以前版本：

```shell
dnf downgrade htop -y
```

在更新的版本存在漏洞或错误时，回滚功能就显得尤为重要了。

删除安装的包：

```shell
dnf remove htop -y
```

> 注意：
>
> 1. Rocky Linux 是 CentOS 项目的创始人 **Gregory Kurtzer** 领导开发的，用以替换 CentOS。红帽宣称 CentOS 8 将在2021年底停止维护。
> 2. 在 CentOS 7 及其之前的版本中，使用 **yum** 作为包管理，在 CentOS 8 及 Rocky Linux 8 中，包管理器为 **dnf**，两者使用方式相似。
> 3. 对学习 Rocky Linux 系统感兴趣的读者，可以阅读我的《Rocky Linux最佳实践》。

Helm 作为 Kubernetes 的软件包管理器，作用与 dnf 相似。dnf 用来管理 Rocky Linux 上的应用程序，Helm 用来管理 Kubernetes 上的应用程序。我将在下文中对此进行更详细的介绍。

### Kubernetes 的包管理器

虽然 Helm 和 dnf 同为包管理器，但实现的细节还是有些差异的。dnf 操作的是 RPM 包，Helm 处理的 Charts。Helm 的 Charts 可以认为是 Kubernetes 的 RPM 包。Charts 包含部署应用所需的声明性资源文件，与 RPM 类似，它也可以包含应用程序的依赖项。

Helm 通过存储库获取 Charts。运维人员在创建完成声明性的 YAML 文件后，打包为 Charts，然后将 Charts 包存放在存储库中，用户使用 Helm 搜索要部署到 Kubernetes 上的 Charts，这类似用户使用 dnf 搜索要部署到 Rocky Linux 的 RPM 软件包。

以部署 Redis 为例，我们使用已发布到存储库中的 Redis Chart，通过 Helm 将其部署到 Kubernetes 上：

```shell
helm install redis bitnami/redis --namespace=redis
```

这条命令把 bitnami chart 仓库中的 redis chart 安装到 Kubernetes 的 redis 命名空间中。这是初始安装。

如果有新版的 Redis Chart 可用，可以使用如下命令升级到新版本：

```shell
helm upgrade redis bitnami/redis --namespace=redis
```

如果发现新版本存在 bug，用户会比较关心如何回滚，Helm提供了 rollback 命令来处理回滚操作：

```shell
helm rollback redis 1 --namespace=redis
```

这个命令会将 Redis 回滚到第一个版本。

当你不再需要 Redis 的时候，可以通过 uninstall 命令删除：

```shell
helm uninstall redis --namespace=redis
```

了解了 Helm 作为包管理器的功能后，我们更详细的讨论下 Helm 给 Kubernetes 带来的好处。在上一节，我们讨论了创建 Kubernetes 资源时的挑战，现在来看看 Helm 是如何解决这些挑战的。

#### Kubernetes 资源的抽象复杂性

假设我们接到了在 Kubernetes 上部署 MySQL 的任务，我们需要配置创建容器、网络、存储所需的资源。从头配置这样的应用程序，有很高的 Kubernetes 相关技术要求，对于新手或者经验不丰富的用户来说，会是一个不小的挑战。

有了 Helm，我们可以从 Charts 库中搜索 MySQL Charts。这些 Charts 由社区中的 Charts 开发人员编写，包含了部署 MySQL 所需的声明性配置。所以我们可以直接使用 Helm 安装MySQL，非常简单。

#### 历史版本

Helm 有一个概念叫做发布历史。当第一次安装 Helm chart 时，Helm 会记录一个初始版本。随着版本更新，历史记录也随着修改，从而保存了应用程序不同版本的快照。
下图是版本更新的历史记录，蓝色方块表示更新的版本：

<img src="https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/1.6-Upgrade.jpg" alt="1.6-Upgrade" style="zoom:80%;" />

图1.6 - 版本历史

记录每次版本更新便于回滚操作。Helm 中的版本回滚非常简单，用户只需指向先前的版本，Helm 就会将运行状态恢复到所选版本。有了 Helm，n-1 备份的日子已经一去不复返了。Helm 可以回滚到任意的版本，甚至可以回滚到应用第一次部署时的状态。

#### 配置动态声明式资源

Kubernetes 的声明式资源文件是静态的，无法参数化，哪怕相似度很高的两个资源文件，都得独立配置，增加了很多工作量和维护成本。Helm 引入了 values 和 template 解决了这个问题。

values 是 Helm Charts 的参数，template 是基于 values 动态生成的资源文件，有了它们，Helm 管理应用程序就灵活多了，并且更易于维护。

values 和 template 允许用户执行以下操作：

* 参数化公共字段，例如 Deployment 中的镜像名称和 Service 中的端口
* 根据用户输入生成较长的 YAML 配置片段，例如 Deployment 中的卷挂载或 ConfigMap 中的数据
* 包含或排除资源

动态生成声明式资源文件使创建基于 YAML 格式的资源变得更简单，又容易复制。

#### 本地状态和运行时一致性

包管理器有效避免了用户手动修改应用程序及依赖，所有对应用程序包及依赖的修改，都可以通过包管理器完成。这个优势同样适用于 Helm，因为 Helm charts 包含了 Kubernetes 资源的各种配置，所以用户无需直接修改 Kubernetes 的资源。如果用户需要修改应用程序，可以通过向 Helm charts 提供新的 values，并对版本进行更新来实现。这有效保证了本地状态和运行时状态在修改过程中的一致性。

#### 智能部署

Helm 能够识别要创建的 Kubernetes 资源的先后顺序，简化应用程序部署过程。Helm 分析 Charts 包含的资源，并根据资源类型排序，避免因依赖关系而导致资源部署失败。例如 Secret 和 Configmap 应在 Deployment  之前创建，因为 Deployment 可能会以卷的形式挂载这些资源。Helm 无需用户干预就能按先后顺序创建它们，这种复杂性被抽象出来，用户无需关心这些资源的创建顺序。

#### 生命周期 hooks 管理

与其他包管理器类似，Helm 提供了定义生命周期钩子(hooks)的能力。生命周期钩子可以在应用程序生命周期的不同阶段自动触发事先定义好的操作。它们可以做以下事情：

* 升级时执行备份
* 回滚时恢复数据
* 在安装应用程序前验证 Kubernetes 环境

生命周期钩子很有用，因为它们将任务的复杂性抽象出来，而这些任务可能不是 kubernetes 特有的。例如，Kubernetes 用户可能不熟悉数据库备份的最佳实践，或者不清楚何时该执行这样的任务。生命周期钩子可以执行专业人员编写的最佳实践的自动化程序，这样用户就可以继续高效地专注于自己的工作，而不必担心其他细节。
