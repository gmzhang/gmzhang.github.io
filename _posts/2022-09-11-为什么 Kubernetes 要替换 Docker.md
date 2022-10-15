---
layout: post
title: 为什么 Kubernetes 要替换 Docker
---

![](/public/images/k8s_docker/00.jpg)

Kubernetes 是今天容器编排领域的事实标准，而 Docker 从诞生之日到今天都在容器中扮演着举足轻重的地位，也都是 Kubernetes 中的默认容器引擎。然而在 2020 年 12 月，Kubernetes 社区决定着手移除仓库中 Dockershim 相关代码[1](#fn:1)，这对于 Kubernetes 和 Docker 两个社区来说都意义重大。

![kubelet-and-containers](/public/images/k8s_docker/kubelet-and-containers-2021-03-10-16153845432597.png)

**图 1 - Dockershim**

相信大多数的开发者都听说过 Kubernetes 和 Docker，也知道我们可以使用 Kubernetes 管理 Docker 容器，但是可能没有听说过 Dockershim，即 Docker 垫片。如上图所示，Kubernetes 中的节点代理 Kubelet 为了访问 Docker 提供的服务需要先经过社区维护的 Dockershim，Dockershim 会将请求转发给管理容器的 Docker 服务。

其实从上面的架构图中，我们就能猜测出 Kubernetes 社区从代码仓库移除 Dockershim 的原因：

*   Kubernetes 引入容器运行时接口（Container Runtime Interface、CRI）隔离不同容器运行时的实现机制，容器编排系统不应该依赖于某个具体的运行时实现；
*   Docker 没有支持也不打算支持 Kubernetes 的 CRI 接口，需要 Kubernetes 社区在仓库中维护 Dockershim；

可扩展性
----

Kubernetes 通过引入新的容器运行时接口将容器管理与具体的运行时解耦，不再依赖于某个具体的运行时实现。很多开源项目在早期为了降低用户的使用成本，都会提供开箱即用的体验，而随着用户群体的扩大，为了满足更多定制化的需求、提供更强的可扩展性，会引入更多的接口。Kubernetes 通过下面的一系列接口为不同模块提供了扩展性：

![kubernetes-extensions](/public/images/k8s_docker/kubernetes-extensions-2021-03-10-16153845432607.png)

**图 2 - Kubernetes 接口和可扩展性**

Kubernetes 在较早期的版本中就引入了 CRD、CNI、CRI 和 CSI 等接口，只有用于扩展调度器的调度框架是 Kubernetes 中比较新的特性。我们在这里就不展开分析其他的接口和扩展了，简单介绍一下容器运行时接口。

Kubernetes 早在 1.3 就在代码仓库中同时支持了 rkt 和 Docker 两种运行时，但是这些代码为 Kubelet 组件的维护带来了很大的困难，不仅需要维护不同的运行时，接入新的运行时也很困难；容器运行时接口（Container Runtime Interface、CRI）是 Kubernetes 在 1.5 中引入的新接口，Kubelet 可以通过这个新接口使用各种各样的容器运行时。其实 CRI 的发布就意味着 Kubernetes 一定会将 Dockershim 的代码从仓库中移除。

CRI 是一系列用于管理容器运行时和镜像的 gRPC 接口，我们能在它的定义中找到 `RuntimeService` 和 `ImageService` 两个服务[2](#fn:2)，它们的名字很好地解释了各自的作用：

    service RuntimeService {
        rpc Version(VersionRequest) returns (VersionResponse) {}
    
        rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse) {}
        rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse) {}
        rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse) {}
        rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse) {}
        rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse) {}
    
        rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse) {}
        rpc StartContainer(StartContainerRequest) returns (StartContainerResponse) {}
        rpc StopContainer(StopContainerRequest) returns (StopContainerResponse) {}
        rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse) {}
        rpc ListContainers(ListContainersRequest) returns (ListContainersResponse) {}
        rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse) {}
        rpc UpdateContainerResources(UpdateContainerResourcesRequest) returns (UpdateContainerResourcesResponse) {}
        rpc ReopenContainerLog(ReopenContainerLogRequest) returns (ReopenContainerLogResponse) {}
    
        ...
    }
    
    service ImageService {
        rpc ListImages(ListImagesRequest) returns (ListImagesResponse) {}
        rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse) {}
        rpc PullImage(PullImageRequest) returns (PullImageResponse) {}
        rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse) {}
        rpc ImageFsInfo(ImageFsInfoRequest) returns (ImageFsInfoResponse) {}
    }
    

对 Kubernetes 稍有了解的人都能从上面的定义中找到一些熟悉的方法，它们都是容器运行时需要暴露给 Kubelet 的接口。Kubernetes 将 CRI 垫片实现成 gRPC 服务器与 Kubelet 中的客户端通信，所有的请求都会被转发给容器运行时处理。

![cri-and-container-runtimes](/public/images/k8s_docker/cri-and-container-runtimes-2021-03-10-16153845432613.png)

**图 3 - Kubernetes 和 CRI**

Kubernetes 中的声明式接口非常常见，作为声明式接口的拥趸，CRI 没有使用声明式的接口是一件听起来『非常怪异』的事情[3](#fn:3)。不过 Kubernetes 社区考虑过让容器运行时重用 Pod 资源，这样容器运行时可以实现不同的控制逻辑来管理容器，能够极大地简化 Kubelet 和容器运行时之间的接口，但是社区出于以下两点考虑，最终没有选择声明式的接口：

1.  所有的运行时都需要重新实现相同的逻辑支持很多 Pod 级别的功能和机制；
2.  Pod 的定义在 CRI 设计时演进地非常快，初始化容器等功能都需要运行时的配合；

虽然社区最终为 CRI 选择了命令式的接口，但是 Kubelet 仍然会保证 Pod 的状态会不断地向期望状态迁移。

不兼容接口
-----

与容器运行时相比，Docker 更像是一个复杂的开发者工具，它提供了从构建到运行的全套功能。开发者可以很快地上手 Docker 并在本地运行并管理一些 Docker 容器，然而在集群中运行的容器运行时往往不需要这么复杂的功能，Kubernetes 需要的只是 CRI 中定义的那些接口。

![docker-and-cri](/public/images/k8s_docker/docker-and-cri-2021-03-10-16153845432620.png)

**图 4 - Docker & CRI**

Docker 的官方文档加起来可能有一本书的厚度，相信没有任何开发者可以熟练运用 Docker 提供的全部功能。而作为开发者工具，虽然 Docker 中包含 CRI 需要的所有功能，但是都需要实现一层包装以兼容 CRI。除此之外，社区提出的很多新功能都没有办法在 Dockershim 中实现，例如 cgroups v2 以及用户命名空间。

Kubernetes 作为比较松散的开源社区，每个成员尤其是各个 SIG 的成员都只会在开源社区上花费有限的时间，而维护 Kubelet 的 sig-node 又尤其繁忙，很多新的功能都因为维护者没有足够的精力而被搁置，所以既然 Docker 社区看起来没有打算支持 Kubernetes 的 CRI 接口，维护 Dockershim 又需要花费很多精力，那么我们就能理解为什么 Kubernetes 会移除 Dockershim 了。

总结
--

今天的 Kubernetes 已经是非常成熟的项目，它的关注点也逐渐从提供更完善的功能转变到提供更好的扩展性，这样才能满足不同场景和不同公司定制化的业务需求。Kubernetes 在过去因为 Docker 的热门而选择 Docker，而在今天又因为高昂的维护成本而放弃 Docker，我们能够从这个过程中体会到容器领域的发展和进步。

移除 Docker 的种子其实从 CRI 发布时就种下了，Dockershim 一直都是 Kubernetes 为了兼容 Docker 获得市场采取的临时决定，对于今天已经统治市场的 Kubernetes 来说，Docker 的支持显得非常鸡肋，移除代码也就顺理成章了。我们在这里重新回顾一下 Kubernetes 在仓库中移除 Docker 支持的两个原因：

*   Kubernetes 在早期版本中引入 CRI 摆脱依赖某个具体的容器运行时依赖，屏蔽底层的诸多实现细节，让 Kubernetes 能够更关注容器的编排；
*   Docker 本身不兼容 CRI 接口，而且官方并没有实现 CRI 的打算，同时也不支持容器的一些新需求，所以 Dockershim 的维护成为了社区的想要摆脱负担；

原文链接：https://draveness.me/whys-the-design-kubernetes-deprecate-docker/