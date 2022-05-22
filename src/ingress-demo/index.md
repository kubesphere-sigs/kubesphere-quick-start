# 应用路由与服务示例

本文档介绍了如何在 KubeSphere 上开启集群网关，以及如何创建、使用和编辑应用路由。

KubeSphere 上的应用路由和 Kubernetes 上的 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress) 相同，您可以使用应用路由和单个 IP 地址来聚合和暴露多个服务。

本文将构建一个比较完整的应用路由案例，它的大致架构如下图所示：

![Case Study Overview](ingress-demo.png)

> 注意：上图中的多个 `httpbin` 服务可以是同一个服务，也可以是完全不同的服务。

## 准备工作

### 准备企业空间、项目及用户

您需要创建一个企业空间（例如 `playground`）、一个项目（例如 `demo`）以及两个用户（例如，`admin` 和 `operator`）。

在本教程中，`admin` 用户必须具有 `platform-admin` 角色；`operator` 用户必须至少具有 `platform-regular` 角色，并已被添加到 `playground` 企业空间的 `demo` 项目中（项目角色至少为 `operator`）。

关于企业空间、项目及用户的更多信息，请参见 [多租户管理快速入门](../admin-quick-start.md)。

### 部署 httpbin 示例服务

由于需要演示网关的访问控制能力，我们必须要先有一个可以访问的应用作为网关的后台服务。这里我们使用 [httpbin.org](http://httpbin.org/) 提供的 [kennethreitz/httpbin](https://hub.docker.com/r/kennethreitz/httpbin/) 容器应用作为演示应用。

在 KubeSphere 中，我们可以先创建新的项目或使用已有的项目，进入项目页面后，选择「应用负载」下的「服务」直接创建「无状态服务」并生成配套的服务。

![Create Stateless Service](svc-create.png)

使用 `kennethreitz/httpbin` 容器默认的 `80` 端口作为服务端口，创建 2 个容器组副本，完成后确保在「工作负载」和「服务」页面下都可以看到 `httpbin` 的对应条目，大致过程及最终结果如下图所示（图中未展示的部分默认「下一步」即可）。

![Create Service - Metadata](svc-create-meta.png)

![Create Service - Container](svc-create-container.png)

![Create Service - Pod](svc-create-pod.png)

![Service Created](svc-created.png)

### 启用集群网关

> KubeSphere 从 3.1 版本开始支持项目级别的网关，进而在 3.2.0 版本中开始支持集群级别的全局网关，所有项目可共用同一个网关，之前已创建的项目网关也不会受到集群网关的影响。
>
> 如果您在开始实验本文的案例前已经部署了 Ingress Controller，那么不管集群网关还是项目网关都不是必须的。为了更完整的讲解应用路由的使用，此处仍会概述如何启用 KubeSphere 集群网关。

[项目网关](https://kubesphere.com.cn/docs/project-administration/project-gateway/) 是 KubeSphere 3.0 以来就有的功能：“KubeSphere 项目中的网关是一个 [NGINX Ingress 控制器](https://www.nginx.com/products/nginx-ingress-controller/)。KubeSphere 内置的用于 HTTP 负载均衡的机制称为 [应用路由](https://kubesphere.com.cn/docs/project-user-guide/application-workloads/routes/)，它定义了从外部到集群服务的连接规则。如需允许从外部访问服务，用户可创建路由资源来定义 URI 路径、后端服务名称等信息。”

进入 KubeSphere 3.2.0 版本之后，我们更推荐大家使用集群网关的功能来统一整个集群的应用路由。要启用集群网关其实也非常简单：使用具备集群管理权限的账号（如本文中的 `admin`），进入其可管理的某个集群（我们这里以 `default` 集群为例），在「集群设置」的「网关设置」中即可「开启网关」。

方便起见，本例中直接选择 `NodePort` 作为「访问方式」，完成开启后回到设置页面，稍等片刻后刷新页面，可以得到如下图这样的部署完成状态，可以看到 NodePort 默认被赋予了两个节点端口，并且通过右上角的「管理」按钮我们可以「查看详情」。

![Cluster Gateway Created](cgw-created.png)

在系统中建立各类应用路由并使用一段时间后，即可通过「查看详情」看到如下图所示的监控数据面板：

![Cluster Gateway Metrics](cgw-metrics.png)

#### 为网关指定 NodePort 节点端口

对于公有云环境，如果使用 NodePort 方式向外暴露访问能力，开放的端口通常会是有限的、受控的、甚至是固定的，因此对于网关所使用的 NodePort 我们需要能够对它进行修改。

由于网关是被 KubeSphere 统一管理的，要修改网关服务的 NodePort，需要具备访问 `kubesphere-controls-system` 项目的权限。进入该项目后，通过「应用负载」的「服务」页面即可找到命名为 `kubesphere-router-kubesphere-system` 且外部访问已开放 NodePort 的网关服务。NodePort 服务端口需要通过「编辑 YAML」来直接修改。

![Cluster Gateway NodePort Customization](cgw-nodeport.png)

## 应用路由的创建及使用

> 下文将以 `demo` 项目中的 `httpbin` 服务为例，讲解如何配置及使用 KubeSphere 应用路由。

### 创建及使用 HTTP 路由

从「应用负载」进入「应用路由」页面，开始「创建」路由。为路由取名为 `httpbin` 后，我们指定一个方便测试的域名（例如 `httpbin.io`），并设置「路径」为 `/`, 选择「服务」`httpbin` 和「端口」`80`，点击下一步跳过「高级设置」后即完成路由创建。

![Route Create](route-create.png)

![Route Create - Rules](route-create-rule.png)

> 通过集群、项目（或自建的）网关从私有网络外部访问应用路由时，您需要特别注意您的网络环境：
>
> - 您可能需要在基础设施环境中配置流量转发和防火墙规则，以便访问应用路由的网关地址和端口号。
> - 若在配置路由规则中选择「自动生成」，则可能需要手动编辑路由规则将路由域名中的网关地址改为您私有网络的外部 IP 地址。
> - 若在配置路由规则中选择「指定域名」，则可能需要改变 DNS 服务器上或者客户端机器上的域名配置（如 Linux / macOS 系统中的 `etc/hosts` 文件），以便将域名解析为您私有网络的外部 IP 地址。

回到「应用路由」页面，点击刚刚创建的 `httpbin` 路由，即可进入详情页面。

![Route Created - HTTP](route-created.png)

您可以在「资源状态」标签页上通过右侧的「访问服务」按钮访问 httpbin 页面，在域名解析配置正确的情况下，可以得到下图的页面结果。

![Route - httpbin.io](route-httpbin.png)

### 创建及使用 HTTPS 路由

由于要在应用路由中绑定的域名为 HTTPS 协议，因此需要预先创建一个携带 TLS 证书的 Kubernetes Secret。

首先，我们回到项目首页，在「配置」下的「保密字典」中创建一个名为 `httpbin-tls` 的保密字典（即 Kubernetes Secret）。如下图所示，选择字典「类型」为 `TLS 信息`。

![Secret Create - Data](secret-create-data.png)

在「凭证」和「私钥」文本框中填入您的凭证和私钥数据。

> 注意，您可以使用任意您熟悉的工具来创建 TLS 证书密钥，本例也会提供一份实验用凭证和私钥，请勿在生产环境中使用！
>
> ```plain
> -----BEGIN CERTIFICATE-----
> MIIDLjCCAhYCCQDAOF9tLsaXWjANBgkqhkiG9w0BAQsFADBaMQswCQYDVQQGEwJV
> UzELMAkGA1UECAwCQ0ExITAfBgNVBAoMGEludGVybmV0IFdpZGdpdHMgUHR5IEx0
> ZDEbMBkGA1UEAwwSY2FmZS5leGFtcGxlLmNvbSAgMB4XDTE4MDkxMjE2MTUzNVoX
> DTIzMDkxMTE2MTUzNVowWDELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAkNBMSEwHwYD
> VQQKDBhJbnRlcm5ldCBXaWRnaXRzIFB0eSBMdGQxGTAXBgNVBAMMEGNhZmUuZXhh
> bXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCp6Kn7sy81
> p0juJ/cyk+vCAmlsfjtFM2muZNK0KtecqG2fjWQb55xQ1YFA2XOSwHAYvSdwI2jZ
> ruW8qXXCL2rb4CZCFxwpVECrcxdjm3teViRXVsYImmJHPPSyQgpiobs9x7DlLc6I
> BA0ZjUOyl0PqG9SJexMV73WIIa5rDVSF2r4kSkbAj4Dcj7LXeFlVXH2I5XwXCptC
> n67JCg42f+k8wgzcRVp8XZkZWZVjwq9RUKDXmFB2YyN1XEWdZ0ewRuKYUJlsm692
> skOrKQj0vkoPn41EE/+TaVEpqLTRoUY3rzg7DkdzfdBizFO2dsPNFx2CW0jXkNLv
> Ko25CZrOhXAHAgMBAAEwDQYJKoZIhvcNAQELBQADggEBAKHFCcyOjZvoHswUBMdL
> RdHIb383pWFynZq/LuUovsVA58B0Cg7BEfy5vWVVrq5RIkv4lZ81N29x21d1JH6r
> jSnQx+DXCO/TJEV5lSCUpIGzEUYaUPgRyjsM/NUdCJ8uHVhZJ+S6FA+CnOD9rn2i
> ZBePCI5rHwEXwnnl8ywij3vvQ5zHIuyBglWr/Qyui9fjPpwWUvUm4nv5SMG9zCV7
> PpuwvuatqjO1208BjfE/cZHIg8Hw9mvW9x9C+IQMIMDE7b/g6OcK7LGTLwlFxvA8
> 7WjEequnayIphMhKRXVf1N349eN98Ez38fOTHTPbdJjFA/PcC+Gyme+iGt5OQdFh
> yRE=
> -----END CERTIFICATE-----
> ```
>
> ```plain
> -----BEGIN RSA PRIVATE KEY-----
> MIIEowIBAAKCAQEAqeip+7MvNadI7if3MpPrwgJpbH47RTNprmTStCrXnKhtn41k
> G+ecUNWBQNlzksBwGL0ncCNo2a7lvKl1wi9q2+AmQhccKVRAq3MXY5t7XlYkV1bG
> CJpiRzz0skIKYqG7Pcew5S3OiAQNGY1DspdD6hvUiXsTFe91iCGuaw1Uhdq+JEpG
> wI+A3I+y13hZVVx9iOV8FwqbQp+uyQoONn/pPMIM3EVafF2ZGVmVY8KvUVCg15hQ
> dmMjdVxFnWdHsEbimFCZbJuvdrJDqykI9L5KD5+NRBP/k2lRKai00aFGN684Ow5H
> c33QYsxTtnbDzRcdgltI15DS7yqNuQmazoVwBwIDAQABAoIBAQCPSdSYnQtSPyql
> FfVFpTOsoOYRhf8sI+ibFxIOuRauWehhJxdm5RORpAzmCLyL5VhjtJme223gLrw2
> N99EjUKb/VOmZuDsBc6oCF6QNR58dz8cnORTewcotsJR1pn1hhlnR5HqJJBJask1
> ZEnUQfcXZrL94lo9JH3E+Uqjo1FFs8xxE8woPBqjZsV7pRUZgC3LhxnwLSExyFo4
> cxb9SOG5OmAJozStFoQ2GJOes8rJ5qfdvytgg9xbLaQL/x0kpQ62BoFMBDdqOePW
> KfP5zZ6/07/vpj48yA1Q32PzobubsBLd3Kcn32jfm1E7prtWl+JeOFiOznBQFJbN
> 4qPVRz5hAoGBANtWyxhNCSLu4P+XgKyckljJ6F5668fNj5CzgFRqJ09zn0TlsNro
> FTLZcxDqnR3HPYM42JERh2J/qDFZynRQo3cg3oeivUdBVGY8+FI1W0qdub/L9+yu
> edOZTQ5XmGGp6r6jexymcJim/OsB3ZnYOpOrlD7SPmBvzNLk4MF6gxbXAoGBAMZO
> 0p6HbBmcP0tjFXfcKE77ImLm0sAG4uHoUx0ePj/2qrnTnOBBNE4MvgDuTJzy+caU
> k8RqmdHCbHzTe6fzYq/9it8sZ77KVN1qkbIcuc+RTxA9nNh1TjsRne74Z0j1FCLk
> hHcqH0ri7PYSKHTE8FvFCxZYdbuB84CmZihvxbpRAoGAIbjqaMYPTYuklCda5S79
> YSFJ1JzZe1Kja//tDw1zFcgVCKa31jAwciz0f/lSRq3HS1GGGmezhPVTiqLfeZqc
> R0iKbhgbOcVVkJJ3K0yAyKwPTumxKHZ6zImZS0c0am+RY9YGq5T7YrzpzcfvpiOU
> ffe3RyFT7cfCmfoOhDCtzukCgYB30oLC1RLFOrqn43vCS51zc5zoY44uBzspwwYN
> TwvP/ExWMf3VJrDjBCH+T/6sysePbJEImlzM+IwytFpANfiIXEt/48Xf60Nx8gWM
> uHyxZZx/NKtDw0V8vX1POnq2A5eiKa+8jRARYKJLYNdfDuwolxvG6bZhkPi/4EtT
> 3Y18sQKBgHtKbk+7lNJVeswXE5cUG6EDUsDe/2Ua7fXp7FcjqBEoap1LSw+6TXp0
> ZgrmKE8ARzM47+EJHUviiq/nupE15g0kJW3syhpU9zZLO7ltB0KIkO9ZRcmUjo8Q
> cpLlHMAqbLJ8WYGJCkhiWxyal6hYTyWY4cVkC0xtTl/hUE9IeNKo
> -----END RSA PRIVATE KEY-----
> ```

创建完成后，可以在「保密字典」页面看到如下图所示的 `httpbin-tls` 字典项已经被生成。

![Secret Created](secret-created.png)

然后，我们再次回到「应用路由」页面，再新建一个名为 `httpbin-tls` 的路由，并为它设置路由规则为：`HTTPS` 协议，使用 `httpbin-tls` 保密字典，域名绑定改为 `httpbin.tls`。

![Route Create - HTTPS Rules](route-create-rule-https.png)

同样跳过「高级设置」后创建路由，返回路由列表后点击刚刚生成的 `httpbin-tls` 路由来到详情页面，如下面所示。

![Route Created - HTTPS](route-created-https.png)

点击「访问服务」可以同样成功打开 httpbin 服务的页面。至此，我们成功的为同一个服务同时配置了 HTTP 和 HTTPS 路由。本例虽然使用同一个服务作为应用路由的最终端点，但也可以任意替换成不同的服务，包括使用不同的路由路径。

同时，您可能也注意到了：虽然对于客户端访问我们已经启用了 HTTPS，但网关访问 httpbin 服务仍然是通过 `80` 的 HTTP 应用端口，这就是 TLS 终止（TLS Termination）。

> TLS 终止代理（或 SSL 终止代理，或 SSL 卸载）是充当客户端和服务器应用程序之间的中间点的代理服务器，用于终止和/或建立 TLS（或 DTLS）通过解密和/或加密通信来建立隧道。这与 TLS 传递不同 —— 通过代理在客户端和服务器之间转发加密的 (D)TLS 流量而不终止隧道。
>
> —— [TLS termination proxy](https://en.wikipedia.org/wiki/TLS_termination_proxy)，维基百科

### 通过路由注释实现高级配置

> KubeSphere 的集群及项目网关本身是基于 Kubernetes 官方提供的 [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) 实现的，目前业内各类 Ingress Controller 在 Ingress 标准以外提供各自更为丰富的能力，这些能力主要是通过 Ingress 资源的注释（[Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)）来体现的。

KubeSphere 应用路由的「高级配置」能力便是提供了 Ingress 注释的编辑功能，所以最后我们通过一个案例来展示如何在应用路由中使用 NGINX Ingress Controller 的一些扩展能力。

让我们来看一个现实世界中经常出现的需求：为您的服务路径绑定一个特定的前缀。例如在本文的案例中，我们现在希望更改我们 HTTPS 的应用路由的访问方式为 `https://httpbin.tls/prefix`，即：

- 当请求 `https://httpbin.tls/prefix` 或 `https://httpbin.tls/prefix` 时，仍然可以得到 httpbin 服务的首页
- 当请求 `https://httpbin.tls/prefix/<resource-uri>`（例如 `https://httpbin.tls/prefix/ip`）时，仍然得到对应的 httpbin 资源路径下的页面

> 在开始实验前，让我们先访问 `https://httpbin.tls/ip` 获取数据作为后续的目标结果：`{"origin": "10.0.169.20"}`。

回到「应用路由」页面，找到并点击 `httpbin-tls` 路由项进入详情页面。在左侧的「更多操作」拉下按钮中，我们首先选择「编辑注释」。

![Route Annotations](route-anno-add.png)

如下图所示，注释是一组或多组键值对。我们可以搜索 `rewrite`，找到并使用 `nginx.ingress.kubernetes.io/rewrite-target` 键；在值的部分填入 `/$2` 后即可保存注释（稍后我们会介绍这个值的含义）。

![Route Annotations Added](route-anno-added.png)

> KubeSphere 的应用路由提供了很棒的注释键搜索和自动补全功能，您也可以查阅 NGINX Ingress Controller 官方文档中对于所有 [注释](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) 的详细说明。

完成注释添加后，我们再回到详情页面，同样找到「更多操作」按钮，这次我们选择「编辑路由规则」。进入编辑面板后选择已有的这个路由规则进行编辑：

![Route Rule Edit](route-rule-edit.png)

这里我们把「路径」修改为 `/prefix(/|$)(.*)`。可以注意到这里实际使用了正则表达式的语法，由此也不能理解在注释中的 `$2` 的含义，即指代了第二个匹配组，即 `(.*)` 部分所匹配的内容，也就是 `/prefix` 前缀后的 URI 资源。那么 `rewrite-target: /$2` 即表示将前缀后匹配到的 URI 作为新的请求路径发送给后端服务；如果匹配不到，则一直请求 `/` 路径。

同时正则路径中第一个匹配组 `(/|S)` 表示新的根路径既可以是 `/prefix` 也可以是 `/prefix/`。由此，我们前文所提到的两个路由前缀需求都得到了满足。

保持路由更改后让我们打开浏览器，直接输入地址 <https://httpbin.tls:31357/prefix/ip> 即可检验更新后的路由已经生效，您也可以使用其它 httpbin 服务的资源路径来检验，比如 <https://httpbin.tls:31357/prefix/headers> 会返回您的请求头信息。

> **路由更新后 httpbin 服务首页为何显示不正常？**
>
> ![Route with Proxy - httpbin.tls](route-prefix-httpbin.png)
>
> 相信细心的您一定还会发现虽然 <https://httpbin.tls:31357/prefix> 可以访问，但是界面好像和之前不太一样。没错，这是因为页面的静态资源文件都没有获取到的缘故。
>
> 我们不是已经设置了前缀路由了吗？是的，但这个路由的能力生效的前提是请求也必须带上 `/prefix` 这个前缀。很可惜，我们页面中的静态资源仍然是以 `/` 为根路径来加载的，所以它们没办法被正确的路由。要解决这个问题还需要前端的支持，现在一般主流的前端框架都支持为静态资源设定根访问路径，将其和路由规则对齐后即可解决这个问题。
