你好，我是孔令飞。

如果你想学习 Kubernetes 相关技术，例如 Kubernetes 运维、Kubernetes 开发，基本上都需要一个 Kubernetes 集群。由于 Kubernetes 集群组件多，部署复杂，为了简化部署流程，降低部署难度，社区提供了不少部署方案和工具供我们选择。

这节课我们就来详细聊聊，社区当前都有哪些部署 Kubernetes 集群的方法，这些方法有什么特点，又分别适用什么场景，让你能够快速选择出合适的方法，搭建出满足需求的 Kubernetes 集群。

## Kubernetes 集群部署方法

当前社区常用的部署工具有：

- hack/local-up-cluster.sh
- Minikube
- Kind
- 二进制部署
- kubeadm
- cluster-api
- 云服务商

接下来，我们就来简单介绍下这些方法。

### local-up-cluster.sh

如果你对 Kubernetes 源码不熟悉，你可能没听过 hack/local-up-cluster.sh 这种部署方式。[hack/local-up-cluster.sh](https://github.com/kubernetes/kubernetes/blob/v1.29.2/hack/local-up-cluster.sh) 是 Kubernetes 源码仓库中的一个脚本，用于在本地快速启动一个单节点的 Kubernetes 集群，这个脚本通常用于开发、测试和调试。

如果你是一名 Kubernetes 源码贡献者，可能经常需要使用 hack/local-up-cluster.sh 脚本来启动一个测试集群，测试 Kubernetes 集群行为是否符合预期，因为该脚本可以基于当前仓库中的代码（master 分支，或者当前项目中的源码）启动一个测试 Kubernetes 集群。

企业社区提供的 Kubernetes 部署工具，基本都是基于正式的 Release 版本来创建 Kubernetes 集群的。

### Minikube VS Kind

Minikube 和 [Kind](https://github.com/kubernetes-sigs/kind)（Kubernetes InDocker）都是 Kubernetes 的官方项目，用来在 macOS、Linux、Windows 上快速部署本地的 Kubernetes 集群。二者当前都处在活跃的更新迭代阶段，很多开发者在刚开始使用时，搞不清二者的区别，以及具体该如何选择。所以接下来，我将它们放到一起来讲。

Minikube 的核心特性如下：

- 支持创建最新的、稳定版的 Kubernetes 集群。
- 跨平台，支持 Linux、macOS、Windows。
- 支持以虚拟机、容器或裸金属的方式部署 Kubernetes 集群。
- 支持 CRI-O、Containerd、Docker 等更多的容器运行时。
- 支持一些高级功能，如负载均衡器、文件系统挂载、FeatureGates 和网络策略。
- 有丰富的 AddOnes 可供选择，这些 AddOnes 可以扩展 Kubernetes 的功能，满足我们特定的需求。Minikube 以 AddOn 的方式，支持很多应用插件，例如 EFK、gVisor、Istio、Kubevirt、Registry、Ingress 等。
- 支持常用的 CI 环境，例如 [Travis CI](https://travis-ci.com/)、[GitHub](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/about-continuous-integration)、[CircleCI](https://circleci.com/)、[GitLab](https://about.gitlab.com/product/continuous-integration/) 等。

这里我要额外解释一下上面提到的虚拟机和容器方式。

- 虚拟机方式：通过 KVM 或 VirtualBox 等虚拟机管理器，在本地创建若干个虚拟机，然后在虚拟机中部署 Kubernetes 组件。
- 容器方式：在本地创建一个容器，然后在容器中部署 Kubernetes 组件。

> 如果你想了解更多关于 Minikube的内容，可阅读其官方文档：[https://minikube.sigs.k8s.io/docs/](https://minikube.sigs.k8s.io/docs/)。

早期的 Minikube 不支持以容器的方式部署 Kubernetes 集群，这直接导致了 Kind 项目的诞生。

Kind 使用 Docker/Podman 驱动，Kubernetes 组件都部署在用 Kindest/Node 镜像启动的 Docker 容器中。

相比 Minikube，Kind 更轻量，需要更少的资源，可以创建轻量的 Kubernetes 集群，大量应用于 CI 流程中。同时，Kind 支持创建 HA 集群，其实就是创建多个 control-plane 类型的 Node 节点，但 Minikube 当前还不支持。不过，社区已经有相关的 PR，只是还没有合并入 Minikube 的主干。而且，Kind 仅支持创建指定版本的 Kubernetes 集群，而非最新的。

> 更多关于 Kind 的介绍和使用，我们会在下节课《如何配置和创建一个 Kind 集群》中介绍，你也可以阅读其官方文档：[https://kind.sigs.k8s.io/](https://kind.sigs.k8s.io/) 来了解。

总的来说，**二者不是一个替代与被替代的关系**。当你对创建 Kubernetes 没有特别需求，且 Minikube 选择 Docker 驱动时，Kind 和 Minikube 其实没有太多区别，你可以通过上面的对比自行选择。

如果你只想快速部署一个测试用的 Kubernetes 集群，没有特殊需求，例如期望在虚拟机上部署，或者期望在集群中部署一些插件（如：EFK、gVisor、Istio、Kubevirt、Registry、Ingress）以增强集群的功能，那么 Kind 是一个好的选择，因为其足够轻量。

如果你想创建一个 HA 集群，当前只能选择 Kind，因为 Minikube 暂时还不支持。如果你想创建一个稍微复杂点的测试集群，并且期望能对集群做更多管控，可以使用 Minikube 来创建。

本课程未来主要使用 Minikube 来创建测试集群，因为 Minikube 功能强大，能创建出功能丰富的 Kubernetes 集群。而且在 Minikube 支持了 Docker 引擎之后，个人感觉 Kind 工具显得有点多余。

### 二进制部署

二进制部署就是直接基于 Kubernetes 官方提供的发布版的二进制文件（例如 [v1.29.2 版本 Kubernetes 二进制文件](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.29.md#v1292)）进行部署。在部署过程中，需要下载这些二进制文件，并根据集群配置创建 Systemd Unit 文件，创建完成后，按照预定的顺序启动 Systemd 服务，完成 Kubernetes 集群的部署。

二进制部署是最复杂、开发工作量最大的部署方式，一般只有在大公司、大团队，或者对部署集群有特殊需求的团队中才会采用这种方式来部署。早期社区没有太多 Kubernetes 集群部署工具，很多团队都采用这种方式来部署集群。但是，随着社区 Kubernetes 部署工具的成熟，例如 kubeadm、cluster-api 等的出现，这些团队逐渐放弃了原始的二进制部署方式，改为基于 kubeadm 或 cluster-api 来部署。

### Kubeadm

[Kubeadm](https://github.com/kubernetes/kubeadm) 是 Kubernetes 官方提供的一个工具，用来便捷地创建生产级可用 Kubernetes 集群。Kubeadm 提供了一种简单快捷的方式，来创建一个最小可行的、安全的 Kubernetes 集群。Kubeadm 是一个运行在当前节点的工具，旨在成为一个集群部署的基础工具，其他集群部署工具可以集成 kubeadm，以完成更加复杂、更加定制化的集群部署。

常用的 kubeadm 命令如下：

- kubeadm init：初始的 Kubernetes 控制面节点。
- kubeadm join：将控制面组件或工作节点添加到已有的 Kubernetes 集群中。
- kubeadm upgrade：升级 Kubernetes 到指定的版本。
- kubeadm reset：撤销 kubeadm init 或 kubeadm join 对机器做的所有更改。

### cluster-api

[cluster-api](https://github.com/kubernetes-sigs/cluster-api)（CAPI） 是一个 Kubernetes 项目，使用声明式 API 的方式来创建、配置和管理 Kubernetes 集群。cluster-api 是更为高阶的集群部署方式，其复杂度也是最高的，通常只有具有很强研发能力的团队才会选择这种方式部署。

现在很多公有云厂商，例如火山引擎、腾讯云、阿里云等都逐渐将集群部署方式迁移为 cluster-api 的部署方式。火山引擎当前已经是使用 cluster-api 来部署，腾讯云的原生节点池节点添加机制跟 cluster-api 很像。

以下是 cluster-api 的一些主要特点和功能：

- **声明式 API：**cluster-api 提供了一组自定义资源（Custom Resources）和控制器，允许用户通过声明式 API 对集群进行管理。用户可以定义集群的规格（Spec）和状态（Status），控制器负责根据规格来实现状态。
- **可扩展性：**cluster-api 是一个可扩展的框架，可以轻松地集成到现有的 Kubernetes 生态系统中。用户可以编写自定义的 Provider 来支持不同的基础设施平台，如 AWS、Azure、GCP 等。
- **多集群管理：**cluster-api 支持多集群管理，允许用户在一个 Kubernetes 集群中管理多个集群实例。这使得跨多个环境和地理位置的集群管理变得更加简单和统一。
- **版本控制：**cluster-api 提供了版本控制机制，可以确保集群配置的一致性和可追踪性。用户可以轻松地管理和追踪集群配置的变更历史。
- **自动化操作：**cluster-api 提供了自动化的集群生命周期管理功能，包括集群的创建、扩展、缩减和删除等操作。用户可以通过简单的 API 调用来执行这些操作。

### 云服务商

如果你对费用不敏感，只想要一个生产级可用的 Kubernetes 集群，并且弱化 Kubernetes 集群或者完全避免 Kubernetes 集群的运维，那么你可以直接向公有云服务商，例如火山引擎、腾讯云、阿里云等购买一个 Kubernetes 集群。

这些云服务商提供了一键购买 Kubernetes 集群的方法，当然你要支付一定的费用。通常你可以在云服务商购买 4 种 Kubernetes 集群，这些集群分别适用于不同的场景。

**第一种是独立集群。**云服务商基于物理机/虚拟机创建一个 Kubernetes 集群，管控面节点和工作节点都交付给用户，用户可以独立控制 Kubernetes 集群，包括但不限于节点登录、Kubernetes 组件升级、参数修改等。

在这种集群提供模式中，因为 Kubernetes 集群控制面归属于用户，所以用户虽然省掉了集群管理（增删改查）相关流程的开发和运维，但是后期集群还需要用户去运维。因此，对于一些对集群有定制化要求的客户，公有云厂商还是需要以某种形式提供独立集群，但厂商更倾向于提供**托管集群。**

**托管集群是，**云服务商创建一个 Kubernetes 集群，工作节点归属于用户，用户可以登录节点、删除文件、变更 Kubernetes 组件版本等（当然不建议这么做，应该也不会有用户擅自更新 kubelet 版本）。管控面节点托管在公有云服务商。因为管控面托管在公有云服务商，所以用户不仅省掉了集群管理相关流程的开发和运维，还省掉了后期的管控面组件运维。托管集群中，工作节点归属于用户，工作节点也需要用户去参与运维。

**第三种是Serverless 集群。**云服务商创建一个完全 Serverless 化的 Kubernetes 集群， 管控节点和工作节点都托管在云厂商，其创建和后期的运维都不需要运维，真正解放用户的双手，使用户只需要关注自己的业务开发即可。Serverless 集群也是未来集群模式演进的一个方向。

**第四种是边缘集群。**云服务商创建一个边缘集群，支持添加边缘节点，把数据中心 Kubernetes 集群能力快速延伸到边缘地域，并使用云原生方式进行资源管理和应用生命周期管理；同时具备独创的多地域边缘自治、流量闭环以及分布式健康状态检查能力。

云服务商在创建上述 4 类集群类型时，会根据集群类型选择不同的集群创建方式，我们一一来说：

- **独立集群：**可以通过 kubeadm、二进制部署方式等方式来创建。
- **托管集群：**托管集群通常采用 Kubernetes In Kubernetes 的方式来创建，这种情况下可以使用 client-go 在管控 Kubernetes 集群中创建一个托管集群，也可以使用 cluster-api 的方式来在管控 Kubernetes 集群中创建一个托管集群。当前越来越多的团队，采用 cluster-api 的方式来创建托管集群。
- **Serverless 集群：**Serverless 基于托管集群进一步将节点云化，Serverless 集群的创建方式，很多时候跟托管集群的创建方式保持一致。
- **边缘集群：**边缘集群本质上也是一个托管集群，其创建方式很多时候跟托管集群保持一致。

个别云厂商还提供了其他的集群形态，例如注册集群、分布式集群等，但其部署形态基本都属于以上 4 种集群部署方式中的某一种。

## 如何选择合适的部署方式？

社区提供了这么多部署 Kubernetes 集群的方式，那么到底该如何选择呢？选择何种方式搭建 Kubernetes 集群，很多时候取决于你使用 Kubernetes 集群的场景。这里给你介绍下我的选择方式。

### Kubernetes 源码开发

当你作为一个 Kubernetes 开发者，需要修改 Kubernetes 源码来添加新的功能或者修复已知 Bug的时候，那么你可能需要频繁修改 Kubernetes 代码，编译 Kubernetes 组件并部署。这时候，你需要一种快捷的方式，来直接更新修改过的 Kubernetes 二进制文件。在开发测试场景中，这种方式最好是轻量的，因为如果能够快速更新和启动 Kubernetes 组件，在一个高频的场景下会极大提高你的开发效率。这时候，最好的选择是使用 Kubernetes 源码仓库下提供的hack/local-up-cluster.sh脚本。

另外，使用 hack/local-up-cluster.sh脚本来创建并测试 Kubernetes 集群还有一个好处，就是当代码合并到 Kubernetes 仓库时，至少通过了该脚本的测试。

当然，如果你不介意初次部署 Kubernetes 集群的复杂操作，也可以花一点时间使用 kubeadm 或二进制部署的方式搭建一个 Kubernetes 集群，专门用作长期的 Kubernetes 开发测试，之后每次更新，只更新目标组件即可。但这种方式有个弊端，就是如果 Kubernetes 集群变得不可用了，你可能要重新使用 kubeadm 或二进制部署的方式来搭建一个 Kubernetes 集群，而这个工作量不小。

### 云原生开发

如果你是一个云原生开发者，需要围绕 Kubernetes 来构建自己的应用，在开发应用的过程中，需要一个 Kubernetes 集群来部署你的应用。这时候，你需要一种可以快速搭建一个测试用的 Kubernetes 集群的方式，你可以选择 Minikube 或 Kind。因为 kind 工具更轻量，Minikube 可以创建功能强大的 Kubernetes 集群，更多的特点你可以回顾下我们刚才讲的内容。

### 部署生产级应用

如果你想在集群中部署生产环境使用的应用，那么你需要创建一个生产级可用的 Kubernetes 集群。搭建生产环境用的 Kubernetes 集群，当前用得比较多的是 kubeadm 工具。很多公有云厂商会有自己的一套 Kubernetes 集群部署系统，这些系统通常都是基于 kubeadm 工具封装的，例如可以使用 Ansiable、ArgoCD 等来封装 kubeadm。kubeadm 工具已经提供了功能强大的集群部署方式，所以，企业只需要进行简单的封装即可。

另外，一些企业或团队，期望完全掌控 Kubernetes 集群的部署方式，或者觉得 kubeadm 不能满足企业部署 Kubernetes 集群的某些需求，这些企业也可以直接使用 Kubernetes 二进制文件，并自定义启动参数来部署 Kubernetes 集群。

将集群部署在物理机或虚拟机上，可以获得更好的性能和隔离性，但是不利于运维和管理，并且资源利用率不高。所以，当前越来越多的企业选择将 Kubernetes 集群部署在 Kubernetes 集群中（也即 Kubernetes In Kubernetes）。将集群部署在 Kubernetes 中可以直接使用 client-go 调用管理集群的 kube-apiserver 接口，来创建 Kubernetes 集群（其实就是部署 kube-apiserver、kube-controller-mananger、kube-scheduler 等管控组件），也可以使用 cluster-api 来部署。

当然，如果你是初创公司，没有人力来维护一套复杂的集群管理系统，或者你愿意付费，以减小集群创建和运维成本。你也可以选择在公有云厂商付费一键购买一个功能完备的、生产可用的 Kubernetes 集群。

## 课程总结

这节课，我们讲了社区当前主流的 Kubernetes 部署方法，以及这些方法的特点及适用什么场景，如下表所示：

![](https://static001.geekbang.org/resource/image/67/89/6797e3abd29032a531383ae80a65f089.jpg?wh=1420x1258)

针对这些集群部署方式，我将选择思路汇总成了一张集群部署选择图，如下所示：

![](https://static001.geekbang.org/resource/image/d8/9d/d88b132e2187635c13fb06194736799d.jpg?wh=3118x746)

图中标注有具体的选择场景，你可以看看。

## 课后练习

1. 请思考下 Kubernetes 还有哪些集群部署方式，欢迎评论区讨论、分享。
2. 请安装部署 Minikube 工具，并使用 Minikube 工具快速创建一个测试集群。

欢迎你在留言区与我交流讨论，如果今天的内容让你有所收获，也欢迎转发给有需要的朋友，我们下节课再见！