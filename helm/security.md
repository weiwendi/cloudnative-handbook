# 10 Helm 安全

经过这么多章的学习，你可能体会到 Helm 是一个功能强大的工具了，它为用户提供了部署各种应用程序的可能性。但如果未能意识到某些安全规范，或者未能严格遵守这些规范，可能会导致这种强大的功能被滥用，最终导致灾难性的结果。幸运的是，从下载 Helm CLI 那一刻起，到在 Kubernetes 集群中安装 Helm 图表，Helm 都提供了将安全性融入其中的便捷方法。

本章主要介绍的知识点如下：

* 数据来源和完整性
* Helm 图表安全
* 关于 RBAC、值和图表仓库的注意事项

## 10.1 技术要求

本章会用到以下技术：

* **minikube**
* **kubectl**
* **helm**
* **GNU Privacy Guard(GPG)**

我们还会用到 https://github.com/weiwendi/learn-helm 中的 player-stats 图表，作为演示示例，如果你还没有将其 clone 到本地，需要确保主机能执行以下命令：

```bash
$ git clone https://github.com/weiwendi/learn-helm.git
```

## 10.2 数据来源及完整性

在处理任何类型的数据时，都应该考虑两个经常被忽视的问题：

* 数据来源是可靠的吗，或者数据是来自你期望的数据源吗？
* 数据是完整的吗？

第一个问题涉及数据源，用来确定数据的来源。

第二个问题涉及数据完整性。数据完整性是为了确定远程获取的数据是否是你期望的数据，以及数据在传输过程中是否被篡改。可以使用数字签名的概念来验证数据来源及数据完整性。作者可以创建一个基于密码学的唯一签名对数据进行签名，数据的使用者可以使用密码工具来验证签名的真实性。

如果验证了签名的真实性，说明数据来自预期的数据源，并且在传输过程中没有被篡改。

作者可以通过创建一个 Pretty Good Privacy 密钥对创建数据签名。这里的 PGP 指的是 OpenPGP，它是一种基于密钥的加密方式。PGP 侧重于建立非对称加密，使用两种不同的密钥——私钥和公钥。

私钥是保密的，而公钥是被设计为共享的。对于数字签名，私钥用于加密数据，公钥用于解密数据。PGP 密钥对通常使用一种称为 GPG 的工具创建，它是一种实现 OpenPGP 标准的开源工具。

一旦创建了 PGP 密钥对，作者就可以使用 GPG 对数据进行签名。在数据签名完成后，GPG 在后台执行以下步骤：

1. 根据内容计算出一个 hash 值，输出为一个固定的长度的字符串，称为消息摘要。
2. 消息摘要使用作者的私钥进行加密。输出的是数字签名。

要验证签名，使用者必须使用作者的公钥对其解密。这种验证也可以使用 GPG 执行。

数字签名在 Helm 中起着两种作用：

* 首先，每个 Helm 下载都有一个来自维护人员的数字签名，可以用来验证二进制文件的真实性。签名可用于验证下载的来源以及完整性。
* 其次，也可以对 Helm 图表进行数字签名，这样就可以验证它的来源及完整性了。图表作者在打包过程中对图表进行签名，图表用户使用作者的公钥验证图表的有效性。

了解了数据来源、完整性与数字签名的关系后，我们在本地创建一个 GPG 密钥对(它将用于详细说明前面描述的许多概念。)。

## 10.3 创建密钥对

创建密钥对，需要先安装 GPG(Linux系统中可能已经默认安装了)：

Windows：

```powershell
> choco install gnupg
```

也可以从 https://gpg4win.org/download.html 页面下载安装。

macOS：

```bash
$ brew install gpg
```

也可以从 https://sourceforge.net/p/gpgosx/docu/Download/ 页面下载安装。

Debian-based Linux发行版：

```bash
$ sudo apt install gnupg
```

RPM-based Linux发行版：

```bash
$ sudo dnf install gpg
```

安装完成后，就可以创建 GPG 密钥对了，步骤如下：

1. 运行以下命令创建新的密钥对(该命令可以在任何目录下运行)：

   ```bash
   $ gpg --generate-key
   ```

2. 安装提示输入姓名及邮箱。这两个信息将你标识为密钥对的所有者，接收公钥的人可以看到这两个信息。

3. 按 O 键继续。

4. 然后，系统将提示您输入私钥密码。输入并确认用于加密和解密操作的密码。

一旦创建了GPG密钥对，你会看到类似如下的输出：

![9.1 gpg outputs](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/9.1%20gpg%20infors.jpg)

pub：公钥信息，第二行是公钥的指纹。指纹是唯一的标识符，用于将你标识为该秘钥的所有者。

uid：显示生成 GPG 秘钥对时输入的名称及邮箱。

sub：私钥信息。

秘钥对已经创建完了，我们继续下一节的学习，如何验证 helm 下载。

## 10.4 验证 Helm 下载

在第二章中，我们讨论了 Helm 的部署，可以从 https://github.com/helm/helm/releases 页面下载 Helm 的不同版本，页面内容如下：

![9.2 helm versions](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/9.2%20helm%20release%20download.jpg)

在 Installation and  Upgrading 底部，有一段内容说明了该版本已签名。每个 Helm 发行版都有维护人员签名，并可以与下载的 Helm 版本对应的数字签名进行验证。每个数字签名都位于 Assets 部分下。并可根据与下载的 Helm 发行版对应的数字签名进行验证。

每个数字签名都位于 Assets 下，截图显示了它们是如何表示的：

![9.3 digital signature](https://gitee.com/sretech/images/raw/master/cloudnative-handbook/helm/9.3%20digital%20signature.jpg)

要验证下载的 Helm 的来源和完整性，还应该下载相应的 `.asc` 文件。`.sha256.asc` 文件仅用于验证完整性。在本例中，我们下载相应的 `.asc` 文件，以验证数据来源及完整性。

按照以下步骤开始验证 Helm 发行版：

1. 下载与操作系统对应的 Helm 压缩文件。如果你第二章安装后保留了该文件，则无需重新下载。
2. 下载与操作系统对应的 `.asc` 文件，例如，你的操作系统是基于 `amd64` 的 Linux 系统，则需要下载 `helm-v3.0.0-linux-amd64.tar.gz.asc` 文件(文件名中包含的版本对应于你下载的 Helm 版本)。

下载完成后，应该在下载目录中看到以下两个文件：

```bash
[aiops@aiops0113 software]$ ll -h
total 12M
-rw-rw-r--. 1 aiops aiops 12M Apr 14 17:20 helm-v3.5.4-linux-amd64.tar.gz
-rw-rw-r--. 1 aiops aiops 833 Apr 14 17:29 helm-v3.5.4-linux-amd64.tar.gz.asc
```

下一步是将 Helm 维护者的公钥导入到本地秘钥环，解密 `.asc` 文件中包含的数字签名，以验证你下载的来源及完整性。可以通过链接到维护人员的 keybase account 来检索维护人员的公钥。将光标悬停在 keybase account (图9-2)上会出现该链接 https://keybase.io/mattfarina，在连接后面添加 /pgp_keys.asc 下载公钥 https://keybase.io/mattfarina/pgp_keys.asc。

因为有不同的 Helm 维护者，如果你验证的是其它版本，链接可能会和示例中不同。请确保下载的公钥与签署发行版的秘钥对应。

继续进行验证：

1. 使用命令行下载 Helm 发行版签名对应的公钥：

   ```bash
   [aiops@aiops0113 software]$ curl -o release_key.asc https://keybase.io/mattfarina/pgp_keys.asc
   ```

2. 下载之后，需要将公钥导入到 gpg 密钥环中：

   ```bash
   [aiops@aiops0113 software]$ gpg --import release_key.asc
   gpg: key 461449C25E36B98E: 3 signatures not checked due to missing keys
   gpg: key 461449C25E36B98E: public key "Matthew Farina <matt@mattfarina.com>" imported
   gpg: Total number processed: 1
   gpg:               imported: 1
   gpg: marginals needed: 3  completes needed: 1  trust model: pgp
   gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
   gpg: next trustdb check due at 2023-09-01
   ```

3. 导入数字签名的公钥后，就可以使用 GPG 的 **--verify** 子命令验证Helm的发行版了。针对 `helm*.asc` 文件执行以下命令：

   ```bash
   [aiops@aiops0113 software]$ gpg --verify helm-v3.5.4-linux-amd64.tar.gz.asc 
   gpg: assuming signed data in 'helm-v3.5.4-linux-amd64.tar.gz'
   gpg: Signature made Wed 14 Apr 2021 05:25:20 PM EDT
   gpg:                using RSA key 711F28D510E1E0BCBD5F6BFE9436E80BFBA46909
   gpg: Good signature from "Matthew Farina <matt@mattfarina.com>" [unknown]
   gpg: WARNING: This key is not certified with a trusted signature!
   gpg:          There is no indication that the signature belongs to the owner.
   Primary key fingerprint: 672C 657B E06B 4B30 969C  4A57 4614 49C2 5E36 B98E
        Subkey fingerprint: 711F 28D5 10E1 E0BC BD5F  6BFE 9436 E80B FBA4 6909
   ```

   该命令会尝试解密 `.asc` 文件中包含的数字签名，如果成功，意味着下载的 Helm 发行版(.tar.gz)源是我们期望的，且数据没有被篡改。

在上面的输出中，有一条 WARNING 消息，*This key is not certified with a trusted signature!*，此密钥没有通过可信签名认证！这条消息并不影响结果，验证是成功的，只是我们执行时，没有确认维护者的公钥是属于该维护者的。

可以通过以下步骤执行可信签名验证：

1. 检查输出内容中的 Primary key fingerprint 一行最后64位(8个字符)，与 Helm releases 页面中显示的 64 位指纹是否匹配。指纹显示如下：

   *This release was signed with `672C 657B E06B 4B30 969C 4A57 4614 49C2 5E36 B98E ` and can be found at [@mattfarina](https://github.com/mattfarina) [keybase account](https://keybase.io/mattfarina).*

2. 指纹匹配，确认了是属于我们期望的人，因此我们可以安全地认证维护者的公钥。通过使用自己的 gpg 秘钥对公钥进行签名来实现：

   ```bash
   $ gpg --sign-key 461449C25E36B98E # Last 64 bits of fingerprint
   ```

3. 在 `Really sign?` 提示中输入 `y`。

   签署了维护者公钥，现在密钥已经被认证。我们再次执行 **--verify** 子命令验证 Helm 发行版：

   ```bash
   $ gpg --verify helm-v3.5.4-linux-amd64.tar.gz.asc
   gpg: assuming signed data in 'helm-v3.5.4-linux-amd64.tar.gz'
   gpg: Signature made Wed 14 Apr 2021 05:25:20 PM EDT
   gpg:                using RSA key 711F28D510E1E0BCBD5F6BFE9436E80BFBA46909
   gpg: checking the trustdb
   gpg: marginals needed: 3  completes needed: 1  trust model: pgp
   gpg: depth: 0  valid:   1  signed:   1  trust: 0-, 0q, 0n, 0m, 0f, 1u
   gpg: depth: 1  valid:   1  signed:   0  trust: 1-, 0q, 0n, 0m, 0f, 0u
   gpg: next trustdb check due at 2023-09-01
   gpg: Good signature from "Matthew Farina <matt@mattfarina.com>" [full]
   ```

数字签名在验证 Helm 图表的来源及完整性方面同样发挥着重要作用，我们在下一节讨论。

## 10.5 签名和验证 Helm 图表

与 Helm 维护人员签署发行版类似，你可以签署自己的 Helm 图表，以便用户可以验证他们安装图表是来自于你的。要对图表签名，首先需要有 gpg 密钥对。在先前的部分，我们已经创建了 gpg 密钥对，可以直接使用。

接下来，可以使用 **helm package** 命令的某些标识，指定密钥为图表签名。

我们使用包存储库中的 player-stats 图表作为示例，图表位于 learn-helm/helm-charts/charts/player-stats 目录。

在签名 player-stats 图表之前需要注意一点，如果你使用了GPG v2 及更高的版本，需要将 public 和 secret 钥匙环导出为老版本的格式。较新版本的 GPG 以 `.kbx` 文件格式存储钥匙环，而老版本以 `.gpg` 文件格式存储钥匙环，目前Helm仅支持 `.gpg` 格式。

将 GPG public 和 secret 钥匙环转换为 `.gpg` 文件格式：

1. 查看 gpg 版本：

   ```bash
   $ gpg --version
   gpg (GnuPG) 2.2.20
   libgcrypt 1.8.5
   Copyright (C) 2020 Free Software Foundation, Inc.
   ```

2. 如果你的 gpg 版本是 `2` 或更大，使用以下命令导出 public 和 secret 钥匙环：

   ```bash
   $ gpg --export > ~/.gnupg/pubring.gpg
   $ gpg --export-secret-keys > ~/.gnupg/secring.gpg
   ```

   导出完成，就可以对 Helm 图表执行签名和打包操作了。**helm package** 命令提供了三个标志，用于签名和打包图表：

   **--sign**: 允许你使用 GPG 私钥对图表签名

   **--key**: 签名时使用的密钥名称

   **--keyring**: 指定包含 PGP 私钥的密匙环的路径

   在下一步中，这些标志将与 **helm package** 命令一起用于签名和打包 player-stats Helm 图表。

3. 执行 **helm package** 命令：

   ```bash
   $ helm package --sign --key '$KEY_NAME' --keyring ~/.gnupg/secring.gpg player-stats
   ```

   **$KEY_NAME** 变量可以是所需密钥关联的邮箱、名称或指纹。可以通过 **gpg --list-keys** 命令查看。

   当使用不带签名标识的 **helm package** 命令时，会生成一个包含 helm 图表的 `tgz` 归档文件，使用带签名标识的打包命令时，会产生两个文件：

   ```bash
   player-stats-1.0.0.tgz
   player-stats-1.0.0.tgz.prov
   ```

   player-stats-1.0.0.tgz.prov 文件称为出处文件。出处文件中包含出处记录，内容如下：

   1. 来自 Chart.yaml 文件的图表元数据

   2. player-stats-1.0.0.tgz sha256 hash值

   3. player-stats-1.0.0.tgz 文件的 PGP 数字签名

      Helm 图表的用户可以使用出处文件验证图表的数据来源和完整性。当把图表推送到图表存储库时，应确保同时推送了 `.tgz` 归档文件和 `.tgz.prov` 出处文件。

      一旦 Helm 图表签名并打包完成，你需要导出用于加密数字签名的私钥对应的公钥。用户下载此公钥进行签名验证。

   4. 使用以下命令将公钥导出为 ascii-armor 格式：

      ```bash
      $ gpg --armor --export $KEY_NAME > pubkey.asc
      ```

      如果你的 player-stats 图表发行版是公开的，那么这个公钥是可以被图表用户下载的，比如 Keybase。用户可以使用 **gpg --import** 命令导入这个公钥。

      在安装图表前，图表用户可以使用 **helm verify** 命令验证图表的数据来源及完整性。此命令针对下载的 `.tgz` 归档文件和 `.tgz.prov` 出处文件运行。

   5. 下面的命令演示了在 player-stats Helm 图表上运行此过程，并假设你的公钥已经导入到名为 ~/.gnupg/pubring.gpg 的密匙环中：

      ```bash
      $ helm verify --keyring ~/.gnupg/pubring.gpg player-stats-1.0.0.tgz
      Signed by: weiwendi <weiwendi@aiops.red>
      Using Key With Fingerprint: DC06795BA6B4C0526F085753519584CA7F2EFA86
      Chart Hash Verified: sha256:d47cac011e7d85ad21df6c2b00844de74f3954bdd857087cd257fc370b824da0
      ```

      如果将返回错误信息，验证失败的原因有多种，包括：

      * `.tgz` 和 `.tgz.prov 文件不在同一个目录
      * `.tgz.prov` 文件被损坏
      * 文件 hash 不匹配，表示文件完整性有问题
      * 用于解密签名的公钥与最初用于加密签名的私钥不匹配。

**helm verify** 命令用于验证本地下载的图表，因此有一个更简便的方式，使用 **helm install --verify** 命令，该命令同时执行了验证和安装。假设 `.tgz` 和 `.tgz.prov` 文件已经下载到本地。

下面是 **helm install --verify** 命令的示例：

```bash
$ helm install my-playerstats $CHART_REPO/player-stats --verify --keyring ~/.gnupg/pubring.gpg
```

通过使用本节中描述的方法来签署和验证 Helm 图表，你和用户都可以确保正在安装的图表是可靠的，并且没有被篡改。

在了解了数据来源和完整性在 Helm 安全中的作用之后，我们继续讨论 Helm 的其它安全事项，我们的下一个主题是 Helm 图表开发相关的安全性。

## 10.6 开发安全的 Helm 图表

虽然来源和完整性在 Helm 的安全中扮演着重要的角色，但它们仅是其中的一环。图表开发人员应该确保在开发过程中，遵守有关安全性的最佳实践，以防止用户在 Kubernetes 集群中安装图表时引入漏洞。在本节中，我们将讨论与 Helm 图表开发有关的主要安全问题，以及作为开发人员，该如何以安全为最高优先级开发 Helm 图表。

我们先从 Helm 图表使用的容器镜像的安全性开始讨论。

### 10.6.1 使用安全的镜像

由于 Helm(和Kubernetes) 的目标是部署容器镜像，镜像本身就是一个主要的安全问题。首先，图表开发人员应该了解镜像 tags 和镜像摘要的区别。

tag 简单的描述了镜像内容，方便开发人员和使用者能快速获知镜像提供了什么功能。但 tag 也可能带来安全隐患，因为不能保证 tag 和镜像内容始终一致。例如，镜像维护人员可能使用相同的 tag，提交解决了某个安全漏洞的新镜像，这会导致容器在运行时，使用了不同的镜像，从而导致一些问题。除了 tag，镜像也可以使用 digest 获取。镜像 digest 是镜像的 SHA-256 值，不可变，容器运行时可以根据该值验证镜像是否包含预期的内容。这消除了使用 tag 导致拉取错镜像的隐患，还可以消除中间人攻击的风险。

例如，图表模板中配置镜像时，可以把使用 tag 的 quay.io/bitnami/mongodb:4.2.15 替换为 digest 的 quay.io/bitnami/mongodbsha256:bb084ff5c04c6c6b86e39fd7ebefe9af52e9e4663594a2f76fdea707bf5bcf31，以确保镜像内容不会被修改。

随着时间推移，一个安全的镜像可能变得不安全，如该镜像里的操作系统版本被发现有漏洞。有很多方法能够检测镜像是否包含漏洞，利用镜像注册中心的漏洞扫描功能就是常用的一种方式。

Quay 容器注册表（Quay container registry）可以按照指定的时间间隔扫描镜像，以确定镜像包含的漏洞数量。Nexus 和 Artifactory 容器注册表也有这种功能，当然还有 CNCF 中毕业的 Harbor。除了容器注册中心提供的扫描功能外，还可以利用其它工具，如 Clair（Quay、Harbor都使用它进行漏洞扫描）、anchor、Vuls 和 OpenSCAP。当你的镜像 registry 或独立扫描工具扫描出镜像容易被攻击时，就该对镜像进行更新了，以防止将漏洞引入用户的 Kubernetes 群集。

作为镜像维护人员，你需要定期对镜像进行扫描，以防止镜像中存在漏洞。当然，你还需确保镜像是来自于可靠的组织和镜像注册中心，以减少镜像包含漏洞的可能性。此设置是在容器运行时上配置的，不同运行时的配置也不相同。

除了镜像漏洞扫描和镜像来源，你还应该避免部署需要提升权限或功能的镜像。功能是为进程提供根权限的子集。例如 NET_ADMIN，它允许进程执行与网络相关的操作，SYS_TIME 允许进程修改系统时钟。使用 root 身份运行容器可以访问所有功能，所以要尽量避免使用 root 用户。

授予容器功能或以 root 身份运行容器，恶意进程会更有可能破坏底层主机。这不仅会影响包含漏洞的容器，还会影响在该主机上运行的其它容器，甚至可能影响整个 Kubernetes 集群。如果一个容器确实有漏洞，但没有授予它任何功能，则攻击范围要小很多，并且有可能完全被阻止。在开发 Helm 图表时，必须考虑镜像的漏洞和权限要求，以确保你及 Kubernetes 集群中其他租户的安全。

除了镜像外，图表开发人员还应关注分配给应用程序的资源。我们在下一节进行讨论。

### 10.6.2 设置资源限制

Pod 使用底层主机的资源，如果没有设置适当的默认值，Pod 可能会耗尽节点的资源，从而导致 CPU throttling、内存 OOM 或者 Pod 被驱逐等问题，同时也会影响该节点上的其他 Pod。由于资源限制不受控制时可能会出现问题，所以图表开发人员应关注在 Helm 图表或 Kubernetes 集群中设置合理的默认值。

可以在图表的 values.yaml 文件中定义 deployment resources 字段，为图表运行设置一个合理的默认值：

```yaml
resources:
  limits:
    cpu: 500m
    memory: 2Gi
```

在这个默认值中，CPU 限制为 500m，内存限制为 2Gi。这样做，既防止了节点资源被耗尽，又给出了合理的建议值。用户可以根据实际使用情况修改这些值。开发人员也可以提供 requests 的默认值，但它不能避免节点资源被耗尽的风险。

你还应该考虑，在 values.yaml 文件中，对图表要部署到的命名空间设置 limit range(限制范围) 和 resource quotas(资源配额)。这些资源通常不包含在 Helm 图表中，而是由集群管理员在应用程序部署前创建的。limit range 用于定义允许命名空间使用的资源量，还用于部署到尚未定义 limit range 的命名空间的每个容器设置默认资源限制。以下是 LimitRange 对象的定义示例：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits-per-container
spec:
  limits:
    - max:
        cpu: 1
        memory: 4Gi
      default:
        cpu: 500m
        memory: 2Gi
      type: Container
```

LimitRange 在创建 LimitRange 对象的命名空间中强制执行指定的限制。它定义了最大的 CPU 资源设置为 1 核，最大的内存限制为 4Gi。如果没有定义资源限制，它会自动将资源限制设置为 500m 和 2Gi。type 字段也可以设置为 Pod，这样限制级别就是 Pod了，Pod 中所有容器的资源总和应低于指定的值。除了 CPU 和内存的限制外，还可以将 type 字段设置为 PersistentVolumeClaim，将 LimitRange 对象设置为 PersistentVolumeClaim 对象声明的默认存储。

以下是对 PVC 存储的限制配置：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits-per-pvc
spec:
  - max:
      storage: 4Gi
    type: PersistentVolumeClaim
```

当然，在图表的 values.yaml 文件中，也可以设置存储的默认值，LimitRange 对象会强制覆盖用户指定的最大值。

除了限制范围外，你还可以设置资源配额，以对命名空间中的资源使用添加额外的限制。限制范围在每个容器、Pod 或 PVC 级别强制使用资源，而资源配额在每个命名空间级别强制使用资源。它们用于定义命名空间可以利用的最大资源数量。资源配额示例如下:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-and-pvc-quota
spec:
  hard:
    limits.cpu: '4'
    limits.memory: 8Gi
    requests.storage: 20Gi
```

ResourceQuota 对象会把整个命名空间的资源使用量限制在 4 核 CPU 和 8Gi 内存，以及 20Gi 存储。资源配额还可用于 secrets、ConfigMap 和其他 Kubernetes 资源。通过使用资源配额，可以防止单个命名空间过度利用集群资源。

通过在 Helm 图表中设置合理的默认资源限制，以及 LimitRange 和 ResourceQuota 的存在，你可以确保部署该Helm 图表不会耗尽集群资源并导致中断。了解了如何强制执行资源限制后，让我们继续讨论下一个主题，Helm 图表安全——如何处理 Helm 图表中的 secrets。

### 10.6.3 处理 Helm 图表中的 secrets

在 Helm 图表中，如何处理 secrets 是一个常见问题。在第三章中安装 WordPress 时，我们为 admin 用户设置了密码。默认情况下，values.yaml 文件中是不应提供密码的，因为如果你忘记修改此密码，会导致程序更易被攻击。图表开发人员应该养成不为密码提供默认值的习惯，而应要求由最终用户显式提供值。这可以通过 required 函数实现。Helm 还可以使用 randAlphaNum 函数生成随机字符串。

但 randAlphaNum 函数会在每次执行图表升级时，都会生成一个新的随机字符串。因此开发人员在设计图表时，应该要求用户提供密码，并使用 required 函数确保用户提供了该值。

用户在安装图表时用到的 secrets，应该保存在 Kubernetes 的 secret 资源中，它以 Base64 编码对内容加密，而不是纯文本的方式保存在 ConfigMap 资源中。Secret 还可以作为 tmpfs 挂载到 Pod 中，Pod 随着应用的生命周期结束而销毁，要比把 secrets 放在磁盘上更安全。作为图表开发人员，应该确保你的 Helm 图表管理的所有凭证和机密配置都是使用 Kubernetes Secrets 创建的。

图表开发人员应该使用 Kubernetes secrets 和 required 函数恰当的处理 secrets，而图表用户应该安全地向Helm 图表提供凭证和私密数据。Helm 图表的值通常以 **--values** 标志提供，附加值或重写值通常在 values 文件中声明，并在安装期间传递给 Helm CLI。在处理常规值时，这是一种适当的方式，但在使用这种方式处理 secrets 值时应该足够谨慎，不能把包含 secrets 的值文件检入到 git 存储库或其他可能暴露 secrets 的公共空间。

避免暴露 secrets 的一种方法是使用 **--set** 标志，从本地命令行传递 secrets。这降低了凭证暴露的风险，但是用户应该意识到，凭证会暴露在 **bash** 历史命令中。

避免暴露 secrets 的另一种方法是使用加密工具对包含 secrets 的值文件加密。这样就可以把加密后的值文件推送到远程 git 仓库了。值文件只能由手握密钥的用户解密，只允许受信任的成员访问。用户可以使用 GPG 对值文件加密，也可以使用 sops 等工具。sops 用于加密 YAML 或 JSON 的值，但不加密键。以下是 sops 加密文件的键值对：

```yaml
password:ENC[AES256GCM,data:xhdUx7DVUG8bitGnqjGvPMygpw==,iv:3LR9KcttchCvZNpRKqE5LcXRyWD1I00v2kEAIl1ttco=,tag:9HEwxhT9s1pxo9lg19wyNg==,type:str]
```

password 键并没有被加密，它的值是被加密的。这样既可以方便查看文件包含了哪些键，又不会暴露其值。

还有其它工具可以加密包含 secrets 的值文件，如 git-crypt(https://github.com/AGWA/git-crypt)、blackbox(https://github.com/StackExchange/blackbox)。此外，HashiCorp 的 Vault、CyberArk Conjur 之类的工具可用于 key/value 存储的形式加密。然后，可以通过使用秘钥管理系统进行身份验证，通过 **--set** 传递它们，在 Helm 中使用它们来检索加密的值。

了解了 Helm 图表开发过程中的安全注意事项后，现在我们开始讨论如何在 Kubernetes 中使用基于角色的访问控制来为用户提供更大的安全性。

## 10.7 配置 RBAC 规则

Kubernetes 中经过身份验证的用户，要操作集群，还需要一组 RBAC 策略。在第二章我们介绍过策略，也就是角色，可以与 user 或者 service accounts 相关联，Kubernetes 包含几个可以关联的默认角色。从 1.6 版本开始，Kubernetes 中默认启用了 RBAC。在为 Helm 配置 RBAC 时，需要考虑两个因素：

* 安装 Helm 图表的使用的 user
* 与运行的 Pod 关联的 service account

在大多数情况下，负责安装 Helm 图表的个人与 Kubernetes user 有关。不过，Helm 图表可以通过其他方式安装，例如由 Kubernetes operator 使用关联的 service account 安装。

默认情况下，users 和 service account 在 Kubernetes 集群中的权限是最小的。通过使用作用域为单个命名空间的角色或在集群级别授予访问权的集群角色，可以授予其他权限。然后使用角色绑定或集群角色绑定将它们与 users 或 service account 关联，具体取决于目标策略的类型。虽然 Kubernetes 包含了许多可以应用的角色，但是应该尽可能使用最小特权访问的概念。最小特权访问是指仅授予用户或应用程序正常工作所需的最小权限集。以我们之前开发的 player-stats 图表为例，假设我们想要添加新的功能，可以查询 player-stats 应用程序命名空间中 Pods 的元数据。

虽然 Kubernetes 有一个名为 view 的内置角色，提供查询指定命名空间中 Pod 清单所需的权限，但它也提供对其它资源的访问，比如 ConfigMap 和 Deployment。为了最小化授权应用程序的访问级别，可以创建一个角色或集群角色形式的自定义策略，该策略仅提供应用程序所需的必要权限。由于 Kubernetes 集群的大多数用户没有在集群级别上创建资源的权限，所以让我们创建一个应用于 Helm 图表所在命名空间的角色。

**kubectl create role** 命令用于创建角色，基本角色包含两个因素：

* 针对 Kubernetes API 的动作(动词)类型
* Kubernetes 的目标资源列表

我们为经过身份验证的 user 添加查看命名空间中 Pod 的权限，以此来演示如何创建 RBAC。

在此之前，确保已经使用 **minikube start** 命令启动了 Minikube。

创建一个新的命名空间，名称为 testrbac：

```bash
$ kubectl create ns testrbac
```

1. 使用 Kubectl CLI 创建一个名称为 playerstats-pod-viewer 的角色：

   ```bash
   $ kubectl create role playerstats-pod-viewer --resource=pods --verb=get,list -n testrbac
   ```

   创建新角色后，需要将其与 user 或者 service account 关联。因为我们要把它与 Kubernetes 中运行的应用程序关联，所以我们把这个角色应用到一个 service account 上。在创建 Pod 时，它使用名为 default 的 service account。在尝试遵循最小权限访问时，建议使用一个单独的 service account。这是为了确保没有其它应用部署在与 player-stats 相同的命名空间中，因为它也将继承相同的权限。

2. 创建名为 player-stats 的 service account：

   ```bash
   $ kubectl create sa player-stats -n testrbac
   ```

3. 接下来，创建一个名为 playerstats-pod-viewer 的角色绑定(role binding)，将 playerstats-pod-viewer 角色与 player-stats ServiceAccount 关联起来：

   ```bash
   $ kubectl create rolebinding playerstats-pod-viewer --role=playerstats-pod-viewer --serviceaccount=testrbac:player-stats -n testrbac
   ```

   要使用新创建的 player-stats ServiceAccount 运行 player-stats 应用程序，需要将 service account 的名称应用于 Deployment。

   下面显示了 Deployment YAML中的 serviceAccount 配置：

   ```yaml
   serviceAccountName: player-stats
   ```

   配置完这些，就可以安装 player-stats 应用程序了。

   https://github.com/weiwendi/learn-helm/tree/main/helm-charts/charts/player-stats 这个player-stats  图表设置了 deployment 的 service account 的值。

4. 运行以下命令安装 player-stats 图表：

   ```bash
   $ helm install my-playerstats learn-helm/helm-charts/charts/player-stats \
   --set serviceAccount.name=player-stats \
   --set serviceAccount.create=false \
   -n testrbac
   ```

   因为我们在之前的步骤中创建了 serviceAccount，所以 --set serviceAccount.create 的值设置为了 false。

5. 此时 player-stats 应用程序已拥有了获取和列出所有 Pod 的权限，为了验证这个假设，我们可以使用 kubectl 命令查询 user 或 service account 是否有权执行某个操作。执行以下命令验证 ServiceAccount player-stats 是否有权限查询 player-stats 命名空间中所有的 Pod：

   ```bash
   $ kubectl auth can-i list pods --as=system:serviceaccount:testrbac:player-stats -n testrbac
   ```

   **--as** 标志利用 Kubernetes 中的用户模拟特性来允许调试授权策略。

6. 以上命令的输出应该为 yes。要验证 service account 无法访问其不该访问的资源，例如列出 deployment，可以执行以下命令：

   ```bash
   $ kubectl auth can-i list deployments --as=system:serviceaccount:player-stats:player-stats -n testrbac
   # 这次会输出 no.
   ```

7. 测试完毕，可以删除我们的部署了：

   ```bash
   $ helm uninstall my-playerstats -n testrbac
   ```

   也可以停在 Minikube 了，它在本章中剩余部分已经不再需要了：

   ```bash
   $ minikube stop
   ```

正确使用 Kubernetes RBAC，可以帮助 Helm 开发人员总是以最小权限访问所需的工具，从而保护用户和应用程序免受潜在的意外或恶意行为的影响。

接下来我们讨论如何以增强 Helm 整体安全性的方式，保护和访问图表存储库。

## 10.8 图表存储库的安全设置

可以通过图表存储库搜索 Helm 图表，并将它们安装到 Kubernetes 集群上。第一章中讨论了 Helm 图表存储库，它是一个 HTTTP 服务，包含 index.yaml 文件，记录了存储库中的图表元数据。在前面的章节中，我们使用了来自不同存储库的图表，还使用 GitHub 页面实现了自己的存储库。这些存储库都可以免费供人使用。但是，Helm支持额外的安全措施来保护存储库中存储的内容，包括：

* Authentication
* SSL/TLS

虽然大多数公共存储库不需要任何形式的身份验证，但 Helm 支持用户对安全的图表存储库执行基于证书和基本的身份验证功能。对于基本身份验证，当使用 **helm repo add** 添加存储库时，可以使用 **--username** 和 **--password** 标志提供用户名和密码。例如，如果要访问使用基本身份验证保护的存储库，需要使用以下方式添加存储库：

```bash
$ helm repo add $REPO_URL --username=<username> --password=<password>
```

接着，就可以与存储库交互，而无需每次提供凭证。

对于基于证书的身份认证，**helm repo add** 命令提供了 **--ca-file**, **--cert-file **和 **--key-file** 标志。**--ca-file** 标志用于验证图表存储库的权威证书，**--cert-file** 和 **--key-file** 标志分别用于指定客户端证书和密钥。

在图表存储库上启用基本身份认证和证书认证的方式，取决于存储库本身。主流的 ChartMuseum 存储库使用 **--basic-auth-user** 和 **--basic-auth-pass** 标志，可以在启动时配置基本身份验证的用户名和密码。它还提供了 **--tls-ca-cert** 标志配置证书颁发结构(CA)以进行证书验证。其他图表存储库实现可能会提供其他标志或要求你提供配置文件。

开启了身份验证，也要保证 HTTP 服务器与 Helm 之间的传输是可靠的，可以使用 SSL 或 TLS 对传输进行加密，以确保 Helm 客户端和 Helm 图表存储库之间的通信安全。这样使用基本身份验证的存储库(以及未使用身份验证的存储库)都可以从加密网络中受益，因为这会尝试保护身份验证和存储库的内容。与身份验证一样，在图表存储库上配置 TLS 取决于所使用的存储库实现。ChartMuseum 提供 **--tls-cert** 和 **--tls-key** 标志来提供证书链和密钥文件。更常用的 Nginx 等 Web 服务器，通过一个配置文件指定证书及密钥文件的位置。GitHub 页面已经配置了TLS。

到目前为止，我们使用的每个 Helm 存储库都使用了由公共可用 CA 颁发的证书，这些证书存储在 Web 浏览器和底层操作系统中。许多大型组织都有自己的 CA，可用于生成在图表存储库中配置的证书。由于证书可能不是来自公开可用的 CA，因此 Helm CLI 可能不信任该证书，添加存储库会导致以下错误：

```bash
Error: looks like '$REPO_URL' is not a valid chart repository or cannot be reached: Get $REPO_URL/index.yaml: x509: certificate signed by unknown authority
```

要允许 Helm CLI 信任图表存储库的证书，可以将 CA 证书或包含多个证书的 CA 包添加到操作系统的信任存储中，也可以使用 **helm repo add** 命令的 **--ca-file** 标志显式指定，这样就可以无误地执行命令。

最后，根据图表存储库的配置方式，还可以获得额外的 Metrics 来执行请求审计和日志记录，以确定谁试图访问存储库。

通过使用身份验证和管理控制传输层的证书，实现了增强 Helm 存储库安全性的额外功能。