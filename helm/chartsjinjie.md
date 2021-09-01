# Helm 图表进阶

在上一章中，我从使用者角度讨论了 Helm 的用法，并将其作为包管理器在 Kubernetes 集群中安装了 WordPress 应用程序。安装过程非常简单，不需要任何 Kubernetes 专业知识，也无需对安装的应用程序有所了解，因为所需资源和逻辑都封装在了 Helm 图表中。我们只需要了解图表中的值，以便自定义配置一些选项。

知其然，只能勉强成为合格的执行者；知其所以然才能有更好的发展。在这一章中，我会深入介绍图表的工作原理，以及如何创建图表。

准备好与上一章相同的 Kubernetes 集群和 Helm 环境，我们开始一探图表究竟。

## YAML 语法介绍

YAML(YAML Ain't Markup Language)一种易于人类阅读的文件格式，是配置 Kubernetes 资源最常用的文件格式，也是 Helm 图表中大多数文件的格式。

YAML 遵循键值对(key-value)格式声明配置，我们来看一下 YAML 的键值对结构。

### 定义键值对

下面是 YAML 最基本的键值对示例：

```yaml
blog: "cloudnative.aiops.red"
```

在这个示例中，定义了 blog 键，它的值是 cloudnative.aiops.red。在 YAML 中键和值之前用冒号 `:` 分隔。冒号左边的字符是键，冒号右边的字符是值。

在 YAML 格式中，空格非常重要。下面是一个错误的 YAML 键值对示例：

```yaml
blog:"cloudnative.aiops.red"
```

冒号和 cloudnative.aiops.red 字符串之间缺少空格，会导致解析错误。YAML 格式要求冒号和值之间必须要有空格。

YAML 也支持嵌套元素或块定义更复杂的键值对，示例：

```yaml
resources:
  limits:
    cpu: 100m
    memory: 512Mi
```

示例中的 resources 对象映射了两个键值对，如下表：

| Key                     | Value |
| :---------------------- | ----- |
| resources.limits.cpu    | 100m  |
| resources.limits.memory | 512Mi |

键是通过 YAML 块下的缩进来确定的，每个缩进都向键名添加一个点 `.` 分隔符。当 YAML 块中不再有任何缩进的时候，就是值。按照使用规范，YAML 的缩进使用两个空格表示，但用户可以使用任意数量的空格，只要在整个文件中空格数是一致的即可。

YAML 不支持制表符(Tab 键)，使用制表符将会导致解析错误。

### 值类型

YAML 文件中的值可以是不同类型，常见的类型是字符串，它是文本值。可以通过引号将值引起来，声明字符串类型(如第一个示例中的 "cloudnative.aiops.red")。但这不是必须的，只要值中包含字母或特殊字符，不论是否有引号，就会被视为字符串。可以使用管道符(|)设置多行字符串，示例：

```yaml
configuration: |
  server.port=8443
  logging.file.path=/var/log
```

值也可以是整数。当值是没有用引号括起来的数字时，这个值就是整数类型。这个 YAML 示例声明了一个整数类型的值：

```yaml
replicas: 1
```

将其与下面的 YAML 进行比较，后者为 replicas 分配给一个字符串类型的值：

```yaml
replicas: '1'
```

布尔值也经常被使用，可以用 true 或 false 声明：

```yaml
ingress:
  enable: true
```

这个 YAML 示例设置了 ingress.enable 为 true 的布尔值类型。其他布尔值包括：yes、no、on、off、y、n、Y、N。 

值也可以是更复杂的类型，比如列表(list)。YAML 中的列表项由 `-` (减号)标识。下面是一个值为列表类型的示例：

```yaml
servicePorts:
  - 8080
  - 8443
```

servicePorts 的值是一个整数列表 8080 和 8443。这种语法也可以用在对象上：

```yaml
deployment:
  env:
    - name: MY_VAR
      value: MY_VALUE
    - name: SERVICE_NAME
      value: MY_SERVICE
```

在本例中，env 是一个包含 name 和 value 字段的对象列表。在配置 Kubernetes 和 Helm 的 YAML 文件时，我们会经常用到列表，所以要记住这种格式的写法。

YAML 便于人类阅读的特性，使它在 Kubernetes 和 Helm 的配置中比较流行，但也可以使用 JSON 格式替代 YAML。JSON 也遵循键值对格式，类似 YAML，主要的区别在于 YAML 依赖空格缩进配置键值对，而 JSON 依赖于大括号和方括号。下面是将 YAML 格式转换为 JSON 格式的示例：

```json
{
  'deployment': {
    'env': [
      {
        'name': 'MY_VAR',
        'value': 'MY_VALUE'
      },
      {
        'name': 'SERVICE_NAME',
        'value': 'MY_SERVICE'
      }
      ]
  }
}
```

JSON 中的所有键都用引号括起来，放在冒号前面：

* 使用花括号({)表示块，这与 YAML 中以缩进表示块的方式类似。
* 使用方括号([)表示列表，这与 YAML 中以减号表示列表的方式类似。

YAML 和 JSON 格式还有很多结构，但本文介绍的知识点已足够我们在 Helm 中使用了。

在下一节中，我将讨论 Helm 图表的文件结构，你可能会注意到它包含了 YAML 和 JSON 文件。

## Helm 图表结构

图表作为 Kubernetes 的包，必须要遵循一定的文件结构。

```shell
$ ls my-chart/
  chart 文件和目录...
```

把 Helm 图表的顶级目录作为 Helm 图表的名称，这是最佳命名规则。虽然 Helm 不严格要求这么做，但这样命名能增加图表的辨识度。在上个示例中，my-chart 是图表的顶级目录，也是这个图表的名称。

顶级目录里包含了组成图表的文件和目录。下表列出了可能包括的文件和目录：

| 文件/目录           | 定义                                                         | 是否必须                                          |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------- |
| Chart.yaml          | 该文件中包含了图表的元数据                                   | 是                                                |
| templates/          | 存放图表的资源模板文件，这些文件是 YAML 格式                 | 是(除非依赖项在 Chart.yaml 文件中声明)            |
| templates/NOTES.txt | 图表的说明文件。安装、升级或执行 notes 命令会输出此文件内容  | 否                                                |
| values.yaml         | 图表的默认值文件                                             | 否，但作为最佳实践，最好包含此文件                |
| .helmignore         | 指定忽略 Helm 包中哪些文件和目录                             | 否                                                |
| charts/             | 存放图表依赖的其他图表                                       | 无需明确提供，Helm 的依赖管理系统会自动创建此目录 |
| Chart.lock          | 保存先前版本的依赖                                           | 无需明确提供，Helm 的依赖管理系统会自动创建此文件 |
| crds/               | 存储自定义(CRD)资源的目录。YAML 格式。这些资源会在 templates/ 中的资源之前安装。 | 否                                                |
| README.md           | 图表安装和使用说明的自述文件                                 | 否，但作为最佳实践，最好包含此文件                |
| LICENSE             | 图表 license 文件                                            | 否                                                |
| values.schema.json  | JSON 格式的图表值文件                                        | 否                                                |

我会逐个介绍这些文件或目录，以便大家了解如何创建 Helm 图表。首先从 templates/ 目录开始，它是动态生成 Kubernetes 资源的关键。

### 图表模板

Helm 图表之所以能够创建和管理在 Kubernetes 集群中部署应用程序所需的资源，主要是通过模板(templates)实现的。用户将自定义的值传递给模板，就可以生成所需的 Kubernetes 资源。在本节中，我会详细介绍模板和值是如何结合的。

Helm 图表中必须包含 templates/ 目录，存放生成 Kubernetes 资源的 YAML 模板文件，文件列表大致如下：

```yaml
templates/
  configmap.yaml
  deployment.yaml
  service.yaml
```

configmap.yaml 文件的内容大致如下：

```yaml
{% raw %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}
data:
  configuration.txt: |-
    {{ .Values.configurationData }}
{% endraw %}
```
你可能觉得这段代码不是有效的 YAML 文件，这是因为 configmap.yaml 文件仅是一个 Helm 模板文件，它可以根据特定的值修改资源配置，然后生成有效的 YAML 资源文件。两个花括号 `{{ ... }}` 中的内容是 Go 语言模板的输入文本，会在图表安装或更新期间被替换成对应的值。

Go 模板是什么？如何使用它生成 Kubernetes 资源文件呢？带着这两个问题，我们进入下一小节。

#### Go 模板

Go 是谷歌开发的编程语言，Kubernetes、Helm、Istio 以及云原生的许多生态系统都是使用 Go 开发的，对 Go 感兴趣的朋友，欢迎阅读我的《Go 入门与实战》系列文章。模板是 Go 语言的核心组件，可以生成不同格式的文件。对于 Helm，Go 模板用于在 Helm 图表的 templates/ 目录下生成 Kubernetes YAML 资源文件。

Go 模板以两个左花括号 `{{` 开始、两个右花括号 `}}` 结束，这些花括号虽然在 YAML 文件中可见，但在图表安装或升级的时候，会被自动删除。

在第五章，我会详细介绍 Go 模板的使用，并带你从零到一完成图表开发。

#### 使用值和内置对象参数化字段

Helm 图表中包含一个名为 values.yaml 的文件，提供了图表所需的默认值。这些值会被 Go 模板引用，并交给 Helm 处理，动态生成 Kubernetes 资源。

values.yaml 文件的格式大致如下：

```yaml
## weChatPublicId is my WeChat public number
weChatPublicId: sretech

## weChatPublicName is my WeChat public name
weChatPublicName: 魏文弟
```

以井号(#)开头的行是注释行，对键值对进行说明，在执行时会被忽略。我们应该为每个值都提供注释，以便用户理解如何使用这些值。文件中的其他行表示键值对。

以 .Values 开头的 Go 模板引用 values.yaml 文件中定义的值，或者在安装、升级过程中，使用 --set、--values 参数传入的值。

下面的示例是模板文件中的内容：

```yaml
env:
  - name: WECHATPUBLIC_ID
    value: {% raw %}{{ .Values.weChatPublicId }}{% endraw %}
  - name: WECHATPUBLIC_NAME
    values: {% raw %}{{ .Values.weChatPublicName }}{% endraw %}
```

模板处理之后的 YAML 资源如下：

```yaml
env:
  - name: WECHATPUBLIC_ID
    value: sretech
  - name: WECHATPUBLIC_NAME
    values: 魏文弟
```

.Values 是一个可用于参数化的内置对象(Built-in Objects)，用来引用图表值。Helm 的所有内置对象可以在 Helm 文档页面(https://helm.sh/docs/chart_template_guide/builtin_objects/)查看。下表是我整理的常用内置对象：

| **内置对象**                                   | **定义**                                                   |
| ---------------------------------------------- | ---------------------------------------------------------- |
| .Release.Name                                  | release 名称                                               |
| .Release.Namespace                             | release 指定的命名空间                                     |
| .Release.Revision                              | 安装或升级时的修订号。安装时是 1，每次升级或回退都会自增。 |
| .Values                                        | 用于引用 values.yaml 或用户提供的值                        |
| .Chart.Name, .Chart.Version, .Chart.AppVersion | 用于引用 Chart.yaml 文件，引用字段包括 Chart.$Field        |
| .Files.Get                                     | 获取图表目录中的文件                                       |
| .Files.AsSecrets                               | 以Base 64编码字符串的形式返回文件内容                      |
| .Files.AsConfig                                | 使用 YAML 格式返回文件内容                                 |
| .Capabilities.APIVersions                      | 返回 kubernetes 集群中可用的 API 版本的列表                |
| .Template.Name                                 | 返回此对象使用的模板的相对文件路径                         |

对象前面的点(.)表示对象作用域。点号后跟对象名，表示将作用域限制为该对象。例如 .Values 仅作用于图表的值；.Release 作用域仅使 Release 对象下的字段可见；.scope 表示全局范围，使所有对象可见。

#### VALUES.SCHEMA.JSON

在学习值和参数化前，我们先了解下 values.schema.json 文件，它并不是图表目录中必须存在的文件。values.schema.json 强制在值文件中执行特定的模式，这种模式可用于在图表安装或更新期间验证用户所提供的值。

values.schema.json 文件的内容片段：

```json
{
  '$schema': 'https://json-schema.org/draft-07/schema#',
  'properties': {
    'replicas': {
      'description': 'number of application instances to deploy',
      'minimum': 0
      'type' 'integer'
    },
    . . .
  'title': 'values',
  'type': 'object'
}
```

有了 values.schema.json 模式文件，replicas 的值最小只能设置为 0，这就完成了对值的限制。该文件确保了用户只能提供图表模板支持的值。

Go 模板不仅支持图表开发人员参数化 Helm 图表，也支持开发人员在 YAML 文件中使用条件逻辑。接下来我们介绍这个特性。

#### 模板中的流程控制

模板的参数化功能允许使用值替换模板中的字段，而流程控制语句，进一步增加了配置灵活性。以下是在模板中定义流程控制的几个关键词：

| 动作    | 定义                       |
| ------- | -------------------------- |
| if/else | 有条件地包含或排除某些资源 |
| with    | 用于修改值引用的范围       |
| range   | 用于循环遍历列表中的值     |

在图表模板的开发过程中，有时可能需要包含或排除某些 Kubernetes 资源或资源的某些部分。if…else 动作可满足此需求。以下是 Deployment 模板中包含条件块的代码内容：

```yaml
readinessProbe:
{{- if .Values.probeType.httpGet }}
  httpGet:
    path: /healthz
    port: 8080
    scheme: HTTP
{{- else }}
  tcpSocket:
    port: 8080
{{- end }}
  initialDelaySeconds: 30
  periodSeconds: 10
```

if 语句可以根据条件判断设置 readinessProbe，如果 probeType.httpGet 的值为 true 或非 null，httpGet  readinessProbe 将被模板化。否则，readinessProbe 将为 tcpSocket 类型。花括号中的减号表示在处理后应该删除空格。左花括号后的减号可以删除花括号前面的空格，右花括号前面的减号可以删除花括号后面的空格。

图表开发人员还可以使用 with 动作修改值的作用域。当引用深度嵌套的值时，非常适合使用 with。通过减少引用深度嵌套值所需的字符数量，提高模板文件的可读性和可维护性。

下面的代码是值文件的一部分，包含了深度嵌套的值：

```yaml
application:
  resources:
    limits:
      cpu: 100m
      memory: 512Mi
```

如果没有 with 动作，这些值在模板文件中被引用的方式如下：

```yaml
cpu: {% raw %}{{ .Values.application.resources.limits.cpu }}{% endraw %}	
memory: {% raw %}{{ .Values.application.resources.limits.memory }}{% endraw %}
```

with 动作允许开发人员修改这些值的作用域，使用简短的语法引用它们：

```yaml
{% raw %}{{- with .Values.application.resources.limits }}{% endraw %}
cpu: {% raw %}{{ .cpu }}{% endraw %}
memory: {% raw %}{{ .memory }}{% endraw %}
{{- end }}
```

开发人员还可以使用 range 动作执行重复操作，遍历循环列表类型的值。假设某个图表包含以下值：

```yaml
servicePorts:
  - name: http
    port: 8080
  - name: https
    port: 8443
  - name: monitor
    port: 8881
```

上面的代码提供了一个 servicePort 列表，它可以被循环使用，如下所示：

```yaml
spec:
  ports:
{{- range .Values.servicePorts }}
  - name: {% raw %}{{ - name }}{% endraw %}
  port: {% raw %}{{ .port }}{% endraw %}
{{- end }}
```

with 和 range 的作用域是所提供的对象。在 range 示例中，range 作用于 .Values.servicePorts 对象，点 `.`号将作用域限制在此对象下定义的值。要引用所有值或是设置全局作用域，开发人员应该在引用前加上美元 `$` 符号，示例：

```yaml
{{- range .Values.servicePorts }}
  - name: {% raw %}{{ $.Release.Name }}{% endraw %}-{% raw %}{{ .name }}{% endraw %}
  port: {% raw %}{{ .port }}{% endraw %}
{{- end }}
```

除了图表值外，还可以为模板传递变量。

#### 模板变量

图表模板也支持变量，定义如下：

```yaml
{% raw %}{{ $weChatPublicId := 'sretech' }}{% endraw %}
```

设置 weChatPublicId 变量的值为 sretech(本人公众号 ID，欢迎关注☺) 。变量也可以赋值给对象，比如图表值：

```yaml
{% raw %}{{ $myvar := .Values.greeting }}{% endraw %}
```

在模板中引用变量的方式如下：

```yaml
data:
  greeting.txt: |
    {% raw %}{{ $myvar }}{% endraw %}
```

使用变量的最佳场景之一是在 range 块中，变量被设置为捕获列表迭代的索引和值：

```yaml
data:
  greetings.txt: |
{{- range $index, $value := .Values.greetings }}
    Greeting {% raw %}{{ $index }}{% endraw %}: {% raw %}{{ $value }}{% endraw %}
{{- end }}
```

结果如下：

```yaml
data:
  greetings.txt: |
    Greeting 0: Hello
    Greeting 1: Hola
    Greeting 2: Hallo
```

变量还可以简化对映射的迭代处理，如下所示：

```yaml
data:
  greetings.txt: |
{{- range $key, $val := .Values.greetings }}
    Greeting in {% raw %}{{ $key }}{% endraw %}: {% raw %}{{ $val }}{% endraw %}
{{- end }}
```

可能的结果如下：

```yaml
data:
  greetings.txt: |
    Greeting in English: Hello
    Greeting in Spanish: Hola
    Greeting in German: Hallo
```

也可以使用变量引用当前作用域之外的值。

考虑下面的 with 代码块：

```yaml
{{- with .Values.application.configuration }}
My application is called {% raw %}{{ .Release.Name }}{% endraw %}
{{- end }}
```

在这个模板中，.Release.Name 是不会被处理的，因为它不在 .Values.application.configuration 范围内。解决这个问题的一种方式是在 with 块上方，设置一个值为 .Release.Name 的变量：

```yaml
{% raw %}{{ $appName := .Release.Name }}{% endraw %}
{{- with .Values.application.configuration }}
My application is called {{ $appName }}
{{- end }}
```

当然，这种方式虽然能解决问题，但不是最好的。可以使用 `$` 符引用全局范围的资源，代码行数会更少、更易阅读。

流程控制和变量为动态生成资源提供了有力的支持，除此之外，开发人员还可以使用函数和管道来协助资源的渲染。

#### 使用函数和管道处理复杂数据

Go 提供了函数和管道的概念，以支持在模板中处理复杂的数据。

Go 模板中的函数和其他结构或编程语言中的函数类似，旨在处理某些输入，并根据输入提供输出的逻辑。

Go 模板通过以下方法调用函数：

```go
functionName arg1 arg2 . . .
```

indent 是常用的 Go 模板函数之一，该函数用于缩进字符串，以确保字符串被正确格式化，因为 YAML 对空格很敏感。indent 函数接收两个参数，第一个参数是要缩进的空格数，第二个参数要缩进的字符串。下面是 indent 函数的模板示例：

```yaml
data:
  application-config: |-
{% raw %}{{ indent 4 .Values.config }}{% endraw %}
```

这个示例将 config 值中的字符串缩进 4 个空格，以确保字符串在 application-config YAML 键下正确缩进。

Helm 提供的另一种构造是管道。管道是从 UNIX 借来的概念：一个命令的输出被作为另一个命令的输入：

```shell
cat file.txt | grep helm
```

上面的例子是一个 UNIX 管道。管道(|)的左边是第一个命令，右边是第二个命令。

第一个命令 **cat file.txt** 输出 file.txt 文件的内容，并将其作为第二个命令 **grep helm** 的输入，第二个命令用来在第一个命令的输出中过滤出包含 helm 单词的行。

Go 管道以类似的方式工作，再次以 indent 函数来演示：

```yaml
data:
  application-config: |-
{% raw %}{{ .Values.config | indent 4 }}{% endraw %}
```

同样将 config 的值缩进 4 个空格，管道适合将多个命令连接在一起执行，阅读起来也连续自然。

有许多 Go 模板函数可以在 Helm 图表中使用。这些函数可以在 https://golang.org/pkg/text/template/#hdr-Functions 的 Go 文档和 http://masterminds.github.io/sprig/ 的 Sprig 模板库中找到。以下是一些常见的 Go 模板函数，你可能会在图表开发中用到：

* **date：**格式化日期
* **default：**设置默认值
* **fail：**模板渲染失败
* **include：**执行 Go 模板并返回结果
* **nindent：**将文本缩进指定数量的空格，并在缩进之前添加一个新行
* **indent：**将文本缩进指定数量的空格
* **now：**显示当前日期/时间
* **quote：**用引号将字符串括起来
* **required：**要求用户输入
* **splitList：**将字符串拆分为字符串列表
* **toYaml：**将字符串转换为 YAML 格式

Go 模板语言还包含布尔操作符，可以在 if 动作中使用它们来进一步控制 YAML 资源的生成：

* **and**
* **or**
* **not**
* **eq**（相等）
* **ne**（不等）
* **lt**（小于）
* **le**（小于或等于）
* **gt**（大于）
* **ge**（大于或等于）

除了生成 Kubernetes 资源，Go 模板还可以重用具有相同模板的 YAML 资源。这是通过命名模板实现的。

#### 使用命名模板实现代码重用

在创建模板文件时，可能存在重复的 YAML 块。我们以标签资源作为示例：

```yaml
labels:
  'app.kubernetes.io/instance': {% raw %}{{ .Release.Name }}{% endraw %}
  'app.kubernetes.io/managed-by': {% raw %}{{ .Release.Service }}{% endraw %}
```

标签可以附加到任意资源对象上，一个资源对象上也可以添加任意数量的标签。将资源对象与标签捆绑，可以实现多维度的资源分组管理。如果图表包含许多不同的 Kubernetes 资源，那么在每个文件中定义所需的标签就会很麻烦，特别是在需要修改或者向资源添加新标签的时候。

Helm 提供了命名模板的概念，允许图表开发人员创建可重用的模板，从而减少重复的配置块。命名模板在 templates/ 目录下定义，这些文件以下划线开头，以 .tpl 作为文件扩展名。许多图表包含 _helpers.tpl 的命名模板文件，尽管该文件不需要被命名为 helpers。

可以使用 define 动作在 tpl 文件中创建命名模板。下面代码创建了一个封装资源标签的命名模板：

```yaml
{% raw %}{{- define 'mychart.labels' }}{% endraw %}
labels:
  'app.kubernetes.io/instance': {% raw %}{{ .Release.Name }}{% endraw %}
  'app.kubernetes.io/managed-by': {% raw %}{{ .Release.Service }}{% endraw %}
{{- end }}
```

define 动作以模板名作为参数，在上面的示例中，模板名就是 mychart.labels。模板名的命名约定是 $CHART_NAME.$TEMPLATE_NAME，其中 $CHART_NAME 是 Helm 图表的名称，$TEMPLATE_NAME 是一个简短的描述性名称，用来描述模板的用途。

mychart.labels 这个名称说明了模板来自于 mychart 图表，并且它会生成所需的标签资源。

要在 Kubernetes YAML 模板中使用命名模板，需要 include 函数，语法如下：

```yaml
include [TEMPLATE_NAME] [SCOPE]
```

TEMPLATE_NAME 参数是命名模板的名称，SCOPE 参数定义了应用的值和内置对象的范围。大多数情况下，这个参数是一个点(.)，表示当前的顶级作用域，如果指定的模板引用了当前作用域之外的值，则应该使用 $ 符号。

下面的示例演示如何使用 include 函数处理命名模板：

```yaml
metadata:
  name: {% raw %}{{ .Release.Name }}{% endraw %}
{% raw %}{{- include 'mychart.labels' . | indent 2 }}{% endraw %}
```

这个示例中，首先设置了模板名称为 release 名称。接着使用 include 函数处理标签(labels)，并使用管道将标签行缩进两个空格。当处理完成后，一个名为 template-demonstration 的 release 中的资源可能会如下所示：

```yaml
metadata:
  name: template-demonstration
  labels:
    'app.kubernetes.io/instance': template-demonstration
    'app.kubernetes.io/managed-by': Helm
```

Helm 还提供了 template 动作，可以扩展命名模板。template 的用法与 include 相同，但不能在管道中使用。由于这个原因，开发人员应该尽量使用 include 而不是 template，因为 include 不仅具有与 template 相同的特性，还提供了管道的功能。

在下一节中，我们将讨论如何使用命名模板来减少多个不同图表中重复的模板块。

#### 图表库

Helm 图表分为应用程序(application)和库(library)两种类型，可以通过 Chart.yaml 文件中的 type 字段设置。应用程序类型的图表用于在 Kubernetes 集群中部署应用程序，也是常见的、默认设置的图表类型。图表也可以定义为库类型，这种类型的图表不能部署应用程序，而是提供可跨多个图表使用的命名模板。我们依然使用上一个小结的标签示例，开发人员可能维护成百上千的图表，每个图表都具有相同的资源标签，单独维护每个图表的 _helpers.tpl 文件就变得复杂且多余了。在这种情况下，图表开发人员可以声明一个库类型的图表，用来提供生成资源标签的命名模板。

虽然 Helm 常用来创建 Kubernetes 的原生资源，但也可以用它创建自定义资源(CRs)，我将在下一节详细介绍。

#### CR 模板

CR 用于创建非原生的 Kubernetes API 资源，扩展 Kubernetes 提供的功能。CR 可以使用 Helm 模板创建，就像创建原生的 Kubernetes 资源一样，但创建前必须有对应的 Custom Resource Definition (CRD)，用来定义 CR。 如果在创建 CR 之前没有 CRD，CR 将会创建失败。

在 Helm 图表中创建 crds/ 目录，存放 CRD。crds/ 目录示例如下：

```
crds/
  my-custom-resource-crd.yaml
```

my-custom-resource-crd.yaml 文件的内容大致如下：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: my-custom-resources.learnhelm.io
spec:
  group: cloudnative.aiops.red
  names:
    kind: MyCustomResource
    listKind: MyCustomResourceList
    plural: MyCustomResources
    singular: MyCustomResource
    scope: Namespaced
    version: v1
```

 templates/ 目录中可以包含 MyCustomResource 资源的实例。

```yaml
templates/
  my-custom-resource.yaml
```

这样的结构确保了在安装自定义资源(CR)前已经定义了 MyCustomResource  CRD。

创建 CRD 需要管理员权限，如果你不是管理员，需要让管理员帮你创建好 CRD。如果是这样， crds/ 目录就不需要包含在图表中了，因为 CRD 在集群中已经存在了。

到现在为止，我已经详细介绍了 Helm 模板，总而言之，它是图表的大脑，用于生成 Kubernetes 资源。

接下来，我会继续介绍 Helm 图表的其它基础知识，与模板同样重要的 Chart.yaml 文件。

### 图表定义

Chart.yaml 文件又被称为图表定义，用于配置图表元数据。该文件在图表中是必须存在的，否则会报错：

```yaml
Error: validation: chart.metadata is required
```

在第三章中，我们执行 **helm show chart** 命令查看了 Bitnami's WordPress 图表的定义，再次执行此命令，重新查看图表的定义：

```shell
[aiops@aiops0113 ~]$ helm show chart bitnami/wordpress --version 11.0.5
annotations:
  category: CMS
apiVersion: v2
appVersion: 5.7.2
dependencies:
- condition: mariadb.enabled
  name: mariadb
  repository: https://charts.bitnami.com/bitnami
  version: 9.x.x
- condition: memcached.enabled
  name: memcached
  repository: https://charts.bitnami.com/bitnami
  version: 5.x.x
- name: common
  repository: https://charts.bitnami.com/bitnami
  tags:
  - bitnami-common
  version: 1.x.x
description: Web publishing platform for building blogs and websites.
home: https://github.com/bitnami/charts/tree/master/bitnami/wordpress
icon: https://bitnami.com/assets/stacks/wordpress/img/wordpress-stack-220x234.png
keywords:
- application
- blog
- cms
- http
- php
- web
- wordpress
maintainers:
- email: containers@bitnami.com
  name: Bitnami
name: wordpress
sources:
- https://github.com/bitnami/bitnami-docker-wordpress
- https://wordpress.org/
version: 11.0.
```

从输出中可以看到，图表定义或者说 Chart.yaml 文件，包含许多字段。有些字段是必要的，而大部分字段是可选的，只需在使用时提供。

我们对 Chart.yaml 文件有了基本的了解，在下一节中我们讨论必要字段。

#### 必要字段

Chart.yaml 文件必须包含以下字段，这些字段都是重要的图表元数据：

| 字段       | 描述                                 |
| ---------- | ------------------------------------ |
| apiVersion | 图表 API 版本                        |
| name       | 图表名                               |
| version    | 图表版本，示例中的版本是 11.0.5 版本 |

我们看看这些字段都代表了什么：

* apiVersion 字段有两个值可选：
  * v1
  * v2
* aipVersion  v1 版本是 Helm3 发布前使用的 apiVersion 版本。如果设置成 v1，意味着图表将遵循老的图表结构，老结构支持 requirement.yaml 文件，但不支持 type 字段。虽然 Helm 3 向后兼容 apiVersion v1，但还是建议直接使用 v2 版本，以免使用了被弃用的特性。
* name 字段定义 Helm 图表的名称，该值应该与 Helm 图表的顶级目录名相同。**helm search** 命令和 **helm list** 命令的搜索结果包含了 Helm 图表的名称。这个字段的值应该简单且具有描述性，用一个简单的名字描述图表安装的应用程序，比如 wordpress 或 redis-cluster。减号是连接名称中不同单词的常用方式。有时，名称也会被写成一个单词，如 rediscluster。
* version 字段定义 Helm 图表的版本，版本必须遵循 Semantic Versioning (SemVer) 2.0.0 格式。SemVer 格式为 Major.Minor.Patch(主版本号.次版本号.补丁版本号)。当版本改动较大，不向后兼容时，应该修改主版本号；当发布向后兼容的功能时，应该修改次版本号；当修复 bug 时，应该修改补丁版本号。次版本增大时，需将补丁版本号置为 0；主版本增大时，次版本号和补丁版本号都需置为 0。图表开发人员在更改版本号时要注意这些规范，因为它们代表了新版本是何种程度的更新。

虽然这三个字段在 Chart.yaml 文件中是必要字段，但还有很多定义图表元数据的可选字段：

| 字段         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| appVersion   | 图表部署的应用程序版本。                                     |
| dependencies | 图表依赖的其他图表列表。                                     |
| deprecated   | 图表是否被弃用。                                             |
| description  | 图表的简短描述。                                             |
| home         | 项目主页的 URL。                                             |
| icon         | SVG 或 PNG 格式的图标，用于表示 Helm 图表。显示在 Helm Hub 的图表页上。 |
| keywords     | 关键字列表，可以通过 **helm search** 命令搜索到。            |
| kubeVersion  | 在 SemVer 中兼容 Kubernetes 的一系列版本。                   |
| maintainers  | 图表的维护人员名单。                                         |
| sources      | 链接到 Helm 图表或应用程序源代码的 URL 列表。                |
| type         | 定义图表的类型，可以是 application 或 library。              |

 一些字段仅仅提供描述信息，但有些字段可以修改 Helm 图表的行为。比如 type 字段，可以设置图表类型为 application 或 library。如果设置为 application，图表将用来部署应用程序；如果设置为 library，图表可通过 helper 模板的形式向其他图表提供功能。

第二个可以修改 Helm 图表行为的字段是 dependencies(依赖项) 字段，我们在下一节讨论。

#### 管理图表依赖

Helm 可以自行安装依赖的图表，但具体依赖哪些图表需要图表开发人员在 Chart.yaml 文件中的 dependencies 字段设置。比如 wordpress 图表，它在依赖项中声明了 mariadb。通过配置依赖，免去了重头定义资源的麻烦。以下是 wordpress 图表中定义 mariadb 依赖的代码段：

```yaml
dependencies:
- condition: mariadb.enabled
  name: mariadb
  repository: https://charts.bitnami.com/bitnami
  version: 9.x.x
```

在这里，我仅仅粘贴了 mariadb 的依赖。通过执行 **helm show chart bitnami/wordpress --version 11.0.5** 命令，可以看到 dependencies 块是一个可定义多个依赖项的列表。

dependencies 块包含多个字段，这些字段用来修改图表的依赖项管理行为。可以在下表中查看这些字段：

| 字段          | 定义                         | 是否必须 |
| ------------- | ---------------------------- | -------- |
| name          | 依赖的图表名称               | 是       |
| repository    | 依赖的图表位置(存储库)       | 是       |
| version       | 依赖的图表版本               | 是       |
| alias         | 依赖的图表别名               | 否       |
| condition     | 布尔值，它判断是否包含依赖项 | 否       |
| import-values | 将依赖图表的值导入到父图表   | 否       |
| tags          | 标签                         | 否       |

dependencies 块中的必要字段只有 name、repository 和 version。我们在 wordpress 依赖项中看到，依赖的图表名称是 mariadb，存储库(repository)地址是 https://charts.bitnami.com/bitnami。它会从该存储库中搜索名称为 mariadb 的依赖。version 字段定义了依赖图表的版本，它的值可以是一个固定的版本号，比如 9.0.0；也可以是通配符形式，比如 9.x.x，如果定义的是通配符，会下载与通配符匹配的最新图表版本。

在了解了 dependencies 的必要字段后，我们来学习如何下载依赖项。

#### 下载依赖项

可以使用 **helm dependency** 子命令下载依赖项，子命令列表如下：

| 命令                   | 作用                                                         |
| ---------------------- | ------------------------------------------------------------ |
| helm dependency build  | 基于 Chart.lock 文件重建 chart/ 目录，如果 Chart.lock 文件不存在，则执行 "helm dependency update" 命令。 |
| helm dependency list   | 列出图表的依赖项。                                           |
| helm dependency update | 基于 Chart.yaml 文件的内容更新 chart/ 目录，并生成 Chart.lock 文件。 |

第一次下载依赖时，可以使用 **helm dependency update** 命令，它会将每个依赖项下载到指定图表的 charts/ 目录下：

```shell
$ helm dependency update $CHART_PATH
```

**helm dependency update** 命令从图表存储库中下载 .tgz 扩展名的 GZip 格式压缩的依赖项。这个命令还会生成 Chart.lock 的文件。Chart.lock 文件类似 Chart.yaml 文件。Chart.yaml 文件定义是否需要依赖项，Chart.lock 文件定义依赖项的实际状态。

Chart.lock 文件内容如下：

```yaml
dependencies:
- name: mariadb
  repository: https://charts.bitnami.com
  version: 9.3.11
digest: xxxxx
generated: "2021-xxxx"
```

与其对应的 Chart.yaml 文件示例：

```yaml
apiVersion: v2
version: 0.0.1
name: dependencies-demonstration
dependencies:
  - name: mariadb
    version: 9.x.x
    repository: https://charts.bitnami.com
```

在 Chart.yaml 文件中，指定依赖的 mariadb 版本是 9.x.x，但在 Chart.lock 文件中，版本是 9.3.11，这是因为 Chart.yaml 文件用了通配符定义版本，Helm 会下载 9.x.x 的最新版本，而当前最新版本正是 9.3.11。

根据 Chart.lock 文件，Helm 可以在 charts/ 目录被删除或者需要重新构建时，自动下载依赖项：

```shell
$ helm dependency build $CHART_PATH
```

既然能够通过 **helm dependency build** 命令下载依赖项，就可以从我们的图表源码中删除 charts/ 目录，以减少代码仓库的大小。

随着功能迭代，总会产生新的版本。这时可以使用 **helm dependency update** 命令更新依赖项，它会下载最新的可用版，并重新生成 Chart.lock 文件。如果将来要下载 10.x.x 版本，或者希望将依赖项固定到特定版本，如 9.0.0，可以在 Chart.yaml 文件中定义，然后执行 **helm dependency update** 命令。

**helm dependency list** 命令可用于查看下载并保存在本地主机上的 Helm 图表依赖项：

```shell
[aiops@aiops0113 testChart]$ helm dependency list wordpress
NAME     	VERSION	REPOSITORY                        	STATUS
mariadb  	9.x.x  	https://charts.bitnami.com/bitnami	ok    
memcached	5.x.x  	https://charts.bitnami.com/bitnami	ok    
common   	1.x.x  	https://charts.bitnami.com/bitnami	ok
```

STATUS 列表示依赖项是否已经成功下载到 charts/ 目录。如果状态显示为 ok，表示依赖项已下载。如果状态为missing，则表示该依赖项还没有被下载。

默认会下载 Chart.yaml 文件中定义的所有依赖项，可以通过 dependencies 块提供的 condition 或者 tags 字段选择下载哪些依赖。

#### 选择依赖项

使用 condition 和 tags 字段，可以在安装或升级时，有条件地选择依赖项。以下是 Chart.yaml 文件的 dependencies 块示例：

```yaml
dependencies:
- name: dependency1
  repository: https://example.com
  version: 1.x.x
  condition: dependency1.enabled
  tags:
    - monitoring
- name: dependency2
  repository: https://example.com
  version: 2.x.x
  condition: dependency2.enabled
  tags:
    - monitoring
```

在这个示例中， dependencies 块增加了 condition 和 tags 两个字段。condition 字段的值由用户或图表的 values.yaml 文件提供。如果评估结果为 true，依赖项会被使用；如果为 false，则不包含该依赖项。可以通过逗号 (**,**) 添加多个条件，如下所示：

```yaml
condition: dependency1.enabled, global.dependency1.enabled
```

condition 的设置应遵循 chartname.enabled 的命名规范，每个依赖项都应根据依赖的图表名称设置唯一条件。这可以让用户更直观的启用或禁用某个图表依赖。如果条件值未包含在 values.yaml 文件中、或者用户未提供，则将忽略 condition 字段。

condition 字段用于启用或禁用某个依赖项，而 tags 字段用于启用或禁用依赖项组。在上面的示例中，dependencies 块中的两个依赖项都配置了 monitoring 标签(tag)，这意味着如果启用了 monitoring 标签，两个依赖项都会包含在内。如果 monitoring 设置为 false，则忽略这两个依赖项。通过在父图表的 values.yaml 文件中的 tags 对象下，设置标签是启用还是禁用：

```yaml
tags:
  monitoring: true
```

在 Chart.yaml 文件中，遵循 YAML list 语法，依赖项可以定义多个标签。其中一个为 true，此依赖项就会被使用。如果一个依赖项的所有标签都被忽略，那么依赖项将被默认包含。

在本节中，我们讨论了如何有条件地声明依赖关系。接下来，我们将讨论如何覆盖和引用依赖项中的值。

#### 覆盖和引用子图表中的值

默认情况下，可以引用或覆盖依赖项(子图表) 的值，方法是将这些值包装到与子图表同名的映射(map)中。假设一个名为 my-dep 的子图表包含以下值：

```yaml
## 子图表中的配置
replicas: 1
servicePorts:
  - 8080
  - 8443
```

当 my-dep 图表作为依赖项安装时，可以在父图表的 my-dep YAML 对象中设置这些值：

```yaml
## 父图表的配置
my-dep:
  replicas: 3
  servicePorts:
    - 8080
    - 8443
    - 8881
```

上面的示例覆盖了 my-dep 图表中定义的 replicas 和 servicePorts 的值，将 replicas 设置为 3，并向 servicePorts 添加了一个 8881 的端口。这些值可以在父图表的模板中引用，方式是使用点号(.)，例如 my-dep.replicas。除了覆盖和引用值外，还可以通过 import-values 字段直接导入依赖项的值。

#### 使用 import-values 导入值

Chart.yaml 文件中的 dependencies 块支持 import-values 字段，用来在父图表中导入子图表的值。这个字段有几种使用方式，第一种方式是提供要从子图表导入的键的列表。要确保其能正常工作，子图表必须在 exports 块下声明了该值，如下所示：

```yaml
exports:
  image:
    registry: 'my-registry.io'
    name: learnhelm/my-image
    tag: latest
```

然后父图表可以在 Chart.yaml 文件中定义 import-values 字段：

```yaml
dependencies:
  - name: mariadb
    repository: https://charts.bitnami.com/bitnami
    version: 7.x.x
    import-values:
      - image
```

子图表中 exports.image 的默认值会在父图表中被引用，内容如下：

```yaml
registry: 'my-registry.io'
name: learnhelm/my-image
tag: latest
```

需要注意的是，这种导入方式，删除了 image 映射，只保留了它的键值对。如果要保留 image，可以在 import-values 字段里使用 child-parent 格式。它会从指定的子图表中导入值，并提供它们在父图表中应引用的名称：

```yaml
dependencies:
  - name: mariadb
    repository: https://charts.bitnami.com/bitnami
    version: 7.x.x
    import-values:
      - child: image
        parent: image
```

这个示例会获取子图表中 image 块下的所有值，并将其导入父图表的 image 块中。

使用 import-values 字段导入的值不能在父图表中被覆盖。如果子图表中的值需要覆盖，则不应该使用 import-values 字段，而是应该通过在每个子图表的名称前添加前缀来覆盖所需值。

在本节中，我们讨论了如何在 Chart.yaml 文件中管理依赖项，接着，我们来学习如何在 Helm 图表中定义 release 的生命周期管理钩子(hooks)。

### Release 生命周期管理

Helm 图表的功能之一是能够管理 Kubernetes 上的复杂应用程序。每个应用程序的 release 都会经历生命周期的各个阶段。为了对 release 的生命周期进行管理，Helm 提供了钩子(hooks)机制，以便在 release 生命周期的不同阶段执行操作。在本节中，我会介绍应用程序生命周期的不同阶段，以及如何使用钩子与 release 及整个 Kubernetes 集群交互。

在第三章中，我们介绍了 Helm release 生命周期的几个阶段，包括安装、升级、删除和回退。有的 Helm 图表很复杂，它们管理将要部署到 Kubernetes 的一个或多个应用程序，除了部署资源之外，往往还有其它操作，如：

* 完成应用程序所需的先决条件，例如管理证书或者 secrets
* 在升级图表前，执行数据库的备份、恢复操作
* 在删除图表前，清理部署的资源

当然，还会有很多具体操作，因此我们必须要了解 Helm 钩子的基本知识以及何时执行钩子，这也是我们下一节的目标。

#### 钩子基础知识

钩子在 release 生命周期的指定阶段作为一次性动作执行，与 Helm 中的大多数特性一样，钩子也是作为 Kubernetes 的资源实现的，更具体地说是在容器中实现的。虽然 Kubernetes 中大多数工作负载都是为长期存在的进程而设计的，例如提供 API 请求服务的应用程序，但工作负载也可以由单个任务或一组任务组成，这些任务使用脚本执行，并在执行完成后显示成功或失败。

在 Kubernetes 集群中创建短期任务通常有两种方式：裸 Pod 或者 Job。裸 Pod 是一种运行完成即终止的 Pod，缺陷是如果工作(Worker)节点发生故障，裸 Pod 不会被重新调度。由于这个原因，最好将生命周期钩子作为 Job 运行，这样工作节点故障时，钩子依然会被重新调度。

钩子被定义为 Kubernetes 的资源，存放在 templates/ 目录中，并用 helm.sh/hook 注解(Annotations)。此注解确保在标准处理过程中，钩子不会与 Kubernetes 集群中的其他资源一起渲染。helm.sh/hook 注解决定了钩子何时作为 release 生命周期的一部分执行。

将钩子定义为 Job 的示例：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: helm-auditing
  annotations:
    'helm.sh/hook': pre-install,post-install
spec:
  template:
    metadata:
      name: helm-auditing
    spec:
      restartPolicy: Never
      containers:
      - name: helm-auditing
        command: ["/bin/sh", "-c", "echo Hook Executed at $(date)", "sleep 10"]
        image: alpine
```

这个简单的示例会打印容器当前的日期和时间，然后休眠 10 秒钟。Helm 在图表安装前(pre-install)和安装后(post-install)执行这个钩子。这类钩子可以绑定到一个审计系统，跟踪应用程序在 Kubernetes 集群中的安装进度，安装完成后，触发钩子，统计图表安装完成所耗费的时间。

介绍了钩子的基础知识，我们来看看如何在 Helm 中定义钩子。

#### 执行钩子

正如你在上一节的“将钩子定义为 Job”的示例中看到的，helm.sh/hook 注解的值是 pre-install。pre-install 是 Helm 图表生命周期中可以执行钩子的一个点。

下表列出了 helm.sh/hook 注释的所有值，它们决定了何时执行钩子。这些信息来自 Helm 官方文档，可以在 https://helm.sh/docs/topics/charts_hooks/#the-available-hooks 页面查看：

| 注解值        | 描述                                    |
| ------------- | --------------------------------------- |
| pre-install   | 在模板渲染后、Kubernetes 创建资源前执行 |
| post-install  | 在 Kubernetes 创建资源后执行            |
| pre-delete    | 在 Kubernetes 删除资源前执行            |
| post-delete   | 在该 release 相关的资源被删除后执行     |
| pre-upgrade   | 在模板渲染后、任何资源更新前执行        |
| post-upgrade  | 在所有资源更新后执行                    |
| pre-rollback  | 在模板渲染后、任何资源回退前执行        |
| post-rollback | 在所有资源回退后执行                    |
| test          | 当运行 **helm test** 子命令时执行       |

helm.sh/hook 注解可以包含多个值，这些值定义了同一资源在图表发布周期内的不同时间点执行。例如，要在图表安装前后执行钩子，可以在 Pod 或 Job 上定义以下注解：

```yaml
annotations:
  'helm.sh/hook': pre-install,post-install
```

了解如何执行钩子、以及在图表生命周期中的哪个阶段执行钩子非常重要。如上面示例，当指定在 pre-install 和 post-install 钩子之间执行 **helm install** 命令时，会发生以下操作：

1. 用户安装一个 Helm 图表(例如，**helm install bitnami/wordpress --version 11.0.5**)。
2. Helm API 被调用。
3. crds/ 目录中的 CRDs 被加载到 Kubernetes 中。
4. 执行图表模板资源验证，并渲染资源。
5. pre-install 钩子按权重排序，然后被渲染并加载到 Kubernetes 中。
6. Helm 等待 pre-install 钩子执行完成。
7. 模板资源被渲染并应用到 Kubernetes 环境。
8. post-install 钩子被执行。
9. Helm 等待 post-install 钩子完成。
10. 返回 **helm install** 命令的执行结果。

在了解了 Helm 钩子执行的基础知识后，我们来学习 Helm 钩子的高级知识。

#### 钩子进阶

在 Helm 图表的生命周期内执行的钩子数量是没有限制的，并且在某些情况下，可能会为同一生命周期的某个阶段配置多个钩子。此时，钩子的默认执行顺序是按照钩子的首字母顺序。通过 helm.sh/weight 注解可以为每个钩子设置权重，这样就可以按权重顺序执行钩子。权重按升序排序，如果多个钩子的权重值相同，则继续按名称字母顺序执行。

钩子提供了管理 Helm 图表生命周期的机制，但与模板资源不同的是，钩子不会被 Helm 跟踪和管理，在执行 **helm uninstall** 命令时，钩子不会随着图表一起删除。有两种方案删除 release 生命周期中的钩子：配置删除策略和在 Job 上设置生存时间控制器(TTL,time to live controller)。

首先，可以在钩子关联的 Pod 或者 Job 上添加 helm.sh/hook-delete-policy 注解，该注解定义了 Helm 应何时从 Kubernetes 中删除资源。下表是具体的注解值(https://helm.sh/docs/topics/charts_hooks/#hook-deletion-policies)：

| 注释值               | 描述                                     |
| -------------------- | ---------------------------------------- |
| before-hook-creation | 在启动一个新钩子前删除之前的资源（默认） |
| hook-succeeded       | 钩子执行成功后删除资源                   |
| hook-failed          | 如果钩子在执行过程中失败，则删除资源     |

此外，Kubernetes 的生存时间控制器还提供了生存时间(TTL)机制，限制已完成执行的资源对象的寿命。生存时间控制器目前只支持 Job，使用 Job 的 ttlSecondsAfterFinished 属性来控制资源完成后的保留时间，如下所示：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ttl-job
  annotations:
    'helm.sh/hook': post-install
spec:
  ttlSecondsAfterFinished: 60
```

在本例中，钩子资源在完成或失败后60秒内被删除。

删除应用程序是 release 生命周期管理的最后一个阶段，尽管 **helm uninstall** 命令可以删除已安装的图表，但你可能希望保留某些资源，比如在应用程序生命周期开始时，通过 PersistentVolumeClaim 创建的持久卷，保留着应用程序程序运行期间的数据，不应该一起被删除。可以使用 helm.sh/resource-policy 注解对此进行设置：

```yaml
'helm.sh/resource-policy': keep
```

这时再执行 **helm uninstall** 命令，持久卷就不会被删除了。但需要注意的是，该资源会变成一个孤立资源。如果再次使用 **helm install** 命令，可能会出现老资源未删除而导致的资源名字冲突，进而部署失败的现象。删除孤立资源，可以使用 **kubectl delete** 命令，手动删除。

本节介绍了钩子的创建和自动化管理图表的生命周期，在下一节中，我会介绍编写 Helm 图表文档的软技能。描述准确的文档，有助于用户获得流畅的使用体验。

### 图表文档

完善的文档可以帮助用户更便捷的使用 Helm 图表。Helm 图表结构支持使用自述文件(README.md)记录图表的用法、LICENSE 文件授权使用、tepmlate/NOTES.txt 文件在图表安装完成后输出使用说明。

#### README.md

自述文件(README.md)是软件开发中常用的文件，描述应用程序的部署、使用及其他信息。Helm 图表的 README 文件通常包含以下信息：

* **Prerequisites**：安装图表(应用程序)的先决条件。常见示例是在 Kubernetes 集群中安装图表时，需要先创建 Secret。通过 README 文件，用户会注意到这一要求。
* **Values**：图表通常包含不同的值，每个值都应在 README 文件中进行说明。包括值的名称、描述信息、默认值、在安装或升级过程中，是否需要提供该值等。
* **Application-specific information**：使用 Helm 图表安装应用程序后，可能需要应用程序自身的信息，例如该如何访问这个应用。这些信息也可以在 README 文件中提供。

Helm README 文件使用 Markdown 格式语法编写，Markdown 通常在 GitHub 项目和开放源代码软件中使用，是一种轻松编写文本的方式，以优雅的格式显示文本。在 Markdown 指南网站上，可以学习更多 Markdown 的使用方法，网址为 https://www.markdownguide.org。

#### LICENSE

除了包含技术说明的 README 文件，有时图表中也会包含一个许可文件，说明图表的使用和分发权限。许可文件的名称是 LICENSE，在图表的顶级目录下。

LICENSE 文件是包含一个软件许可证的纯文本文件，许可证可以是自己编写的，也可以是开源软件中常用的许可证副本，例如 “Apache License 2.0” 或者 “MIT License”。 如果你想了解更多许可证的内容，可以访问 https://choosealicense.com 网站。

#### templates/NOTES.txt

类似 README.md 文件，templates/NOTES.txt 文件用于在使用 Helm 安装应用程序后提供使用说明。不同的是，README.md 文件是静态的，NOTES.txt 文件可以通过 Go 模板动态生成。

假设一个 Helm 图表的 values.yaml 文件包含如下配置：

```yaml
## serviceType can be set to NodePort or LoadBalancer
serviceType: NodePort
```

设置不同的 Service 类型，应用的访问方式会有所不同。如果设置的 Service 类型为 Nodeport，则可以使用 Kubernetes 集群中任一节点的 IP:NodePort 访问。如果 Service 的类型为 LoadBalancer，则将使用在创建 Service 时自动设置的负载均衡器的地址访问应用程序。对于经验欠缺的 Kubernetes 用户来说，理解如何基于 Service 类型访问应用程序可能很困难，因此图表维护人员应该在 templates/ 目录下提供 NOTES.txt 文件，来说明该如何访问应用程序。

示例：

```yaml
请按照下述说明访问你的应用程序：

{{- if eq .Values.serviceType 'NodePort' }}

export NODE_PORT=$(kubectl get --namespace {% raw %}{{ .Release.Namespace }}{% endraw %} -o jsonpath='{.spec.ports[0].nodePort}' services {% raw %}{{.Release.Name }}{% endraw %})

export NODE_IP=$(kubectl get nodes --namespace {% raw %}{{ .Release.Namespace }}{% endraw %} -o jsonpath='{.items[0].status.addresses[0].address}')

echo "URL: http://$NODE_IP:$NODE_PORT"

{{- else }}

export SERVICE_IP=$(kubectl get svc --namespace {% raw %}{{ .Release.Name }}{% endraw %} wordpress --template '{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}')

echo "URL: http://$SERVICE_IP"

{{- end }}
```

该内容会在应用程序安装、升级、回退时生成并输出到终端，也可以通过 **helm get notes** 命令获取。通过显示的内容，指导用户如何访问应用程序。

到目前为止，我已经介绍了 Helm 图表的大部分内容，除了将图表打包。包使图表易于分发，在下一节，我们讨论包的概念。

### 图表的压缩包

虽然 Helm 图表遵循通用的文件结构，但考虑到分发与传播，我们应该把图表打成 tgz 格式的压缩包。可以使用 bash 中的 **tar** 命令手动完成，不过 Helm 提供了 **helm package** 命令，简化了打包的过程。**helm package** 命令的语法如下：

```shell
$ helm package [CHART_NAME] [...] [flags]
```

执行 **helm package** 命令时需要指定打包的本地图表目录。命令执行成功后，将生成一个 tgz 的压缩文件，文件名称格式如下：

```shell
$ CHART_NAME-$CHART_VERSION.tgz
```

可以把压缩文件推送到图表存储库中，其他人就可以从库中获取并使用这个图表了，就像我们使用 WordPress 图表一样。这部分内容我会在第五章详细介绍。

**helm package** 命令会打包图表目录下的所有文件，但有时图表目录中包含了非必须打包的文件或目录，在打包时最好把它们过滤掉。比如 .git 目录，它存在于 Git SCM 管理的项目中，把它打包进 tgz 压缩文件中，没有任何意义，还会增加压缩文件的大小，造成资源浪费。可以通过 .helmignore 文件，在打包时忽略某些文件或目录。.helmignore 文件内容如下：

```yaml
# 忽略 .git/ 目录和 .gitignore 文件.
.git/
.gitignore
```

包含在 .helmignore 文件中的目录或文件名，会被 **helm package** 命令忽略，不会被打包进 tgz 压缩文件中。以`#` 开头的行为注释行。如果你的图表目录中包含了与 helm 图表功能无关的文件或目录，请确保使用 .helmignore 文件将其过滤掉。
