---
type: docs
title: "背景"
linkTitle: "背景"
weight: 1
---
Dubbo3 的设计与开发有两个大的背景。

**首先，如何更好的满足企业实践诉求。** Dubbo 自 2011 由阿里巴巴捐献开源以来，一直是众多大型企业微服务实践的首选开源服务框架。在此期间，企业架构经历了从 SOA 架构到微服务架构变迁，Dubbo 社区自身也在不断的更新迭代以更好的满足企业诉求。然而 Dubbo2 架构上的局限逐渐在实践中凸显：1.协议，Dubbo2 协议以性能、简洁著称，但却在云原生时代遇到越来越多的通用性、穿透性问题；2.可伸缩性，Dubbo2 在可伸缩性上依旧远超很多其他框架，但随着微服务带来更多应用与实例我们不得不思考如何应对更大规模集群的实战；3.服务治理易用性，如更丰富的流量治理、可观测性、智能负载均衡等。

**其次，适配云原生技术栈的发展。** 微服务让业务开发演进更灵活、快捷的同时，也带来了一些它独有的特征和需求：如微服务之后组件数量越来越多，如何解决各个组件的稳定性，如何快速的水平扩容等，以 Docker、Kubernetes、Service Mesh 为代表的云原生基础设施为解决这些问题带来了一些新的选择。随着更多的微服务组件及能力正下沉到以 Kubernetes 为代表的基础设施层，传统微服务开发框架应剔除一些冗余机制，积极的适配到基础设施层以做到能力复用，微服务框架生命周期、服务治理等能力应更好地与 Kubernetes 服务编排机制融合； 以 Service Mesh 为代表微服务架构给微服务开发带来的新的选择，Sidecar 给多语言、透明升级、流量管控等带来的优势，但同时也带来运维复杂性、性能损耗等弊端，因此基于服务框架的传统微服务体系还将是主流，长期仍将占据半壁江山，在长时间内将会维持混合部署将会维持混合部署状态。

### 总体目标
Dubbo3 依旧保持了 2.x 的经典架构，以解决微服务进程间通信为主要职责，通过丰富的服务治理（如地址发现、流量管理等）能力来更好的管控微服务集群；Dubbo3 对原有框架的升级是全面的，体现在核心 Dubbo 特性的几乎每个环节，通过升级实现了稳定性、性能、伸缩性、易用性的全面提升。

![architecture-1](/imgs/v3/concepts/architecture-1.png)

* **通用的通信协议。** 全新的 RPC 协议应摒弃私有协议栈，以更通用的 HTTP/2 协议为传输层载体，借助 HTTP 协议的标准化特性，解决流量通用性、穿透性等问题，让协议能更好的应对前后端对接、网关代理等场景；支持 Stream 通信模式，满足不同业务通信模型诉求的同时给集群带来更大的吞吐量。
* **面向百万集群实例，集群高度可伸缩。** 随着微服务实践的推广，微服务集群实例的规模也在不停的扩展，这得益于微服务轻量化、易于水平扩容的特性，同时也给整个集群容量带来了负担，尤其是一些中心化的服务治理组件；Dubbo3 需要解决实例规模扩展带来的种种资源瓶颈问题，实现真正的无限水平扩容。
* **更丰富的编程模型，更小的业务侵入。** 在开发态业务应用面向 Dubbo SDK 编程，在运行态 SDK 与业务应用运行在同一个进程，SDK 的易用性、稳定性与资源消耗将在很大程度上影响业务应用；因此 3.0 应该具备更抽象的 API、更友好的配置模式、更少的侵占业务应用资源、具备更高的可用性。
* **更易用、更丰富的服务治理能力。** 微服务的动态特性给治理工作带来了很高的复杂性，而 Dubbo 这方面一直做的不错，是最早的一批治理能力定义者与实践者；3.0 需面向更丰富的场景化，提供诸如可观测性、安全性、灰度发布、错误注入、外部化配置、统一的治理规则等能力。
* **全面拥抱云原生。**

### 面向企业生产实践痛点
Dubbo2 仍旧是国内首选开源服务框架，被广泛应用在互联网、金融保险、软件企业、传统企业等几乎所有数字化转型企业中，久经规模化生产环境检验。以 Dubbo2 的贡献者和典型用户阿里巴巴为例，阿里巴巴基于 Dubbo2 在内部维护的 HSF2 框架经历了历次双十一峰值考验，每天数十亿次的 RPC 调用，治理着超过千万的服务实例。在长期的优化和实践积累中，阿里巴巴有了对下一代服务框架的设想与方案，在内部开始了快速演进，并快速的被贡献到 Apache 社区，如同阿里巴巴一样，其他用户的实践诉求与痛点也在开源社区快速的积累，形成了一致的方向和技术方案，可以说 Dubbo3 的诞生就来自于超大基数的企业用户积累，为了更好的满足他们的实践诉求。

![dubbo3-hsf](/imgs/v3/concepts/dubbo-hsf.png)

Dubbo3 融合了阿里巴巴 HSF2 及其他社区企业的大量服务治理经验，当前 Dubbo3 已经被全面应用到生产实践环境，用户包括阿里巴巴电商、饿了么、钉钉、考拉、阿里云、小米、工商银行、风火递、平安健康等。社区与用户的合作形成的良性循环极大的促进了 Dubbo3 的发展，阿里巴巴已经以社区版 Dubbo3 完全取代了内部维护的 HSF2 框架，他们的实践经验一方面推动 Dubbo3 的稳定性，另一方面正够源源不断的将服务治理实践经验输出到开源社区。

### 面向百万集群实例，横向可扩容
随着微服务实践经验的积累，微服务被拆分成更细粒度，部署到越来越多的机器实例，以支撑不断增长的业务规模。在众多的 Dubbo2 企业用户中，尤其是以金融保险、互联网为代表的规模化企业开始遇到集群容量瓶颈问题（典型的请参照[工商银行实践案例](/zh/users/icbc/)）：
* 服务发现过程
    * 注册中心数据存储规模达到容量瓶颈
    * 数据注册&推送效率严重下降
* Dubbo 进程
    * 侵占更多机器资源，导致业务资源利用率降低
    * 频繁 GC 影响业务稳定性

Dubbo3 在设计上很好的解决了这些问题，通过全新设计实现的服务治理（服务发现）模型，可以实现服务发现链路上的数据传输、数据存储量平均下降 90% 左右；同时 Dubbo3 自身在业务进程中变得更轻量、更稳定，实现提升资源利用率 50%。

Dubbo3 一个更大的优势在于其对整体架构稳定性的提升，新的服务发现架构使得对于整个集群容量、可伸缩性评估变得更容易、更准确。

![capacity](/imgs/v3/concepts/capacity.png)

如果将应用开发粗略划分为业务开发、运维部署两个层次，其中变化比较频繁的因素包括服务（接口）、应用、机器实例。在 2.x 时代，所有这三个因素的增长都会影响微服务集群的总体容量，尤其是接口增减带来的波动，对整体容量评估是非常不透明的。而在 3.0 中集群容量变化仅与应用名、机器实例两个因素相关，而我们容量评估的对象往往都是应用与实例，因此整个集群集群变的更稳定透明。

### 云原生
在云原生时代，底层基础设施的变革正深刻影响应用的部署、运维甚至开发过程，往上也影响了 Dubbo3 微服务技术方案的选型与部署模式。

#### 下一待 RPC 协议
新一代的 Triple 协议基于 HTTP/2 作为传输层，具备更好的网关、代理穿透性，原生支持 Stream 通信语义，兼容 gRPC 协议。

#### 多语言友好
Dubbo3 从服务定义、RPC 协议、序列化、服务治理等多个方面都已经将多语言友好性作为重点考量因素，目前提供了 Java、Golang 稳定的多语言版本，更多语言版本的 3.0 实现如 Rust、Javascript、C/C++、C# 等在开发建设中。

#### Kubernetes
Dubbo3 开发的应用可以原生部署到 Kubernetes 平台，Dubbo3 在地址、生命周期等已设计可与 Kubernetes 等容器调度平台对齐；对于要进一步复用 Kubernetes 底层基础设施能力的用户来说，Dubbo3 也已对接到了原生的 Kubernetes Service 体系。

#### Service Mesh
Service Mesh 强调控制面在微服务治理中的作用，在一定程度上推动了控制面通信协议、职责范围的扩展与标准化；传统 Mesh 架构下的 Sidecar 模型强调旁路代理对于流量的统一管控，以实现透明升级、多语言无感、无业务侵入等特性。

Dubbo3 提供了基于自身思考的 Dubbo Mesh 解决方案，强调了控制面对微服务集群的统一管控，而在部署架构上，同时支持 sicecar 与无 sidecar 的 proxyless 部署架构，使用 Dubbo Mesh 的用户基于自身的业务特点将有更多的部署架构选择。

#### 异构体系互通
我们正看到越来越多的异构微服务体系互通的诉求，典型如 Dubbo、Spring Cloud、gRPC 等。有些是因为技术栈迁移，有些是组织合并后需要实现业务互调，Dubbo3 借助于新的服务发现模型以及可灵活扩展的 RPC 协议，可以成为
