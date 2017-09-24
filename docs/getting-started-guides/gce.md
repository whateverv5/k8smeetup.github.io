---
approvers:
- brendandburns
- jbeda
- mikedanese
- thockin
title: 在 Google 计算引擎上运行 Kubernetes
---

下面的例子创建了4个工作节点虚拟机和一个主虚拟机（例如，你的集群里有5个虚拟机）。这个集群从你的工作平台建立和控制（或者你方便的任何地方）。

* TOC
{:toc}



### 开始之前

如果你想要一个精简的入门体验和 GUI 管理集群，请考虑尝试 [Google 容器引擎](https://cloud.google.com/container-engine/)（GKE）来托管集群安装和管理。

如果你想使用自定义二进制文件或纯开源 Kubernetes，请继续参考下面。



### 先决条件

1. 你需要一个开启账单的 Google 云平台账户。浏览 [Google 开发控制台](https://console.cloud.google.com)获取更多详细内容。
1. 安装 `gcloud` 是必要。`gcloud` 作为 [Google 云 SDK](https://cloud.google.com/sdk/) 的一部分，可以被安装。
1. 在 [Google 云开发控制台](https://console.developers.google.com/apis/library)开启[计算引擎实例组管理 API](https://console.developers.google.com/apis/api/replicapool.googleapis.com/overview)
1. 确保 gcloud 被设置为你想要使用的 Google 云平台工程。你可以使用 `gcloud config list project` 检查当前的工程，通过 `gcloud config set project <project-id>` 来修改。
1. 通过运行 `gcloud auth login` 确保你有 GCloud 权限。
1. 确保你能通过命令行启动一个 GCE VM。至少确保你能做[创建一个实例](https://cloud.google.com/compute/docs/instances/#startinstancegcloud)的 GCE 快速开始部分。
1. 确保你能无交互提示地 ssh 进 VM。看[登录到实例](https://cloud.google.com/compute/docs/instances/#sshing)的 GCE 快速开始部分。



### 启动一个集群

你可以安装一个客户端并且使用下面任一命令启动一个集群（我们列出两个以防只有一个命令安装在你的机器上）：

```shell
curl -sS https://get.k8s.io | bash
```

或

```shell
wget -q -O - https://get.k8s.io | bash
```

一旦这个命令完成，你将有一个主 VM 和四个工作 VM，作为 Kubernetes 集群运行。

默认的，一些容器已经运行在你的集群。像容器 `fluentd` 提供[日志](/docs/user-guide/logging/overview)，同时 `heapster` 提供[监控](http://releases.k8s.io/{{page.githubbranch}}/cluster/addons/cluster-monitoring/README.md)服务。

通过上面命令运行脚本创建一个集群命名／签准为 “kubernetes”。它定义了一个指定的集群配置，所以你不能多次运行它。

或者，你可以从[这个页面](https://github.com/kubernetes/kubernetes/releases)下载安装最新的 Kubernetes 版本，然后运行 `<kubernetes>/cluster/kube-up.sh` 脚本来开启集群：

```shell
cd kubernetes
cluster/kube-up.sh
```

如果你想要多于一个集群运行在你的工程，想使用一个不同的名字，或者想要一个不同数量的工作节点，在你启动你的集群之前更多细粒度配置看 `<kubernetes>/cluster/gce/config-default.sh` 文件。

如果你运行出现问题，请看[问题分析](/docs/getting-started-guides/gce/#troubleshooting)部分，提交到 [kubernetes 用户组](https://groups.google.com/forum/#!forum/kubernetes-users)，或者在 [Slack](/docs/troubleshooting/#slack) 上提问题。

接下来的一些步骤将展示给你：

1. 如何在你的工作平台建立命令行客户端来管理你的集群
1. 如何使用集群的例子
1. 如何删除集群
1. 如何启动非默认选项的集群（像更大的集群）



### 在你的工作站安装 Kubernetes 命令行工具

集群启动脚本将运行集群并在你的工作站上留下一个 `kubernetes` 目录。

[kubectl](/docs/user-guide/kubectl/) 工具管理 Kubernetes 集群。它可以让你审查你的集群资源，创建，删除，和更新组建，还有更多。你将使用它来查看你的新集群和启动例子应用。

你可以在你的工作站使用 `gcloud` 来安装 `kubectl` 命令行工具：

     gcloud components install kubectl

**注：** 与 `gcloud` 绑定的 kubectl 版本也许比从 get.k8s.io 下载安装脚本要旧。看[安装 kubectl](/docs/tasks/kubectl/install/) 文档来了解你如何在工作站安装最新的 `kubectl`。



### 集群入门

#### 查看你的集群

一旦 `kubectl` 在你的路径里，你就可以使用它查看你的集群。例如运行：

```shell
$ kubectl get --all-namespaces services
```

应该输出一系列[服务](/docs/user-guide/services)，看起来像这样：

```shell
NAMESPACE     NAME                  CLUSTER_IP       EXTERNAL_IP       PORT(S)        AGE
default       kubernetes            10.0.0.1         <none>            443/TCP        1d
kube-system   kube-dns              10.0.0.2         <none>            53/TCP,53/UDP  1d
kube-system   kube-ui               10.0.0.3         <none>            80/TCP         1d
...
```

类似的，你可以查看一系列在启动集群期间创建的 [pods](/docs/user-guide/pods)。你可以通过这个

```shell
$ kubectl get --all-namespaces pods
```

命令查看。

你将看到 pods 列表，看起来像这样（具体的名字是不一样的）：

```shell
NAMESPACE     NAME                                           READY     STATUS    RESTARTS   AGE
kube-system   fluentd-cloud-logging-kubernetes-minion-63uo   1/1       Running   0          14m
kube-system   fluentd-cloud-logging-kubernetes-minion-c1n9   1/1       Running   0          14m
kube-system   fluentd-cloud-logging-kubernetes-minion-c4og   1/1       Running   0          14m
kube-system   fluentd-cloud-logging-kubernetes-minion-ngua   1/1       Running   0          14m
kube-system   kube-dns-v5-7ztia                              3/3       Running   0          15m
kube-system   kube-ui-v1-curt1                               1/1       Running   0          15m
kube-system   monitoring-heapster-v5-ex4u3                   1/1       Running   1          15m
kube-system   monitoring-influx-grafana-v1-piled             2/2       Running   0          15m
```

一些 pod 也许要花一些时间来启动（在这期间他们将显示为 `Pending`），但是短暂的检查后他们所有显示为 `Running` 



#### 运行一些例子

接下来，看[一个简单的 nginx 例子](/docs/user-guide/simple-nginx) 来试试你的新集群。

为更多完整应用，请看[试例目录](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/examples/)。[留言簿试例](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/examples/guestbook/)是一个好的“入门”演练。



### 销毁集群

移走／删除／销毁集群，使用 `kube-down.sh` 脚本。

```shell
cd kubernetes
cluster/kube-down.sh
```

同样的，在同目录下 `kube-up.sh` 会重新启动回来。你不需要重新运行 `curl` 或 `wget` 命令：所有东西需要部署到 Kubernetes 集群已经在你的工作站上了。




### 定制

上面的脚本依赖于 Google 存储预制的 Kubernetes 发行版。它将启动一个单独的主 VM 和 4 个工作 VM。你可以通过编辑 `kubernetes/cluster/gce/config-default.sh` 修改一些参数.[这里](https://gist.github.com/satnam6502/fc689d1b46db9772adea)你可以浏览一个成功创建集群的副本。



### 问题排查

#### 工程设置

你需要有 Google 云存储 API，开启 Google 云存储 JSON API。新工程默认是激活的。另一方面，也可以在 Google 云控制台来完成。更多详细可以查看 [Google 云存储 JSON API 概览](https://cloud.google.com/storage/docs/json_api/)。

还要确保 -- 在[先决条件部分]（＃先决条件）中列出 -- 您启用了 “Compute Engine Instance Group Manager API”，并可以从命令行启动 GCE VM，如 [GCE 快速入门](https://cloud.google.com/compute/docs/quickstart)教程。



#### 集群初始化挂起

如果 Kubernetes 启动脚本挂起等待获取 API，你可以通过 SSHing 进入主和节点 VM 查看日志，如 `/var/log/startupscript.log`。

在运行 `kube-up.sh` 再次尝试之前，**一旦你解决了问题，你应该运行 `kube-down.sh` 来清理**部分集群创建。



#### SSH

如果你 SSHing 到你的实例有问题，确保 GCE 防火墙没有限制你的 VM 22端口。默认这个应该是工作的，但是如果你编辑了防火墙规则或创建了一个新的非默认网络，你将需要暴露它：`gcloud compute firewall-rules create default-ssh --network=<network-name> --description "SSH allowed from anywhere" --allow tcp:22`

另外，你的 GCE SSH 密钥必须没有密码或你需要使用 `ssh-agent`。



#### 网络

实例必须能使用他们的私网 IP 连接到每个彼此。脚本使用 “default” 网络，它应该有一个防火墙规则叫做 “default-allow-internal”,它允许私有 IP 的所有端口流量。如果这个规则从默认网络丢失或者如果你改变了网络，使用 `cluster/config-default.sh` 创建一个新规则字段如下：

* Source Ranges: `10.0.0.0/8`
* Allowed Protocols and Port: `tcp:1-65535;udp:1-65535;icmp`



## 支持级别

IaaS Provider        | Config. Mgmt | OS     | Networking  | Docs                                              | Conforms | Support Level
-------------------- | ------------ | ------ | ----------  | ---------------------------------------------     | ---------| ----------------------------
GCE                  | Saltstack    | Debian | GCE         | [docs](/docs/getting-started-guides/gce)                                    |   | Project

所有解决方案的支持等级信息，看[解决方案表](/docs/getting-started-guides/#table-of-solutions)表格.

## 进一步阅读

请看 [Kubernetes 文档](/docs/)获取关于管理使用 Kubernetes 集群更多详细。
