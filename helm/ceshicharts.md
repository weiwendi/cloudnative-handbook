# 6. 测试 Helm 图表

测试是软件开发过程中必不可少的环节。对软件进行测试，既可以验证功能，又可以防止软件在运行过程中潜在的 Bug。实践证明，经过良好测试的软件更易于维护，开发人员也可以更自信地向用户提供新功能。

我们开发的 Helm 图表也应该经过适当的测试，以确保达到了预期。这也是本章的重点，我会在本章讨论测试 Helm 图表的方法。

## 6.1 环境准备

还是老套路，咱们先把部署 Helm 图表的环境准备好：

1. 启动 Minikube：

   ```bash
   [aiops@aiops0113 ~]$ minikube start
   ```

2. 创建 website 命名空间：

   ```bash
   [aiops@aiops0113 ~]$ kubectl create ns website
   ```

Minikube 环境就绪后，我们开始讨论如何测试 Helm 图表。

## 6.2 验证 Helm 模板

在上一章中，我们从头构建了一个 Helm 图表。在开发完成后，最终产品还是相当复杂的：包括参数化、条件模板、生命周期钩子。由于 Helm 的主要目的之一是创建 Kubernetes 资源，因此在将资源模板应用到 Kubernetes 集群前，应该确保这些模板都已正确生成。可以通过多种方式验证模板，我会在下一节具体讨论。

### 6.2.1 使用 helm template 验证模板

验证图表模板的第一种方法是使用 **helm template** 命令，该命令可以在本地渲染图表模板，并在标准输出中显示模板渲染后的结果。

**helm template** 命令语法如下：

```bash
$ helm template [NAME] [CHART] [flags]
```

NAME 参数用来指定 release 名称，CHART 参数用来指定图表库。我们使用存储库中的 aiops/player-stats 作为示例，演示 **helm template** 命令。aiops/player-stats 就是咱们在上一章从 GitHub 添加到本地的图表存储库。

运行以下命令在本地渲染 player-stats 图表：

```bash
[aiops@aiops0113 ~]$ helm template myplayer-stats aiops/player-stats -n website
---
# Source: player-stats/charts/mongodb/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mongo
  namespace: website
  labels:
    app.kubernetes.io/name: mongodb
    helm.sh/chart: mongodb-10.15.2
    app.kubernetes.io/instance: myplayer-stats
    app.kubernetes.io/managed-by: Helm
secrets:
  - name: mongo
...      
```

该命令会打印出应用到 Kubernetes 集群的所有资源，上面的内容截取了 **helm template** 命令输出内容的开始部分，显示了一个渲染后的 ServiceAccount 资源类型。在本地渲染这些资源，开发人员可以清晰的看到创建的资源和规范。

在图表开发过程中，你可能会经常使用 **helm template** 命令验证是否正确生成了 Kubernetes 资源。

需要验证的常见内容如下：

* 参数化的字段是否被默认值或者覆盖值成功替换
* 流程控制语句，如 if、range 和 with 是否成功地根据提供的值生成了 YAML 文件
* 资源是否包含适当的间距和缩进
* 是否使用函数和管道正确格式化和操作 YAML 文件
* 诸如 required 和 fail 之类的函数能否根据用户输入正确地验证值

了解了在本地渲染图表模板后，我们再来看看在一些特定方面如何使用 **helm template** 命令进行测试和验证。

### 6.2.2 模板参数化测试

检查模板中的参数是否被正确的值填充，也是很重要的一项测试。因为图表可能包含多个值，如果值不合理、或者没有提供值，会导致图表渲染失败。

试想以下部署：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
  replicas: {{ .Values.replicas }}
<skipping>
          ports:
            - containerPort: {{ .Values.port }}
```

replicas 和 port 的默认值在 values.yaml 文件中：

```yaml
replicas: 1
port: 8080
```

针对此模板资源，执行 **helm template** 命令，模板渲染结果如下，replaces 和 port 的值都被替换成了 values.yaml 文件中定义的默认值：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
  replicas: 1
<skipping>
          ports:
            - containerPort: 8080
```

根据输出结果，我们可以验证参数是否被默认值正确替换了。还可以在执行 **helm template** 命令时通过 --values 或 --set 传递值，验证参数是否被成功覆盖：

```bash
$ helm template my-chart $CHART_DIRECTORY --set replicas=2
```

执行结果中涵盖了我们提供的值：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
  replicas: 2
<skipping>
          ports:
            - containerPort: 8080
```

使用 **helm template** 测试默认值很容易，难的是测试特定值，因为这种情况下，提供了无效值会导致图表安装失败。可以结合 required 和 fail 函数完成验证。

假设有这么一个 Deployment 模板：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
  replicas: {% raw %}{{ .Values.replicas }}{% endraw %}
<skipping>
      containers:
        - name: main
          image: {% raw %}{{ .Values.imageRegistry }}{% endraw %}/{% raw %}{{ .Values.imageName }}{% endraw %}
          ports:
            - containerPort: {{ .Values.port }}
```

如果此 deployment 和我们上面的代码块使用同一个 values.yaml 文件，并且在安装图表时，你希望用户提供 imageRegistry 和 imageName 的值。我们在执行 **helm template** 命令时不提供这些值，执行结果就和我们期望的不一样了，会输出如下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
  replicas: 1
<skipping>
      containers:
        - name: main
          image: /
          ports:
            - containerPort: 8080
```

通过 **helm template** 命令，我们发现因为没有传入值，deployment 模板被渲染后，image 的值是 `/`，这是一个无效值。为了避免所需值没有被正确定义的情况发生，我们可以使用 required 函数进行验证：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
  replicas: {% raw %}{{ .Values.replicas }}{% endraw %}
<skipping>
      containers:
        - name: main
          image: {% raw %}{{ required 'value 'imageRegistry' is required' .Values.imageRegistry }}{% endraw %}/{% raw %}{{ required 'value 'imageName' is required' .Values.imageName }}{% endraw %}
          ports:
            - containerPort: {% raw %}{{ .Values.port }}{% endraw %}
```

更新图表后，再次执行 **helm template** 命令，会返回一个错误信息，提示用户缺少了值：

```bash
$ helm template my-chart $CHART_DIRECTORY
Error: execution error at (player-stats/templates/deployment.yaml:34:21): value 'image.repository' is required
```

可以通过为 **helm template** 命令传递一个值文件，来进一步测试。对于此示例，我们假设值文件的内容如下：

```yaml
imageRegistry: my-registry.example.com
imageName: learnhelm/my-image
```

将用户定义的值文件传递给 **helm template** 的命令如下：

```bash
$ helm template my-chart $CHART_DIRECTORY --values my-values.yaml
```

这次模板渲染后的内容，image 的值就是值文件中提供的值了。

在进行参数化测试时，请确保验证了图表中使用的每个值，对于无法在 values.yaml 文件中提供的默认值，可以使用 required 函数验证。使用 **helm template** 命令确保正确渲染并生成了所需的 Kubernetes 资源配置。

顺便说一句，可以在 values.yaml 文件中为必填字段添加注释，这样用户在查看 values.yaml 文件时就能知道哪些字段是必须的：

```yaml
replicas: 1
port: 8080
## REQUIRED
imageRegistry:
## REQUIRED
imageName:
```

注释内容是不会出现在 **helm template** 命令的输出中的。

接下来，我们探讨如何在本地生成图表模板来帮助我们测试图表的控制操作。

### 6.2.3 测试控制操作

除了测试参数化外，你还应该使用 **helm template** 命令验证控制操作(特别是 if 和 range)是否得到了正确的处理，以产生所需的结果。

假设有以下 deployment 模板：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
{{- range .Values.env }}
          env:
            - name: {% raw %}{{ .name }}{% endraw %}
              value: {% raw %}{{ .value }}{% endraw %}
{{- end }}
{{- if .Values.enableLiveness }}
          livenessProbe:
            httpGet:
              path: /
              port: {% raw %}{{ .Values.port }}{% endraw %}
            initialDelaySeconds: 5
            periodSeconds: 10
{{- end }}
          ports:
            containerPort: 8080
```

当 env 和 enableLiveness 的值都为 null 时，通过 **helm template** 命令来测试模板渲染是否成功：

```bash
$ helm template my-chart $CHART_DIRECTORY --values my-values.yaml

---

# Source: test-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
          ports:
            - containerPort: 8080
```

可以看到 range 和 if 操作没有生成任何内容，这是因为 range 子句不会对 Null 或空值执行任何操作，即使提供给 if 执行，也会被当做 false。通过向 **helm template** 提供 env 和 enableLiveness 的值，可以验证已经编写好的模板能否正确生成 YAML。

可以将这些值添加到 values 文件中，如下所示：

```yaml
env:
  - name: weChatPublicId
    value: sretech

enableLiveness: true
```

完成这些更改后，使用这些值验证 **helm template** 命令的执行结果，以证明模板是正确编写的：

```yaml
---
# Source: test-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
          env:
            - name: weChatPublicId
              value: sretech
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          ports:
            - containerPort: 8080
```

在向图表中添加其他控制结构时，你应该确保使用 **helm template** 命令及时对模板渲染进行了测试，因为这些结构会增大图表开发的难度，特别是控制结构比较多或比较复杂时。

除了检查控制结构是否正确生成之外，你还应该检查用到的函数和管道是否按计划工作。

### 6.2.4 测试管道和函数

**helm template** 命令也可用于测试管道和函数的渲染，这些管道和函数通常用于生成格式化的 YAML。

以下面的模板为例：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
          resources:
{% raw %}{{ .Values.resources | toYaml | indent 12 }}{% endraw %}
```

该模板使用管道格式化和参数化资源的值，以满足容器的需求。比较合理的做法是图表的 values.yaml 文件中设置一些合理的默认值，确保应用程序都有适当的资源限制，避免滥用集群资源造成资源浪费。

包含 resources 值的模板示例如下：

```yaml
resources:
  limits:
    cpu: 200m
    memory: 256Mi
```

执行 **helm template** 命令，查看这些值是否按照正确的缩进生成了有效的 YAML 文件。输出如下：

```yaml
apiVersion: apps/v1
kind: Deployment
<skipping>
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
```

接下来，我们将讨论在使用 **helm template** 渲染资源时如何启用服务器端验证。

### 6.2.5 为图表渲染添加服务器端验证

虽然 **helm template** 命令对于开发图表很有帮助，并且在图表开发过程中你应该经常用它验证图表的渲染，但它也有一个很大的限制。**helm template** 主要用于客户端本地渲染，不与 Kubernetes API 进行交互验证资源。如果你想确保生产的资源是有效的，可以使用 **--validate** 标志指示 **helm template** 在资源生成后与 Kubernetes API 进行通信：

```bash
$ helm template my-chart $CHART_DIRECTORY --validate
```

任何生成的 Kubernetes 模板，如果不是有效的 Kubernetes 资源，都会在错误信息中打印出来。假设，在一个 deployment 模板中 apiVersion 的值设为了 aipVersion: v1。我们知道，在 Kubernetes API 中，apiVersion 的值是 apps/v1。所以这是一个错误的资源，使用 **--validate** 标志，我们会看到下面的错误：

```bash
Error: unable to build kubernetes objects from release manifest: unable to recognize '': no matches for kind 'Deployment' in version 'v1'
```

**--validate** 用于打印生成的错误资源，使用此标志需要有访问 Kubernetes 的权限。

在执行 **install/upgrade/rollback/uninstall** 命令时，也可以使用 **--dry-run** 标志验证。

下面是使用 **install** 命令使用此标志的示例：

```bash
$ helm install my-chart $CHART --dry-run
```

此标志会生成图表的模板，并执行验证，类似于 **helm template --validate** 命令。**--dry-run** 会把生成的每个资源打印到终端，而不会在 Kubernetes 环境中创建这些资源。主要用于在用户最终安装前，执行此命令以确保是按期望进行安装的。图表开发人员可以使用 **--dry-run** 标志测试图表的渲染，也可以使用 **helm template --validate** 命令进行验证。

验证了模板是否按期望生成，我们也需要确保模板是最优的，以降低开发和维护成本。Helm 提供了 **helm lint** 命令，可用于此，我们在下一节探讨。

### 6.2.6 helm lint

对图表进行整理，能有效防止图表格式或图表定义文件出错，并在使用 Helm 图表时提供最佳实践指导。**helm lint** 命令语法如下：

```bash
$ helm lint PATH [flags]
```

**helm lint** 命令需要指定图表目录，用来验证该图表的格式是否正确。

**helm lint** 命令不验证已渲染的 API 模式，也不检查 YAML 格式，而只是检查图表中是否包含有效的 Helm 图表应该包含的文件和设置。

可以针对 player-stats 图表执行 **helm lint** 命令，在 helm-charts 目录下执行以下命令：

```bash
[aiops@aiops0113 helm-charts]$ helm lint player-stats
==> Linting player-stats
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

1 chart(s) linted, 0 chart(s) failed 说明图表是有效的。[INFO] 消息建议图表在 values.yaml 文件中包含一个 icon 字段，但这不是必需的。其它的消息类型有 [WARNING]，表示违反了图表惯例；[ERROR] 表示图表会安装失败。

让我们看几个例子。假设有一个结构如下的图表：

```bash
player-stats/
  templates/
  values.yaml
```

这个图表结构是有问题的，缺少了定义图表元数据的 Chart.yaml 文件，对这样的图表执行 **lint** 命令，会有以下报错：

```yaml
==> Linting .
Error unable to check Chart.yaml file in chart: stat Chart.yaml: no such file or directory
Error: 1 chart(s) linted, 1 chart(s) failed
```

错误信息显示在图表中找不到 Chart.yaml 文件。我们试着在图表中创建一个空的 Chart.yaml 文件，再次执行看看是什么情况：

```yaml
==> Linting .
[ERROR] Chart.yaml: name is required
[ERROR] Chart.yaml: apiVersion is required. The value must be either 'v1' or 'v2'
[ERROR] Chart.yaml: version is required
[INFO] Chart.yaml: icon is recommended
[ERROR] templates/: validation: chart.metadata.name is required
Error: 1 chart(s) linted, 1 chart(s) failed
```

这段输出显示了 Chart.yaml 文件中缺少的必要的字段：name、apiVersion、version，这些字段是 Chart.yaml 文件中的必要字段，用来生成有效的 Helm 图表。**helm lint** 还反馈了额外的信息，apiVersion 的值必须是 v1 或 v2，version 字段的值是否为适当的 Semver 版本。**lint** 还会检查其他需要的文件，比如 values.yaml 和 template 目录。它还会确保 template 目录下的文件的扩展名是否为 .yaml/.yml/.tpl/.txt。**helm lint** 对于检查图表的内容很有用，但不会检查文件中具体的 YAML 格式。

要执行 YAML 检查，可以使用 **yamllint** 工具，该工具可以在 https://github.com/adrienverge/yamllint 上找到。可以通过 **pip** 包管理器安装：

```bash
pip install yamllint --user
```

它也可以使用操作系统的包管理器安装，如 https://yamllint.readthedocs.io/en/stable/quickstart.html 上的 yamllint 快速入门所述。

rockyLinux 安装 yamllint：

```bash
[root@aiops0113 ~]# dnf install -y epel-release
[root@aiops0113 ~]# dnf install -y yamllint
```

为了在图表 YAML 资源上使用 **yamllint**，必须结合 **helm template** 命令渲染 Go 模板并生成 YAML 资源。因为图表的 YAML 资源中包含了 Go 模板，所以 **yamllint** 需要和 **helm template** 结合使用。下面是对 GitHub 包存储库中的 player-stats 图表运行 **yamllint** 命令的示例：

```bash
[aiops@aiops0113 helm-charts]$ helm template myplayer-stats player-stats |yamllint -
stdin
  83:1      warning  comment not indented like content  (comments-indentation)
  124:1     error    trailing spaces  (trailing-spaces)
  128:1     error    trailing spaces  (trailing-spaces)
  142:1     error    trailing spaces  (trailing-spaces)
  191:81    error    line too long (141 > 80 characters)  (line-length)
  203:23    error    trailing spaces  (trailing-spaces)
```

该命令将 templates/ 目录中的模板文件渲染成 Kubernetes 的资源文件，并将输出通过管道传递给 **yamllint** 命令。

这些是 **yamllint** 输出的 Helm 模板的内容，虽然有行号，但要和 YAML 资源文件中的行对应上，我们还需通过以下命令：

```bash
[aiops@aiops0113 helm-charts]$ cat -n <(helm template myplayer-stats player-stats)
     1	---
     2	# Source: player-stats/charts/mongodb/templates/serviceaccount.yaml
     3	apiVersion: v1
     4	kind: ServiceAccount
     5	metadata:
     6	  name: mongo
     7	  namespace: default
     8	  labels:
     9	    app.kubernetes.io/name: mongodb
    10	    helm.sh/chart: mongodb-10.15.2
    11	    app.kubernetes.io/instance: myplayer-stats
    ...
```

**yamllint** 可以根据以下规则进行 YAML 检查：

* 缩进
* 行长度
* 空格
* 空行
* 注释格式

可以通过创建以下文件之一设置规则，从而覆盖默认规则：

* **.yamllint, .yamllint.yaml, and .yamllint.yml **，当前工作目录下
* **$XDB_CONFIG_HOME/yamllint/config**
* **~/.config/yamllint/config**

要重写针对 player-stats 图表的缩进规则报告，可以在当前目录下创建 **.yamllint.yaml** 文件，文件内容如下：

```yaml
rules:
  indentation:
    # Allow      myList
    #            - item1
    #            - item2
    # Or
    #            myList
    #              - item1
    #              - item2
    indent-sequences: whatever
```

这个配置会覆盖 yamllint，这样在添加列表条目时，就不会强制一个特定的缩进方法了。这由 indent-sequences: whatever 行定义。创建该文件后，再次执行 **yamllint**，就不会有缩进报错了：

```bash
$ helm template myplayer-stats player-stats | yamllint -
```

在本节中，我们讨论了如何通过 **helm template** 和 **helm lint** 命令，验证图表的本地渲染。然而，这实际上并没有测试图表的功能或使用图表创建的应用程序资源的运行情况。

在下一节中，我们将学习如何在 Kubernetes 环境中创建测试，以测试 Helm 图表能否正常安装并运行。

### 6.2.7 在集群中进行测试

为图表创建测试，是维护和开发图表的重要组成部分。图表测试有助于验证图表是否按照预期运行，避免在添加功能或修复程序时回退。

测试包括两个步骤。首先需要在图表的 templates/ 目录下创建包含 helm.sh/hook: test 注解的 Pod 模板，这些 Pod 用于执行测试图表和应用程序功能的命令。第二步需要执行 **helm test** 命令，它会使用第一步中的注解创建测试钩子。

在这一节中，我们通过在 player-stats 图表中添加测试，来学习如何在 Kubernetes 集群中进行测试。参考样例在 https://github.com/weiwendi/learn-helm/tree/main/helm-charts/charts/player-stats 代码仓库中。

首先在 templates/test/ 目录中添加 frontend-connection.yaml 和 mongodb-connection.yaml 文件。图表测试虽然不必放在 test 子目录下，但将它们放在那里既能保持测试组织有序，又能与图表主模板分离：

```bash
[aiops@aiops0113 helm-charts]$ mkdir player-stats/templates/test
[aiops@aiops0113 helm-charts]$ touch player-stats/templates/test/frontend-connection.yaml
[aiops@aiops0113 helm-charts]$ touch player-stats/templates/test/mongodb-connection.yaml
```

 接下来，我们向这两个文件中添加内容，以验证其关联的应用程序组件。

#### 6.2.7.1 创建图表测试

Player-stats 图表由 Go 前端和 MongoDB 后端组成，用户在前端对话框中输入信息，提交后被存储到后端。我们写几个测试，确保安装后前后端可用。先从 MongoDB 开始，编辑 templates/test/mongodb-connection.yaml 文件，添加以下内容，这些内容也可以在 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/templates/test/mongodb-connection.yaml 页面查看：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {% raw %}{{ include "player-stats.fullname" . }}{% endraw %}-test-mongodb-connection
  labels:
    {% raw %}{{- include "player-stats.labels" . | nindent 4 }}{% endraw %}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  containers:
    - name: test-mongodb-connection
      image: bitnami/mongodb:4.4.6-debian-10-r0
      # image: registry.cn-beijing.aliyuncs.com/sretech/mongo:4.4.6
      command: ["/bin/sh", "-c"]
      args: ['mongo --host {% raw %}{{ .Values.mongodb.fullnameOverride }}{% endraw %}:{% raw %}{{ .Values.mongodb.service.port }}{% endraw %} --quiet --eval "db.infor.find()" stats']
  restartPolicy: Never
```

该模板定义了在 test 生命周期钩子期间创建 Pod，模板中还定义了一个删除钩子策略，用来指示何时删除以前的 test Pod。

containers 对象下的 args 字段是用于测试的命令，使用 mongo 客户端工具连接到 MongoDB，并执行 find() 命令。前端会把用户提交的信息存储为 MongoDB 的文档消息。这个简单的测试，旨在检查是否可以连接到 MongoDB 数据库，它返回的结果和用户通过页面查询到的内容是相同的。

再来配置前端的可用性测试，因为它是直面用户的。编辑 templates/test/frontend-connection.yaml 文件添加以下内容，这些内容也可以在 https://github.com/weiwendi/learn-helm/blob/main/helm-charts/charts/player-stats/templates/test/frontend-connection.yaml 页面查看：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {% raw %}{{ include "player-stats.fullname" . }}{% endraw %}-test-frontend-connection
  labels:
    {% raw %}{{- include "player-stats.labels" . | nindent 4 }}{% endraw %}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  containers:
    - name: test-frontend-connection
      image: curlimages/curl:7.68.0
      command: ["/bin/sh", "-c"]
      args: ['curl {% raw %}{{ include "player-stats.fullname" . }}{% endraw %}']
  restartPolicy: Never
```

这是一个非常简单的测试，对 player-stats 服务发起 HTTP 请求，来检查前端是否可用。

到此，我们完成了图表测试所需的模板。当然，这些模板可以使用 **helm template** 命令在本地渲染，也可以使用 **helm lint** 和 **yamllint** 进行规范，就像本章之前介绍的。在开发图表时，这种更高级的测试，有时会非常有用。

测试已经写好了，我们在 Minikube 环境中运行它们。

#### 6.2.7.2 执行图表测试

要执行图表测试，必须先执行 **helm install** 命令在 Kubernetes 集群中安装图表。因为我们编写的测试用例是在安装完成后运行的。可以在安装图表时使用 --wait 标志，方便确定 Pod 何时准备就绪：

```bash
[aiops@aiops0113 helm-charts]$ helm install myplayer-stats player-stats -n website --wait
```

一旦图表安装完成，就可以使用 **helm test** 命令执行 test 生命周期钩子并创建测试资源。 **helm test** 命令的语法如下：

```bash
helm test [RELEASE] [flags]
```

对已安装的 myplayer-stats release 进行测试：

```bash
[aiops@aiops0113 helm-charts]$ helm test myplayer-stats -n website
```

如果测试成功，你将在输出中看到以下内容：

```yaml
NAME: myplayer-stats
LAST DEPLOYED: Wed Jun 16 02:42:49 2021
NAMESPACE: website
STATUS: deployed
REVISION: 1
TEST SUITE:     myplayer-stats-test-frontend-connection
Last Started:   Wed Jun 16 02:44:54 2021
Last Completed: Wed Jun 16 02:44:56 2021
Phase:          Succeeded
TEST SUITE:     myplayer-stats-test-mongodb-connection
Last Started:   Wed Jun 16 02:44:56 2021
Last Completed: Wed Jun 16 02:44:58 2021
Phase:          Succeeded
```

在运行测试时，可以使用 --logs 标志，将执行测试时的日志打印到命令行。使用 --logs 标志，再次运行测试：

```bash
[aiops@aiops0113 helm-charts]$ helm test myplayer-stats -n website --logs
```

这次的输出中，除了与前面相同的输出内容外，还输出了进行测试的容器日志，这些额外的输出，和使用 **kubectl logs** 看到的是一样的。以下是测试前端连接的部分日志内容：

```yaml
POD LOGS: myplayer-stats-test-frontend-connection
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>球员统计</title>
```

下面是测试后端连接的部分日志输出：

```yaml
POD LOGS: myplayer-stats-test-mongodb-connection
```

后端测试的日志内容是空的，因为我们尚未向 player-stats 提交任何信息。可以在前端页面提交信息后，再次执行此测试。

在本小节，我们编写了两个简单的测试用例，对安装的图表进行冒烟测试。这些测试帮助我们确认修改后的图表，能够正常运行。

在下一节中，我们来讨论如何使用 **ct** 工具改进测试过程。

## 6.3 Chart testing project

我们已经编写了测试 player-stats 应用程序是否被成功安装的测试用例，但这些测试用例在执行的过程中，是有一些限制的。

第一个限制是如何测试图表值可能出现的不同排列。**helm test** 命令无法修改 release 的值，不能测试安装或更新时未指定的值，因此在对不同的值执行 **helm test** 时，需要遵循以下流程：

1. 使用一组初始值安装图表。
2. 针对 release 执行 **helm test**。
3. 删除执行完 **helm test** 的 release。
4. 使用另一组值安装图表。
5. 重复 2 - 4 步，直到对大部分值进行了测试。

除了值的不同排列外，还应确保对图表的修改不会发生回退。在测试图表的较新版本时，防止回退的最佳方式是按照以下流程执行：

1. 安装之前的图表版本。
2. 将 release 升级到较新的图表版本。
3. 删除 release。
4. 安装较新的图表版本。

针对排列组合中的每一组值重复此流程，确保对图表的修改不会发生回退或中断。

这些过程很乏味，尤其是在维护众多不同的 Helm 图表时，图表开发人员可能会承受额外的压力，在这些图表上应该进行仔细的测试。在维护多个 Helm 图表时，图表开发人员倾向于使用 git monorepo 设计，也就是把多个或所有的图表存储在一个 git 仓库中。

一个存储 Helm 图表的 monorepo 文件结构类似如下：

```bash
helm-charts/
  player-stats/
    Chart.yaml
    templates/
    README.md
    values.yaml
  mongodb/           
  wordpress/       
  README.md
```

Monorepo 中的 Helm 图表在修改后应进行测试，以确保修改后的图表能正常运行。修改图表时，Chart.yaml 文件中的 version 字段也应该按照 SemVer 规范进行增加，以表示本次代码更改的程度。SemVer 规范遵循 MAJOR.MINOR.PATCH 版本编号格式。

以下是增加 SemVer 版本号的操作指南：

* 对图表进行突破性的更改需要增加 MAJOR 版本号。突破性的更改是指当前做的修改会使图表不向后兼容。
* 添加一个非突破性的功能，与之前的版本兼容，则应增加 MINOR 版本号。
* 对错误或安全漏洞进行修补，且更改后与之前的图表版本向后兼容，则应增加 PATCH 版本号。

如果没有完善的自动化工具，很难确保每次修改图表时都进行了测试，并且版本号也做了增加，尤其是在维护存储了多个 Helm 图表的 monorepo 时。这一挑战促使 Helm 社区创建了 **ct** 项目，用于在测试和维护图表时提供结构化和自动化的功能。

### 6.3.1 Chart testing project 介绍

图表测试项目(chart testing project，可以在 https://github.com/helm/chart-testing 页面找到)用来对 git monorepo 中的图表执行自动检测、验证和测试。自动化测试是通过 git 检测图表目标分支的更改来实现的。更改的图表应该经过测试，而没有更改的图表则不需要测试。

**ct** 提供了四个主要命令：

* **lint：**检查并验证更改的图表。
* **install：**安装和测试更改的图表。
* **lint-and-install：**梳理、安装和测试更改的图表。
* **list-changed：**列出更改的图表。

**list-changed** 命令不执行任何验证或测试，**lint-and-install** 是 **lint** 命令和 **install** 命令的组合，它还会检查每个 Chart.yaml 文件的 version 字段是否增加了。内容更改了，但 version 没有增加的图表，将测试失败。这种验证有助于维护人员根据所做更改的类型严格地修改图表版本。

除了检查图表版本外，chart testing 还可以对多个值文件进行测试。在调用 **lint、install、lint-and-install** 命令时，chart testing 循环遍历每个需要测试的值文件，并根据提供的不同值排列执行验证和测试。为了区分这些值文件与图表的 values.yaml 文件，需要将测试的值文件存放在名为 ci/ 的文件夹下，文件结构如下：

```yaml
player-stats/
  Chart.yaml
  ci/
    nodeport-service-values.yaml
    ingress-values.yaml
  templates/
  values.yaml
```

Chart testing 会应用 ci/ 目录下的所有值文件，不论这些文件的名称是什么。根据被覆盖的值命名每个值文件是个很好的习惯，这样维护者和贡献者就能很直观的知道值文件的内容。

实际工作中，常用的 **ct** 命令是 **lint-and-install**。下面列出了该命令用于检测、安装和测试 git monorepo 中修改的图表的步骤：

1. 检测被修改的图表。
2. 使用 **helm repo update** 命令更新本地 Helm 缓存。
3. 使用 **helm dependency build** 命令下载修改过的图表的依赖项。
4. 检查每个修改过的图表 version 是否有增加。
5. 对于在步骤 4 中计算为 true 的图表，检查 ci/ 目录中的每个值文件。
6. 对于步骤 4 中计算为 true 的图表，执行以下步骤：
   - 在自动创建的命名空间中安装图表。
   - 运行 **helm test** 执行测试。
   - 删除命名空间。 
   - 对 ci/ 目录下的每个值文件重复此操作。

**lint-and-install** 命令执行了多个操作，在命名空间中安装和测试每个修改过的图表，并对 ci/ 目录中的值文件重复此过程，以确保图表得到充分、正确的检查和测试。但是，默认情况下，**lint-and-install** 命令不会从图表的旧版本执行升级检查向后兼容性。可以通过 **--upgrade** 标志启用该特性：

如果未指定是突破性更改，则 **--upgrade** 标志会以以下步骤代替上面的第六步：

1. 在自动创建的命名空间中安装旧版本的图表。
2. 执行 **helm test** 对 release 进行测试。
3. 将 release 升级到图表的修改版本，并再次执行测试。
4. 删除命名空间。
5. 在一个新的自动创建的命名空间中安装修改的图表版本。
6. 执行 **helm test** 对 release 进行测试。
7. 使用相同的图表版本再次升级 release，并重新执行测试。
8. 删除命名空间。
9. 对 ci/ 目录下的每个值文件重复上述操作。

建议在执行 chart testing 时使用 **--upgrade** 标志，以便对 Helm 的升级进行额外的测试，避免升级后可能出现的回退。

如果增加了 Helm 图表的 MAJOR 版本，**--upgrade** 标志将失效，因为这表明你做了一个突破性的更改，在当前版本上进行原地升级是不会成功的。

我们在本地安装 chart testing CLI 及其依赖的工具，以便观察测试过程。

### 6.3.2 安装 chart testing  工具

为了使用 chart testing CLI，你必须在本地机器上安装以下工具：

* **helm**
* **git(version 2.17.0 or later)**
* **yamllint**
* **yamale**
* **kubectl**

Chart testing 在测试过程中用到了这些工具。**helm** & **kubectl** & **yamllint** & **git** 我们已经使用过了，所以现在只需要安装 **Yamale**。**yamale** 是一个 chart testing 工具，用来对照 Chart.yaml 模式文件验证图表中的 Chart.yaml 文件。

**Yamale** 可以通过 **pip** 包管理器安装，如下所示：

```bash
$ pip install yamale --user
```

你也可以通过从 https://github.com/23andMe/Yamale/archive/master.zip 下载压缩文件手动安装 Yamale。

下载后，解压压缩文件并执行安装脚本：

```bash
# 此命令需要使用 root 用户执行
$ python setup.py install
```

这些工具安装完成后，就可以下载 chart testing 工具了 ，GitHub 下载地址：https://github.com/helm/chart-testing/releases。在该页面下载和操作系统相符的最新版本。

解压下载的文件，会有以下内容：

```bash
LICENSE
README.md
etc/chart_schema.yaml
etc/lintconf.yaml
ct
```

LICENSE、README.md 文件不是必须的，可以删除。把 etc/chart_schema.yaml、etc/lintconf.yaml 两个文件移动到 /etc/ct/ 目录中。**ct** 文件移动到系统的 $PATH 变量路径中，我们把它放到 /usr/local/bin/ 目录下。

```bash
$ mkdir $HOME/.ct
$ mv $HOME/Downloads/etc/* /etc/ct/
$ mv $HOME/Downloads/ct /usr/local/bin/
```

所有的工具都已准备完成，接下来我们演示如何对本地的图表存储库进行修改，并使用 chart testing 整理和安装修改后的图表。

如果你还没有 clone 存储库到本地主机，可以使用以下命令进行 clone：

```bash
$ git clone https://github.com/weiwendi/learn-helm.git
```

clone 完成后，你会发现，该存储库中有一个 ct.yaml 的文件，内容如下：

```yaml
target-branch: main
chart-dirs:
  - helm-charts/charts
chart-repos:
  - bitnami=https://charts.bitnami.com/bitnami
  - mycharts=https://weiwendi.github.io/charts
validate-maintainers: false
```

chart-dirs 字段定义了 **ct** 的 helm-charts/charts 目录，相对于 ct.yaml 文件是图表库 monorepo 的根。chart-repos 字段提供了一个存储库列表，chart testing 会运行 **helm repo add**，以确保能够下载依赖。

还有其他配置可以添加到这个文件中，这里不做讨论，可以在 https://github.com/helm/chart-testing 的 chart testing 文档中查看。每次调用 **ct** 命令都会引用 ct.yaml 文件。

现在工具已经安装好了，包存储库也 clone 到了本地，我们可以执行 **ct lint-and-install** 命令来测试图表了。

### 6.3.3 执行 ct lint-and-install

**lint-and-install** 命令作用于 learn-helm/helm-charts 目录下的 Helm 图表：

* player-stats：这是我们前一章创建的图表。
* nginx：这是通过 **helm create** 命令创建的、用于演示的 Helm 图表。

要执行测试，首先切换到  learn-helm 存储库的顶层目录：

```bash
[aiops@aiops0113 ~]$ cd learn-helm
```

ct.yaml 文件通过 chart-dirs 字段定义 Helm 图表的 monorepo 的位置，因此，你可以从顶层路径运行 ct lint-and-install 命令。

```bash
[aiops@aiops0113 learn-helm]$ ct lint-and-install
Linting and installing charts...
-------------------------------------
No chart changes detected.
-------------------------------------
All charts linted and installed successfully
```

运行此命令后，你会看到类似上面的输出内容。

因为没有任何图表被修改，所以 **ct** 不会对图表执行任何操作。我们修改图表，观察 **ct** 的执行过程。修改图表应该在非 main 分支进行，我们先创建一个名为 test 的新分支：

```bash
[aiops@aiops0113 learn-helm]$ git checkout -b test
Switched to a new branch 'test'
```

修改范围可大可小，我们简单地修改每个图表的 Chart.yaml 文件。修改 learn-helm/helm-charts/charts/player-stats/Chart.yaml 中的 description 字段为：

```yaml
description: Used to deploy the player-stats application.
# 原来的值为：A Helm chart for Kubernetes.
```

修改 learn-helm/helm-charts/charts/nginx/Chart.yaml 文件中的 description 字段为：

```yaml
description: Deploys an NGINX instance to Kubernetes.
# 原来的值为：A Helm chart for Kubernetes.
```

修改这些图表后，再次运行 **lint-and-install** 命令：

```bash
[aiops@aiops0113 learn-helm]$ ct lint-and-install
Linting and installing charts...
...
 ✖︎ nginx => (version: "0.1.0", path: "helm-charts/nginx") > Chart version not ok. Needs a version bump!
 ✖︎ player-stats => (version: "1.0.0", path: "helm-charts/player-stats") > Chart version not ok. Needs a version bump!
...
Error: Error linting and installing charts: Error processing charts
Error linting and installing charts: Error processing charts
```

执行报错，因为 **ct** 检测到 monorepo 中的两个图表发生了更改，而 version 并没有增加。

这个问题可以通过增加 player-stats 和 nginx 图表的版本来解决。由于此更改并没有引入新功能，因此我们将增加 PATCH 的版本号。将两个图表的 Chart.yaml 文件中的 version 字段分别修改为 1.0.1：

```yaml
version: 1.0.1
```

执行 **git diff** 命令，确认每个图表都进行了修改后，再次执行 **lint-and-install** 命令：

```bash
[aiops@aiops0113 learn-helm]$ ct lint-and-install
```

在增加了图表版本后，**lint-and-install** 命令遵循了完整的 chart testing 工作流。从输出中可以看到，修改过的图表被 lint 并部署到了自动创建的命名空间中。一旦部署的应用程序的 Pod 就绪，**ct** 会根据 helm.sh/hook: test 注解自动运行图表的测试用例。Chart testing 输出了每个测试 Pod 的日志及命名空间事件。

从 **lint-and-install** 的输出中， 你可能注意到了，nginx 图表部署了两次，player-stats 图表只部署和测试了一次。这是因为 nginx 图表的 ci/ 目录中，包含了两个值文件。ci/ 目录中的值文件通过 chart testing 进行迭代，有几个值文件，就会部署几次，以确保每组值的组合都能成功安装。Player-stats 图表没有 ci/ 目录，因此只安装了一次。

这些可以在 **lint-and-install** 输出的下面几行中观察到：

```bash
Linting chart with values file 'helm-charts/charts/nginx/ci/clusterip-values.yaml'...
Linting chart with values file 'helm-charts/charts/nginx/ci/nodeport-values.yaml'...
Installing chart with values file 'helm-charts/charts/nginx/ci/clusterip-values.yaml'...
Installing chart with values file 'helm-charts/charts/nginx/ci/nodeport-values.yaml'...
```

此命令仅能测试图表的功能，不能验证能否对图表进行升级。为此，我们需要在执行 **lint-and-install** 命令时指定 **--upgrade** 标志。添加 **--upgrade** 标志再次执行该命令：

```bash
[aiops@aiops0113 learn-helm]$ ct lint-and-install --upgrade
```

该命令使用 ci/ 目录下的值文件原地升级 release。这可以在输出中看到，如下所示：

```bash
esting upgrades of chart 'player-stats => (version: "1.0.1", path: "helm-charts/charts/player-stats")' relative to previous revision 'player-stats => (version: "1.0.0", path: "ct_previous_revision074583074/helm-charts/charts/player-stats")'...
```

只有 MAJOR 版本相同时，才可以执行升级。在 MAJOR 版本不同时，执行 --upgrade，会收到如下错误提示：

```bash
Skipping upgrade test of 'nginx => (version: "1.0.1", path: "helm-charts/charts/nginx")' because: 1 error occurred:
        * 1.0.1 does not have same major and minor version as 0.1.0
```

测试的相关知识，到这里就介绍完了。