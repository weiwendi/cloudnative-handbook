# 使用 CI/CD 和 GitOps 实现 Helm 自动化
截止到本章，我们已经讨论了两个 Helm 的高级用法。第一个是将 Helm 作为包管理器，在 Kubernetes 集群中部署各种复杂的应用程序。第二个是开发和测试 Helm 图表，将 Kubernetes 的复杂性封装在 Helm 图表中，并对图表执行测试，确保成功交付了用户所需的功能。

这两种操作，都使用了多个不同的 Helm Cli 命令，这些命令虽然能有效地完成任务，但不够有效率，需要在命令行手动执行。在管理多个图表或应用程序时，手动执行会成为瓶颈，导致环境难以维护和扩展。因此，我们应该在 Helm 的基础上提供额外的自动化功能。本章我们讨论持续集成和持续交付(CI/CD)以及 GitOps 的相关概念，这些概念是自动调用 Helm Cli 和其他命令的方法论，实现对 Git 仓库执行自动化流程，这些流程包括在图表开发过程中构建、测试和打包图表，使用 Helm 自动部署应用程序。

本章的学习重点：

* 理解 CI/CD 和 GitOps
* 设置所需环境
* 创建 CI 管道构建 Helm 图表
* 创建 CD 管道部署 Helm 图表

## CI/CD 和 GitOps 介绍

迄今为止，我们讨论了许多 Helm 开发、测试、部署的相关概念，但这些仅限于手动配置和调用 Helm Cli 实现。虽然在初学 Helm 时可以这样做，但当你准备将图表部署到生产环境，在生产环境一展身手时，就不得不考虑以下两个问题了：

* 如何才能确保图表开发和部署遵循了最佳实践？
* 新成员加入时会对开发和部署过程产生什么影响？

这两个问题适用于任何软件项目，而不仅是 Helm 图表开发。虽然到目前为止我们已经介绍了很多最佳实践，但在有新成员加入时，他们可能对最佳实践有不同的理解，甚至在执行关键步骤时缺乏纪律性。通过使用自动化和可重复的流程，建立诸如 CI/CD 之类的概念，可以解决其中的一些挑战。

### CI/CD

每次软件变更时，都期望有自动化的软件开发过程，这就是 CI 产生的背景。CI 不仅确保了软件变更过程遵循了最佳实践，而且有助于避免许多开发人员面临的共同问题：“在我的机器上就能正常运行，怎么部署在服务器上就不行了呢？”。我们之前讨论过使用版本控制系统(如 git)存储代码，通常开发人员都有自己独立的源代码副本，这就导致维护代码库会很困难，因为其他开发人员也会提交代码。

CI 可以通过自动化工具实现，每次代码发生更改时，自动化工具都会检索代码并执行预定步骤。常用的自动化工具包括 Jenkins、TeamCity，以及各种基于软件即服务的解决方案。有了这些自动化工具的加持，开发人员可以将更多精力用于代码编写，并能够频繁地提交代码，进而更快的、更安全的交付软件。

这些工具都有一个关键特性，就是能够及时报告项目的当前状态。与在软件开发周期的最后阶段发现破坏性的变更不同，通过使用 CI，一旦在代码分支中合并了变更，就会执行流程，并将结果通知到相关人员。快速通知的机制，为引入变更的开发人员提供了解决问题的时机，让这些问题在代码的开发阶段就能被发现，而不是在交付阶段。

应用程序最终交付是要运行在生产环境的，借鉴软件交付生命周期中许多 CI 的概念，诞生了 CD。CD 是一组定义好的步骤，通过发布过程(通常是管道)发布软件。CI 和 CD 经常搭配在一起，因为 CI 的执行引擎也可以实现 CD。CD 已经得到了许多组织和公司的认可，并越来越流行，在这些公司中，为了使软件的发布过程更进一步，需要实施适当的变更控制或审批。由于 CI/CD 的许多概念都是以可重复的方式实现自动化的，因此一旦团队确信他们有了一个可靠的框架，就完全不需要手动审批了。

实现构建、测试、部署和发布过程的完全自动化，而不需要任何人为干预的过程称为持续部署(continuous deployment)。虽然许多软件项目从来没有完全实现持续部署，但只要实现 CI/CD 强调的概念，团队就能够更快地产生真正的业务价值。在下一节中，我们将介绍 GitOps，用于改进应用程序的管理及其配置。

## GitOps 将 CI/CD 提升到新高度

Kubernetes 支持声明式配置，和其他编程语言编写的程序一样，Kubernetes 清单也可以通过 CI/CD 管道进行管理。清单应该存储在代码仓库中(如 git)，并且每次的构建、测试、部署步骤应该相同。在 Git 存储库中管理 Kubernetes 集群配置的生命周期，然后以自动化的方式应用这些资源，这一趋势促使了 GitOps 概念的流行。GitOps 于 2017 年首次由 WeaveWorks 软件公司提出，此后作为管理 Kubernetes 配置的一种方式，它的受欢迎程度不断提高。虽然 GitOps 在 Kubernetes 的上下文中广为人知，但它的原理可以应用于任何本地云原生环境。

和 CI/CD 类似，目前已经有了很多管理 GitOps 流程的工具，如 Intuit 的 ArgoCD、WeaveWorks 的 Flux。当然，你无需特意使用专门为 GitOps 设计的工具，因为任何 CI/CD 自动化流程管理工具都可以。传统的 CI/CD 工具和为 GitOps 设计的工具之间的关键区别在于，GitOps 工具能够持续观察 Kubernetes 集群的状态，并在当前状态与 Git 库中清单所定义的状态不符时，应用所需的配置。这些工具有效利用了 Kubernetes 原生的控制器。

由于 Helm 图表最终是作为 Kubernetes 资源呈现的，因此它们也适用于 GitOps 流程，并且很多 GitOps 工具原生支持 Helm。在本章的剩余部分，我将讨论如何使用 CI/CD 和 GitOps 部署 Helm 图表，并把 Jenkins 作为 CI 和 CD 的首选工具。

## 设置环境

在本章中，我们将配置两个管道，演示围绕 Helm 实现不同程度的自动化。

按照以下步骤设置本地环境：

1. 删除原来的 Minikube，启动一个 4G 内存的 Minikube：

   ```shell
   [aiops@aiops0113 ~]$ minikube delete
   [aiops@aiops0113 ~]$ minikube start --memory=4g --image-mirror-country='cn' \
                        --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.20.0.iso
   ```

2. 在 Minikube 启动完成后，创建名为 cicd 的命名空间：

   ```shell
   $ kubectl create ns cicd
   ```

此外，你还需要 Fork Helm 包存储库，以便根据步骤操作：

包存储库地址：https://github.com/weiwendi/charts

1. 单击右上角的 “Fork” 按钮创建包存储库的 Fork：

   ![7.1 fork](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.1fork.png)

2. 创建包存储库的分支后，运行以下命令将该分支克隆到本地机器：

   ```shell
   $ git clone https://github.com/$GITHUB_USERNAME/charts
   ```

使用以下步骤从你的图表存储库中删除图表：

1. 切换到本地图表存储库的目录，我们的示例中，目录名为 charts：

   ```shell
   [aiops@aiops0113 ~]$ cd charts/
   [aiops@aiops0113 ~]$ ls
      ......
   # CNAME 文件和图表无关，只因我为该仓库设置了自定义域名，GitHub 自动生成了 CNAME 文件。
   ```

2. 从存储库中删除 README.md 之外的文件：

   ```shell
   [aiops@aiops0113 charts]$ rm -rf $(ls |egrep -v '(README.md)')
   [aiops@aiops0113 charts]$ ls
   README.md
   ```

3. 把更改推送到远程 GitHub 仓库：

   ```shell
   [aiops@aiops0113 charts]$ git add --all
   [aiops@aiops0113 charts]$ git commit -m 'Preparing for ci/cd.'
   [aiops@aiops0113 charts]$ git push origin main
   ```

4. 在 GitHub 上确认图表和索引文件已被删除，只剩 README.md 文件：

   ![7.2 delete tar and index file](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.2delete.jpg)


经过以上四步，你已经启动了 Minikube，Fork 了图表包存储库，并从 charts 中删除了 player-stats 图表及索引等文件。现在，我们开始学习创建 CI 管道发布 Helm 图表。

## 创建 CI 管道构建 Helm 图表

以图表开发者的角度，CI 的概念需要实现构建、测试、打包和发布 Helm 图表到图表存储库。在本节中，我们将讨论如何使用端到端的 CI 管道简化这一过程，并逐步引导你创建示例管道。首先，我们从设计示例管道所需的组件开始。

### CI 管道设计

在之前的章节中，我们讨论了Helm 图表的开发，整个开发过程很大程度是手动完成的。虽然 Helm 可以在 Kubernetes 集群中创建自动化的测试钩子，但在代码更改后，仍需手动执行 **helm lint**、**helm test** 或者 **ct lint-and-test** 命令，以确保能够通过测试。一旦更改后的代码经过了测试，就可以执行 **helm package** 命令对图表打包，再执行 **helm repo index** 命令创建 index.yaml 文件。如果图表库是 GitHub，就可以把包文件和索引文件推送到 GitHub 仓库中了。

虽然这些步骤可以通过手动执行命令完成，但当你开发的图表越来越多，或者有更多的开发人员参与其中，这个工作流就会逐渐变得难以掌控。在手动执行时，很可能提交未经测试的代码，而且很难保证每位开发者都遵守了测试规范。幸运的是，可以通过创建一个自动发布流程的 CI 管道避免这些问题。

下面的步骤概述了一个 CI 工作流程，它使用了本书到目前为止介绍的命令和工具。在这个示例中，假设了图表被保存在 GitHub 仓库：

1. 图表开发人员对 git monorepo 中对一个或多个图表代码进行了修改。
2. 开发人员推送修改后的代码到远程仓库。
3. 执行 **ct lint** 和 **ct install** 命令，在 Kubernetes 命名空间中自动检测和测试修改的图表。
4. 如果经过测试，自动执行 **helm package** 命令将图表打包。
5. 使用 **helm repo index** 命令生成 index.yaml 索引文件。
6. 打包后的图表和更新后的 index.yaml 文件会自动推送到存储库。它们被推送到 job 对应的分支上。

在下一节中，我们将使用 Jenkins 执行此过程。首先我们先了解 Jenkins 是什么以及它是如何工作的。

### Jenkins 介绍

Jenkins 是一个开源服务，用于执行自动化任务和工作流程。它通过 Jenkins 的管道即代码功能创建 CI/CD 管道，该功能通过名为 Jenkinsfile 的文件实现，Jenkinsfile 文件用于定义 Jenkins 管道。

Jenkins 管道是用 Groovy 特定领域语言(Domain-Specific Language DSL)编写的。Groovy 是一种类似于 Java 的语言，但与 Java 不同的是，它可以作为一种面向对象的脚本语言，适合编写易于阅读的自动化脚本。在本章中，我将带你学习两个已经编写好的 Jenkinsfile 文件：ciJenkinsfile 和 cdJenkinsfile。你不需要有任何编写 Jenkinsfile 文件的经验，因为对 Jenkins 的深入研究超出了本书的范围。也就是说，在本章结束时，你应该能够把学到的概念应用到你选择的自动化工具中。虽然本章以 Jenkins 为例，但其概念可以应用于任何其他自动化工具。

当 Jenkinsfile 被创建后，定义的工作流步骤集会在 Jenkins 主服务器或一台单独的 Agent 上执行。可以把 Agent 作为 Kubernetes 的一个 Pod 启动，从而简化 Agent 的创建和管理。在 Agent 执行完 Job 后，可以配置为自动终止。当有新的构建任务时，会在一个新创建的干净 Pod 中执行。在本章中，我们将使用 Jenkins Agent 运行示例管道。

Jenkins 可以扫描代码仓库中是否包含 Jenkinsfile 文件，这一功能使它很适合 GitOps 的概念。对于包含 Jenkinsfile 文件的分支，会自动创建一个新 Job，克隆所需分支的代码。这让测试新功能变得更加简单，因为新的 Job 可以与它们相应的分支一起自动创建。

对 Jenkins 有了基本的了解之后，现在让我们将 Jenkins 安装到 Minikube 环境中。

### 安装 Jenkins

与在 Kubernetes 上部署的其他应用程序一样，Jenkins 也可以使用 Helm Hub 上的图表部署。在本章中，我们使用 Jenkins 官方提供的 Jenkins Helm 图表。首先添加 jenkins 图表仓库：

```shell
[aiops@aiops0113 charts]$ helm repo add jenkins https://charts.jenkins.io
"jenkins" has been added to your repositories
```

Jenkins 图表除了提供 Kubernetes 的相关值，如资源限制、service 类型等，还包含了与 Jenkins 相关的值，用于自动配置不同的 Jenkins 组件。在安装前，我们定义一个值文件，用来修改 Jenkins Master 的 service 类型，默认是 ClusterIP，我们把它改成 NodePort。在我们的示例中，用的是 Multibranch 类型的 Job，Jenkins 默认并没有安装此插件，所以也需要在值文件中，提供此插件的安装信息。最终的值文件内容如下：

```shell
[aiops@aiops0113 jenkins]$ cat values.yaml
controller:
  serviceType: NodePort
  installPlugins:
    - workflow-multibranch:2.26
```

执行 **helm install** 命令安装 Jenkins：

```shell
[aiops@aiops0113 jenkins]$ helm install jenkins jenkins/jenkins --values ~/learn-helm/jenkins/values.yaml -n cicd

NAME: jenkins
LAST DEPLOYED: Thu Jul  8 02:24:10 2021
NAMESPACE: cicd
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace cicd -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export NODE_PORT=$(kubectl get --namespace cicd -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
  export NODE_IP=$(kubectl get nodes --namespace cicd -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT/login

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http:///configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-cod
e-plugin/tree/master/demos
For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
```

命令执行完成后，会输出 NOTES 信息，这时 Jenkins Master Pod 可能还没启动完成，它需要一个初始化的阶段。通过以下命令监听 Jenkins Master Pod 的状态，当全部 READY 时，就可以通过 NOTES 提供的信息访问 Jenkins Master 了：

```shell
[aiops@aiops0113 jenkins]$ kubectl get pod -w -n cicd

# 获取 admin 账号的登录密码
[aiops@aiops0113 jenkins]$ kubectl exec --namespace cicd -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo

# 获取 node IP
[aiops@aiops0113 jenkins]$ kubectl get nodes --namespace cicd -o jsonpath="{.items[0].status.addresses[0].address}"

# 获取 node port
[aiops@aiops0113 jenkins]$kubectl get --namespace cicd -o jsonpath="{.spec.ports[0].nodePort}" services jenkins
```

有了这三个信息，就可以在浏览器登录 Jenkins 控制台了。下一节，我们讨论如何配置 Jenkins。

### 配置 Jenkins

根据 NOTES 提供的信息登录 Jenkins 控制台，我们对 Jenkins 做一些必要的配置。

首先，我们添加两个凭据，一个用于向 GitHub 提交代码，一个用于访问 Kubernetes 集群。

凭据创建路径大致如此：“Manage Jenkins ” --> “Manage Credentials”  --> “Jenkins” --> “全局凭据 (unrestricted)” --> “Add Credentials”。

#### 创建 GitHub 凭据

在 CI 过程中，我们要把经过测试的 Helm 图表包推送到 Fork 出的 GitHub 包存储库，这个操作需要向 GitHub 提供凭据。在 2021 年 8 月 13 日之后，GitHub 将不再支持密码验证，需要使用 **personal access token** 替换。

我们先创建一个 token，登录 GitHub 页面，点击右上角头像旁的倒三角，会弹出一个窗口，点击其中的 “Settings” 按钮，在弹出的新页面中，点击左侧栏的 “Developer settings”，点击 “Personal access tokens”，再点击 “Generate new token”，填写 Note 信息，设置过期时间，勾选 repo 相关权限，最后点击底部的 “Generate token” 创建 token，然后把生产的 token 复制出来，我们会在创建 GitHub 凭据时用到。

![7.3 create github token.](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.3%20create%20github%20token.jpg )



有了 GitHub token，我们到 Jenkins 控制台中创建凭据，配置信息请参考下图：

![7.4 create jenkins github credentials](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.4%20create%20github%20credentials.jpg)

* Kind：使用何种方式创建该凭据，我们选择用户名、密码的方式。
* Scope：凭据的使用范围。
* Username：创建该凭据使用的用户名，这里需要填写你 GitHub 的用户名。
* Password：创建该凭据使用的密码/token，这里需要把在 GitHub 上复制的 token 粘贴进去。
* ID：凭据的标识符，可以填写一个有标示性的名称。在 Jenkinsfile 中会通过凭据 ID 引用凭据。
* Description：凭据描述信息。

#### 创建 kubeconfig 凭据

在 CI/CD 过程中，我们会通过 kubectl 和 helm 命令操作 Kubernetes 集群，这就需要在 Jenkins 创建一个访问 Kubernetes 集群的凭据，配置信息见下图：

![7.5 create jenkins kubeconfig credentials](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.5%20create%20kubernetes%20credentials.jpg)

* Kind：因为我们上传的是 kubeconfig 文件，所以类型选择为 Secret file。
* File：选择要上传的文件，在演示中我直接使用了 `~/.kube/config` 文件。

#### 创建 CI 管道

CI 是通过具体的 item 实现的，在安装 Jenkins 后，需要我们手动创建 CI item。

Item 的类型有多种，我们使用多分支流水线类型。在首页点击 “New Item”，输入 item 名称，我们 CI 示例的 item 名称为 “testAndRelease”，点击 ”多分支流水线“，然后点击 ”OK“ 创建。会弹出一个新的配置页面，我们只需配置 “Branch Sources” 和 “Build Configuration” 两个部分，其他默认即可。

在 “Branch Sources” 下点击 ”Add source“ --> “GitHub”，在 “Repository HTTPS URL” 框中输入源码仓库地址，其他地方保持默认即可，如下图：

![7.6 add repo](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.6%20add%20https%20github%20repository.png)

在 “Build Configuration” 部分，“Mode” 选择默认的 “by Jenkinsfile”，“Script Path” 是 Jenkinsfile 文件的路径及名称，我把 CI 用到的 Jenkinsfile 放在了源码仓库中的根目录下，起名为 ciJenkinsfile，所以 “Script Path” 部分只需填写文件名 ciJenkinsfile 即可，如下图：

 ![7.7 add ci jenkinsfile](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.7%20add%20ciJenkinsfile.jpg)

点击 Save，保存配置后，Jenkins 会自动对我们的配置做检查，输出以下内容就是配置完成了：

```shell
Started
[Tue Jul 13 07:06:29 UTC 2021] Starting branch indexing...
07:06:31 Connecting to https://api.github.com with no credentials, anonymous access
Examining weiwendi/learn-helm

  Checking branches...

  Getting remote branches...

    Checking branch main

  Getting remote pull requests...
      ‘ciJenkinsfile’ found
    Met criteria
Scheduled build for branch: main

  1 branches were processed

  Checking pull-requests...

  0 pull requests were processed

Finished examining weiwendi/learn-helm

[Tue Jul 13 07:06:34 UTC 2021] Finished branch indexing. Indexing took 5.3 sec
Finished: SUCCESS
```

这时，如果你细心的话，会发现 Jenkins 控制台左下角 “Build Executor Status” 会有构建任务在执行，点击进度条，能够看到管道执行过程的日志信息。

![7.8 scan repo](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.8%20scan%20repository%20build.jpg)

接下来，结合日志输出与 ciJenkinsfile 文件，我们讨论下 CI 管道的执行过程。

### CI 管道介绍

通过 Jenkinsfile 创建 CI 管道，Jenkinsfile 文件位于 learn-helm 目录下，名称为 ciJenkinsfile，可以在 https://github.com/weiwendi/learn-helm/blob/master/ciJenkinsfile 查看文件。

ciJenkinsfile 内容如下：

```groovy
// 声明 CI Pipeline.
pipeline {

    // environment 用于设置变量,变量作用域是该管道.
    environment {

        // 为 Helm Charts 源代码仓库及 Helm Charts Package 仓库设置变量.
        // gitSourceRepo 仓库包含了 Helm Charts 的源代码，也有本书其他相关代码，如 Jenkinsfile.
        // gitPackageRepo 是 Helm Package 存储库.
        // 这两个仓库地址，你都需要修改成 Fork 后的地址.
        gitSourceRepo = 'https://github.com/weiwendi/learn-helm.git'
        gitPackageRepo = 'https://github.com/weiwendi/charts.git'

        // 定义凭证变量，我们在 Jenkins 控制台创建了两个凭证，ID 分别为 github、kubeconfig.
        // 在向 gitPackageRepo 推送 Charts 包时会用到 GITAUTH 变量.
        // KUBECONFIG 会被挂在到 Pod 中，与 Kubernetes 集群进行交互.
        GITAUTH = credentials('github')
        KUBECONFIG = credentials('kubeconfig')
    }


    // 定义执行 Job 的 jenkins agent;
    // agent 是运行在 Kubernetes 集群中的一个 Pod. 
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
    # 定义容器的名称
  - name: test-and-release-ci
    # 指定容器的镜像,该镜像包含了 helm/ct/yamllint/yamale/git/kubectl.
    # 镜像 Dockerfile 可以在源码仓库的 docker 目录下查看.
    image: registry.cn-beijing.aliyuncs.com/sretech/cttools:latest
    command:
    - sleep
    args:
    - infinity
    resources:
      requests:
        memory: "1024Mi"
        cpu: "1000m"
      limits:
        memory: "1024Mi"
        cpu: "1000m"
'''
            // 设置执行各 stages 时默认使用 test-and-release-ci 容器
            defaultContainer 'test-and-release-ci'
        }
    }
	
    // Jenkins 选项
    options {

        // 显示执行的时间，需要 Timestamper 插件支持.
        timestamps()

        // 在 Stage View 中展示近  10 次构建信息
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    // 定义管道执行的各个阶段、每个阶段的具体步骤，建议包含至少一个 stage.
    stages {

        // stage 定义了 pipeline 完成的所有实际工作.
        // 首先，我们定义一个打印构建信息的 stage.
        stage('Build Messages') {

            // steps 定义了管道中具体执行的步骤.
            steps {
                
                // container 选择哪个容器执行该 steps，不是用 container 选项的话，会使用默认容器执行 steps.
                container("test-and-release-ci") {
                    echo " workspace: ${WORKSPACE}\n gitPackageRepo: ${gitPackageRepo}\n gitSourceRepo: ${gitSourceRepo}\n branch: ${env.BRANCH_NAME}\n buildId: ${BUILD_ID}"
                }
            }
        }	

        stage("Copy Kubeconfig") {
            steps {
                script {
                    sh '''
                        mkdir -p ~/.kube
                        cp ${KUBECONFIG} ~/.kube/config
                    '''
                }
            }
        }
        stage("List Changed Charts") {
            steps {
                script {
                    sh "ct list-changed"
                }
            }
        }

        stage("Lint") {
            steps {
                container("test-and-release-ci") {
                    sh "ct lint"
                }
            }
        }

        stage("Install & Test") {
            steps {
                script {
                    sh "ct install --upgrade"
                }
            }
        }
        stage("Package Charts") {
            steps {
                script {
                    sh "helm package --dependency-update helm-charts/charts/*"
                    sh "ls -l"
                }
            }
        }
        stage("Push Charts to Chart Repo") {
            steps {
                script {
                    def baseBranch = "main"

                    // 克隆 Helm Chart Package 包存储库到 chart-repo 目录.
                    sh "git clone ${env.gitPackageRepo} chart-repo"

                    // 使用 if 语句，根据分支判断图表包应该推送到 stable 或是 staging 目录.
                    def repoType
                    if (env.BRANCH_NAME == baseBranch) {
                        repoType = "stable"
                    } else {
                        repoType = "staging"
                    }

                    // 如果不存在，创建 stable 或 staging 目录.
                    def files = sh(script: "ls chart-repo", returnStdout: true)
                    if (!files.contains(repoType)) {
                        sh "mkdir chart-repo/${repoType}"
                    }

                        // 移动图表包到 stable 或 staging 目录.
                        sh "mv *.tgz chart-repo/${repoType}/"

                        // 更新索引文件.
                        sh "helm repo index chart-repo/${repoType}"
                        
                        // 更新 git 配置信息，需要填写你自己的 github 邮箱及用户名.
                        sh """
                            git config --global user.email 'weiwendi@aiops.red'
                            git config --global user.name 'weiwendi'
                        """

                    dir("chart-repo") {
		        // Add and commit the changes
		        sh "git add --all"
			sh "git commit -m 'pushing charts from branch ${env.BRANCH_NAME}'"

                        // 推送图表包到 gitPackageRepo 仓库.
			script {
			    // Inject GitHub auth and push to the repo where charts are being served
		            def authRepo = env.gitPackageRepo.replace('://', '://${GITAUTH_USR}:${GITAUTH_PSW}@')
			    sh "git push ${authRepo} ${baseBranch}"
		        }

                    }

                }
            }
        }
    }
}
```

管道在执行时，首先在 Kubernetes 的 cicd 命名空间中创建一个 Pod，作为 jenkins agent，用来完成接下来的构建任务。Pod 的命名规范是 “itemname-branchname-buildnumber-xx-xx”。

Pod 中包含了两个容器，一个是我在 ciJenkinsfile 文件中定义的，名称为 “test-and-release-ci”，使用指定的 cttools 镜像；另一个容器是 jnlp，与 jenkins master 通信。

cttools 镜像是我构建的，可以在 https://github.com/weiwendi/learn-helm/blob/master/docker/cttools/Dockerfile 中查看 Dockerfile 文件内容。镜像地址是 registry.cn-beijing.aliyuncs.com/sretech/cttools，包含了第六章中介绍的一些工具：

* **helm**
* **ct**
* **yamllint**
* **yamale**
* **git**
* **kubectl**

由于这个镜像包含了测试 Helm 图表所需的工具，因此可以将其作为执行 Helm 图表的 CI 的镜像。

当 Jenkins agent 运行时，会自动克隆指定的代码仓库。这是由 Jenkins 执行的，不需要在 Jenkinsfile 中配置。可以在日志中看到这一过程 ，在 Pod 创建完成后，有一个叫 SCM 的 stage 被执行，拉取我们在 item 中配置的 github 仓库代码。



在 Jenkins agent 完成克隆后，就开始执行 Jenkinsfile 文件中定义的阶段(stages)。阶段是管道中的逻辑分组，能可视化步骤。执行的第一个和 lint 相关的阶段是 “lint stage”，包含以下命令：

```shell
sh 'ct lint'
```

**sh** 用于运行 bash shell 命令或者脚本，本处用于执行 **ct lint** 子命令，用来检测所有针对 main 分支修改的图表中的 Chart.yaml、values.yaml 文件，我们在第六章讨论过。

如果检测成功，管道将执行下一个阶段，也就是 “test stage”，该阶段执行的命令如下：

```shell
sh 'ct install --upgrade'
```

这个命令看起来也很熟悉。它安装主分支上的每个修改过的图表版本，并执行定义好的测试步骤。它还能确保从上一版本的任何升级都是成功的，这有助于向后兼容。

虽然前面两个阶段可以通过运行 **ct lind-and-install --upgrade** 命令实现，也就是说两个阶段可以合并成一个，但为了更好的可视化所执行的操作，我们还是把它分解成了不同的阶段。

如果 “test stage” 执行成功，管道将继续执行 “package charts stage” 阶段，命令如下：

```shell
sh 'helm package --dependency-update helm-charts/charts/*'
```

这个命令将 helm-charts/charts 目录下的图表打包，它还将更新和下载声明的依赖项。

如果打包成功，管道将进入最后一个阶段 “Push Charts to gitPackageRepo”。这是最复杂的阶段，所以我们将它拆分成更小的步骤，第一步如下：

```shell
// 克隆 Helm Chart Package 包存储库到 chart-repo 目录.
sh "git clone ${env.gitPackageRepo} chart-repo"

// 使用 if 语句，根据分支判断图表包应该推送到 stable 或是 staging 目录.
def repoType
if (env.BRANCH_NAME == baseBranch) {
    repoType = "stable"
} else {
    repoType = "staging"
}

// 如果不存在，创建 stable 或 staging 目录.
def files = sh(script: "ls chart-repo", returnStdout: true)
if (!files.contains(repoType)) {
    sh "mkdir chart-repo/${repoType}"
}

```

由于我们要推送到的 Helm 图表存储库是单独的 GitHub 页面，因此必须先将存储库克隆到 Jenkins agent 本地，以便添加新图表并推送修改。克隆 GitHub 存储库后，将根据 CI/CD 管道运行的分支设置名为 repoType 的变量。此变量用于确定将前一阶段打包的图表推送到 stable(稳定) 或 staging(临时) 图表存储库中。

stable 和 staging 作为两个单独的图表存储库，可以通过在 GitHub 页面创建两个单独的目录来实现：

```shell
  charts/
  stable/
  staging/
  
```

stable 和 staging 目录中会包含自己的 index.yaml 文件，用于把它们区分为不同的图表存储库。

如果基于分支的管道执行依赖于这两个目录，为方便起见，可以在上一个阶段自动创建它们。

现在已经确定了图表应推送到的存储库，继续执行下一个阶段：

```shell
sh """
    // 将图表包移动到 chart-repo 的 stable 或 staging 目录中.
    mv *.tgz chart-repo/${repoType}/

    // 更新 index.html.
    helm repo index chart-repo/${repoType}

    // 更新 git 配置信息，需要把邮箱、用户名改为你自己的.
    git config --global user.email 'weiwendi@aiops.red'
    git config --global user.name 'weiwendi'
"""

```

第一个命令将上一个阶段中打的图表包复制到 stable/ 和 staging/ 目录中。接下来，使用 **helm repo index** 命令更新 stable/ 或 staging/ 目录中的 index.yaml 文件，以反映更改或增加的图表。

需要记住的一点是，如果我们使用不同的图表存储库解决方案，比如 ChartMuseum(一个由Helm社区维护的图表库解决方案)，那么 **helm repo index** 命令就不需要了。当 ChartMuseum 收到一个新打包的 Helm 图表时， index.yaml 文件会自动更新。对于不自动更新 index.yaml 文件的，例如 GitHub 页面，**helm repo index** 命令是必须的，正如我们在这个管道中看到的。

该阶段中的最后两条命令用来设置 git 邮箱和用户名，这是提交内容到 git 存储库必须的步骤。在本例中，我们将用户名设置为 weiwendi，邮箱设置为 weiwendi@aiops.red，这些都是示例，在操作过程中，你需要把它们改成你的真实邮箱和用户名。

最后一步是 push 操作：

```shell
dir("chart-repo") {
    // Add and commit the changes.
    sh """
        git add --all
        git commit -m 'pushing charts from branch ${env.BRANCH_NAME}'
    """

    // 推送图表包到 gitPackageRepo 仓库.
    script {
        // Inject GitHub auth and push to the repo where charts are being served
        def authRepo = env.gitPackageRepo.replace("://", "://${GITAUTH_USR}:${GITAUTH_PSW}@")
        sh "git push ${authRepo} ${baseBranch}"
    }

}

```

首先使用 **git add** 和 **git commit** 命令添加和提交图表包，接下来使用 **git push** 命令推送到存储库，推送时使用一个名为 GITAUTH 的凭据变量。这个凭证是安装期间通过 Jenkins 控制台创建的凭据。GITAUTH 凭据可以安全的引用密码，从而避免在管道代码中以明文的形式显示密码。

Helm 社区发布了一个名为 Chart Releaser 的工具，它通过调用 **helm repo index** 命令生成 index.yaml 文件，并使用 **git push** 将其推送到 GitHub。Chart Releaser 工具旨在通过管理 GitHub 页面中包含的 Helm 图表，来抽象这些额外的复杂性。不过，我们决定不在本章中使用这个工具来实现管道，因为 Chart Releaser 不支持 Helm 3(在撰写本文时)。

在构建日志中，Stage 的 steps 前都会显示执行时间，这是由 options.timestamps() 定义的。所有 stage 执行完成，最后输出 `Finished：SUCCESS` 字段，整个 CI 管道也就成功执行完成了。

现在我们已经熟悉了 CI 管道的概述，让我们来执行一个示例。

### 运行 CI 管道

我们前面已经介绍过，在保存 item 配置后，会触发 “Scan Repositor“ 的操作，针对配置的分支自动运行 CI 管道，可以通过 Jenkins 页面的 ”testAndRelease“ 连接查看。你会看到一个成功执行的 Job，运行在 main 分支上。

Jenkins 中的每个 pipeline 构建都包含执行输出的相关日志。你可以通过点击左侧蓝色圆圈旁边的链接，然后在下一个屏幕上选择 **Console Output** 查看构建日志。这个构建日志显示，第一个 stage **Lint** 成功地显示了以下信息：

```shell
All charts linted successfully

----------------------------------

No chart changes detected.

This is what we would expect because no charts were changed from the perspective of the master branch. A similar output can be seen under the install stage as well:

All charts installed successfully

-----------------------------------

No chart changes detected.

Because both the Lint and Install stages completed without error, the pipeline continued to the Package Charts stage. Here, you can view the output::

+ helm package --dependency-update helm-charts/charts/player-stats helm-charts/charts/nginx

Successfully packaged chart and saved it to: /home/jenkins/agent/workspace/t_and_Release_Helm_Charts_master/player-stats-1.0.0.tgz

Successfully packaged chart and saved it to: /home/jenkins/agent/workspace/t_and_Release_Helm_Charts_master/nginx-1.0.0.tgz

Finally, the pipeline concludes by cloning your GitHub Pages repository, creating a stable folder within it, copying the packaged charts over to the stable folder, committing the changes to the GitHub Pages repository locally, and pushing the changes to GitHub. We can observe that each file that was added to our repository is outputted in the following lines:

+ git commit -m 'pushing charts from branch master'

[master 9769f5a] pushing charts from branch master

3 files changed, 32 insertions(+)

create mode 100644 stable/player-stats-1.0.0.tgz

create mode 100644 stable/index.yaml

create mode 100644 stable/nginx-1.0.0.tgz
```

GitHub 存储库在自动推送后，页面内容如下，请忽略 CNAME 文件：

![7.9 charts repo list](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.9%20charts%20repo%20list.jpg)

在 stable 文件夹中，你应该能够看到三个不同的文件：两个独立的图表和一个 index.yaml 文件：

![7.10 stable list](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/7.10%20stable%20list.jpg)

第一个 pipeline 构建成功地创建了初始的 stable 图表集，但它没有演示如何在新图表被认为稳定，并准备好供最终用户使用前对其进行检查和测试。为了演示这一点，我们需要从 main 分支上检出一个 feature 分支，用来修改一个或多个图表，并将更改推送到 feature 分支，然后在 Jenkins 中开始一个新的构建。

首先，在 main 分支上创建一个名为 chapter7 的新分支：

```shell
$ cd charts
$ git checkout main
$ git checkout -b chapter7
```

在这个分支上，我们简单的修改 nginx 图表的版本，以触发图表的 linting 和 testing。Nginx 是一个 WEB 服务器和反向代理，它比 player-stats 应用程序要轻量的多，因此，在这个例子中，我们使用包存储库中的 nginx 图表，以避免 Minikube 中运行 Jenkins 时可能出现的资源限制。

将 helm-charts/charts/nginx/Chart.yaml 文件中 version 的值从 1.0.0 改为 1.0.1：

```yaml
version: 1.0.1
```

执行 **git status** 命令查看更改：

```shell
$ git status

On branch chapter7
Changes not staged for commit:
  (use 'git add <file>...' to update what will be committed)
  (use 'git checkout -- <file>...' to discard changes in working directory)
        modified:   helm-charts/charts/nginx/Chart.yaml
no changes added to commit (use 'git add' and/or 'git commit -a')
```

nginx 图表的 Chart.yaml 文件已经被修改，add 修改后的文件，然后执行 commit，最后将其推送到你的 Fork：

```shell
$ git add helm-charts
$ git commit -m 'bumping NGINX chart version to demonstrate chart testing pipeline'
$ git push origin chapter7
```

我们需要在 Jenkins 中触发扫描存储库，以便 Jenkins 能够检测并针对这个分支启动一个新的构建。点击 “testAndRelease” 按钮，点击左侧栏菜单中的  “Scan Repository Now” 按钮，触发 Jenkins 检测新分支，并自动启动一个新的构建。扫描大约10秒内完成。刷新页面，新的 chapter7 分支将出现在页面上。

chapter7 job 会比 main job 执行更长的时间，因为 chapter7 job 包含一个修改的 Helm 图表，该图表会使用 testing 工具进行测试。可以在 chapter7 job 的输出中查看 pipeline 的操作。pipeline 执行结束，会显示以下信息：

```yaml
Finished: SUCCESS
```

在控制台输出的日志开头部分，请注意 **ct lint** 和 **ct install** 命令是如何针对 nginx chart 运行的，因为这是唯一发生修改的 chart：

```shell
Charts to be processed:

---------------------------------------------------------------

nginx => (version: '1.0.1', path: 'helm-charts/charts/nginx')
```

这些命令的输出你应该已经很熟悉了，因为在第六章中，我们已经接触过了。

在 GitHub 页面，你应该会看到 staging 文件夹中新版本的 nginx chart，因为它不是基于 main 分支构建的。

要发布 nginx-1.0.1.tgz 图表，需要将 chapter7 分支合并到 main 分支，并推送到远程库中，命令如下：

```shell
$ git checkout main
$ git merge chapter7
$ git push origin main
```

返回 “testAndRelease” Jenkins 页面，点击 main job，点击该页面左侧栏的 “Build Now” 按钮。从输出日志中看到，图表测试被跳过了，因为 chart testing tool 将 clone 与 main 分支进行了比较，由于内容相同，tool 决定不需要进行测试。当构建完成后，可以在 GitHub 页面看到 **nginx-1.0.1.tgz** 图表在 stable 存储库中。

使用 **helm repo add** 命令在本地添加存储库，验证这些图表是否已正确发布到 GitHub stable 存储库中：

```shell
$ helm repo add aiops $gitPackageRepoPage/stable
```

$gitPackageRepoPage 的值是 GitHub 站点的地址，而不是实际的 git 存储库。地址格式类似于 https://$GITHUB_USERNAME.github.io/charts/stable，该链接可以在 GitHub 页面的 “Settings” 标签中找到，因为我在 “Settings” 的 “Page” 中配置了 CNAME，所以我的地址可以是 https://charts.aiops.red/stable。

添加了 stable 存储库后，运行以下命令查看在两次 main 构建过程中构建和推送的存储库中的图表：

```shell
$ helm search repo aiops --versions
```

在本节中，我们讨论了如何通过 CI pipeline 管理 Helm 图表的生命周期。根据示例，通过遵循自动化工作流，你可以在向用户发布图表之前轻松地执行例行检查和测试了。

下一节，我们介绍使用 CD pipeline，将 helm 图表部署到不同的环境中。

## 创建 CD 管道部署 Helm 图表

CD 管道是一组可重复的步骤，可以在单个或多个环境中，以自动化的方式部署应用程序。在本节中，我们将创建一个 CD 管道来部署 Nginx 图表，在上一节中，我们已经把它推送到了 GitHub 存储库。GitOps 还可以应用保存到 **git** 存储库中的值文件。

我们来梳理一下此管道中有哪些步骤：

1. 添加包含 Nginx 图表 release 的 GitHub stable 库。
2. 将 Nginx 图表部署到开发环境中。
3. 将 Nginx 图表部署到 QA(Quality Assurance ) 环境中。
4. 等待用户批准管道进行生产环境部署。
5. 将 Nginx 图表部署到生产环境中。

CD 工作流包含在一个单独的 Jenkinsfile 文件中，该文件的名称是 cdJenkinsfile。与 ciJenkinsfile 不同，在创建 cdJenkinsfile 前，我们先更新 Minikube 和 Jenkins 环境，以便执行 CD 流程。

### 更新环境

在 Minikube 中创建三个命名空间，对应开发、QA和生产环境。我们这里只是为了演示 CD 流程，在实际使用中，不建议把非生产环境(开发和QA)和生产环境部署在一个集群中。

创建 dev、qa、prod 命名空间，部署不同的环境：

```shell
$ kubectl create ns dev
$ kubectl create ns qa
$ kubectl create ns prod
```

删除 chapter7 分支，因为在创建新的 CD 管道时，Jenkins 会尝试在每个分支上执行它，为了简单和避免资源受限，我们仅使用 main 分支。

使用以下命令从存储库中移除 chapter7 分支：

```shell
$ git push -d origin chapter7
$ git branch -D chapter7
```

最后，在 Jenkins 控制台创建一个名为 “deployNginxChart” 的 item，步骤和创建 “testAndRelease” 一样，唯一不同的是，需要把 Jenkinsfile 的名字从 “ciJenkinsfile” 改为 “cdJenkinsfile”。

保存配置后，同样会执行代码库扫描，并自动执行 CD 管道。

### CD 管道介绍

在本节中，我们只介绍管道的主要部分，完整的 CD 管道可以参考 https://github.com/weiwendi/learn-helm/blob/master/cdJenkinsfile。

与 CI 管道一样，为了测试和发布 Helm 图表，CD 管道首先在 Kubernetes 中动态的创建一个 Jenkins Agent Pod。

Agent 创建完成后，Jenkins 就会隐式地 clone 你 fork 的源代码仓库了，就像之前在 CI pipeline 中所作的那样。

Pipeline 定义的第三个 stage 是 Add Chart，它将 GitHub stable 存储库中的图表添加到 Jenkins Agent 的本地 Helm 客户端：

```shell
sh "helm repo add aiops ${env.gitPackageRepoPage}/stable"
```

添加存储库之后，管道就可以将 Nginx 部署到不同的环境中了。下一步，部署 Nginx 图表到 dev 命名空间中，并打印出环境变量，方便我们查看：

```shell
dir("nginx-cd") {
    sh """
        helm upgrade --install nginx-${env.BRANCH_NAME} aiops/nginx --values common-values.yaml --values dev/values.yaml -n dev --wait
        kubectl -n dev exec deploy/nginx-${env.BRANCH_NAME} -- env | grep ENVIRONMENT
    """
}

```

**dir('nginx-cd')** 是 Jenkins 的语法，用于设置执行命令的工作目录。在这个 stage 中，我们给 **helm upgrade** 命令提供了 **--install** 标志。当 release 存在时，执行 **helm upgrade** 会对 release 进行升级，如果 release 不存在，则会执行失败，所以我们提供了 **--install** 标志，当 release 不存在时，会安装图表，这大大增加了自动化的程度。

在这个命令中，我们传递了两个值文件：commmon-values.yaml 和 dev/values.yaml。这两个文件都在 nginx-cd 目录下，目录内容如下：

```shell
nginx-cd/
├── common-values.yaml
├── dev
│   └── values.yaml
├── prod
│   └── values.yaml
└── qa
    └── values.yaml
```

将应用程序部署到不同的环境时，我们需要稍微修改应用程序的配置，以便与环境中的其它服务集成，比如数据库地址。dev、qa 和 prod 文件夹下的值文件，都包含了一个环境变量，该变量是在 Nginx 部署时设置的，具体取决于要部署到的环境。例如 dev/values.yaml 文件内容：

```yaml
envVars:
- name: ENVIRONMENT
  value: dev
```

同样的，qa/values.yaml 文件内容如下：

```yaml
envVars:
  - name: ENVIRONMENT
    value: qa
```

prod/values.yaml 文件内容如下：

```yaml
envVars:
  - name: ENVIRONMENT
    value: prod
```

虽然本示例中部署的 Nginx 图表很简单，并不严格要求指定这些值，但可能你已经发现了，使用这种在单独的值文件中分离特定于某个环境配置的方法，有助于处理实际环境中更复杂的用例，比如不同环境，有不同的中间件地址。相应的值文件可以通过变量的形式传递，如 **helm upgrade --install command with --values ${env}/values.yaml**，其中 ${env} 表示 dev、qa、prod。

common-values.yaml 文件从文件名可以看出，用于配置各环境的通用值。common-values.yaml 文件内容如下：

```yaml
service:
  type: NodePort
```

这个文件定义了在图表安装过程中，创建的 Nginx service 类型为 NodePort，在 values.yaml 文件中设置的其它值，也会应用于每个环境，因为它们没有被 common-values.yaml 或独立的 values.yaml 文件覆盖。

需要注意的是，应用程序在每个环境中应该以相同的方式部署。任何改变运行中的 Pod 或容器的属性值，都应该在 common-values.yaml 文件中定义，这些配置包括但不限于：

* replica 数量
* 资源的 requests 和 limits 值
* service 类型
* image 名称
* image tag
* imagePullPolicy
* values 挂载

可以针对特定环境独立修改的值如下：

* 度量或监控服务的地址
* 数据库或后端服务地址
* 应用对外暴露的 ingress URL
* 通知服务

继续介绍 “Deploy to Dev” stage 中的 **helm** 命令， 它使用了 common-values.yaml、dev/values.yaml 两个值文件，这两个值文件被用来在 dev 中安装 nginx 图表。这个命令还使用了 **-n dev** 标志，用来指定在 dev 命名空间中部署 Nginx 图表。**--wait** 标志用于暂缓输出创建信息，直到 Pod 处于 reday 状态。

在 CD 管道中，把应用部署到 dev 之后的下一个 stage 是冒烟测试，执行以下命令：

```shell
sh "helm test nginx-${env.BRANCH_NAME} -n dev"
```

Nginx 图表包含一个检查 Nginx Pod 连接的 test 钩子。如果 test 钩子能够连接到 Pod，则 test 返回成功。虽然 **helm test** 命令常用于图表测试，但它也可以用于 CD 过程中执行基本冒烟测试。冒烟测试主要用于确保程序部署后关键功能是按照设计工作的。由于 Nginx 图表测试不会以任何方式干扰正在运行的应用程序或部署环境的其它应用，因此 **helm test** 命令是确保 Nginx 图表成功部署的适当方法。

冒烟测试完成后，CD 管道将执行下一个 stage——“Deploy to QA”。此 stage 包含一个条件，用于评估管道正在执行的当前分支是否为 main 分支，如下所示：

```shell
when {
  expression {
    return env.BRANCH_NAME == 'main'
  }
}
```

此条件可以使用 feature 分支来测试 values.yaml 中的值，而不是将其放在生产环境中测试，main 分支中包含的 helm 值应该是生产就绪的。

**Deploy to QA** 使用的命令如下：

```shell
dir("nginx-cd") {
    sh """
        helm upgrade --install nginx-${env.BRANCH_NAME} aiops/nginx --values common-values.yaml --values qa/values.yaml -n qa --wait
        kubectl -n qa exec deploy/nginx-${env.BRANCH_NAME} -- env | grep ENVIRONMENT
    """
}
```

这条命令和 “Deploy to DEV” stage 里的命令相似，这里就不再详述了。

在部署到 QA 或类似的测试环境之后，可以再次运行冒烟测试，以确保 QA 部署的基本功能都能正常工作。在这个 stage，你还可以执行其他的自动化测试，在将应用程序部署到 prod 之前验证其功能是非常必要的。我在这个示例管道中省略了这些测试细节。

管道的下一个 stage 是 **Wait for Input**：

```shell
stage("Wait for Input") {
    when {
        expression {
            return env.BRANCH_NAME == "main"
        }
    }
    steps {
        input "Deploy to Prod?"
    }
}
```

这一步会暂停执行 CD Pipeline，并提示用户是否继续 “Deploy to Prod”？在运行 job 的控制台日志中，给用户两个选项——“Proceed” 和 “Abort”。虽然生产环境的部署可以自动执行，但许多开发人员可能更倾向在非生产部署和生产部署之间设置一道闸。这个 **input** 命令为用户提供了一个机会，来决定是继续部署还是在部署 QA 之后终止 CD 管道。

如果用户决定继续，则执行最后一个 stage：Deploy to Prod：

```shell
dir("nginx-cd") {
    sh """
        helm upgrade --install nginx-${env.BRANCH_NAME} aiops/nginx --values common-values.yaml --values prod/values.yaml -n prod --wait
        kubectl -n prod exec deploy/nginx-${env.BRANCH_NAME} -- env | grep ENVIRONMENT
    """
}
```

这个 stage 与 “Deploy to Dev” 和 “Deploy to QA” 两个 stage 几乎相同，除了值和命名空间。

整个 CD 管道介绍完了，我们看一下它的执行。

### 运行 CD 管道

要查看 CD pipeline 的实际操作，在 Jenkins 页面点击 “deployNginxChart” job 的 main 分支，单击 **#1** 查看日志输出。当导航到日志时，你会看到一个提示，显示 “Deploy to Prod”？我们将很快解决这个问题。首先，让我们回顾日志的开头部分，查看管道到目前为止的执行情况。

你已看到第一个部署的是 **dev** 环境：

```shell
19:14:51  + helm upgrade --install nginx-main aiops/nginx --values common-values.yaml --values dev/values.yaml -n dev --wait
19:14:56  Release "nginx-main" has been upgraded. Happy Helming!
19:14:56  NAME: nginx-main
19:14:56  LAST DEPLOYED: Fri Aug 13 11:14:53 2021
19:14:56  NAMESPACE: dev
19:14:56  STATUS: deployed
19:14:56  REVISION: 34
19:14:56  NOTES:
19:14:56  1. Get the application URL by running these commands weiwendi:
19:14:56    export NODE_PORT=$(kubectl get --namespace dev -o jsonpath="{.spec.ports[0].nodePort}" services nginx-main)
19:14:56    export NODE_IP=$(kubectl get nodes --namespace dev -o jsonpath="{.items[0].status.addresses[0].address}")
19:14:56    echo http://$NODE_IP:$NODE_PORT
19:14:56  + kubectl -n dev exec deploy/nginx-main -- env
19:14:56  + grep ENVIRONMENT
19:14:56  ENVIRONMENT=dev
```

然后，你应该看到冒烟测试，这是由 **helm test** 命令运行的：

```shell
+ helm test nginx-master -n dev

Pod nginx-master-test-connection pending

Pod nginx-master-test-connection pending

Pod nginx-master-test-connection succeeded

NAME: nginx-master

LAST DEPLOYED: Thu Apr 30 02:07:55 2020

NAMESPACE: dev

STATUS: deployed

REVISION: 1

TEST SUITE:     nginx-master-test-connection

Last Started:   Thu Apr 30 02:08:03 2020

Last Completed: Thu Apr 30 02:08:05 2020

Phase:          Succeeded19:14:57  + helm test nginx-main -n dev
19:15:05  NAME: nginx-main
19:15:05  LAST DEPLOYED: Fri Aug 13 11:14:53 2021
19:15:05  NAMESPACE: dev
19:15:05  STATUS: deployed
19:15:05  REVISION: 34
19:15:05  TEST SUITE:     nginx-main-test-connection
19:15:05  Last Started:   Fri Aug 13 11:14:57 2021
19:15:05  Last Completed: Fri Aug 13 11:15:04 2021
19:15:05  Phase:          Succeeded

```

冒烟测试之后，是 qa 环境的部署：

```shell
19:15:06  + helm upgrade --install nginx-main aiops/nginx --values common-values.yaml --values qa/values.yaml -n qa --wait
19:15:32  Release "nginx-main" has been upgraded. Happy Helming!
19:15:32  NAME: nginx-main
19:15:32  LAST DEPLOYED: Fri Aug 13 11:15:28 2021
19:15:32  NAMESPACE: qa
19:15:32  STATUS: deployed
19:15:32  REVISION: 17
19:15:32  NOTES:
19:15:32  1. Get the application URL by running these commands weiwendi:
19:15:32    export NODE_PORT=$(kubectl get --namespace qa -o jsonpath="{.spec.ports[0].nodePort}" services nginx-main)
19:15:32    export NODE_IP=$(kubectl get nodes --namespace qa -o jsonpath="{.items[0].status.addresses[0].address}")
19:15:32    echo http://$NODE_IP:$NODE_PORT
19:15:32  + kubectl -n qa exec deploy/nginx-main -- env
19:15:32  + grep ENVIRONMENT
19:15:32  ENVIRONMENT=qa
```

然后是 **input** stage，就是最初我们看到的内容：

![7.11 Proceed or Abort](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/Figure_7.17.jpg)

在 “Stage View” 页面单击 “Proceed” 继续 pipeline 的执行（单击 “Abort” 将导致 pipeline 终止，并阻止生产环境的部署）。然后，你将看到 prod 部署的提示如下：

```shell
19:19:31  + helm upgrade --install nginx-main aiops/nginx --values common-values.yaml --values prod/values.yaml -n prod --wait
19:19:35  Release "nginx-main" has been upgraded. Happy Helming!
19:19:35  NAME: nginx-main
19:19:35  LAST DEPLOYED: Fri Aug 13 11:19:33 2021
19:19:35  NAMESPACE: prod
19:19:35  STATUS: deployed
19:19:35  REVISION: 1
```

最后，如果生产部署成功，您将在管道的末尾看到以下消息：

```shell
[Pipeline] End of Pipeline

Finished: SUCCESS
```

可以从命令行手动验证部署是否成功。使用 **helm list** 命令查看 **nginx-master** 版本：

```shell
$ helm list -n dev
$ helm list -n qa
$ helm list -n prod
```

每个命令都应该在对应的命名空间中列出 nginx release：

```shell
NAME             NAMESPACE     REVISION  

nginx-master       dev           1
```

你还可以使用 kubectl 列出每个命名空间中的 Pods，并验证是否部署了 Nginx：

```shell
$ kubectl get Pods -n dev
$ kubectl get Pods -n qa
$ kubectl get Pods -n prod
```

每个命名空间的结果如下（dev 还将有一个冒烟测试阶段的 test pod）：

```shell
NAME                    READY   STATUS    RESTARTS   AGE

nginx-fcb5d6b64-rmc2j   1/1     Running   0          46m
```

在本节中，我们讨论了如何在 CD 管道中使用 Helm 在 Kubernetes 中的多个环境中部署应用程序。管道依赖于 GitOps 存储配置(values.yaml 文件)的实践。并引用这些文件来正确配置 Nginx。了解了 Helm 如何在 CD 环境中使用后，现在就可以清理 Minikube 集群了。

## 环境清理

要清理本章练习中的 Minikube 集群，请删除 **chapter7**、**dev**、**qa** 和 **prod** namespaces：

```shell
$ kubectl delete ns chapter7
$ kubectl delete ns dev
$ kubectl delete ns qa
$ kubectl delete ns prod
```

关闭 Minikube VM：
```shell
$ minikube stop
```
