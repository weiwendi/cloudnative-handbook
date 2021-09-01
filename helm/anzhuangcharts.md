# 安装 Helm 图表
在本书的第一章，我把 Helm 定义为 Kubernetes 的包管理器，并与操作系统的包管理器进行了比较。用户使用包管理器可以便捷地安装应用程序、管理它们可能需要的依赖。这也是 Helm 的作用。

在 Kubernetes 上安装应用程序时，用户只需输入对应的图表名称，Helm 会完成剩余工作。Helm 图表包含安装应用程序所需的逻辑和组件，用户无需知道具体的细节即可执行安装。用户还可以将值(values)作为参数传递给 Helm 图表，用来配置应用程序，实现自定义安装。在本章中，我们通过使用 Helm 在 Kubernetes 集群中安装 WordPress 博客系统，来学习 Helm 包管理器的特性。

## WordPress 介绍

WordPress 是一个开源的内容管理系统(CMS)，可以创建网站和博客。它有两个版本：SaaS 版本和自维护版本。SaaS 版需要用户花钱购买服务，自维护就是用户下载软件包自行部署维护。笔者的 https://www.aiops.red 就是使用 WordPress 的自维护版本搭建的。

WordPress 将网站数据存储在数据库中，数据库要求 MySQL 或者 MariaDB。在 Kubernetes 集群中部署一套 WordPress 服务需要创建的资源如下：

* Secret：管理数据库和控制台的凭证
* ConfigMap：配置数据库
* Service：负载均衡
* PersistentVolumeClaim：持久化存储数据 
* StatefulSet：以有状态的方式部署数据库
* Deployment：部署 WordPress 前端服务

创建这些资源需要对 WordPress 和 Kubernetes 都有深入的了解，要知道如何配置 WordPress，以及如何将 WordPress 的需求作为 Kubernetes 的资源表述出来。考虑到所需资源的复杂性以及需要的资源类型较多，在 Kubernetes 上部署 WordPress 可能是一项令人生畏的任务。

完成这一挑战是 Helm 的完美使用案例。用户不必专注于创建和配置每个 Kubernetes 资源，也无需 WordPress 的专业知识，只需使用 Helm 包管理器，在 Kubernetes 上部署和配置 WordPress。首先，我们在 Helm Hub 平台上找到 WordPress 的 Helm 图表，之后使用 Helm 将 WordPress 部署到 Kubernetes 集群中， 并在部署的过程中，探索 Helm 的基本特性。

## 搜索 WordPress 图表

Helm 图表可以存储在图表存储库中，供其他人使用。图表存储库是一个 HTTP 服务，如 GitHub、NGINX Web 服务器，可以存储和共享图表包。

在使用 Helm 图表安装 WordPress 前，我们需要配置 Helm 能够从哪些存储库中获取图表。添加存储库的命令就是之前介绍过的 **helm repo add**。为了方便查找图表，Helm 社区创建了 Helm Hub 平台。

Helm Hub 旨在聚合所有已知的公共图表，并提供了搜索功能。在本章中，我们使用 Helm Hub 平台搜索所需的 WordPress Helm 图表。在找到合适的图表后，添加该图表的存储库，以便执行安装。

可以通过命令或 Web 浏览器与 Helm Hub 交互。当使用命令行搜索 Helm 图表时，返回的结果包括了 Helm Hub 的 URL（ URL 可用于查找有关 chart 的其他信息）、版本及说明。

### 从命令行搜索 WordPress 图表

Helm 提供了两个不同的搜索命令帮助我们查找 Helm 图表：

* 在 Helm Hub 中搜索图表

  ```shell
  $ helm search hub
  ```

* 通过图表的关键字在存储库中搜索：

  ```shell
  $ helm search repo
  ```

如果没有添加过图表存储库，用户应使用 **helm search hub** 命令从所有公共图表存储库中搜索可用的 helm 图表。在添加存储库后，用户可以使用 **helm search repo** 命令在已添加的存储库中搜索图表。

我们从 Helm Hub 中搜索所有的 WordPress 图表。Helm Hub 中的每个图表都包括一组可以搜索的关键字。执行以下命令搜索包含 wordpress 关键字的图表：

```shell
[aiops@aiops0113 ~]$ helm search hub wordpress
```

执行此命令会显示以下输出：

![3.1 helm search hub wordpress outputs](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/3.1hubouts.jpg)

该命令返回的每一行都来自 Helm Hub 的一个图表 。输出内容包括图表 Helm Hub 页面的 URL，Helm 图表的最新版本，图表部署的应用程序版本，以及描述信息。

你可能注意到有些信息被截断了，这是因为 **helm  search hub** 以表格形式返回结果，默认超过 50 个字符的列将被截断。可以通过 **--max-col-width=0** 参数，显示完整信息：

```shell
$ [aiops@aiops0113 ~]$ helm search hub wordpress --max-col-width=0
```

这次以表格的形式完整输出了字段信息。

还可以通过 **--output** 参数指定 **yaml** 或 **json** 格式输出：

```shell
[aiops@aiops0113 ~]$ helm search hub wordpress --output yaml
```

结果将以 YAML 格式显示，类似下图：

![3.2 helm search hub wordpress yaml outputs](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/3.2hubyamloutps.png)

我们选择返回的第一个图表进行安装。要了解更多该图表的信息，可以访问 https://artifacthub.io/packages/helm/bitnami/wordpress 页面。

### 从浏览器搜索 WordPress 图表

使用 **helm search hub** 命令是在 helm hub 中搜索图表的最快方式，但它并不能提供完整的信息。用户需要添加图表存储库的 URL，以便安装图表。图表的Helm Hub 页面提供了此 URL，以及其他内容。

把 WordPress 图表的 URL 粘贴到浏览器中：

![3.3 wordpress page details](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/3.3wordpressdetails.jpg)

Helm Hub 的 WordPress 图表页面提供了很多内容，包括图表简介、维护者、安装信息等。

点击右侧栏的 INSTALL 按钮，可以看到添加 Bitnami 图表库的命令：

1. 在命令行中执行以下命令添加图表库：

   ```shell
   [aiops@aiops0113 ~]$ helm repo add bitnami https://charts.bitnami.com/bitnami
   ```

2. 查看添加的图表库列表：

   ```shell
   [aiops@aiops0113 ~]$ helm repo list
   ```

3. 搜索 bitnami 存储库中或其它存储库中包含 bitnami 关键字的图表，并以 YAML 格式显示：

   ```shell
   [aiops@aiops0113 ~]$ helm search repo bitnami --output yaml
   ```

与 **helm search hub** 命令类似，**helm search repo** 命令也使用图表关键字作为参数。将 bitnami 作为关键字将返回 bitnami 存储库中所有的图表，以及该存储库之外包含 bitnami 关键字的图表。

为了确保我们搜到的是 WordPress 相关的图表，可以将 wordpress 作为 **helm search repo** 命令的参数：

```shell
[aiops@aiops0113 ~]$ helm search repo wordpress
```

返回的是我们在浏览器上看到的 Helm Hub 上的 WordPress chart：

![3.4 bitnami wordpress](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/3.4bitnamiwordpress.jpg)

NAME 列中，`/` 前的字段表示 Helm 图表存储库的名称，写作此文时最新的图表版本是 11.0.5，也是我们要安装的版本。可以使用 version 参数，查看历史版本。

### 从命令行查看图表信息

在 Helm Hub 页面，我们看到了很多有用的图表信息。将图表存储库添加到本地，就可以使用 **helm show** 的四个子命令从命令行查看这些信息了：

* 显示图表的元数据：

  ```shell
  $ helm show chart
  ```

* 显示图表的自述(README)文件

  ```shell
  $ helm show readme
  ```

* 显示图表的值(values)：

  ```shell
  $ helm show values
  ```

* 显示图表的定义、自述文件和值：

  ```shell
  $ helm show all
  ```

我们通过查看 Bitnami WordPress 的图表信息来熟悉一下这 4 个子命令，图表名称为 bitnami/wordpress。如果要查看特定版本的信息，需要使用 --version 标志，后跟版本号。如果省略此标志，将返回最新版本的图表信息。

运行 **helm show chart** 命令查看图表的元数据：

```shell
[aiops@aiops0113 ~]$ helm show chart bitnami/wordpress --version 11.0.5
```

这个命令输出了图表的定义信息，包括图表版本、依赖关系、关键字和维护人员信息等：

![3.5 show chart](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/3.5showchart.jpg)

运行 **helm show readme** 命令查看图表自述文件中的内容：

```shell
[aiops@aiops0113 ~]$ helm show readme bitnami/wordpress --version 11.0.5
```

运行 **helm show values** 命令查看图表的值。用户通过提供值，实现图表的自定义配置。

运行 **helm show all** 命令，将前面三个命令中的信息聚合在一起。如果你想一次性查看图表的所有信息，请使用此命令。

我们已经搜索到了 WordPress 图表，并检查了它的内容，现在我们来配置 Kubernetes 环境，然后安装图表。

## 启动 Kubernetes 集群

在上一章中，我们使用 Minikube 创建了 Kubernetes 集群，如果你没有保持环境的运行，可以通过以下步骤重新启动 Kubernetes 集群：

1. 启动 Kubernetes 集群：

   ```shell
   [aiops@aiops0113 ~]$ minikube start
   ```

2. 在集群启动之后，我们创建一个名为 website 的 namespace，专门用来安装 WordPress：

   ```shell
   [aiops@aiops0113 ~]$ kubectl create namespace website
   ```

## 安装 WordPress 图表

Helm 图表的安装非常简单，只需要一条命令。但在安装前，需要我们确认图表包含的值，以及自定义哪些值，才能达到我们期望的安装效果。

### 创建值文件

可以通过创建 YAML 格式的值文件覆盖图表的默认值。为了创建正确的值文件，我们首先使用 **helm show values** 子命令查看图表支持的值：

```shell
[aiops@aiops0113 ~]$ helm show values bitnami/wordpress --version 11.0.5
## @section Global parameters
......
image:
  registry: docker.io
  repository: bitnami/wordpress
  tag: 5.7.2-debian-10-r2
  ## Specify a imagePullPolicy
  ## Defaults to 'Always' if image tag is 'latest', else set to 'IfNotPresent'
  ## ref: http://kubernetes.io/docs/user-guide/images/#pre-pulling-images
  ##
  pullPolicy: IfNotPresent
  ## Optionally specify an array of imagePullSecrets.
  ## Secrets must be manually created in the namespace.
  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
  ## e.g:
  ## pullSecrets:
  ##   - myRegistryKeySecretName
  ##
  pullSecrets: []
  ## Enable debug mode
  ##
  debug: false

## @section WordPress Configuration parameters
## WordPress settings based on environment variables
## ref: https://github.com/bitnami/bitnami-docker-wordpress#environment-variables

## @param wordpressUsername WordPress username
##
wordpressUsername: user
## @param wordpressPassword WordPress user password
## Defaults to a random 10-character alphanumeric string if not set
##
wordpressPassword: ""
```

该命令输出了 wordpress 图表能够设置的所有值，其中有很多已经提供了默认值。在安装图表时，如果我们没有提供值，图表将使用这些默认值。例如，在我们提供的值文件中没有覆盖 image 的值，WordPress 图表将使用默认的镜像配置：docker hub 镜像仓库中的 bitnami/wordpress 镜像，镜像 tag 是 5.7.2-debian-10-r2。

图表值文件中以 `#` 开头的是注释，注释可以作为某个值或块的说明，便于用户理解；也可以用来注释某个值，使其不再生效。

继续浏览输出内容，会看到设置 WordPress 元数据的值。这些值对于配置 WordPress 博客很重要，我们创建一个值文件覆盖它们。在你的机器上创建一个名为 wordpress-values.yaml 的文件，并将以下内容写入该文件：

```yaml
wordpressUsername: weiwendi  # WordPress 控制台登录用户
wordpressPassword: Z5tOfyh   # WordPress 控制台登录密码
wordpressEmail: weiwendi@aiops.red
wordpressFirstName: AiOps
wordpressBlogName: Learn Helm!
```

除了这些，还有一个重要的值我们要修改：

```yaml
service:
    type: LoadBalancer
```

这个值在 minikube 安装的 Kubernetes 集群中，需要改为 NodePort：

```yaml
service:
    type: NodePort
```

最终，完整的值文件内容为：

```yaml
wordpressUsername: weiwendi
wordpressPassword: Z5tOfyh
wordpressEmail: weiwendi@aiops.red
wordpressFirstName: AiOps
wordpressBlogName: Learn Helm!
service:
        type: NodePort
```

创建完值文件，我们开始安装 WordPress 图表。

### 安装图表

**helm install** 命令用来安装 Helm 图表。语法如下：

```shell
helm install [NAME] [CHART] [flags]
```

* **NAME** ：使用 Helm 安装的应用程序，在 Helm 中被称为 release，NAME 参数是我们为 release 起的名称。Release 会把 Kubernetes 资源与图表一起安装，并跟踪应用程序的生命周期。本章会介绍 release 是如何工作的。 

* **CHART**： Helm 图表的名称，对于图表存储库中的图表，遵循 \<repo name>/\<chart name>。
* **flags**：代指 **helm install** 命令支持的一系列参数，方便用户定制安装图表。flags 允许用户设置、覆盖值，指定命名空间(namespace) 等。可以通过 **helm install --help** 命令查看 flags 列表。--help 参数也可以传递给其它命令，查看它们的用法和支持的选项。

我们已经正确掌握了 **helm install** 命令的用法，现在执行以下命令安装 WordPress 图表：

```shell
[aiops@aiops0113 valuesFile]$ helm install wordpress bitnami/wordpress --values=wordpress-values.yaml --namespace website --version 11.0.5
```

在该命令中，我们使用 bitnami/wordpress 图表安装了一个名为 wordpress 的 release，并把自定义的值通过 wordpress-values.yaml 文件传递给了图表，以覆盖默认值，图表被安装到了 website 命名空间中。--version 标志指定了要部署的版本是 11.0.5。在不指定版本号的情况下，将安装最新版本的 Helm 图表。

如果图表安装成功，你会看到类似以下的输出内容：

```yaml
NAME: wordpress
LAST DEPLOYED: Mon May 17 11:28:51 2021
NAMESPACE: website
STATUS: deployed
REVISION: 1
NOTES:
** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    wordpress.website.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

   export NODE_PORT=$(kubectl get --namespace website -o jsonpath="{.spec.ports[0].nodePort}" services wordpress)
   export NODE_IP=$(kubectl get nodes --namespace website -o jsonpath="{.items[0].status.addresses[0].address}")
   echo "WordPress URL: http://$NODE_IP:$NODE_PORT/"
   echo "WordPress Admin URL: http://$NODE_IP:$NODE_PORT/admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: weiwendi
  echo Password: $(kubectl get secret --namespace website wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```

输出内容显示了安装信息，包括 release 名称、部署时间、部署的命名空间、部署状态(已部署)和修订号(Revision)，由于该 release 是初次安装，所以修订号为 1。

输出内容还包括 Notes，Notes 用于向用户提供图表安装的附加信息。在 WordPress 图表的例子中，Notes 提示了如何访问和验证 WordPress 应用程序。可以通过 **helm get notes** 命令随时查看 Notes 信息。

Helm 图表安装完成后，我们可以通过 release 查看应用程序的资源清单和配置信息。

### 查看 release

可以在 Kubernetes 命名空间中查看所有已安装的 release：

```shell
[aiops@aiops0113 valuesFile]$ helm list --namespace website
NAME     	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART           	APP VERSION
wordpress	website  	1       	2021-05-17 10:50:12.321181442 +0800 CST	deployed	wordpress-11.0.5	5.7.2
```

**list** 子命令字段的含义：

* **NAME**：release 名称
* **NAMESPACE**：release 所在的命名空间
* **REVISION**：release 的最新修订号
* **UPDATED**：release 发布的最新时间戳
* **STATUS**：release 的状态
* **CHART**：图表名称及版本
* **APP VERSION**：图表中应用的版本

除了 **list** 子命令可以查看 release 的信息，Helm 还提供了 **get** 子命令，用来获取 release 的更多信息。下面的命令列表用来查看指定 release 的更详细信息：

* 或取 release 的所有钩子：

  ```shell
  helm get hooks
  ```

* 获取 release 的资源清单：

  ```shell
  helm get manifest
  ```

* 获取 release 的 notes：

  ```shell
  helm get notes
  ```

* 获取 release 的值：

  ```shell
  helm get values
  ```

* 获取 release 的所有信息，涵盖了以上 4 个命令：

  ```shell
  hlem get all
  ```

通过 **helm get manifest** 命令，可以查看到安装 release 时创建的 Kubernetes 资源清单：

```shell
[aiops@aiops0113 valuesFile]$ helm get manifest wordpress --namespace website
```

执行此命令，可以看到以下 Kubernetes 资源清单：

* 两个 Secret
* 两个 Service
* 一个 ConfigMap
* 一个 ServiceAccount
* 一个 Deployment
* 一个 StatefulSet
* 一个 PersistentVolumeClaim

并且从输出中，还可以看到我们自定义的值，如：NodePort。

图表提供的大多数默认值都保持不变，这些值已经被应用到了 Kubernetes 资源上，可以通过 **helm get manifest** 命令查看到。如果修改了这些值，那么 Kubernetes 资源的配置也会改变。

**helm get notes** 命令显示了 release 安装完成后的说明信息。你可能还记得，在安装完成 WordPress 图表时显示过 release notes。这些 notes 引导用户如何访问应用程序，我们通过命令再次查看 notes：

```shell
[aiops@aiops0113 valuesFile]$ helm get notes wordpress --namespace website
```

**helm get values** 命令用来查看安装 release 时提供的值，也就是我们通过 wordpress-values.yaml 文件传递的值：

```shell
[aiops@aiops0113 valuesFile]$ helm get values wordpress --namespace website
```

加上 --all 参数，可以显示所有值，包括默认值：

```shell
[aiops@aiops0113 valuesFile]$ helm get values wordpress --all --namespace website
```

**helm get all** 命令，可以打印所有来自各个 **helm get** 命令的信息：

```shell
[aiops@aiops0113 valuesFile]$ helm get all wordpress --namespace website
```

除了 Helm 的命令外，**kubectl** 命令可以更方便的查看安装的资源。**kubectl** 可以精确查询某种类型的资源，如 Deployment，而不是获取所有的资源。为了确保返回的资源是 Helm 部署 release 时创建的，可以给 **kubectl** 提供部署时定义的标签，Helm 图表通常会为 Kubernetes 资源设置标签。通过以下命令，查看带有 app.kubernetes.io/instance=wordpress 标签的 Kubernetes 资源：

```shell
[aiops@aiops0113 valuesFile]$ kubectl get all -l app.kubernetes.io/instance=wordpress --namespace website
```

## install 的更多用法

### -n 标志

-n 标志可以替代 --namespace，以减少输入命令时的工作量，如下示例：

```shell
[aiops@aiops0113 valuesFile]$ helm list --namespace website 
# 可以写成
[aiops@aiops0113 valuesFile]$ helm list -n website
```

### HELM_NAMESPACE 环境变量

如果不指定命名空间，Helm 的操作都会在 default 命名空间进行。可以修改 **HELM_NAMESPACE** 变量的值，设置 Helm 默认交互的命名空间。在不同操作系统上设置环境变量的方法略有不同：

* macOS 和 Linux：

  ```shell
  $ export HELM_NAMESPACE=website
  ```

* Windows 用户使用 PowerShell：

  ```powershell
  > $env:HELM_NAMESPACE = 'website'
  ```

修改后，使用 **helm env** 命令验证：

```shell
[aiops@aiops0113 valuesFile]$ helm env
```

在本书中，我们不依赖于 HELM_NAMESPACE 变量，而是在每个命令上使用 -n 标志，这样能更清楚地知道我们使用的命名空间。使用 -n 标志也是为 Helm 指定命名空间的最佳方式，因为它便于我们确认操作的命名空间是否正确。

### --set 和 --values

对于 **install**、**upgrade**、**rollback** 命令，可以使用两种方式将值传递给图表：

* **--set**：从命令行传入值。
* **--values**：从 YAML 值文件或 URL 中传入值

--values 标志是本书中配置图表值的首选方法，用这种方式配置多个值会更方便，还可以将值文件保存在源码管理(SCM)系统(如git)中。但对于密码等敏感值，建议使用 --set 标志。

--set 直接从命令行传递值，对于密码类型的、少量的值，这种方式可以接受，但 --set 不方便重复安装图表。

## 访问 WordPress

WordPress 图表安装完成后，Notes 输出了四个命令，用于配置访问 WordPress 程序。

* macOS 或者 Linux：

  ```shell
  $ export NODE_PORT=$(kubectl get --namespace website -o jsonpath="{.spec.ports[0].nodePort}" services wordpress)
  
  $ export NODE_IP=$(kubectl get nodes --namespace website -o jsonpath="{.items[0].status.addresses[0].address}")
  
  $ echo "WordPress URL: http://$NODE_IP:$NODE_PORT/"
  
  $ echo "WordPress Admin URL: http://$NODE_IP:$NODE_PORT/admin"
  ```

* Windows PowerShell：

  ```powershell
  > $NODE_PORT = kubectl get --namespace website -o jsonpath="{.spec.ports[0].nodePort}" services wordpress | Out-String
  
  > $NODE_IP = kubectl get nodes --namespace website -o jsonpath="{.items[0].status.addresses[0].address}" | Out-String
  
  > echo "WordPress URL: http://$NODE_IP:$NODE_PORT/"
  
  > echo "WordPress Admin URL: http://$NODE_IP:$NODE_PORT/admin"
  ```

把 **kubectl** 的执行结果传递给两个变量，echo 命令打印拼接在一起的两个变量，输出访问 WordPress 的 URL。第一个 RUL 是网站主页，用户通过该页面查看网站内容；第二个 URL 是网站的管理控制台，网站管理人员用它来配置和管理网站内容。

将第一个 URL 粘贴到浏览器中，你应该会看到类似下图显示的页面：

![3.6 wordpress 首页](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/3.8wordpresspage.jpg)

这个页面中有些内容看起来很熟悉，在居中位置最上方，博客的标题叫 Learn Helm! ，这不仅是本书的主题，也是你在安装过程中设置的 wordpressBlogName 的值。

影响页面的另一个自定义值是 wordpressUsername。请注意，“Hello world！” 的作者是 weiwendi。这是提供给 wordpressUsername 的值，如果该值是其他用户名，则会显示不同的名称。

将第二个echo命令中的链接粘贴到浏览器中，访问控制台，页面内容如下：

![3.7 admin pages](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/3.9adminpage.jpg)

要登录管理控制台，需要输入安装时提供的 wordpressUsername 和 wordpressPassword 的值。这些值可以在 wordpress-values.yaml 文件中查看，也可以运行以下命令查看，这两条命令也是 WordPress 图表 notes 的内容：

```shell
echo Username: weiwendi
echo Password: $(kubectl get secret --namespace website wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```

通过身份验证后，将显示管理控制台仪表板，如下所示：

![3.8 dashboard](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/3.10adminpage2.png)

图3-8 管理控制台仪表盘

## 更新 WordPress 版本

修改值或升级图表到最新版本，都需要更新 WordPress 版本实现。在本节中，我们通过修改 WordPress 副本数及资源请求的值，来演示版本更新。

### 修改值

通过 Helm 图表的值配置应用程序的实例副本数及资源请求大小是很常见的情况，下面的内容是 **helm show values** 命令的部分输出，这是当前运行的副本数：

```shell
[aiops@aiops0113 ~]$ helm show values bitnami/wordpress |grep replicaCount
replicaCount: 1
```

我们需要把 replicaCount 的值修改成 2。

需要修改的第二部分的值是 YAML 文件的 “WordPress container” resources 块的值，该块中 cpu 和 memory 的值如下：

```shell
[aiops@aiops0113 ~]$ helm show values bitnami/wordpress |grep resources -C 4
## WordPress containers' resource requests and limits
## ref: http://kubernetes.io/docs/user-guide/compute-resources/
## @param resources.limits The resources limits for the WordPress container
## @param resources.requests [object] The requested resources for the WordPress container
##
resources:
  limits: {}
  requests:
    memory: 512Mi
    cpu: 300m
```

值通过缩进进行逻辑分组，如 resources 部分的 requests 块，定义了 WordPress 应用程序在启动时， Kubernetes 为其分配的 memory 和 cpu 的值。我们将 requests.memory 的值修改为 256Mi (256 mebibytes)，将 requests.cpu 的修改为100m(100毫秒)。将这些修改添加到 wordpress-values.yaml 文件中，如下所示：

```yaml
resources:
  requests:
    memory: 256Mi
    cpu: 100m
```

修改完这两部分，新的 wordpress-values.yaml 内容如下：

```yaml
wordpressUsername: weiwendi
wordpressPassword: Z5tOfyh
wordpressEmail: weiwendi@aiops.red
wordpressFirstName: AiOps
wordpressBlogName: Learn Helm!
service:
        type: NodePort
replicaCount: 2
resources:
        requests:
                memory: 256Mi
                cpu: 100m
```

更新了 wordpress-values.yaml 文件的内容，在下一节中，我们使用 **helm upgrade** 命令升级 release。

### 执行 upgrade

**helm upgrade** 命令的语法与 **helm install** 命令几乎相同，如下例所示：

```shell
helm upgrade [RELEASE] [CHART] [flags]
```

执行 **helm install** 命令时，需要我们提供一个在该命名空间中不存在的 release 名称，而执行 **helm upgrade** 命令时，需要我们指定更新(已存在)的 release 名称。

与 **helm install** 命令相同，**helm upgrade** 也是通过 --values 标志传递值文件。使用我们修改后的值文件更新 WordPress 版本。

但如果直接使用我们修改后的 wordpress-values.yaml 文件更新 WordPress 会收到包含以下内容的错误：

```shell
Error: UPGRADE FAILED: template: wordpress/templates/NOTES.txt:83:4: executing "wordpress/templates/NOTES.txt" at <include "common.errors.upgrade.passwords.empty" (dict "validationErrors" $passwordValida
tionErrors "context" $)>: error calling include: template: wordpress/charts/common/templates/_errors.tpl:21:48: executing "common.errors.upgrade.passwords.empty" at <fail>: error calling fail: PASSWORDS ERROR: You must provide your current passwords when upgrading the release.
                 Note that even after reinstallation, old credentials may be needed as they may be kept in persistent volume claims.
                 Further information can be obtained at https://docs.bitnami.com/general/how-to/troubleshoot-helm-chart-issues/#credential-errors-while-upgrading-chart-releases

    'mariadb.auth.rootPassword' must not be empty, please add '--set mariadb.auth.rootPassword=$MARIADB_ROOT_PASSWORD' to the command. To get the current value:

        export MARIADB_ROOT_PASSWORD=$(kubectl get secret --namespace "website" wordpress-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)

    'mariadb.auth.password' must not be empty, please add '--set mariadb.auth.password=$MARIADB_PASSWORD' to the command. To get the current value:

        export MARIADB_PASSWORD=$(kubectl get secret --namespace "website" wordpress-mariadb -o jsonpath="{.data.mariadb-password}" | base64 --decode)

```

图表要求在更新 release 时，必须提供数据库当前的密码，我们根据提示获取密码：

```shell
[aiops@aiops0113 valuesFile]$ kubectl get secret --namespace "website" wordpress-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode
[aiops@aiops0113 valuesFile]$ kubectl get secret --namespace "website" wordpress-mariadb -o jsonpath="{.data.mariadb-password}" | base64 --decode
```

将 Mariadb 的两个密码也添加到 wordpress-values.yaml 文件中，最终内容如下：

```yaml
wordpressUsername: weiwendi
wordpressPassword: Z5tOfyh
wordpressEmail: weiwendi@aiops.red
wordpressFirstName: AiOps
wordpressBlogName: Learn Helm!
service:
        type: NodePort
mariadb:
        auth:
                rootPassword: szopyrAlqR  # Mariadb root 账号密码
                password: ePMtyJarxb      # Mariadb 普通账号密码
replicaCount: 2
resources:
        requests:
                memory: 256Mi
                cpu: 100m
```

修改完 wordpress-values.yaml 文件后，执行 **upgrade** 命令： 

```shell
[aiops@aiops0113 valuesFile]$ helm upgrade wordpress bitnami/wordpress --values wordpress-values.yaml -n website --version 11.0.5
```

一旦命令执行完成，你会看到类似 3.4小节中 **helm install** 时的 Notes 输出。

使用 **kubectl** 命令查看 WordPress Pod 的信息：

```shell
[aiops@aiops0113 valuesFile]$ kubectl get pods -n website
```

在 Kubernetes 中，当 Deployment 被修改时会创建新的 Pod。同样的行为也能在 Helm 中观察到。升级过程中添加的值更改了 WordPress 配置，因此，Helm 使用更新后的配置创建了新的 WordPress Pod。这些更改可以通过 **helm get manifest** 和 **kubectl get deployment** 命令来查看。

在下一小节中，我们将再执行几次升级，演示在升级过程中值的设置方式。

### upgrade 时值的重用与重置

**helm upgrade** 命令有两个标志，用于操作值。

这两个标志是：

* **--reuse-values**：升级时，重用上一个版本的值。
* **--reset-values**：升级时，将值重置为 chart 默认值。

如果执行 **upgrade** 时没有使用 --set 或 --values 提供值，则会默认添加 --reuse-values 标志。换句话说，如果在更新时没有提供值，将使用与之前版本相同的值：

1. 重新执行升级命令，不指定任何参数：

   ```shell
   [aiops@aiops0113 valuesFile]$ helm upgrade wordpress bitnami/wordpress -n website --version 11.0.5
   ```

2. 使用 **helm get values** 命令查看升级过程中使用的值：

   ```shell
   [aiops@aiops0113 valuesFile]$ helm get values wordpress -n website
   USER-SUPPLIED VALUES:
   mariadb:
     auth:
       password: ePMtyJarxb
       rootPassword: szopyrAlqR
     primary:
       persistence:
         enabled: false
   persistence:
     enabled: false
   replicaCount: 2
   resources:
     requests:
       cpu: 100m
       memory: 256Mi
   service:
     type: NodePort
   wordpressBlogName: Learn Helm!
   wordpressEmail: weiwendi@aiops.red
   wordpressFirstName: AiOps
   wordpressPassword: Z5tOfyh
   wordpressUsername: weiwendi
   ```

   显示的值与前一个版本相同。

   当升级过程中从命令行提供值时，可以观察到不同的结果。如果值是通过 --set 或 --values 传递的，那么所有没有提供的图表值都将重置为默认值。

3. 使用 --set 提供单个值再次执行升级。当然，别忘记获取并传递 Mariadb 账号密码，不然更新会执行失败：

   ```shell
   [aiops@aiops0113 valuesFile]$ export MARIADB_ROOT_PASSWORD=$(kubectl get secret --namespace "website" wordpress-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)
   [aiops@aiops0113 valuesFile]$ export MARIADB_PASSWORD=$(kubectl get secret --namespace "website" wordpress-mariadb -o jsonpath="{.data.mariadb-password}" | base64 --decode)
   [aiops@aiops0113 valuesFile]$ export WORDPRESS_PASSWORD=$(kubectl get secret --namespace "website" wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
   
   [aiops@aiops0113 valuesFile]$ helm upgrade wordpress bitnami/wordpress --set replicaCount=1 --set mariadb.auth.rootPassword=$MARIADB_ROOT_PASSWORD --set mariadb.auth.password=$MARIADB_PASSWORD --set wordpressPassword=$WORDPRESS_PASSWORD -n website --version 11.0.5
   ```

4. 升级完成后，执行 **helm get values** 命令：

   ```shell
   [aiops@aiops0113 valuesFile]$ helm get values wordpress -n website
   ```

在升级过程中提供了值时，Helm 会自动使用 --reset-values 标志，把除了提供的值之外的其它值置为默认值。

用户可以使用 --reset-values 或 --reuse-values，在升级过程中显式地定义值的行为。如果在升级时希望除了从命令行覆盖的值之外，其他值都被重置为默认值，可以使用 --reset-values；如果希望重用 release 上一版本的值，同时又想重新设置部分值，可以使用 --reuse-values。把值保存在值文件中，能有效简化值的管理，并在更新时以声明性的方式设置值。

按照本章的操作，现在 release 应该有了 4 个版本。第 4 个修订版并不符合我们的期望，因为它只配置了 replicaCount(除了图表要求的必须传入的密码相关值)，其它都被置为了默认值。在下一节中，我会介绍如何将 wordpress 回退到包含所需值的 release 版本。

## 回退 WordPress 版本

有时后退是为了更好的前进。在某些情况下，回退到应用程序的前一个版本非常有必要。**helm rollback** 命令就为满足这种场景。接下来我们把 WordPress 版本回退到之前的状态。

### 查看 WordPress 历史版本

Helm 的每个 release 都有历史发布记录，用于统计不同版本中使用的 values、Kubernetes 资源、图表版本。在安装、升级或回退图表时，会创建新的修订记录。默认情况下，修订数据存储在 Kubernetes 的 Secret 中。

使用 **kubectl** 命令查看 website 命名空间中的 Secret：

```shell
[aiops@aiops0113 valuesFile]$ kubectl get secret -n website
```

这条命令会返回 website 命名空间中所有的 Secret，你至少会看到以下 4 个 Secret：

```shell
sh.helm.release.v1.wordpress.v1
sh.helm.release.v1.wordpress.v2
sh.helm.release.v1.wordpress.v3
sh.helm.release.v1.wordpress.v4
```

这些 Secret 都对应一个 release 的历史修订条目，可以通过 **helm history** 命令查看：

```shell
[aiops@aiops0113 valuesFile]$ helm history wordpress -n website
```

该命令以表格的方式输出每个 release 的条目，以下内容只是截取了部分字段的信息：

```shell
REVISION    STATUS       DESCRIPTION
1           superseded   Install complete

2           superseded   Upgrade complete

3           superseded   Upgrade complete

4           deployed     Upgrade complete   
```

完整的列表字段如下：

* **REVISION**：修订版本号，1 为最初版(install 时的版本)，更新或回退一次则加 1 
* **UPDATED**：更新日期及时间
* **STATUS**：状态
* **CHART**：图表版本
* **APP VERSION**：图表部署的应用程序版本
* **DESCRIPTION**：描述信息

在这个输出中，STATUS 为 “superseded” 表示已升级的版本，为 “deployed” 表示当前运行的版本。STATUS 的其他类型还包括 pending、pending_upgrade(这两个表示安装、升级正在进行中)、failed(安装或升级失败，DESCRIPTION 会显示详细的失败信息)、unknown(状态未知，很难遇到这种状态)。

前面介绍过，**helm get** 命令通过 --revision 标志可以操作指定的修订版本。在执行回退前，我们使用此命令确认指定的修订版本是否包含了我们需要的值。你可能还记得，修订号为 4 的版本仅包含了 replicacount 值，而修订号为 3 的版本包含了我们需要的所有值，可以使用以下命令验证：

```shell
[aiops@aiops0113 valuesFile]$ helm get values wordpress --revision 3 -n website
```

### 执行回退

**helm rollback** 是 Helm 提供的回退命令，语法如下：

```shell
helm rollback <RELEASE> [REVISION] [flags]
```

在执行回退时，用户需要提供回退的 release 名称、回退到的修订版本号。执行以下命令可以将 wordpress 回退到修订号为 3 的版本：

```shell
[aiops@aiops0113 valuesFile]$ helm rollback wordpress 3 -n website
Rollback was a success! Happy Helming!
```

执行 **helm history** 命令，可以看到 release 回退到了修订号为 3 的版本：

```shell
[aiops@aiops0113 valuesFile]$ helm history wordpress -n website
```

从输出内容可以看到，release 记录中多了一条修订号为 5 的记录，STATUS 是 deployed，DESCRIPTION 中的内容是 “Rollback to 3”。当回退应用程序时，会添加一条新的 release 历史修订记录。不要将此与升级混淆，最高修订版本号仅表示当前部署的版本，从 DESCRIPTION 中才能确定这条记录是由升级还是回退创建的。

再次查看当前版本使用的值，确认回退的版本是我们希望的：

```shell
[aiops@aiops0113 valuesFile]$ helm get values wordpress -n website
```

在 **rollback** 子命令中，我们没有显式指定图表版本和它的值，这是因为 **rollback** 子命令无须接收这些信息；它的目的是将图表回退到之前的修订版，并使用之前修订版的值和图表版本。**rollback** 子命令不是 Helm 日常实践的一部分，它仅适用于在当前应用程序的状态不稳定、且必须恢复到之前稳定版的紧急情况。

在成功演示了版本回退功能后，本章的内容也即将结束。最后一步是使用 **uninstall** 子命令从 Kubernetes 集群中删除 WordPress 应用程序。我将在下一小节介绍 **uninstall** 子命令。

## 卸载 WordPress

卸载 Helm release，意味着要删除它管理的 Kubernetes 资源。默认情况下，**uninstall** 命令会同时删除 release 的历史记录，这是合理操作，如果想保留这些记录，也可以使用 **--keep-history** 标志。

**uninstall** 的语法非常简单：

```shell
helm uninstall RELEASE_NAME [...] [flags]
```

执行 **helm uninstall ** 命令卸载 WordPress 版本：

```shell
[aiops@aiops0113 valuesFile]$ helm uninstall wordpress -n website
release 'wordpress' uninstalled
```

通过 **list** 子命令查看 website 命名空间中的 release，wordpress 已经不存在了：

```shell
[aiops@aiops0113 valuesFile]$ helm list -n website
```

输出一个空表。也可以使用 kubectl 获取 WordPress 的 Deployment，来确认 wordpress 已经不存在了：

```shell
[aiops@aiops0113 valuesFile]$ kubectl get deployments -l app.kubernetes.io/instance=wordpress --namespace website
No resources found in website namespace.
```

正如预期的那样，已没有 WordPress 了。

```shell
[aiops@aiops0113 valuesFile]$ kubectl get pvc -n website
```

然而，你会看到在 website 命名空间中仍然有两个 PersistentVolumeClaim 可用。

这个 **PersistentVolumeClaim** 资源没有被删除，因为它是由 **StatefulSet** 在后台创建的。在 Kubernetes 中，如果 **StatefulSet** 被删除，那么由 **StatefulSet** 创建的 **PersistentVolumeClaim** 资源不会被自动删除。在  Helm 卸载过程中，**StatefulSet** 被删除，但相关的 **PersistentVolumeClaim** 没有被删除。这是我们所期望的。可以通过以下命令手动删除 **PersistentVolumeClaim** 资源：

```shell
[aiops@aiops0113 valuesFile]$ kubectl delete pvc -l app.kubernetes.io/instance=wordpress -n website
```

现在我们已经安装和卸载了 Wordpress，让我们清理你的 Kubernetes 环境，这样我们就有了一个干净的设置，我们将在本书后面的章节中进行练习。

我们已经完成了 WordPress 的安装和卸载，本章的所有试验都已经完成，接下来我们清理 Kubernetes 环境，打扫战场。

## 清扫战场

我们所有操作都是在 website namespace 中完成的，要清理 Kubernetes 环境，直接删除该 namespace 即可：

```shell
[aiops@aiops0113 valuesFile]$ kubectl delete ns website
namespace "website" deleted
```

在删除 wordpress 命名空间后，也可以关闭 Minikube 虚拟机：

```shell
$ minikube stop
```

VM 被关闭，但会保留 Minikube 状态，以便在下一个练习中快速启动。

## Minikube 的一个小 Bug

本书写作时，最新版本也是本书演示示例所用的版本：minikube version: v1.20.0，在使用了 --image-mirror-country=cn 标志时，会从错误的镜像地址下载 storage-provisioner 镜像，如果你够仔细的话，会留意到执行完 **minikube start** 命令后，输出内容包含以下信息：

```shell
...
  - Using image registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-minikube/storage-provisioner:v5 (global image repository)
* Enabled addons: storage-provisioner, default-storageclass
```

registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-minikube/storage-provisioner:v5 这个地址是错误的，在阿里云镜像仓库中，正确的镜像地址是 registry.cn-hangzhou.aliyuncs.com/google_containers/storage-provisioner:v5，执行以下命令可以验证这一点：

```shell
[aiops@aiops0113 valuesFile]$ kubectl get pod -n kube-system
```

从输出中查看 storage-provisioner Pod 的状态为 ImagePullBackOff。编辑 storage-provisioner Pod 的 YAML 文件：

```shell
[aiops@aiops0113 valuesFile]$ kubectl edit pod/storage-provisioner -n kube-system
```

把 image 的值修改为正确的地址，保存退出即可。

关于这个问题，也有人提了 PR：https://github.com/kubernetes/minikube/pull/10770。
