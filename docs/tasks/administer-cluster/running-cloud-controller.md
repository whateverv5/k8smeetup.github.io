---
approvers:
- luxas
- thockin
- wlan0
cn-approvers:
- xiaosuiba
cn-reviewers:
- zhangqx2010
title: Kubernetes 云管理控制器

---


**云管理控制器（Cloud Controller Manager）是 1.8 版本中的一个 alpha 特性。在以后的版本中，它将成为 Kubernetes 与任何云服务进行集成的首选方式。这将保证云服务提供商（cloud provider）能够在 Kubernetes 核心发布周期外独立开发他们的特性。**

* TOC
{:toc}


## 云管理控制器


Kubernetes v1.6 包含一个新的二进制文件，叫做 `cloud-controller-manager`。`cloud-controller-manager` 是一个嵌入了特定云服务（cloud-specific）控制循环逻辑的守护进程。这些特定云服务控制循环逻辑最初存在于 `kube-controller-manager` 中。由于云服务提供商开发和发布的速度与 Kubernetes 项目不同，将服务提供商专用代码从 `cloud-controller-manager` 二进制中抽象出来有助于云服务厂商在 Kubernetes 核心代码之外独立进行开发。


`cloud-controller-manager` 可以被链接到任何满足 [cloudprovider.Interface](https://git.k8s.io/kubernetes/pkg/cloudprovider/cloud.go) 约束的云服务提供商。为了兼容旧版本，Kubernetes 核心项目中提供的 [cloud-controller-manager](https://github.com/kubernetes/kubernetes/tree/master/cmd/cloud-controller-manager) 使用和 `kube-controller-manager` 相同的云服务类库。已经在 Kubernetes 核心项目中支持的云服务提供商预计将通过使用 in-tree 的 cloud-controller-manager 过渡到 Kubernetes 核心之外。在将来的 Kubernetes 发布中，所有的云管理控制器将在 Kubernetes 核心项目之外，由 sig 领导者或者云服务厂商进行开发。


## 管理


### 需求


每个云服务都有一套各自的需求用于系统平台的集成，这不应与运行 kube-controller-manager 的需求有太大差异。作为经验法则，你需要：

* 云服务认证/授权：您的云服务可能需要使用令牌或者 IAM 规则以允许对其 API 的访问
* kubernetes 认证/授权：cloud-controller-manager 可能需要 RBAC 规则以访问 kubernetes apiserver
* 高可用：类似于 kube-controller-manager，您可能希望通过主节点选举（leader election，默认开启）配置一个高可用的云管理控制器


### 运行 cloud-controller-manager


您需要对集群配置做适当的修改以成功地运行 cloud-controller-manager：

* **一定不要**为 `kube-apiserver` 和 `kube-controller-manager` 指定 `--cloud-provider` 标志。这将保证它们不会运行任何云服务专用循环逻辑，这将会由云控制管理器运行。未来这个标记将被废弃并去除。
* `kubelet` 必须使用 `--cloud-provider=external` 运行。这是为了保证让 kubelet 知道在执行任何任务前，它必须被云控制管理器初始化。
* `kube-apiserver` **不应该**运行 `PersistentVolumeLabel` 准入控制器，因为云管理控制器将接管对 persistent volume 的标记工作。为了避免运行 PersistentVolumeLabel 准入控制插件，请确认 `kube-apiserver` 的 `--admission-control` 标志的值不包含 `PersistentVolumeLabel`。
* 为了使用 `cloud-controller-manager` 标记 persistent volume，需要启用 initializer 并添加 InitializerConifguration 到系统中。请参照 [这些指导](/docs/admin/extensible-admission-controllers.md#enable-initializers-alpha-feature) 来启用 initializer。使用下面的 YAML 文件创建 InitializerConfiguration：

{% include code.html language="yaml" file="persistent-volume-label-initializer-config.yaml" ghlink="/docs/tasks/administer-cluster/persistent-volume-label-initializer-config.yaml" %}


请记住，配置集群使用云管理控制器将会从几个方面改变集群的行为：

* 指定了 `--cloud-provider=external` 的 kubelet 将被添加一个 `node.cloudprovider.kubernetes.io/uninitialized` 的 taint，导致其在初始化过程中 `不可调度（NoSchedule）`。这将标记该节点在能够正常调度前，需要外部的控制器进行二次初始化。请注意，如果云管理控制器不可用，集群中的新节点会一直处于不可调度的状态。这个 taint 很重要，因为调度器可能需要关于节点的云服务特定的信息，比如他们的区域或类型（high cpu, gpu, high memory, spot instance 等）。
* 集群中节点的云服务信息将不再能够从本地元数据中获取，取而代之的是所有获取节点信息的 API 调用都将通过云管理控制器。这意味着你可以通过限制到 kubelet 云服务 API 的访问来提升安全性。在更大的集群中您可能需要考虑云管理控制器是否会遇到速率限制，因为它现在负责集群中几乎所有到云服务的 API 调用。


对于 v1.8 版本，云管理控制器可以实现：

* node 控制器 - 负责使用云服务 API 更新 kubernetes 节点并删除在云服务上已经删除的 kubernetes 节点。
* service 控制器 - 负责在云服务上为类型为 LoadBalancer 的 service 提供负载均衡器。
* route 控制器 - 负责在云服务上配置网络路由。
* [PersistentVolumeLabel Admission 控制器](/docs/admin/admission-controllers#persistentvolumelabel) - 负责在云服务上标记 persistent volume - 请确保 persistent volume label 准入控制插件没有在您的 kube-apiserver 上启用。
* 如果您使用的是 out-of-tree 提供商，请按需实现其余任意特性。



## 示例


如果当前 Kubernetes 内核支持您使用的云服务，并且想要采用云管理控制器，请参见 [kubernetes 内核中的云管理控制器](https://github.com/kubernetes/kubernetes/tree/master/cmd/cloud-controller-manager)。


对于不在 Kubernetes 内核中的云管理控制器，您可以在云服务厂商或 sig 领导者的源中找到对应的项目。

* [DigitalOcean](https://github.com/digitalocean/digitalocean-cloud-controller-manager)
* [keepalived](https://github.com/munnerz/keepalived-cloud-provider)
* [Rancher](https://github.com/rancher/rancher-cloud-controller-manager)


对于已经存在于 Kubernetes 内核中的提供商，您可以在集群中将 in-tree 云管理控制器作为守护进程运行。请使用如下指南：

{% include code.html language="yaml" file="cloud-controller-manager-daemonset-example.yaml" ghlink="/docs/tasks/administer-cluster/cloud-controller-manager-daemonset-example.yaml" %}



## 限制

运行云管理控制器会有一些可能的限制。虽然以后的版本将处理这些限制，但是知道这些生产负载的限制很重要。


### 对 Volume 的支持

云管理控制器未实现 `kube-controller-manager` 中的任何 volume 控制器，因为和 volume 的集成还需要与 kubelet 协作。由于我们引入了 CSI (容器存储接口，container storage interface) 并对弹性 volume 插件添加了更强大的支持，云管理控制器将添加必要的支持，以使云服务同 volume 更好的集成。请在 [这里](https://github.com/kubernetes/features/issues/178) 了解更多关于 out-of-tree CSI volume 插件的信息。


### 可扩展性

在以前为云服务提供商提供的架构中，我们依赖 kubelet 的本地元数据服务来获取关于它本身的节点信息。通过这个新的架构，现在我们完全依赖云管理控制器来获取所有节点的信息。对于非常大的集群，您需要考虑可能的瓶颈，例如资源需求和 API 速率限制。


### 鸡和蛋的问题


云管理控制器的目标是将云服务特性的开发从 Kubernetes 核心项目中解耦。不幸的是，Kubernetes 项目的许多方面都假设云服务提供商的特性同项目紧密结合。因此，这种新架构的采用可能导致某些场景下，当一个请求需要从云服务提供商获取信息时，在该请求没有完成的情况下云管理控制器不能返回那些信息。


Kubelet 中的 TLS 引导特性是一个很好的例子。目前，TLS 引导认为 Kubelet 有能力从 cloud provider 获取所有的地址类型（私有、公用等），但在被初始化之前，云管理控制器不能设置节点地址类型，而这需要 kubelet 拥有 TLS 证书以和 apiserver 通信。


随着这种措施的演进，将来的发行版中将作出改变来解决这些问题。


## 开发自己的云管理控制器


要构建和开发您自己的云管理控制器，请阅读 [开发云管理控制器](/docs/tasks/administer-cluster/developing-cloud-controller-manager.md) 文档。
