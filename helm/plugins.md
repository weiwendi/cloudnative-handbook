# 9 Helm 插件

如我们所见，Helm 有很多特性和方法，帮助使用者在 Kubernetes 上部署应用程序。当然，我们也可以自定义和扩展 Helm 的功能。

在本章中，我们讨论如何通过插件增强 Helm 功能。

插件可以向 Helm 添加额外的功能，与 Helm CLI 无缝集成，帮助用户完成特殊的工作流。网上有很多常用的第三方插件。此外，使用插件可以非常方便的部署一次性的任务。

通过插件，我们可以基于 Helm 现有的特性构建新功能，以简化和自动化日常工作流程。

## 9.1 插件介绍

Helm 插件是可以直接从 Helm CLI 访问的外部工具，支持自定义的子命令，而无需修改 Helm 源码。在设计上与 kubectl(Kubernetes CLI) 等工具实现插件系统的方法类似。

下载插件可以指定自定义协议与图表存储库通信，如果你有一些自定义身份验证方法，或者你需要以某种方式修改 Helm 从存储库获取图表的方法，那么这将非常有用。

## 9.2 安装第三方插件

大多第三方插件都是开源的，公开在 GitHub 上。这些插件中有很多使用了 **helm-plugin** tag/topic，方便查找。https://github.com/topics/helm-plugin。

一些可以在 GitHub 上找到的 Helm 插件示例：

* **helm/helm-2to3**：把 helm2 转换为 helm3
* **jkroepke/helm-secrets**：管理 YAML 格式的 secrets 的插件
* **maorfr/helm-backup**：备份 Helm releases 到文本文件，或从文本文件恢复 Helm releases
* **karuppiah7890/helm-schema-gen**：基于 values.yaml 生成 values.schema.json 的插件
* **hickeyma/helm-mapkubeapis**：更新 Helm releases 元数据，包含过时的 Kubernetes APIs

找到要安装的插件，在其版本控制 URL 中可以获取 plugin.yaml 文件及其源代码。

例如，一个管理 Helm starters 的插件，位于 https://github.com/salesforce/helm-starter。这个插件的版本控制 URL 是 https://github.com/salesforce/helm-starter.git。

要安装这个插件，执行 **helm plugin install**，将版本控制 URL 作为第一个参数：

```bash
$ helm plugin install https://github.com/salesforce/helm-starter.git

Installed plugin: starter
```

安装成功后，便可以使用插件了：

```bash
$ helm starter --help
Fetch, list, and delete helm starters from github.

Available Commands:
    helm starter fetch GITURL       Install a bare Helm starter from Github
                                      (e.g., git clone)
    helm starter list               List installed Helm starters
    helm starter delete NAME        Delete an installed Helm starter
    --help                          Display this text
```

要列出已安装的插件：

```bash
$ helm plugin list
NAME   	VERSION	DESCRIPTION
starter	1.0.0  	This plugin fetches, lists, and deletes helm starters from github.
```

也可以在插件目录中查看已安装的插件，首先检查插件存储根目录的配置路径：

```bash
$ helm env |grep HELM_PLUGINS
HELM_PLUGINS=/home/aiops/.local/share/helm/plugins
```

如果想使用自定义的插件根目录，可以设置当前环境中的 HELM_PLUGINS 环境变量，以提供自定义的插件根目录。

更新插件：

```bash
$ helm plugin update starter
Updated plugin: starter
```

卸载已安装的插件：

```bash
$ helm plugin remove starter
Uninstalled plugin: starter
```

默认情况下，Helm 在安装插件时，将使用 plugin.yaml 和 Git repo 默认分支上的源码。如果要安装指定的 Git tag，需要使用 **--version** 标志：

```bash
$ helm plugin install https://github.com/databus23/helm-diff.git --version v3.1.0
```

也可以使用 tarball URL 安装插件。Helm 将下载 tarball 并解压到 plugins 目录中：

```bash
$ helm plugin install https://example.com/archives/myplugin-0.6.0.tar.gz
```

从本地目录安装插件：

```bash
$ helm plugin install /path/to/myplugin

# Helm将创建到原始文件的符号链接，而不是复制文件
$ ls -la "$(helm env HELM_PLUGINS)"
total 8
drwxrwxr-x 2 myuser myuser 4096 Jul  3 21:49 .
drwxrwxr-x 4 myuser myuser 4096 Jul  1 21:38 ..
lrwxrwxrwx 1 myuser myuser   21 Jul  3 21:49 myplugin -> /path/to/myplugin
```

这样的好处是，当你在频繁开发、修改一个插件时，符号链接会和源文件保持一致。

## 9.3 自定义子命令

插件有许多特性，可以实现与现有 Helm 用户体验的无缝集成。Helm 插件最显著的特性可能是每个插件都为 Helm 提供了一个自定义的顶级子命令。这些子命令甚至可以使用 **shell** 补全。

一旦插件安装完成，就会新增一个与插件同名的子命令。这个新命令直接与 Helm 集成，甚至可以出现在 **helm help** 中。

假设我们安装了一个名为 **inspect-templates** 的插件，它可以在有关图表中查询 YAML 模板的类型信息。此插件将会提供一个新的子命令：

```bash
$ helm inspect-templates [args]
```

该命令会执行 **inspect-templates** 插件，传递给调用插件时执行的底层工具的任何参数或标志。插件的作者定义了一些命令，Helm 应该在每次调用插件时作为子进程运行。

插件提供了一个很好的替代方案来扩充 Helm 现有的特性集，而不需要对 Helm 本身进行任何修改。

## 9.4 构建插件

构建 Helm 插件是一个相当简单的过程，根据需求和插件的整体复杂性，可能依赖一些编程知识；然而许多插件仅仅运行一个基本的 **shell** 命令即可实现。

### 9.4.1 底层实现

下面的 Bash 脚本 inspect-templates.sh，它是我们示例 inspect-templates 插件的底层实现：

```bash
#!/usr/bin/env bash
set -e

# First argument on the command line, a relative path to a chart directory
CHART_DIRECTORY="${1}"

# Second argument on the command line, is the depth of the directory to look for
MAXDEPTH="${2}"

# Fail if no chart directory provided or is invalid
if [[ "${CHART_DIRECTORY}" == "" ]]; then
    echo "Usage: helm inspect-templates <chart_directory>"
    exit 1
elif [[ ! -d "${CHART_DIRECTORY}" ]]; then
    echo "Invalid chart directory provided: ${CHART_DIRECTORY}"
    exit 1
elif [[ "${MAXDEPTH}" == "" ]]; then
    MAXDEPTH=1
fi

# Print a summary of the chart's templates
cd "${CHART_DIRECTORY}"
cd templates/
echo "----------------------"
echo "Chart template summary"
echo "----------------------"
echo ""
total="$(find . -maxdepth ${MAXDEPTH} -type f -name '*.yaml' | wc -l | tr -d '[:space:]')"
echo " Total number: ${total}"
echo ""
echo " List of templates:"
for filename in $(find . -maxdepth ${MAXDEPTH} -type f -name '*.yaml' | sed 's|^\./||'); do
    kind=$(cat "${filename}" | grep kind: | head -1 | awk '{print $2}')
    echo "  - ${filename} (${kind})"
done
echo ""
```

当运行 **helm inspect-templates** 时，Helm 将在后台执行此脚本。

底层插件实现不局限于 Bash、Go 或任何特定的编程语言编写。对于这个插件的最终用户来说，它应该只是 Helm CLI 的一部分。

### 9.4.2 插件清单(manifest)

每个插件都由一个名为 plugin.yaml 的 YAML 文件定义。此文件包含插件元数据和有关调用插件时要运行的命令的信息。

下面是 inspect-templates 插件 plugin.yaml 的基本示例：

```yaml
name: inspect-templates #1
version: 0.1.0 #2
description: get a summary of a chart's templates #3
command: "${HELM_PLUGIN_DIR}/inspect-templates.sh" #4
```

* 1 插件的名称
* 2 插件的版本
* 3 插件的基本描述
* 4 调用插件时会执行的命令

## 9.6 自动补全

从 Helm 3.2 版本开始，可以为插件提供对 shell 的自动补全。

如果插件提供了自己的参数或者子命令，可以通过位于插件根目录下的 completion.yaml 文件，定义补全。completion.yaml 格式如下：

```yaml
name: <pluginName>
flags:
- <flag 1>
- <flag 2>
validArgs:
- <arg value 1>
- <arg value 2>
commands:
  name: <commandName>
  flags:
  - <flag 1>
  - <flag 2>
  validArgs:
  - <arg value 1>
  - <arg value 2>
  commands:
     <and so on, recursively>
```

需要注意的是：

1. 所有部分都是可选的，应该在适用时提供。
2. 参数不该包含`-` 或 `--`前缀。
3. 应该指定短和长参数。短参数不需要与其对应的长格式关联，但是都应被列出。
4. 参数不需要以任何方式排序，但是需要列举在文件子命令层次结构的正确位置。
5. Helm 现有的全局参数已经由 Helm 的自动补全机制处理，因此插件不需要指定以下参数：`--debug`，`--namespace` 或 `-n`， `--kube-context`，以及 `--kubeconfig`，或者其他全局参数。
6. `validArgs` 列表提供了一个以下子命令的第一个参数可能补全的静态列表。