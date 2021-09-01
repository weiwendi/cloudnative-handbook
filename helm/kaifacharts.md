# 5. 开发 Helm 图表

通过上一章的学习，你对 Helm 图表有了更深入的了解，现在是时候自己开发一个 Helm 图表来检验所学了。学会构建 Helm 图表，就可以更便捷的在 Kubernetes 集群中部署复杂的应用程序了。

为了便于大家学习开发 Helm 图表，我编写了 player-stats 应用程序 ，我们就以此程序为例，完成 Helm 图表开发。开发过程会遵循 Kubernetes 和 Helm 图表的最佳实践，以提供编写良好且易于维护的代码，在整个过程中，你将学到许多构建图表的技能。在本章的最后，会介绍如何打包图表并上传图表到图表存储库，以便让其他用户使用我们开发的图表。

## 5.1 player-stats 应用程序介绍

在本章中，你将创建一个部署 player-stats 应用程序的 Helm 图表。Player-stats 使用 Go 开发，后端存储是 MongoDB，感兴趣的读者可以在 https://github.com/weiwendi/player-stats 页面查看 player-stats 的源码及介绍。部署完成后，可以在页面录入球员的姓名及球衣号码，这些信息会存储到 MongoDB 数据库中，并展示在前端页面。这种架构很常见，很适合作为学习开发 Helm 图表的示例。

Payer-stats 的前端页面由对话框和提交按钮组成，用于将消息持久化到后端的 MongoDB。

要与此应用程序交互，用户可以遵循以下步骤：

1. 在“球员姓名”和“球衣号码”对话框中输入球员信息。
2. 点击“添加信息”按钮提交信息。
3. 信息被保存到 MongoDB 数据库。

MongoDB 是一个基于分布式文件存储的数据库，在本章中，我们以单实例的方式部署 MongoDB。

在基本了解 player-stats 应用程序后，我们开始开发 Helm 图表。在此之前，要确保使用 Minikube 部署的 Kubernetes 集群是可用的。

## 5.2 环境准备

按照以下步骤创建 minikube 环境：

1. 使用 **minikube start** 启动 minikube：

   ```bash
   [aiops@aiops0113 ~]$ minikube start
   ```

2. 在 Kubernetes 集群中创建一个名为 website 的命名空间，用于部署留言板应用程序：

   ```bash
   [aiops@aiops0113 ~]$ kubectl create namespace website
   ```

现在环境已经准备好了，我们开始开发图表。

## 5.3 创建 player-stats 图表

在本节中，我们创建 player-stats 应用程序的 Helm 图表，以便在 Kubernetes 集群中部署该应用。图表的最终版本我已经发布到了 https://github.com/weiwendi/learn-helm/tree/main/helm-charts/charts/player-stats 图表存储库中，在跟随本章学习时，可以随时参考。

开发图表前，需要搭建图表初始文件结构。

### 5.3.1 搭建初始文件结构

上一章我们介绍了图表必须遵循一个特定的文件结构。也就是说，一个图表必须包含以下文件：

* **Chart.yaml**：定义图表元数据
* **values.yaml**：定义图表的默认值
* **templates/**：定义创建 Kubernetes 资源的图表模板

在上一章中，我们已经看到了一个图表可能包含的文件列表，前三个文件是开发图表所必需的。虽然这三个文件可以手动创建，但 Helm 提供了 **helm create** 命令，可以快速创建图表。

**helm create** 命令除了自动创建前面列出的三个文件外，还会生成许多不同的模板文件，开发人员可以利用这些模板更快的编写 helm 图表。现在，我们使用此命令创建一个名为 player-stats 的图表。

**helm create** 命令使用 Helm 图表的名称作为参数，此处的 Helm 图表名称是 player-stats。我们在命令行上创建这个图表：

```bash
[aiops@aiops0113 ~]$ cd helm-charts/

[aiops@aiops0113 chart]$ helm create player-stats
Creating player-stats
```

此命令执行后，会在当前目录生成一个名为 player-stats/ 的目录，这个就是 player-stats 应用程序的图表目录。在这个目录中，你能看到以下四个文件或目录：

* **charts/**
* **Chart.yaml**
* **templates/**
* **values.yaml**

如你所见，除了所需的 Chart.yaml、values.yaml、templates/，**helm create** 命令还创建了 charts/ 目录。charts/ 目前是一个空目录，但当我们声明图表依赖关系时，该目录下会生成对应的文件。你可能还注意到，前面提到的其他文件已经使用默认值进行了填充。在本章中，我们将在开发 player-stats 图表时使用部分默认值。

如果你查看 templates/ 目录下的内容，会发现默认包含了许多不同的模板资源，这些资源大大节省了我们的开发时间。虽然生成了许多有用的模板，但我们需要删除 templates/test/ 文件夹，这个文件夹用来存放 Helm 图表的测试文件，我会在下一章介绍如何对 Helm 图表进行测试。现在，我们先删除该目录：

```bash
[aiops@aiops0113 chart]$ rm -rf player-stats/templates/tests
```

我们已经搭建好了 player-stats 图表，接着配置 Chart.yaml 文件。

### 5.3.2 配置图表定义

图表定义或者 Chart.yaml 文件，包含着 Helm 图表的元数据。在上一章中，我介绍了 Chart.yaml 文件中的每一个选项，现在我们一起回顾下图表定义的主要设置：

* **apiVersion**：设置为 v1 或 v2，使用 Helm3 时，v2 是首选项
* **version**：Helm 图表的版本，应该遵循 Semver 规范
* **appVersion**：Helm 图表部署的应用程序的版本
* **name**：Helm 图表的名称
* **description**：对 Helm 图表的描述信息
* **type**：设置为 application 或 library。Application 图表用于部署应用程序；Library 图表包含一组辅助函数(也称为“命名模板”)，可以跨其他图表使用，以减少相同模板的编写。
* **dependencies**：图表所依赖的图表列表

查看 player-stats 图表中默认生成的 Chart.yaml 文件，你会发现除了依赖项，其它字段已经设置好了。我们使用这些默认值即可。

图表定义中默认没有设置依赖项，在下一节，我们为 player-stats 图表添加 MongoDB 依赖项，来简化我们的开发工作。

### 5.3.3 添加 MongoDB 图表依赖

Player-stats 应用程序前端页面将数据存储到后端的 MongoDB 里，所以我们的 Helm 图表必须能够部署 MongoDB。如果从头创建 MongoDB 的 Helm 图表，除了所需的图表模板，还要对 MongoDB 有一定的了解，以及要知道如何将 MongoDB 部署到 Kubernetes 上。

但通过引入 MongoDB 的依赖，可以大大减少开发 player-stats 图表所涉及的工作量，因为它已经包含了所需的图表模板以及部署逻辑。我们修改 Chart.yaml 文件，添加 MongoDB 依赖。

添加 MongoDB 图表依赖的过程大致需要以下步骤：

1. 在 Helm 图表存储库中搜索 MongoDB 图表：

   ```bash
   [aiops@aiops0113 player-stats]$ helm search hub mongodb
   ```

2. 搜索结果中会出现 Bitnami 的 mongodb 图表，我们使用这个图表作为依赖。如果你没有添加 bitnami 图表库，可以使用 **helm repo add** 命令添加：

   ```bash
   $ helm repo add bitnami https://charts.bitnami.com/bitnami
   ```

3. 确认想要安装的 MongoDB 图表版本：

   ```bash
   [aiops@aiops0113 player-stats]$ helm search repo mongodb --versions
   NAME                   	CHART VERSION	APP VERSION
   bitnami/mongodb        	10.15.2      	4.4.6      
   bitnami/mongodb        	10.15.1      	4.4.5      
   bitnami/mongodb        	10.15.0      	4.4.5      
   bitnami/mongodb        	10.14.0      	4.4.5      
   ```

   需要指定的是图表版本，而不是 APP 版本。APP 版本表示 MongoDB 的版本，而图表版本表示的是 Helm 图表的版本。

   在依赖关系中可以指定具体的图表版本，或者通配符，比如 10.15.x。使用通配符可以轻松地将图表更新到通配符匹配的最新 MongoDB 版本(本例中通配符匹配的版本是 10.15.2)。在本例中，我们使用 10.15.x 版本。

4. 将 dependencies 字段添加到 Chart.yaml 文件中，对于 player-stats 图表，我们仅做最基本的配置：

   **name**：依赖的图表名称

   **version**：依赖的图表版本

   **repository**：依赖的图表仓库的URL

   具体配置如下：

   ```yaml
   dependencies:
     - name: mongodb
       version: 10.15.x
       repository: https://charts.bitnami.com/bitnami
   ```

添加完依赖后，完整的 Chart.yaml 文件配置如下：

```yaml
apiVersion: v2
name: player-stats
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
dependencies:
  - name: mongodb
    version: 10.15.x
    repository: https://charts.bitnami.com/bitnami
```

也可以在 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/Chart.yaml 页面查看该文件(请注意 version 和 appVersion 字段可能不同，因为我们会在本章的后面修改它们)。

既然依赖项已添加到图表定义中了，我们下载此依赖项以确保它已正确配置。

### 5.3.4 下载 MongoDB 图表依赖

第一次下载依赖时，应该使用 **helm dependency update** 命令，此命令会将依赖项下载到 charts/ 目录，并生成 Chart.lock 文件，该文件记录了已下载的图表元数据。

 **helm dependency update** 以 Helm 图表位置作为参数，如下所示：

```bash
[aiops@aiops0113 chart]$ helm dependency update player-stats
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading mongodb from repo https://charts.bitnami.com/bitnami
Deleting outdated charts
```

查看 charts/ 文件夹下是否出现了 MongoDB 图表以验证执行结果：

```bash
[aiops@aiops0113 chart]$ ls player-stats/charts/
mongodb-10.15.2.tgz
```

现在已经包含了 MongoDB 依赖，接下来我们修改 values.yaml 值文件。

### 5.3.5 修改 values.yaml 值文件

Helm 图表中的 values.yaml 文件为图表模板提供了一组默认值，当然，用户在与 Helm 图表交互时，可以使用 **--set** 或 **--values** 标志覆盖这些值，但还是要尽量通过 values.yaml 文件来修改这些值，以便后续维护。除了提供默认值之外，还可以为每个值添加详细、直观的注释，方便使用者和维护者通过 values.yaml 文件快速了解图表的值。

**helm create** 命令生成的 values.yaml 文件，包含了许多模板中会使用到的值。我们需要在此文件中添加一些值，来配置 MongoDB 依赖。

#### 5.3.5.1 配置 MongoDB 图表的值

使用 MongoDB 依赖项省去了很多麻烦，但让它更好的工作，我们还需覆盖它的一些值。首先我们看看 MongoDB 图表都包含了哪些值：

```bash
[aiops@aiops0113 player-stats]$ helm show values charts/mongodb-10.15.2.tgz |more
```

输出的值列表太长，我们使用 more 分屏显示。我们来看看哪些值需要覆盖：

1. 需要覆盖的第一个值是 fullnameOverride：

   ```yaml
   ## String to fully override mongodb.fullname template
   ##
   # fullnameOverride:
   ```

   这个值会被名称为 $CHART_NAME.fullname 的命名模板引用。当为 fullnameOverride 设置了值，命名模板会被赋予此值。否则，此模板将使用 .Release.Name 或者安装 Helm releas时提供的名称。

   在 player-stats 应用程序中，连接 MongoDB 数据的地址是 “mongo”。MongoDB 的 Service 地址会根据此值设置，所以我们需要把此值设置为 “mongo”。

2. 第二个需要覆盖的值是 auth.enabled

   ```bash
   auth:
     ## Enable authentication
     enabled: true
   ```

   Player-stats 应用程序被编写为无须身份验证访问 MongoDB 数据库，所以我们要把这个值设为 false。

   我们将这些修改的值加到 values.yaml 文件的末尾：

   ```yaml
   mongodb:
     fullnameOverride: "mongo"
     auth:
       enabled: false
   ```

   在上一章中介绍了覆盖依赖项中的值，必须在其图表名称下的块中。这就是为什么将每个值都添加在 mongodb 块下的原因。

   你可以参考 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/values.yaml 的内容，来检验 MongoDB 部分的配置是否正确。

#### 5.3.5.2 配置 player-stats 的值

使用 **helm create** 命令创建项目时，会在 templates/ 目录下自动创建一些模板文件，列表如下：

* **deployment.yaml**：部署 player-stats 应用程序到 Kubernetes 集群。
* **ingress.yaml**：提供一种从 Kubernetes 集群外部访问 player-stats 应用程序的选项。
* **serviceaccount.yaml**：为 player-stats 应用创建专用的 ServiceAccount
* **service.yaml**：用于在 player-stats 应用程序的多个实例间进行负载均衡。还可以提供从 Kubernetes 集群外部访问 player-stats 应用程序的选项
* **_helpers.tp**：提供在整个 Helm 图表中使用的一组通用模板
* **NOTES.txt**：提供在安装完成该应用后访问该应用的说明

每个模板都由图表的值配置，一些默认值不能满足使用需求，我们使用所需值对这些默认值进行替换，替换前，先来看看都需要替换哪些值。

在图表模板中有一个 deployment.yaml 文件，定义了部署时使用的镜像：

```yaml
image: '{% raw %}{{ .Values.image.repository }}{% endraw %}:{% raw %}{{ .Chart.AppVersion }}{% endraw %}'
```

从这行代码中可以看到，镜像由 image.repository 和 AppVersion 确定的。在 values.yaml 文件中可以看到 image.repository 配置的是一个 nginx 镜像：

```yaml
image:
  repository: nginx
```

在 Chart.yaml 文件中，可以看到 AppVersion 的值为 1.16.0：

```yaml
appVersion: 1.16.0
```

player-stats 应用程序的镜像已经被我上传到了镜像仓库，地址是 registry.cn-beijing.aliyuncs.com/sretech/player-stats:0.1.0。

因此为了正确的生成 image 字段，image.repository 的值须设置为 registry.cn-beijing.aliyuncs.com/sretech/player-stats，AppVersion 的值须设置为 0.1.0。

需要修改的第二个图表模板是 service.yaml，在这个文件中，定义了 Service 的类型：

```yaml
type: {{ .Values.service.type }}
```

在 values.yaml 中默认的 service.type 的值为：

```yaml
service:
  type: ClusterIP
```

对于 player-stats 图表，我们把 type 值修改为 NodePort，以创建一个 NodePort 类型的 Service。这样用户就可以通过 minikube 虚拟主机上的 IP 和端口，访问 player-stats 应用程序了。

**helm create** 默认生成的也有 ingress.yaml 模板，但在 minikube 环境中，推荐使用 NodePort 类型的 Service，因为不需要额外安装增强组件。

我们已经确定了需要修改的默认值，现在让我们在 values.yaml 文件中修改它们吧：

1. 修改镜像，image 块配置如下：

   ```yaml
   image:
     repository: registry.cn-beijing.aliyuncs.com/sretech/player-stats
     pullPolicy: IfNotPresent
   ```

2. 替换 Service 的类型为 NodePort，service 块配置如下：

   ```yaml
   service:
     type: NodePort
     port: 80
   ```

3. 可以参考 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/Chart.yaml 页面验证你配置的正确与否。

接着更新 Chart.yaml 文件：

1. 替换 appVersion 字段的值为 v0.1.0：

   ```yaml
   appVersion: "0.1.0"
   ```

2. 可以参考 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/Chart.yaml 页面验证你配置的正确与否。

我们已经正确配置了值，现在试着将其部署到 minikube 中。

### 5.3.6 安装 Player-stats 图表

在 player-stats/ 目录外部执行执行以下命令，安装 player-stats 图表：

```bash
[aiops@aiops0113 helm-charts]$ helm install myplayer-stats player-stats -n website
NAME: myplayer-stats
LAST DEPLOYED: Wed Jun  9 03:41:42 2021
NAMESPACE: website
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace website -o jsonpath="{.spec.ports[0].nodePort}" services myplayer-stats)
  export NODE_IP=$(kubectl get nodes --namespace website -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
```

安装完成后，我们需要等待 player-stats、MongoDB 的 Pod 都处于 READY 状态，才能访问。查看 Pod 状态的命令：

```bash
$ kubectl get pods -n website
```

在执行 **helm install** 时也可以加上 **--wait** 标识，以等待 Pod 处于 READY 状态时提示完成安装。**--wait** 可以与 **--timeout** 一起使用，以秒为单位设置 Helm 的等待时间，默认是 5 分钟，这个时间对于启动应用程序来说足够了。

一旦所有 Pod 处于 READY 状态，就可以执行安装完成时的 Notes 信息提供的命令，如果你没记住，可以使用这个命令再次显示这些信息：

```bash
$ helm get notes myplayer-stats -n website
```

将 echo 输出的 IP:port 粘贴到浏览器里，就可以访问 player-stats 应用程序了。界面如下图所示：

![5.1 player stats page](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/5.1player-stats.jpg)

点击“添加信息”按钮，会跳转到添加信息页面。在对应的框中，输入球员姓名及球衣号码，再次点击“添加信息”按钮，数据就提交到了 MongoDB，页面自动返回到首页，并展示提交的信息。能够完成这些操作，说明 player-stats 部署成功了。

做完上面这些之后，我们就可以使用 **helm uninstall** 命令卸载它们了，这条命令会删除 player-stats 安装的所有相关资源以及 release 历史记录：

```bash
[aiops@aiops0113 helm-charts]$ helm uninstall myplayer-stats -n website
release "myplayer-stats" uninstalled
```

**uninstall** 子命令还有另外三个别名，**del、delete、un**。

## 5.4 改进 player-stats 图表

上一节我们成功部署了 player-stats 应用，本节再增加两个功能：

* 添加备份和恢复数据的生命周期钩子
* 验证提供的值是否有效

### 5.4.1 创建生命周期钩子

在本节中，我们将创建两个生命周期钩子：

1. 第一个钩子发生在 pre-upgrade 阶段，此阶段处于 **helm upgrade** 命令执行后、Kubernetes 资源被修改前。用于在执行升级前对 MongoDB 数据进行备份，以便在升级出错后能够恢复可能损坏的数据。
2. 第二个钩子发生在 pre-rollback 阶段，此阶段处于 **helm rollback** 命令执行后、Kubernetes 资源被还原前。用于恢复 MongoDB 数据，并确保 Kubernetes 资源配置还原到与恢复数据匹配的点。

本节创建的钩子比较简单，仅用于教学，不适合在生产环境使用。不过在完成本节的学习后，你将更加熟悉生命周期钩子，并可以用它们执行一些强大的功能。

让我们来看看如何创建 pre-upgrade 生命周期钩子。

### 5.4.2 创建 pre-upgrade 钩子

我们创建一个 pre-upgrade 钩子，在更新应用程序前对 MongoDB 进行数据备份。MongoDB 的备份机制大体分为“延迟节点备份”和“全量备份 + oplog 增量备份”。我们的示例比较简单，仅是一个 MongoDB 的单节点部署，所以我们使用 **mongodump** 命令对数据库进行全量备份即可。在生产环境中，仅做全量备份是不足以保障数据安全的。

我们创建一个钩子，这个钩子首先会在 Kubernetes 的 website 命名空间中创建 PVC，然后创建 Job，Job 用于执行 **mongodump** 命令，我们把 dump 出的文件存储到新创建的 PersistentVolumeClaim 中。

**helm create** 命令虽然创建了许多模板，但它不会创建钩子，因此我们需要按照以下步骤创建 pre-upgrade 钩子：

1. 创建一个文件夹，存放钩子模板。虽然这不是必须的，但新文件夹有助于将钩子模板与常规的图表模板分开，还可以按功能对钩子分组。

   在 player-stats/ 文件结构中创建名为 template/backup 的文件夹：

   ```bash
   [aiops@aiops0113 helm-charts]$ mkdir player-stats/templates/backup
   ```

2. 创建两个模板文件。persistentvolumeclaim.yaml 模板文件用于创建 PVC，存储 dump 出的数据；job.yaml 模板文件是用于执行 **mongodump** 命令的 Job 模板：

   ```bash
   [aiops@aiops0113 helm-charts]$ touch player-stats/templates/backup/persistentvolumeclaim.yaml
   [aiops@aiops0113 helm-charts]$ touch player-stats/templates/backup/job.yaml
   ```

3. 编辑 persistentvolumeclaim 模板文件内容，也可以从 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/templates/backup/persistentvolumeclaim.yaml 页面复制：

   ``` yaml
   {{- if .Values.mongodb.persistence.enabled }}
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: '{% raw %}{{ .Values.mongodb.fullnameOverride }}'{% endraw %}-data-backup-'{% raw %}{{ sub .Release.Revision 1 }}'{% endraw %}
     labels:
       '{% raw %}{{- include "player-stats.labels" . | nindent 4 }}'{% endraw %}
     annotations:
       "helm.sh/hook": pre-upgrade
       "helm.sh/hook-weight": "0"
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: '{% raw %}{{ .Values.mongodb.persistence.size }}'{% endraw %}
   {{- end }}
   ```

   代码释义：

   * 第一行和最后一行是一个 if 语句块，表示只有在 mongodb.persistence.enabled 的值为 true 时，才会创建此资源。该值在依赖的 MongoDB 图表中默认为 true，可以使用 **helm show values** 命令查看。
   * 第五行定义了 PVC 的名称，这个名称基于 MongoDB 的 PVC 名称 `mongo`，在此名称后添加了 data-backup 及版本号，用于备份时命名再妥帖不过。由于钩子是在升级前执行的，因此它会引用要升级到的版本号，我们使用 sub  函数减去 1 个版本号。
   * 第九行是注解行，声明钩子为 pre-upgrade。
   * 第十行同样是注解行，指定与其它 pre-upgrade 的钩子相比，创建此资源的顺序。权重按升序运算，数值越小的优先被执行，因此该钩子将先于其它 pre-upgrade 资源被创建。

4. 编辑 job.yaml 模板文件，也可以从 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/templates/backup/job.yaml 页面复制：

   ```yaml
   {{- if .Values.mongodb.persistence.enabled }}
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: '{% raw %}{{ include "player-stats.fullname" . }}'{% endraw %}-backup
     labels:
       '{% raw %}{{- include "player-stats.labels" . | nindent 4 }}'{% endraw %}
     annotations:
       "helm.sh/hook": pre-upgrade
       "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
       "helm.sh/hook-weight": "1"
   spec:
     template:
       spec:
         containers:
           - name: backup
             image: bitnami/mongodb:4.4.6-debian-10-r0
             command: ["/bin/sh", "-c"]
             args: ["cd /backup/ && mongodump -h '{% raw %}{{ .Values.mongodb.fullnameOverride }}'{% endraw %}:'{% raw %}{{ .Values.mongodb.service.port }}'{% endraw %}"]
             volumeMounts:
               - name: backup
                 mountPath: /backup
         restartPolicy: Never
         volumes:
           - name: backup
             persistentVolumeClaim:
               claimName: '{% raw %}{{ .Values.mongodb.fullnameOverride }}'{% endraw %}-data-backup-'{% raw %}{{ sub .Release.Revision 1 }}'{% endraw %}
   {{- end }}
   ```

   代码释义：

   * 第九行同样定义了是一个 pre-upgrade 的钩子，第十一行的权重设置为 1，表明其将在权重为 0 的 pre-upgrade 钩子创建之后创建。
   * 第十行是新增注解，定义了何时删除此 Job。 默认情况下，Helm 在初始化创建图表后是不管理钩子的，这意味着执行 **helm uninstall** 命令时不会删除钩子。helm.sh/hook-delete-policy 注解用于定义删除资源的条件。before-hook-creation 删除策略指如果命名空间中存在此钩子，则在执行 **helm upgrade** 命令时将其删除，然后创建新 Job。hook-succeeded 表示 Job 执行成功后，删除该 Job。
   * 第十九行执行 dump MongoDB 数据，通过 **mongodump** 客户端命令，连接到 MongoDB 服务，把数据 dump 到用于备份的 PVC。
   * 第二十六行定义了 backup PVC，PVC 由 Job 挂载，存储 dump 出的文件。

遵循以上步骤，就能完成图表的 pre-upgrade 钩子了。 

### 5.4.3 创建 pre-rollback 钩子

通过 pre-upgrade 钩子成功实现 MongoDB 数据备份后，我们再来编写一个 pre-rollback 钩子，将备份的数据还原到 MongoDB 数据库。

按照以下步骤创建 pre-rollback 钩子：

1. 创建 template/restore 文件夹，存放 pre-rollback 钩子：

   ```bash
   [aiops@aiops0113 helm-charts]$ mkdir player-stats/templates/restore
   ```

2. 创建 job.yaml 模板文件，用于恢复 MongoDB 数据：

   ```bash
   [aiops@aiops0113 helm-charts]$ touch player-stats/templates/restore/job.yaml
   ```

3. 编辑 job.yaml 文件，也可以从 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/templates/restore/job.yaml 页面复制内容：

   ```yaml
   {{- if .Values.mongodb.persistence.enabled }}
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: '{% raw %}{{ include "player-stats.fullname" . }}'{% endraw %}-restore
     labels:
       '{% raw %}{{- include "player-stats.labels" . | nindent 4 }}'{% endraw %}
     annotations:
       "helm.sh/hook": pre-rollback
       "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
   spec:
     template:
       spec:
         containers:
           - name: restore
             image: redis:alpine3.11
             command: ["/bin/sh", "-c"]
             args: ["cd /backup/ &&
               mongorestore -h '{% raw %}{{ .Values.mongodb.fullnameOverride }}'{% endraw %}:'{% raw %}{{ .Values.mongodb.service.port }}'{% endraw %}"]
             volumeMounts:
               - name: backup
                 mountPath: /backup
         restartPolicy: Never
         volumes:
           - name: backup
             persistentVolumeClaim:
               claimName: '{% raw %}{{ .Values.mongodb.fullnameOverride }}'{% endraw %}-data-backup-'{% raw %}{{ .Release.Revision }}'{% endraw %}
   {{- end }}
   ```

   代码释义：

   * 第九行声明了该资源为 pre-rollback 钩子。
   * 执行数据恢复是在 18 和 19 行。18 行切换到存放备份文件的目录；第 19 执行 **mongorestore** 命令恢复数据。

* 第 27 行定义了备份卷，备份卷由回滚的版本号确定。

pre-upgrade 和 pre-rollback 钩子都创建完成了，让我们在 minikube 环境中运行它们，看看效果。

## 5.5 执行生命周期的钩子

为了运行创建好的钩子，需要通过 **helm install** 命令再次安装图表：

```bash
[aiops@aiops0113 helm-charts]$ helm install myplayer-stats player-stats -n website
```

当每个 Pod 都处于 1/1 REDAY 状态时，我们就可以按照提示访问 player-stats 应用程序了。当然这次的端口可能会有变化。

我们在 player-stats 的页面上提交一条消息：球员姓名 Stephen Curry，球员号码 30，然后执行 **helm upgrade** 命令触发 pre-upgrade 钩子：

```bash
[aiops@aiops0113 helm-charts]$ helm upgrade myplayer-stats player-stats -n website
```

命令执行成功后，可以看到两个 PVC：

```bash
[aiops@aiops0113 helm-charts]$ kubectl get pvc -n website
NAME                  STATUS   VOLUME                                  
mongo                 Bound    pvc-8451cb95-7272-4939-a85f-9f26bba4dd60
mongo-data-backup-1   Bound    pvc-f6d41be0-c429-47e1-9e54-11dcf0813649
```

在执行 **helm upgrade** 命令后，我们登录 mongo 容器，进入 mongo 数据库命令行，手动删除从页面添加的数据，模拟数据丢失：

```bash
[aiops@aiops0113 helm-charts]$ kubectl get pod  -n website
NAME                             READY   STATUS    RESTARTS   AGE
mongo-6b555fdbb9-zbqfx           1/1     Running   1          20h
myplayer-stats-545dc849c-2sd5k   1/1     Running   19         20h

[aiops@aiops0113 helm-charts]$ kubectl exec -it mongo-6b555fdbb9-zbqfx sh -n website
# 进入容器后，执行 mongo 命令
$ mongo
> use stats;
> db.infor.deleteOne({name: "Stephen Curry"})
{ "acknowledged" : true, "deletedCount" : 1 }
```

执行完成后，刷新 player-stats 页面，你会发现 Stephen Curry 相关的信息都消失了，页面回到了最初安装时的样子。

在执行 **helm upgrade** 命令后，我们通过手动删除数据，模拟了更新后数据损坏的情况。现在执行 **helm rollback** 命令，恢复数据：

```bash
[aiops@aiops0113 helm-charts]$ helm rollback myplayer-stats 1 -n website
Rollback was a success! Happy Helming!
```

命令执行完成后，再刷新页面，你会发现 Stephen Curry  的信息又回来了。

需要注意的是，在执行生命周期命令时(**helm isntall 、helm upgrade、helm rollback、hell uninstall**) ，可以使用 --no-hooks 标志可以跳过钩子。

## 5.6 添加输入验证

使用 Helm 创建 Kubernetes 资源时，Kubernetes API 服务会自动对输入进行验证。如果 Helm 创建了无效的资源，API 服务将返回错误信息，提示资源创建失败。尽管 Kubernetes 自身会执行输入验证，但仍有图表开发人员希望在资源创建请求到达 Kubernetes API 服务前完成验证。

我们可以使用 Helm 图表中的 fail 函数实现输入验证。

### 5.6.1 fail 函数

如果用户提供的值无效，fail 函数可以即刻呈现出模板中的异常。在本节中，我将演示一个限制用户输入的示例。

在 player-stats 图表的 values.yaml 文件中，有一个 service.type，用于配置 service 的类型：

```yaml
service:
  type: NodePort
```

我们将此值设置为 NodePort，但 service 还可以是其他类型。假设你想将 service 类型限制为仅 NodePort 和 ClusterIP 这两种，可以通过 fail 函数实现。

请参照以下步骤限制 player-stats 图表中的 service 类型：

1. 在 template/service.yaml 文件中，有一行定义了根据 service.type 的值设置 service 类型，如下所示：

   ```yaml
   type: '{% raw %}{{ .Values.service.type }}'{% endraw %}
   ```

   在设置 service 类型前，我们应首先检查 service.type 的值是否为 NodePort 或者 ClusterIP，这可以通过在列表中正确的设置变量来实现。然后判断 service.type 的值是否包含在列表中，如果是，则继续设置 service 类型；否则终止图表渲染，并向用户发送一条错误消息，通知用户输入有效的 service 类型。

2. 在 service.yaml 文件中添加以下内容，你也可以从 https://github.com/weiwendi/helm-charts/player-stats/templates/service.yaml 页面复制这些内容到 **service.yaml** 文件中：

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: '{% raw %}{{ include "guestbook.fullname" . }}'{% endraw %}
     labels:
       '{% raw %}{{- include "guestbook.labels" . | nindent 4 }}'{% endraw %}
   spec:
   ## 要添加的内容 
   '{% raw %}{{- $serviceTypes := list "ClusterIP" "NodePort" }}'{% endraw %}
   '{% raw %}{{- if has .Values.service.type $serviceTypes }}'{% endraw %}
     type: { { .Values.service.type } }
   {{- else }}
     '{% raw %}{{- fail "value 'service.type' must be either 'ClusterIP' or 'NodePort'" }}'{% endraw %}
   {{- end }}
   ## 要添加的内容
     ports:
       - port: { { .Values.service.port } }
         targetPort: http
         protocol: TCP
         name: http
     selector:
       '{% raw %}{{- include "guestbook.selectorLabels" . | nindent 4 }}'{% endraw %}
   ```

   代码释义：

   第 8 行到第 13 行是输入验证。第 8 行创建了一个名为 serviceTypes 的变量，它是一个列表，包含了我们期望的 service 类型。第 9 行到第 13 行是 if 语句块。第 9 行中的 has 函数将检查 service.type 值是否包含在 serviceTypes 中。如果是，执行第 10 行，设置 service 类型；否则将执行第 12 行，使用 fail 函数停止模板渲染，并向用户显示期望的 service 类型。

   现在，我们尝试在升级 my-guestbook 版本时提供无效的 service 类型：

   ```bash
   [aiops@aiops0113 helm-charts]$ helm upgrade myplayer-stats player-stats -n website --set service.type=LoadBalancer
   ```

   如果之前的配置正确，现在应该看到类似以下的消息：

   ```bash
   Error: UPGRADE FAILED: template: player-stats/templates/service.yaml:14:6: executing "player-stats/templates/service.yaml" at <fail "value 'service.type' must be either 'ClusterIP' or 'NodePort'">: error
    calling fail: value 'service.type' must be either 'ClusterIP' or 'NodePort'
   ```

   fail 函数是验证用户输入是否符合某一组约束的好的方法，但在某些情况下，你还需要确保用户一开始提就供了某些值。这可以通过 required 函数实现。

### 5.6.2 required 函数

required 与 fail 有些相似，也用于中断模板渲染。不同之处在于，required 函数确保在渲染图表模板时，值不为空。

回想一下，图表包含了名为 image.repository 的值，如下所示：

```yaml
image:
  repository: registry.cn-beijing.aliyuncs.com/sretech/player-stats
```

这个值指定了我们要部署的镜像，考虑到这个值对图表的重要性，我们可以使用 required 函数，强制要求在安装图表时指定该值。尽管我们在图表中提供了一个默认值，但是如果你想确保用户始终使用自己的镜像，required 函数允许你删除这个默认值。

按照以下步骤，为 image.repository 设置 required 函数：

1. template/deployment.yaml 文件包含了通过 image.repository 设置 image 的行，如下所示：

   ```yaml
   image: '{% raw %}{{ .Values.image.repository }}{% endraw %}:{% raw %}{{ .Chart.AppVersion }}{% endraw %}'
   ```

2. required 函数使用以下两个参数：

   1. 显示是否提示该值的错误消息
   2. 必须提供的值。

   给定这两个参数，需要修改 deployment.yaml 文件中 image.repository 的值：

   ```yaml
         containers:
           - name: {% raw %}{{ .Chart.Name }}{% endraw %}
             securityContext:
               {% raw %}{{- toYaml .Values.securityContext | nindent 12 }}{% endraw %}
             image: "{% raw %}{{ required "value 'image.repository' is required" .Values.image.repository }}{% endraw %}:{% raw %}{{ .Chart.AppVersion }}{% endraw %}"
             imagePullPolicy: {% raw %}{{ .Values.image.pullPolicy }}{% endraw %}
             ports:
               - name: http
                 containerPort: 80
                 protocol: TCP
   ```

   完整示例请参考 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/templates/deployment.yaml 页面。

3. 我们给 **image.repository** 提供一个空值，看看会显示什么结果：

   ```bash
   [aiops@aiops0113 helm-charts]$ helm upgrade myplayer-stats player-stats -n website --set image.repository=''
   ```

   如果之前修改成功，你会看到类似以下的错误提示：

   ```bash
   Error: UPGRADE FAILED: execution error at (player-stats/templates/deployment.yaml:34:21): value 'image.repository' is required
   ```

至此，你已经成功开发了一个 Helm 图表，并完成了生命周期钩子和输入验证！

在下一节中，我将介绍如何在 GitHub 页面创建一个简单的 Helm 图表存储库，用于向其他用户公开你的 player-stats 图表。

## 5.7 上传图表到图表存储库

你已完成了 player-stats 图表的开发，可以将图表发布到存储库中，以便其他用户使用它。我们先来创建一个图表存储库。

### 5.7.1 创建图表存储库

图表存储库包含两个组件：

*  图表的 tgz 归档文件
*  index.yaml 文件，包含了存储库中包含的图表元数据

基础的图表存储库需要维护人员生成 index.yaml 文件，而更复杂的解决方案(如 Helm 社区的 ChartMuseum 工具)则在推送新图表到存储库时动态生成 index.yaml 文件。在本示例中，我们将在 GitHub 页面创建一个简单的图表存储库，存储 player-stats 图表。

创建 GitHub 存储库，需要有 GitHub 账号，如果你已有账号，可以在 https://github.com/login 页面登录；如果没有，需要在 https://github.com/join 页面创建。

登录 GitHub 后，请按照以下步骤创建图表存储库：

1. 在 https://github.com/new 页面创建新的存储库
2. 存储库名称 charts
3. 选中 “Add a README file” 前的复选框，这是必须的，因为 GitHub 不允许创建不包含任何内容的静态站点
4. 我们把存储库设为 Public，这样其他用户就可以访问了
5. 点击 “Create repository” 按钮，创建存储库

创建完成后，进入到存储库的页面，点击 “Code“ 按钮，会出现该存储库的 URL，通过此 URL 将存储库克隆到本地计算机：

```bash
$ git clone $REPOSITORY_URL
```

克隆完成后，继续进行下一步，将 player-stats 图表发布到刚创建的图表存储库。

### 5.7.2 发布 player-stats 图表

Helm 提供了几个命令用于发布图表到存储库。但在执行这些命令前，你可能需要在 Chart.yaml 文件中添加 version 字段的值。版本控制是发布图表过程中一个很重要的环节，发布其它类型的软件也是如此。

修改 Chart.yaml 文件中 version 字段的值为 1.0.0：

```yaml
version: 1.0.0
```

版本修改后，我们把图表打包为一个 tgz 归档文件。在 player-stats 目录的上级目录中执行此命令：

```bash
[aiops@aiops0113 helm-charts]$ helm package player-stats
Successfully packaged chart and saved it to: /home/aiops/learn-helm/helm-charts/charts/player-stats-1.0.0.tgz
```

命令执行成功后，当前目录下会多出一个名为 player-stats-1.0.0.tgz 的归档文件。

当依赖有其他图表时，需要将这些依赖项下载到 charts/ 目录中，再执行 **helm package** 命令进行打包。如果 **helm package** 命令执行失败，请检查 MongoDB 依赖项是否已下载到 charts/ 目录中。如果没有下载，你可以使用 --dependency-update 标志，告知打包命令先下载依赖项，再打包：

```bash
[aiops@aiops0113 helm-charts]$ helm package --dependency-update player-stats
```

图表打包后，使用 **cp** 命令将 **tgz** 文件拷贝到 GitHub 图表存储库的克隆目录中：

```bash
$ cp guestbook-1.0.0.tgz $GITHUB_CHART_REPO_CLONE
```

复制完成后，可以使用 **helm repo index** 命令为 Helm 存储库生成 index.yaml 文件。此命令将 GitHub 图表存储库的克隆目录作为参数，命令如下：

```bash
$ helm repo index $GITHUB_CHART_REPO_CLONE
[aiops@aiops0113 charts]$ helm repo index ../charts
```

执行完成后，在 charts/ 目录中会生成 index.yaml 文件。index.yaml 文件包含了 player-stats 图表的元数据。如果 charts/ 目录中还有其它图表，则它们的元数据也将记录在 index.yaml 文件中。

有了 tgz 和 index.yaml 文件，我们就可以使用 **git** 命令将这些文件推送到 GitHub 了：

```bash
$ git add --all
$ git commit -m 'feat: adding the player-stats helm chart'
$ git push origin main
```

执行 push 命令可能会要求你输入 GitHub 用户名密码，push 命令执行完成后，player-stats 图表就发布在了 GitHub 了。

接下来，我们将 Helm 图表存储库添加到本地 Helm 客户端。

### 5.7.3 添加图表存储库

在添加图表存储库之前，我们需要确定它的 URL，这个 URL 在 “Settings” 选项卡中的 “Pages” 页面可以创建或查看。

一旦知道了图表存储库的 URL，就可以使用 **helm repo  add** 命令在本地添加存储库了：

```bash
[aiops@aiops0113 charts]$ helm repo add aiops https://weiwendi.github.io/charts
```

执行完成后，本地 helm 客户端就可以与 “aiops” 存储库交互了。通过本地已经配置的 “repo” 搜索 player-stats 图表，验证图表是否发布完成：

```bash
[aiops@aiops0113 charts]$ helm search repo player-stats
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION                
aiops/player-stats	1.0.0        	0.1.0      	A Helm chart for Kubernetes
```

返回的结果中赫然显示着 aiops/player-stats 图表。

本章的所有试验都已经完成了，祝贺你又 get 了新技能。最后，让我们把试验环境清理一下。

## 5.8 清理环境

可以通过删除命名空间来清理环境：

```bash
$ kubectl delete ns website
```

最后使用 **minikube stop** 命令停止 Minikube 集群。