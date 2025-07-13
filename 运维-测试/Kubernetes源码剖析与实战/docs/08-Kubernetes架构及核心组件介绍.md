你好，我是孔令飞。

在学习 Kubernetes 源码之前，我们需要系统地了解其架构和组件功能。在[第03讲](https://time.geekbang.org/column/article/871131)中，我曾简单介绍过相关内容，而在本节课中，则会对 Kubernetes 的架构和核心组件进行更详细的讲解。

## **Kubernetes 简介**

Kubernetes 是一个可移植、可扩展的开源容器编排平台，用于管理容器化的工作负载和服务，支持声明式配置和自动化。是谷歌保密了十几年的秘密武器 Borg 的开源版本，谷歌一直通过 Borg 系统管理着数量庞大的应用程序集群。由于 Kubernetes 是基于容器技术的分布式架构方案，所以不局限于任何编程语言。

Kubernetes 具有以下核心功能：

![](https://static001.geekbang.org/resource/image/a4/79/a456a6732b5dccb0840f8f4c45bcb679.jpg?wh=1004x910)

## Kubernetes 架构介绍

上面，我介绍了 Kubernetes 及其功能。本小节，我来详细介绍下 Kubernetes 的架构，其架构图如下：

![图片](https://static001.geekbang.org/resource/image/9b/a5/9b15fde858a53eb1c02e8ee4b49463a5.jpg?wh=1920x842)

### Kubernetes 中两大类节点

作为分布式架构的王者，Kubernetes 采用了 Master-Worker 的架构模式。Master 节点即上图中的 Control Plane Node，Worker 节点即上图中的 Worker Node。

- **Master 节点**

Master 节点部署了 Kubernetes 控制面的核心组件。企业 Kubernetes 集群中，Master 节点会部署 kube-apiserver、kube-controller-manager、kube-scheduler 组件，其中 kube-controller-mananger、kube-scheduler 会通过本地回环接口同 kube-apiserver 通信。kube-controller-manager 和 kube-scheduler 之间没有通信。这些核心的控制面组件用来完成 Kubernetes 资源的 CURD、并根据资源的定义执行相应的业务逻辑，例如：创建 Pod、将 Pod 调度到 Worker 节点等。

Kubernetes 中的内置资源，通过 kube-apiserver 进行 CURD 操作，并将数据持久化到 etcd 中。etcd 采用集群化部署，在有些企业中，因为没有专门提供 etcd 集群的中台，也会自己部署 etcd 集群。每个控制面节点部署一个 etcd，不同控制面节点的 etcd 实例，组成一个 etcd 集群。

- **Worker 节点**

Worker 节点主要用来运行 Pod。Worker 节点部署了 Kubernetes 的 kubelet、kube-proxy 组件。kubelet 负责跟底层的容器运行时交互，用来管理容器的生命周期。kube-proxy，作为 Kubernetes 集群内的服务注册中心，负责服务发现和负载均衡。Kubernetes 支持不同的容器运行时，例如：containerd、cri-o 等。当前用的最多的是 containerd。

因为 Master 节点部署了 Kubernetes 的核心控制面组件，为了确保控制面组件的稳定性，Master 节点不会运行 workload。

上面我介绍了 Master 节点和 Worker 节点的工作职责及上面部署的组件。接下来，我就来详细介绍上述架构图中的组件功能及交互方式。

### Kubernetes 中的组件交互

本小节，我会先简单介绍下每个组件的功能，再重点讲解不同组件之间的功能交互。每个组件的详细功能，将会在后面具体展开。

在 Kubernetes 架构图中，我们可以看到，开发者**会使用 client-go 或 kubectl 来访问 kube-apiserver。**

**kube-apiserver** 是非常核心的组件，承载着所有的资源增删改查、认证、鉴权等逻辑，其稳定性非常重要。为了确保 kube-apiserver 的高可用，企业会将 kube-apiserver 实例注册到负载均衡器中，通过负载均衡器来访问。通常建议的 kube-apiserver 副本数至少是 3 个。

**client-go** 其实是一个 Go 语言实现的 SDK，封装了访问 kube-apiserver API 接口的方法，可以使开发者方便的在程序中调用，以访问 kube-apiserver 提供的各种 API 接口。

**kubectl** 是命令行工具，运维人员可以通过 kubectl 来访问 kube-apiserver。kubectl 除了可以很方便的在命令行中直接使用之外，还可以集成在脚本中，使得开发或者运维可以很便捷的编写脚本来自动运维 kubernetes 集群。kubectl 工具，其实也是使用 client-go 来访问 kube-apiserver 的。

所以，client-go、kubectl、kube-apiserver 的关系如下：

![](https://static001.geekbang.org/resource/image/ef/f8/ef0dbf4a23c26f46ff2f2a8381f16bf8.jpg?wh=992x564)

在请求到达 kube-apiserver 后，kube-apiserver 会对请求进行身份认证和资源鉴权，在认证和鉴权通过后，请求还会经过准入控制、设置默认值、校验、版本转换等逻辑，并最终将数据保存在 etcd 中。kube-apiserver 是一个标准的 REST 服务器，内置了很多 REST 资源。kube-apiserver 也支持自定义资源，也即 CRD，通过 CRD 可以极大地提高 kube-apiserver 的扩展能力。访问 kube-apiserver，其实就是对内置资源和自定义资源进行增删改查、Patch、Watch 等操作，并将数据保存或更新在 etcd 中，或者从 etcd 中删除。

而 kube-controller-manager、kube-scheduler、kubelet、kube-proxy 这些组件，则通过 List-Watch 机制，感知 kube-apiserver 中资源的变化。当资源被执行着增删改操作之后，会产生对应的变更事件。上述四个组件，会及时地 Watch 到资源的变更，并根据资源的变更类型和变更内容执行相应的逻辑处理。

- **kube-controller-mananger：**kube-controller-manager 中内置了很多资源的 controller，这些 controller watch 到关注的资源发生变更后，会进行状态调和，根据资源的定义，确保资源的状态始终维持在所声明的状态中。维持状态的逻辑，保存在各个 controller 代码中。
- **kube-scheduler：**kube-scheduler 为 Kubernetes 集群的调度器，用来调度 Pod 到具体的 Node 节点中。单个 Kubernetes 支持多大几千个节点。一个 Pod 被创建出来之后，需要调度到一个具体的节点上运行，那么该如何决定调度到哪个节点上呢？这就是 kube-scheduler 的工作。kube-scheduler 会根据 Pod 的定义、节点的状态等因素，根据调度策略来将 Pod 调度到具体的节点上。
- **kubelet：**在 kube-scheduler 将 Pod 调度到具体的节点之后，会产生一个 Pod UPDATE 事件，kubelet watch Pod 的变更事件后，如果发现该 Pod 的 spec.nodeName 值跟 kubelet 所在的节点名相同，就会根据 Pod 的定义，调用底层的容器运行时，在节点上创建需要的容器，从而将 Pod 运行起来。
- **kube-proxy：**kube-proxy 是 Kubernetes 集群的服务发现组件和负载均衡器。kube-proxy 会 Watch 集群的 Service 和 Pod 资源，并动态维护一个 Service IP（VIP）和 Pod IP（RS） 列表的映射，在 Service 和 Pod 有更新时，会动态的更新这个映射表。在我们通过 Service IP 访问时，kube-proxy 会更具负载均衡策略，选择一个 Pod IP，将请求转发到这个 Pod 中，起到一个负载均衡器的功能。

从上面的介绍，我们可以知道 kube-apiserver 是 Kubernetes 集群中各组件数据交互和通信的枢纽、kube-controller-manager 是资源的控制中心、kube-scheduler 是资源的调度器、kubelet 是节点上 Kubernetes 的代理器、kube-proxy 负责 Pod 的网络访问。

以上就是每个组件功能的详细介绍，这里我想用简短的语句，给你总结下：

- **kube-apiserver：**Web 服务器，提供 RESTful API 接口，完成对内置资源和 CRD 资源的增删改查、Watch、Patch 等操作，并将数据保存/更新到 etcd 中，或者从 etcd 中删除。
- **kube-controller-manager：**Kubernetes 资源的调和器，会 Watch kube-apiserver，在 Watch 到资源变更事件后，会根据事件的类型和内容，对资源进行调和，使资源处于预期的状态。
- **kube-scheduler：**Kubernetes 集群调度器，用来将 Pod 调度到合适的 Worker Node 上。
- **kubelet：**负责所在节点上，Pod 的生命周期管理。
- **kube-proxy：**Kubernetes 集群的服务发现中心和负载均衡器，可以根据服务 IP 和负载均衡策略，访问 RS 列表中具体的某一个 RS（也即 Pod）。

## Kubernetes 组件功能介绍：控制面组件

Kubernetes 控制面是 Kubernetes 集群的核心部分，负责管理和维护整个集群的状态。它由多个组件组成，每个组件都有特定的功能。以下是 Kubernetes 控制面的核心组件：

- kube-apiserver
- kube-controller-manager
- kube-scheduler
- cloud-controller-manager
- etcd

### kube-apiserver

kube-apiserver 提供了 Kubernetes 资源对象的唯一操作入口，其他所有的组件都必须通过它提供的 API 接口来操作资源数据。只有 kube-apiserver 会与 etcd 通信，其他模块都必须通过 kube-apiserver 访问 etcd。kube-apiserver 作为 Kubernetes 系统的入口，封装了核心对象的增删改查等操作。kube-apiserver 提供 RESTful API 给外部客户端和内部组件调用。它还提供了认证、授权和准入控制等安全机制。kube-apiserver 设计上考虑了水平伸缩，也就是说，它可通过部署多个实例进行伸缩。你可以运行 kube-apiserver 的多个实例，并在这些实例之间平衡流量。

kube-apiserver 主要负责以下工作:

- **API 服务：**提供 Kubernetes API 接口，供用户和其他组件（如 Kubelet、Kube-controller-manager 等）进行交互。
- **资源管理：**负责创建、更新、删除和列出集群中的资源（如 Pods、Services、Deployments 等）。
- **认证和授权：**
  
  - 处理用户身份验证，支持多种认证方法（如基本认证、Bearer Tokens、Webhook 等）；
  - 检查用户是否有权限执行特定操作（即授权）。支持 ABAC、RBAC、Webhook 等授权策略。
- **审计日志：**记录所有 API 请求和操作的审计日志，可以用于安全审计和问题排查。
- **API 版本管理：**
  
  - 支持不同版本的 API，确保向后兼容性和逐步迁移；
  - 提供不同的版本（如 v1、v1beta等），允许客户端选择适当的 API 版本。
- **数据存储：**
  
  - 与 etcd 交互，负责持久化存储集群状态和配置数据；
  - 处理所有对 etcd 的读写请求，确保数据一致性和高可用性。
- **对象的 Watch 机制：**
  
  - 支持客户端订阅对象变化，实现实时更新通知（Watch）；
  - 允许系统组件基于资源变化进行响应。
- **API 速率限制：**实现请求速率限制，以确保 API Server 的稳定性和性能，防止过载。

### kube-controller-manager

kube-controller-manager 是一组控制器的集合，用于监控和管理集群中的各种资源。它包括节点控制器、副本控制器、服务控制器、端点控制器等。这些控制器负责监视资源的状态变化，并根据需要采取相应的操作，确保集群中的资源处于期望的状态。

控制器是运行无限控制循环的程序。这意味着它会持续运行并观察对象的实际和期望状态。如果实际状态和期望状态存在差异，控制器会进行操作确保 Kubernetes 资源/对象处于期望状态。

假设你想要创建一个 Deployment，你可以在 YAML 中指定它期望的状态，比如，2 个副本、1 个卷挂载、configmap 等。内置的 controller 将确保 Deployment 始终处于期望的状态。如果用户将副本数更改为 5，Controller 将感知这个变化，并保证 Deployment 的副本的个数变成 5 个。

kube-controller-mananger 内置了很多控制器，这些控制器彼此独立工作，在启动时，也可以通过 `--controllers` 命令行选项，来控制开启/禁用哪些 controller。v1.30.3 版本的 kube-controller-manager 内置了 41 个控制器，你可以在 [controller\_names.go](https://github.com/kubernetes/kubernetes/blob/v1.30.4/cmd/kube-controller-manager/names/controller_names.go#L44) 文件中找到内置的控制器。例如（controller 太多了，功能描述就用 GPT 来生成了，望理解）：

![图片](https://static001.geekbang.org/resource/image/57/cc/5758cfca790cda81aa453a78afd0fbcc.jpg?wh=1700x2400)

### cloud-controller-mananger

当在云环境中部署 Kubernetes 时，cloud-controller-mananger充当云平台 API 接口和 Kubernetes 集群之间的桥梁。Kubernetes 核心组件可以独立工作，也允许通过插件方式与云提供商集成。(例如，Kubernetes 集群和 AWS 云 API 之间的接口)

![图片](https://static001.geekbang.org/resource/image/de/11/de7616ed331c1a237f181ab52bd32c11.jpg?wh=800x1140 "图片来源网络")

cloud-controller-mananger 包含一组云平台控制器，用于保证 cloud 组件（如：nodes，Loadbalancers，storage）的状态符合预期。cloud-controller-mananger 的三个主要控制器为：

- **Node controller：**该控制器通过与云提供商API通信，更新节点相关信息。例如，节点的 Label、Annotation、主机名、CPU和内存、节点运行状况等。
- **Route controller：**负责在云平台上配置网络路由。这样不同节点上的 Pod 就可以互相通信了。
- **Service controller：**负责为Kubernetes服务部署负载均衡器，分配 IP 地址等。

cloud-controller-mananger 的经典使用场景如下：

- **部署负载均衡器类型的 Kubernetes 服务。**在这里，Kubernetes 提供了一个特定于云的负载均衡器，并与 Kubernetes Service 集成。
- **通过云存储解决方案为 Pod 提供存储卷（PV）。**

### kube-sheduler

kube-scheduler 负责将新创建的 Pod 调度到集群中的节点上。它负责监视新创建的、未指定运行节点（node）的 Pods，选择节点让 Pod 在上面运行。调度决策考虑的因素包括单个 Pod 和 Pod 集合的资源需求、硬件/软件/策略约束、亲和性和反亲和性规范、数据位置、工作负载间的干扰和最后时限。

下图显示了调度器工作原理的主要流程：

![图片](https://static001.geekbang.org/resource/image/2d/b5/2d7d99d2f104827bf307f91869bdebb5.jpg?wh=1267x1189 "图片来源网络")

### 非 Kubernetes 组件：etcd

etcd 是 Kubernetes 的分布式键值存储系统，用于存储集群的配置数据和状态信息。它提供高可用性和一致性，并用于存储集群中的各种信息，如节点信息、Pod 信息、服务信息、配置信息等。etcd 是控制平面中唯一的有状态组件。

![图片](https://static001.geekbang.org/resource/image/75/19/7509b3ed53f6e8aaf18bfa9b3a8d1719.jpg?wh=1024x948 "图片来源网络")

etcd 是一个开源的强一致性分布式键值存储：

- **强一致性：**如果对一个节点进行了更新，强一致性将确保它立即更新到集群中的所有其他节点。在 CAP 定理的限制下，在强一致性和分区容忍的情况下，实现 100% 可用性是不可能的。
- **分布式：**etcd 被设计成在保留强一致性的前提下，作为一个集群在多个节点上运行。
- **键值存储：**将数据存储为键和值的非关系数据库。它还公开了一个键值 API。数据存储构建在 BoltDB 之上（BoltDB 是 BoltDB 的一个分支）。

etcd 采用 Raft 共识算法，具有较强的一致性和可用性。它以领导者-成员方式工作，以实现高可用性并承受节点故障。

etcd 在 Kubernetes 集群中的作用包括：

- **etcd 存储 Kubernetes 对象的所有配置、状态和元数据。**包括：Pods、Secrets、Daemonsets、Deployment、Configmaps、Statfulsets 等对象。
- **etcd 允许客户端使用 Watch() API 订阅事件。**kube-apiserver 使用 etcd 的 Watch 功能来跟踪对象状态的变化。
- **etcd 使用 gRPC 公开键值 API。**此外，gRPC 网关作为 RESTful 代理，将所有 HTTP API 调用转换为 gRPC消息。这使得它成为 Kubernetes 的理想数据库。
- **etcd 以键值方式，将所有对象存储在 `/registry` 目录下。** 例如，一个在 `default` 命名空间中，名字为 `nginx` 的 Pod，可以在 `/registry/pods/default/nginx` 下找到。

## Kubernetes 组件功能介绍：数据面组件

Kubernetes 数据面是指与集群内部运行的 Pods 及其网络、存储和其他服务直接相关的部分。与控制面负责状态管理、调度和 API 请求不同，数据面的主要关注点是处理应用程序的实际运行和数据流。以下是数据面的核心组件：

- kubelet
- kube-proxy
- container runtime

### kubelet

kubelet 是运行在每个节点上的代理程序，负责管理和监控节点上的容器。它与 kube-apiserver 通信，接收来自控制平面的指令，并根据指令创建、启动、停止和销毁容器。它还负责监控容器的状态和健康状况，并上报给控制平面。

kubelet 是负责容器真正运行的核心组件，主要职责如下所示：

- 负责 Node 节点上 Pod 的创建、修改、监控、删除等全生命周期的管理。
- 定时上报本地 Node 的状态信息给 kube-apiserver。
- kubelet 是 Master 和 Node 之间的桥梁，接收 kube-apiserver 分配给它的任务并执行。
- kubelet 通过 kube-apiserver 间接与 etcd 集群交互来读取集群配置信息。
- kubelet 在 Node 上做的主要工作具体如下:
  
  - 设置容器的环境变量、给容器绑定 Volume、给容器绑定 Port；
  - 为 Pod 创建，更新，删除容器；
  - 负责处理存活（Liveness）、就绪（Readiness）和启动（Startup）探针；
  - 通过读取 Pod 配置，在主机上为卷挂载创建相应的目录来挂载卷。

Kubelet 除了能够接收来自 kube-apiserver 的 Pod 资源定义外，还可以接受来自文件、HTTP 端口和 HTTP 服务器的 Pod 定义。Kubernetes 的静态 Pod，就是是“从文件中获取 Pod 定义”的一个很好的例子。（静态 Pod 由 kubelet 控制，而不是 kube-apiserver）

### kube-proxy

kube-proxy 负责为 Pod 提供网络代理和负载均衡功能。它维护集群中的网络规则和转发表，并将请求转发到合适的目标 Pod 上。它支持多种代理模式，如用户空间代理、iptables 代理和 IPVS 代理，以满足不同网络环境的需求。

kube-proxy 是集群中每个节点（Node）上所运行的网络代理， 实现 Kubernetes 服务（Service） 概念的一部分。

kube-proxy 是一个 Kubernetes Daemonset，在每个节点上运行。它是实现 Kubernetes Services 概念的代理组件（为每组 Pod 提供具有负载均衡的 DNS）。它主要代理 UDP、TCP 和 SCTP（不能直接代理 HTTP）。

当你使用Service（ClusterIP）公开 Pod 时，kube-proxy 将创建网络规则，将流量发送到分组在 Service 对象下的后端 Pod。也就是说，所有的负载平衡和服务发现都由 kube-rproxy 处理。

![图片](https://static001.geekbang.org/resource/image/a8/e9/a89b798b653a7fbfd462266164435ce9.jpg?wh=800x1095 "图片来源网络")

### 非 Kubernetes 组件：Container Runtime

容器运行时是负责在节点上创建和管理容器的软件。Kubernetes 支持多种容器运行时，如 Docker、containerd、CRI-O 等。容器运行时负责拉取镜像、启动容器、管理容器的生命周期，并提供容器的隔离和资源管理。

Kubernetes 要求容器运行时必须实现 CRI（Container Runtime Interface 容器运行时接口）。CRI 是一个插件接口，它使 kubelet 能够使用各种容器运行时，无需重新编译集群组件，是 kubelet 和容器运行时之间通信的主要协议。

常见的容器运行时有：

- **containerd：**官网推荐的容器运行时，是从 Docker 分离出来的。
- **CRI-O：**是 containerd 的一个替代品，专门为 Kubernetes 创建的一个容器运行时。
- **Docker Engine：**需要先调用 dockershim，然后调用 Docker，再调用 containerd，最后调用底层的 runc。

CRI-O、Containerd、Docker 三者区别见下图：

![图片](https://static001.geekbang.org/resource/image/fa/16/faee11a18c89f457e67d2c996b0def16.jpg?wh=1267x802 "图片来源网络")

以CRI-O为例，容器运行时与 kubernetes 的交互见下图：

![图片](https://static001.geekbang.org/resource/image/9b/40/9b8aedc1fa32c0a2d61ec65a8e4b2340.jpg?wh=1244x853 "图片来源网络")

- 当 kube-apiserver 对 Pod 发出新的请求时，kubelet 与 CRI-O 守护进程交互，通过 Kubernetes 容器运行时接口启动所需的容器。
- CRI-O 使用 containers/image 库，根据配置的容器信息，检查并 Pull 镜像。
- CRI-O 为容器生成 OCI 运行时规范（OCI specification ，JSON格式）。
- CRI-O 启动一个 OCI 兼容的运行时（runc），按照运行时规范启动容器进程。

## 客户端组件：kubectl

kubectl 是 Kubernetes 的命令行工具，它是 Kubernetes 的官方命令行客户端。它允许用户通过命令行与 Kubernetes 集群进行交互，并执行各种操作，包括管理集群中的资源对象、配置集群、故障排查和日志查看等。

kubectl 提供了与 Kubernetes API 进行通信的功能，它可以向 Kubernetes 集群发送命令和请求，然后接收和解析 API 响应，并将结果返回给用户。通过 kubectl，用户可以通过简单的命令操作来管理和控制 Kubernetes 集群，而无需直接与底层的 API 进行交互。

kubectl 支持丰富的命令和选项，可以用于创建、查看、更新和删除 Kubernetes 集群中的各种资源对象，如Pod、Service、Deployment等。它还提供了许多功能，如集群管理、配置管理、故障排查和日志查看等，使得用户可以轻松地管理和操作 Kubernetes 集群。

## 课程总结

本节课详细介绍了 Kubernets 的功能、架构及 Kubernetes 中各个核心组件的功能。最后为了让你对 Kubernetes 的架构有一个更加直观的了解，本节课后半部分，我们还通过 Pod 的创建流程，展示了 Kubernetes 中各个组件的功能和交互。

## 课后练习

- 请回忆一下，Kubernetes 集群中有哪些组件，以及每个组件的功能。
- 请思考下，Kubernetes 具体是如何创建 Pod 的？

欢迎你在留言区与我交流讨论，如果今天的内容让你有所收获，也欢迎转发给有需要的朋友，我们下节课再见！