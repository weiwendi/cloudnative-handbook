# 8 Helm 与 Operator

Helm 的特性之一是能够同步本地状态和运行时状态。本地状态由 Helm 值文件管理，在执行 **install** 或 **upgrade** 命令时提供这些值，这些值就会被应用到 Kubernetes 集群中的程序运行时状态。在前面的章节中，程序需要更改时，我们就调用了这些命令。

同步这些更改，还有另外一种方法，那就是在集群中创建一个应用程序，定期检查所需的状态是否与当前环境中配置的状态相同。如果状态不匹配，应用程序可以自动修改环境中配置的状态，以匹配所需的状态。这个应用程序被称为 Kubernetes Operator。在本章中，我们将创建一个基于 Helm 的 Operator，它确保本地定义的状态始终与集群的运行时状态相同。如果不相同，Operator 将执行相应的 Helm 命令更新运行时状态。

## 8.1 技术要求

本章使用到的命令如下：

* minikube
* helm
* kubectl

除了这些工具外，你还可以在 GitHub 的 https://github.com/weiwendi/learn-helm 上找到包含相关示例资源的资源库。这个存储库将在本章中被引用。

## 8.2 Kubernetes Operators

自动化是 Kubernetes 平台的核心，正如我在第一章中介绍的，Kubernetes 资源可以通过 **kubectl** 隐式管理，也可以通过 YAML 格式的文件进行声明式管理。一旦使用 Kubernetes Command-Line Interface (CLI) 应用了资源，Kubernetes 就会将集群内资源的当前状态与所需状态匹配，这一过程称为控制循环。这种持续的、无终止的集群状态监控模式是通过使用控制器实现的。Kubernetes 包含了许多原生控制器，示例包括从拦截到 Kubernetes API(应用程序编程接口)请求的准入控制器，以及管理正在运行的 Pod 副本数量的副本控制器(ReplicationController)。

为了提高用户的使用热情，Kubernetes 除了提供扩展基础平台功能的能力，以及提供管理应用程序生命周期的更多智能的方法外，还提出了几个重要的发展趋势，这些趋势引导了 Kubernetes 开发的二次浪潮。首先，定制资源定义(CRDs)的引入使用户能够扩展默认的 Kubernetes API(与Kubernetes平台交互的机制)，以便创建和注册新的资源类型。注册一个新的 CRD 会在 Kubernetes API 上创建一个新的标示性状态传输(RESTful)资源路径。因此，与使用 Kubernetes CLI 执行 **kubectl get pods** 查看所有 Pod 对象类似，假如为名称是 player-stats 的对象类型注册一个新的 CRD， 就可以通过 **kubectl get player-stats** 来查看以前创建的所有 player-stats 对象。随着这一新功能的实现，开发人员可以创建自己的控制器来监视这些类型的 CRs，从而管理可以通过使用 CRDs 描述的应用程序生命周期。

第二个主要趋势是，复杂和有状态的应用程序被更频繁地部署在 Kubernetes 上。和小型、简单的应用程序不同，这类应用程序管理和维护起来通常更麻烦，例如部署多个组件，以及围绕 “day2” 的备份与恢复。这些超出了 Kubernetes 原生控制器的处理能力，因为实现这些需要对该应用程序、或该组服务有深入的了解。使用 CR 来管理应用程序及其组件的模式称为 Operator 模式。该项目由 CoreOS 软件公司于 2016 年首创，目的是获取运维人员管理应用程序生命周期所需的知识。Operators 被打包为容器化的应用程序，部署在 Pod 中，这些应用程序会根据 CRs 对 API 的更改做出对应的反应。

可以使用 Operator Framework 工具箱，基于以下三种技术中的一种编写 Operators：

* Ansible
* Helm
* Go

基于 Go 的 Operators 是利用 Go 语言来实现控制循环逻辑。基于 Ansible 的 Operators 利用 Ansible CLI 工具和 Ansible playbooks。Ansible 是一个自动化工具，我在《配置管理工具》部分有过详细介绍，它的逻辑是在 PlayBooks 的 YAML 文件中编写的。

在本章中，我们将重点关注基于 Helm 的 Operators。Helm Operators 基于 Helm 图表和 Helm CLI 提供的功能子集，来建立它们的循环逻辑，简化了 Helm 用户实现 Operators 的复杂度。

在理解了 Operators 之后，让我们使用 Helm 创建一个属于我们自己的 Operator。

## 8.3 创建Helm Operator

在本节中，我们编写一个基于 Helm 的 Operator，用于安装 player-stats Helm 图表。这个图表可以在图表存储库中找到，页面地址：https://github.com/weiwendi/learn-helm/tree/main/helm-charts/charts/player-stats。

Operator 被构建为用于维护应用程序的控制循环逻辑的容器镜像。下图演示了 Player-stats Operator 在部署后是如何工作的：

![8.1 cr install helm chart](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/8.1%20cr%20install%20chart.jpg)

Player-stats Operator 会不断观察 Player-stats CRs 的变化。当一个 Player-stats CR 被创建，Player-stats Operator 将安装 player-stats 图表。如果删除了 Player-stats CR，则 Player-stats Operator 将删除 player-stats Helm 图表。

## 8.4 设置环境

Operator 需要部署到 Kubernetes 中，因此我们应先通过以下命令来启动 Minikube 环境：

```bash
$ minikube start
```

启动 Minikube 后，创建一个名为 operators 的命名空间：

```bash
$ kubectl create ns operators
```

Player-stats Operator 是作为容器镜像构建的，我们需要有一个存放容器镜像的仓库，以便使用镜像。可以在 Quay (quay.io) 中创建一个镜像存储库，这是一个公共的容器注册中心。如果你有其他存储库的账号，也是可以的。我们还需要准备一个包含构建 Operator 镜像所需工具的本地开发环境。

让我们首先在 Quay 中创建一个新的镜像存储库。

### 8.4.1 创建 QUAY 存储库

在 Quay 中创建存储库，首先你需要有 Redhat 帐户。按照以下步骤创建 Redhat 帐户：

1. 在浏览器中访问 https://quay.io/signin/ 页面，也可以直接使用 Redhat 账号登录。

2. 根据提示进行 Redhat 账号创建。

3. 创建完成后，使用新账号登录 Quay。

   创建了 Quay 账号后，就可以继续操作镜像创建新的镜像仓库了。

   要创建一个新的镜像存储库，选择 Quay 页面右上角的 `+` 图标，然后选择 “New Repository”，如下图所示：

   ![8.2 create quay image repo](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/8.2%20create%20quay%20image%20repo.jpg)

4. 接着会进入Create New Repository 页面，输入以下详细信息创建存储库：

   选择 Container Image Repository 输入 Repository 名称：playerstats-operator。

   选择 Public 按钮，表示访问存储库时不需身份验证，简化 Kubernetes 访问镜像的方式。

   其余选项可以保持默认值。一旦完成，Create Public Repository 页面应该出现如下图所示的内容：

   ![8.3 create public image repository](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/8.3%20create%20public%20image%20repository.jpg)

5. 点击 Create Public Repository 按钮，创建 Quay 镜像存储库。

现在已经创建了存储 Player-stats Operator 镜像的存储库，让我们准备一个环境，其中包含构建 Helm Operator 所需的工具。

### 8.4.2 准备本地开发环境

为了创建 Helm Operator，你至少需要以下 CLI 工具：

* **operator-sdk**
* **docker/podman/buildah**

**operator-sdk** CLI 是一个工具包，用于帮助开发 Kubernetes Operators。它包含了简化 Operator 开发过程的内在逻辑。在本质上，**operator-sdk** 需要一个容器管理工具，用来构建 Operator 的镜像。**operator-sdk** CLI支持**docker、podman、build**等底层容器管理工具。

要安装 **operator-sdk** CLI，只需从它的 GitHub 仓库 https://github.com/operator-framework/operator-sdk/releases 下载软件包安装即可。安装 **docker、podman**或者 **buildah** 的过程就会有所不同，这取决于你使用的操作系统。**operator-sdk** 没有Windows版的。

幸运的是，Minikube VM 可以作为许多不同操作系统开发人员的工作环境，因为它是一个 Linux 虚拟机，并且包含了 Docker CLI。在本节中，我们将把 **operator-sdk** 安装到 Minikube VM 中，并使用这个环境创建 Operator。虽然我们是在 VM 中进行的，但这些步骤适用于所有 Linux 和 Mac 主机。

跟着以下步骤，将 **operator-sdk** 安装到 Minikube VM 上：

1. 使用 **minikube ssh** 命令登录虚拟机：

   ```bash
   $ minikube ssh
   ```

2. 登录 VM 后，下载 **operator-sdk** CLI。编写此文档时最新版本为 1.11.0，我们使用 **curl** 命令下载该版本：

   ```bash
   $ curl -o operator-sdk -L https://github.com/operator-framework/operator-sdk/releases/download/v1.11.0/operator-sdk_linux_amd64
   ```

3. 下载完后后，修改为可执行文件，并移动到PATH路径中：

   ```bash
   $ chmod u+x operator-sdk
   $ sudo mv operator-sdk /usr/bin
   ```

4. 最后，通过运行 **operator-sdk version** 命令验证 **operator-sdk** 安装是否成功：

   ```bash
   $ operator-sdk version
   operator-sdk version: "v1.11.0", commit: "28dcd12a776d8a8ff597e1d8527b08792e7312fd", kubernetes version: "1.20.2", go version: "go1.16.7", GOOS: "linux", GOARCH: "amd64"
   ```

   如果执行该命令没有报错，说明 **operator-sdk** CLI 安装成功。

5. 作为附件步骤，在 Minikube VM 中克隆图表存储库，因为我们稍后将使用 player-stats Helm 图表构建 Helm Operator：

   ```bash
   $ git clone https://github.com/weiwendi/learn-helm.git
   ```

现在，Quay 镜像仓库和 Minikube VM 本地开发环境都准备好了，让我们开始编写 Player-stats Operator。这里有一个 Operator 的参考示例：https://github.com/weiwendi/learn-helm/tree/main/playerstats-operator。

### 8.4.3 Operator 文件结构

与 Helm 图表类似，**operator-sdk** CLI 构建的 Helm Operators 也有一个必须遵守的文件结构，如下表所示：

| 文件或目录   | 定义                                                         |
| ------------ | ------------------------------------------------------------ |
| config       | 存放在集群上启动项目的配置文件                               |
| Dockerfile   | Operator 项目的 Dockerfile，用于构建镜像                     |
| helm-charts/ | 包含 Operator 负责安装的 Helm 图表                           |
| Makefile     | 使用 make 时会引用此文件辅助构建项目                         |
| PROJECT      | 包含了项目的配置信息                                         |
| watches.yaml | 定义 Operator 负责监视的自定义资源的文件，包括 Group、Version、Kind、Helm 图表位置 |

创建 playerstats-operator：

```bash
$ midir playerstats-operator
$ cd playerstats-operator
```

使用 **operator-sdk init** 命令轻松创建 Operator 文件结构

这创建了Nginx -operator项目，专门用于使用APIVersion查看Nginx资源

使用 **operator-sdk init** 命令创建基于 Helm 的 playerstats-operator 项目。playerstats-operator 项目专门用于 APIVersion 查看 Player-stats 资源：

```bash
$ operator-sdk init --plugins helm --domain quay.io --helm-chart ~/learn-helm/helm-charts/charts/player-stats --kind Player-stats

Writing kustomize manifests for you to edit...
Creating the API:
$ operator-sdk create api --kind Player-stats --helm-chart /home/aiops/learn-helm/helm-charts/charts/player-stats
Writing kustomize manifests for you to edit...
Created helm-charts/player-stats
Generating RBAC rules
WARN[0010] The RBAC rules generated in config/rbac/role.yaml are based on the chart's default manifest. Some rules may be missing for resources that are only enabled with custom
 values, and some existing rules may be overly broad. Double check the rules generated in config/rbac/role.yaml to ensure they meet the operator's permission requirements.
```

**--plugins helm** 指定创建基于 Helm 的 operator；**--kind** 指定 CR 的名称为 Player-stats，该值必须以大写字母开头；**--domain quay.io** 指定 operator 的镜像仓库地址为 quay.io；**--helm-chart** 指示 **operator-sdk** CLI 将 player-stats 图表拷贝到 playerstats-operator 目录。

成功创建了 player-stats operator 后，我们构建 Operator 镜像并将其推到 Quay 注册中心。

### 8.4.4 构建 Operator 镜像并推送到 Quay

Makefile 文件中 `IMAGE_TAG_BASE` 的值，定义了镜像的注册中心地址、镜像仓库名称以及镜像的 tag。该值是在执行 **operator-sdk init** 命令时生成的，注册中心是 **--domain** 提供的，仓库名称就是所在目录的名称：playerstats-operator，tag 是该 operator 的版本号。当然，如果你想修改这些值，可以编辑 Makefile 文件进行修改，也可以通过命令行修改。

在本地构建完 operator 镜像，需要把它推送到镜像注册中心，以便 Kubernetes 拉取。使用 Docker 将镜像推送到注册中心，需要通过身份验证，使用 **docker login** 命令登录到 Quay：

```bash
$ docker login quay.io --username $YOUR_QUAY_USERNAME --password $YOUR_QUAY_PASSWORD
```

构建 Operator 镜像并推送到注册中心，别忘记把 IMG 中的 sretech 修改成你的 quay 账号：

```bash
$ make docker-build docker-push IMG="quay.io/sretech/playerstats-operator:v0.0.1"
docker build -t quay.io/sretech/playerstats-operator:v0.0.1 .
Sending build context to Docker daemon    149kB
Step 1/5 : FROM quay.io/operator-framework/helm-operator:v1.11.0
 ---> e8bc8fc22395
Step 2/5 : ENV HOME=/opt/helm
 ---> Using cache
 ---> 939f3e320dc2
Step 3/5 : COPY watches.yaml ${HOME}/watches.yaml
 ---> Using cache
 ---> dd1aaf3731cc
Step 4/5 : COPY helm-charts  ${HOME}/helm-charts
 ---> Using cache
 ---> 11c08abe328a
Step 5/5 : WORKDIR ${HOME}
 ---> Using cache
 ---> d50c86585eb0
Successfully built d50c86585eb0
Successfully tagged quay.io/sretech/playerstats-operator:v0.0.1
docker push quay.io/sretech/playerstats-operator:v0.0.1
The push refers to repository [quay.io/sretech/playerstats-operator]
49108043bfd4: Pushed 
1fd26746c559: Pushed 
81c1dccd8779: Mounted from operator-framework/helm-operator 
337cb014e36c: Mounted from operator-framework/helm-operator 
83ab849ae91d: Mounted from operator-framework/helm-operator 
e7ed17121dee: Mounted from operator-framework/helm-operator 
785573c4b945: Mounted from operator-framework/helm-operator 
v0.0.1: digest: sha256:753559ae39712cdf5173f60d7afbd485e6f435dc1f05f709ebb4b65414e87f76 size: 1779
```

再次查看 quay.io 的 playerstats-operator 仓库页面，点击 tag 标签，可以看到我们上传的镜像，如图：

![8.4 player stats operator image](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/8.4%20player%20stats%20operator%20image.jpg)

现在 operator 镜像已经被推送到了容器注册中心，接下来我们将其部署到 Kubernetes 中。

### 8.4.5 部署 playerstats operator

在创建 playerstats operator 时，**operator-sdk** CLI 还创建了一个名为 config/ 的文件夹，并在该文件下生成了部署 operator 所需的文件。

下面是 config/ 文件夹的结构：

```bash
config/
├── crd
│   ├── bases
│   │   └── charts.quay.io_player-stats.yaml
│   └── kustomization.yaml
├── default
│   ├── kustomization.yaml
│   ├── manager_auth_proxy_patch.yaml
│   └── manager_config_patch.yaml
├── manager
│   ├── controller_manager_config.yaml
│   ├── kustomization.yaml
│   └── manager.yaml
├── manifests
│   └── kustomization.yaml
├── prometheus
│   ├── kustomization.yaml
│   └── monitor.yaml
├── rbac
│   ├── auth_proxy_client_clusterrole.yaml
│   ├── auth_proxy_role_binding.yaml
│   ├── auth_proxy_role.yaml
│   ├── auth_proxy_service.yaml
│   ├── kustomization.yaml
│   ├── leader_election_role_binding.yaml
│   ├── leader_election_role.yaml
│   ├── player-stats_editor_role.yaml
│   ├── player-stats_viewer_role.yaml
│   ├── role_binding.yaml
│   ├── role.yaml
│   └── service_account.yaml
├── samples
│   ├── charts_v1alpha1_player-stats.yaml
│   └── kustomization.yaml
└── scorecard
    ├── bases
    │   └── config.yaml
    ├── kustomization.yaml
    └── patches
        ├── basic.config.yaml
        └── olm.config.yaml

11 directories, 29 files
```

crd/ 文件夹包含了创建 player-stats CRD 所需的 YAML 资源(charts.quay.io_player-stats.yaml)。该文件用于向 Kubernetes 注册新的 Player-stats API 端点。此外 samples/ 文件夹还包含一个 Player-stats CR 应用程序示例(charts_v1alpha1_player-stats.yaml)。创建此文件将触发 operator 安装 player-stats Helm 图表。

查看 Player-stats CR 的内容，以便熟悉定义的属性类型：

```bash
$ cat playerstats-operator/config/samples/charts_v1alpha1_player-stats.yaml
apiVersion: charts.quay.io/v1alpha1
kind: Player-stats
metadata:
  name: player-stats-sample
spec:
  # Default values copied from <project_dir>/helm-charts/player-stats/values.yaml
  affinity: {}
  autoscaling:
    enabled: false
    maxReplicas: 100
    minReplicas: 1
    targetCPUUtilizationPercentage: 80
  fullnameOverride: ""
  image:
    pullPolicy: IfNotPresent
    repository: registry.cn-beijing.aliyuncs.com/sretech/player-stats
    tag: ""
  imagePullSecrets: []
  ingress:
    annotations: {}
    enabled: false
......
```

spec 下的条目引用了 player-stats 图表的 values.yaml 文件中的值。**operator-sdk** 工具自动使用该文件中的每个默认值创建这个示例 CR。可以在应用此 CR 前添加或修改条目，以覆盖 player-stats 图表的其他值。operator 在运行时使用这些值部署 player-stats 应用程序。

rbac/ 目录下的 role_binding.yaml、role.yaml、service_account.yaml 文件的创建是为了向 operator 提供必要的权限，以便监视 Player-stats CRs 和安装 player-stats Helm 图表到 Kubernetes。它通过 service_account.yaml 中定义的 service account，使用 Kubernetes API 进行身份验证来执行这些操作。一旦身份验证通过，将根据 role.yaml、role_binding.yaml 资源向 operator 提供授权。role.yaml 文件定义了细粒度的权限，这些权限允许 operator 执行确切的资源和操作。role_binding.yaml 文件将角色绑定到 operator 的 service account 上。

了解了 rbac/ 文件夹中的资源后，就可以按照以下步骤部署 player-stats operator 了：

1. Minikube VM中没有**kubectl**命令，如果你还登录在VM中，需要使用以下命令退出连接：

   ```bash
   $ exit
   ```

2. 前面用 **operator-sdk** 创建的资源位于源码存储库的 playerstats-operator/ 文件夹下。如果你没有在之前的章节中克隆这个存储库，现在使用以下命令克隆它：

   ```bash
   $ git clone https://github.com/weiwendi/learn-helm.git
   ```

   **operator-sdk** CLI 基于 player-stats Helm 图表中的模板文件生成了一个简单的 role.yaml 文件，但如果你还记得的话，player-stats 图表包含一些资源，这些资源只能基于条件值创建，如 Job 钩子资源，只有启用了持久性存储才会包含这些资源。下面的代码片段显示了 Job 模板的一个例子：

   ```yaml
   {{- if .Values.mongodb.persistence.enabled }}
   apiVersion: batch/v1
   kind: Job
   ```

   **operator-sdk** CLI 没有自动为 Jobs 创建基于角色的访问控制(RBAC)规则，因为它不确定要部署的图表中是否包含了这个模板。

   因此，我将这些规则添加到了role.yaml 文件，位于https://github.com/weiwendi/learn-helm/blob/main/playerstats-operator/config/rbac/role.yaml#L62-L67.

   ```yaml
   - verbs:
     - "*"
     apiGroups:
     - "batch"
     resources:
     - "jobs"
   ```

3. 有三种方式运行 Operator，我们采用在集群中运行 Deployment 的方式，其他的方式可以参考官网介绍 https://sdk.operatorframework.io/docs/building-operators/helm/tutorial/。但在运行 Operator 前，我们再做一些辅助性的修改，帮助我们更顺利的运行 Operator。

   1. 修改 Makefile 中 kustomize 工具的获取方式。在 Makefile 原文件中，使用 **curl** 命令从 https://github.com/kubernetes-sigs/kustomize/releases 下载 kustomize 的 tar 包，并解压到 Makefile 同级目录的 bin/ 目录中，这个下载过程耗时比较长，网络不稳定的环境中容易出现异常，我们可以提前准备好该文件，并把 **curl** 命令换成 **cp** 命令：

      ```bash
      # 原命令类似如下：
      curl -sSLo - https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/v3.8.7/kustomize_v3.8.7_$(OS)_$(ARCH).tar.gz | \tar xzf - -C bin/ ;\
      
      # 下载对应的 tar.gz 包并解压到某个目录，从该目录进行拷贝
      # 注意 kustomize 文件要有执行权限
      cp /root/software/kustomize $(dir $(KUSTOMIZE)) ;\
      ```

   2. 修改 kube-rbac-proxy 镜像。运行 Operator 需要 kube-rbac-proxy 镜像，默认是 gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0 的镜像仓库，我们把镜像改成 registry.cn-beijing.aliyuncs.com/sretech/kube-rbac-proxy:v0.8.0：

      ```bash
      [aiops@aiops0113 playerstats-operator]$ sed -i 's/gcr.io\/kubebuilder\/kube-rbac-proxy:v0.8.0/registry.cn-beijing.aliyuncs.com\/sretech\/kube-rbac-proxy:v0.8.0/g' config/default/manager_auth_proxy_patch.yaml
      ```

      需要注意的是，以后镜像 tag 可能会发生变化，不再是 v0.8.0，这时再使用上面的命令，会替换失败。

4. 接下来执行 **make deploy** 命令，安装 config/rbac 中的清单：

   ```bash
   make deploy IMG="quay.io/sretech/playerstats-operator:v0.0.1"
   ```

5. 验证 playerstats-operator 是否启动：

   ```bash
   kubectl get deployment -n playerstats-operator-system
   NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
   playerstats-operator-controller-manager   1/1     1            1           8m
   ```

现在，Player-stats Operator 已经部署，让我们使用它来安装 Player-stats Helm 图表。

### 8.4.6 部署 player-stats 应用程序

将 Helm 用作独立工具时，可以使用 **helm install** 命令安装 Helm 图表。使用 Helm operator，可以通过创建 CR 安装 Helm 图表：

```bash
$ kubectl apply -f playerstats-operator/config/samples/charts_v1alpha1_player-stats.yaml -n operators
```

观察 **operators** 命名空间中的 Pod 启动过程：

```bash
$ kubectl get pods -n operators -w
NAME                                   READY   STATUS              RESTARTS   AGE
mongo-fc6f6b868-s4lfg                  0/1     ContainerCreating   0          10s
player-stats-sample-5599464bfd-6bzzn   0/1     Running             0          10s
player-stats-sample-5599464bfd-6bzzn   0/1     Running             1          42s
player-stats-sample-5599464bfd-6bzzn   0/1     Running             2          74s
player-stats-sample-5599464bfd-6bzzn   0/1     Running             3          112s
mongo-fc6f6b868-s4lfg                  0/1     Running             0          118s
mongo-fc6f6b868-s4lfg                  1/1     Running             0          2m3s
player-stats-sample-5599464bfd-6bzzn   1/1     Running             3          2m13s
```

当创建 Player-stats CR 时，operator 会执行 **helm install** 命令安装 player-stats 图表。可以通过 **helm list** 确认release：

```bash
$ helm list -n operators
NAME               	NAMESPACE	REVISION	UPDATED
player-stats-sample	operators	1       	2021-08-26 07:33:24
```

可以通过修改 player-stats-sample CR 来执行版本升级，将 playerstats-operator/config/samples/charts_v1alpha1_player-stats.yaml 文件中 replicas 的值从 1 改为 2：

```yaml
replicaCount: 2
```

应用更改：

```bash
$ kubectl apply -f playerstats-operator/config/samples/charts_v1alpha1_player-stats.yaml -n operators
```

这时将触发 **helm upgrade** 命令，player-stats Helm 图表的升级又会触发 upgrade hook 备份 MongoDB 数据。如果你在修改 CR 后对 operators 命名空间中的 Pod 进行监控，能看到备份 Job 开始执行。还可以使用 **helm list** 命令查看 player-stats-sample 的版本变成了 2：

```bash
$ helm list -n operators
NAME               	NAMESPACE	REVISION	
player-stats-sample	operators	2       
```

尽管修订号变成了 2，但截至编写此文时，基于 helm operator 的一个限制是，不能像使用 helm cli 那样对以前版本进行回滚。如果你对 player-stats-sample 执行 **helm history**，查看发布历史，只能看到版本 2：

```bash
$ helm history player-stats-sample -n operators
REVISION	UPDATED                 	STATUS 
2       	Thu Aug 26 08:18:27 2021	deployed
```

这是 helm cli 和 helm operator 之间的一个重要区别。因为不保留发布历史，所以基于 Helm operator 不允许执行显示回滚。但是，在升级失败时，将运行 **helm rollback** 命令。在这种情况下，将执行回滚 hook，尝试回滚到升级前版本。

尽管基于 Helm 的 operator 不保留发布历史，但它有一个优势就是同步 application 的期望状态和运行时状态。这是因为 operator 能够不断监视 Kubernetes 环境的状态，并确保 application 的配置始终与 CR 上指定的配置相同。换句话说，如果修改了 Player stats application 的一个资源，operator 会立即恢复更改，使其与 CR 的定义相匹配。你可以通过修改 Player stats 的某个资源来验证这一点。

例如，我们将直接将 Player stats 部署的副本数从 2 更改为 3，并观察 operator 将其自动恢复为 2 个副本，以重新同步为 CR 中定义的所需状态。

执行 **kubectl patch** 命令将 Player stats 部署的副本计数从 2 修改为 3：

```bash
$ kubectl patch deployment player-stats-sample -p '{"spec":{"replicas":3}}' -n operators
```

通常，这会增加一个 player-stats application 副本，但是由于 Player-stats CR 定义的副本数为 2，Operator 会迅速将副本数修改为 2，并终止新增 Pod 的创建。如果你需要把副本数调整为 3，必须更新 Player-stats CR上的 replicaCount 值。这个特性确保了集群运行时状态与配置状态保持一致。

### 8.4.7 删除 Operator

先删除 player-stats-sample CR，并卸载 Helm release：

```bash
$ kubectl delete -f playerstats-operator/config/samples/charts_v1alpha1_player-stats.yaml -n operators
```

该命令会删除 player-stats-sample 和所有依赖的资源。

在资源删除后，就可以删除 Operator 了：

```bash
$ make undeploy
/tmp/test/playerstats-operator/bin/kustomize build config/default | kubectl delete -f -
namespace "playerstats-operator-system" deleted
customresourcedefinition.apiextensions.k8s.io "player-stats.charts.quay.io" deleted
serviceaccount "playerstats-operator-controller-manager" deleted
role.rbac.authorization.k8s.io "playerstats-operator-leader-election-role" deleted
clusterrole.rbac.authorization.k8s.io "playerstats-operator-manager-role" deleted
clusterrole.rbac.authorization.k8s.io "playerstats-operator-metrics-reader" deleted
clusterrole.rbac.authorization.k8s.io "playerstats-operator-proxy-role" deleted
rolebinding.rbac.authorization.k8s.io "playerstats-operator-leader-election-rolebinding" deleted
clusterrolebinding.rbac.authorization.k8s.io "playerstats-operator-manager-rolebinding" deleted
clusterrolebinding.rbac.authorization.k8s.io "playerstats-operator-proxy-rolebinding" deleted
configmap "playerstats-operator-manager-config" deleted
service "playerstats-operator-controller-manager-metrics-service" deleted
deployment.apps "playerstats-operator-controller-manager" deleted
```

应该确保在删除 Operator 之前先删除 CR。当你删除 CR 时，Operator 会执行 **helm uninstall** 命令。如果你不小心先删除了 Operator，需要手动运行 **helm uninstall**，删除相关资源。

在本节中，你创建了一个 Helm operator ，并学习了如何使用基于 operator 的方法部署应用程序。在下一节中，我们将继续讨论如何使用 Helm 来管理 operators。

Operator 的相关知识就讨论到这里，接下来清理实验环境。

## 8.5 清理 Kubernetes 环境

删除 operators 命名空间：

```bash
$ kubectl delete ns operators
```

最后，停止运行Minikube：

```bash
$ minikube stop
```