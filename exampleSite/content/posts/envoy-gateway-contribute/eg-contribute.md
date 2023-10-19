---
weight: 1
title: "Envoy Gateway 指南：架构设计与开源贡献"
date: 2023-03-25T21:40:32+08:00
lastmod: 2023-03-25T21:40:32+08:00
draft: false
author: "Xunzhuo"
authorLink: "https://github.com/xunzhuo"
description: "本文通过介绍 Envoy Gateway 设计与架构，提供开源贡献的指南"

tags: ["Envoy Gateway", "Open Source", "API Gateway", "Toolings"]
categories: ["Envoy Gateway"]

toc:
  auto: false
---

## 前言

Envoy Gateway 是由 EnvoyProxy 社区发起的，联合 Contour、Emissary 等基于 Envoy 的 API 网关开源项目及社区，共同参与并维护的 API 网关项目。

从项目发起到现在，已经有了很多开源贡献者参与到了其中，现在 Issues 和 PRs 总数已经达到 **1200 +**，**GitHub Stars** 已经接近 **1000**，是 EnvoyProxy 社区中发展最快，重视度很高的项目。

很多朋友也私下联系过我，向我询问如何参与到 Envoy Gateway 的贡献之中，所以本文从整体来详细梳理和讲解一下，如何参与到项目贡献之中。

本文的目的是想通过简单介绍 EG 设计与架构后，讲解贡献者如何参与开源贡献之中。

本文主要覆盖以下内容：

+ Envoy Gateway 项目角色
+ Envoy Gateway 架构介绍
+ egctl 命令行工具介绍
+ Envoy Gateway 本地开发指南
+ egctl 本地开发指南
+ GitHub Workflows 介绍
+ 一些建议

## 角色

在开始介绍如何贡献之前，先简单介绍一下 Envoy Gateway 项目的几种角色。

现在项目主要有三种角色：

+ Steering Committee
+ Maintainer
+ Reviewer

#### Steering Committee

**指导委员会成员**，指导委员会是一个由 6 名成员组成的管理机构。

它负责为 Envoy Gateway 做出方向性和战略性的决定。它还将决定任何关于其管理、营销和未来方向的变化。

指导委员会席位将每年评估一次，新的人/组织可以根据对项目的贡献或成为有价值的最终用户而被提名并投票加入指导委员会。

**目前指导委员会成员：**

+ Envoy core [Member: **Matt Klein**] @mattklein123
+ Ambassador Labs [Member: **Richard Li**]
+ Tetrate [Member: **Varun Talwar**] @zinuga
+ Tencent Holdings Limited [Member: **Xunzhuo Liu**] @Xunzhuo
+ VMware [Member: **Winnie Kwon**] @pydctw
+ Fidelity Investments [Member: **Venkat Kasisomayajula**] @skasisom6

#### Maintainer

**项目维护者**，拥有项目的 **Maintain** 权限，负责项目的开发、测试、发布、Roadmap、文档等各个方面的工作。

维护者不仅需要对 Envoy Gateway、EnvoyProxy、Gateway API 有很深的理解、同时需具备较好的代码能力，对 EG 有过多个重要的特性变更，以及需要积极参与代码 Review，社区讨论，同时具备很好的开源协作能力。

维护者由大多数维护者投票赞成通过而产生。

**目前维护者成员：**

+ @AliceProxy 
+ @arkodg 
+ @skriss 
+ @Xunzhuo 
+ @youngnick 
+ @zirain

#### Reviewer

**项目审查者**，拥有项目的 **Triage** 权限，同时是 EnvoyProxy 的 Member。负责对项目的 Issue 和 PR 进行审查，协助 Maintainer 保证代码的健康和稳定性。

Reviewer 是对 Envoy Gateway 有长期贡献的积极贡献者，参与过重大特性的开发，以及积极参与 Code Review、以及社区讨论。

Reviewer 目前这个角色是我主要在社区推动的一个新角色，目的是降低贡献者参与社区的门槛，让贡献者的贡献能得到社区阶段性的认可。

应该在最近几周就会正式宣布，参见 [PR](https://github.com/envoyproxy/gateway/pull/1156) 和 [Issue](https://github.com/envoyproxy/gateway/issues/927).

**提名的第一批 Reviewer 目前有：**

- @chauhanshubham 
- @kflynn      
- @LanceEa     
- @qicz        
- @zhaohuabing 

## Envoy Gateway 架构

首先我们需要明白 Envoy Gateway 本质上是一个**控制面**，数据面为 EnvoyProxy。它的定位是作为一个轻量级的南北向网关，**轻量化、易扩展、可观测性、中立**，是 EG 的主题。

EG 之所以能在 Contour（VMware） 以及 Emissary （Ambassador）这些网关供应商共同参与下进行，是在设计上 EG 是一款高扩展性的控制平面，能够简单的被终端用户使用，以及方便的被网关供应商所集成。

EG 是基于 Gateway API 作为其配置语言，Gateway API 相比传统 Ingress，提供了更先进的 API 设计，其面向角色的设计，更符合现在 DevOps 的理念，运维和开发能做到关注点分离：

<div align="center">
<img src="https://liuxunzhuo.oss-cn-chengdu.aliyuncs.com/blog/api-model.png" alt="api-model" />
</div>

运维只需要关注 **Gateway** 和 **GatewayClass**，而开发只需要关注各种 **Route**。

同时它提供了更清晰的权限控制模型，如通过 **ReferenceGrant** 实现跨命名空间资源访问的控制。

同时 Gateway API 相比 Ingress，提供了更明确的 API 语义，更多通用的能力被 API 统一定义，而不是像 Ingress 一样，需要通过 Annotations，这种非结构话的数据结构去定义，或者通过各种自定义 CRDs，这些大大降低了 API 的**可移植性**。

当然 Gateway API 也提供了基于 CRD 的扩展，不过 Gateway API 是通过定义 **parametersRef**，**extensionRef** 等标准化的方式去接入 CRD。

当然 Gateway API 目前也不够成熟，社区也是积极的发展中，在各个 API Gateway Implementations 的推动下，我认为发展速度是很快的，未来值得期待。

<div align="center">
<img src="https://liuxunzhuo.oss-cn-chengdu.aliyuncs.com/blog/architecture-20230326140724411-20230326142333260.png" alt="architecture" />
</div>

Envoy Gateway 是一个单进程模型，各个内部的 Component 是以协程的方式工作，主要是有这几个组件组成：

+ Provider：提供从 Kubernetes、File、配置中心 List / Watch Gateway API 资源的能力。
+ Gateway API Translator：提供将 Gateway API 资源翻译为 IR 的能力，IR 全称 Intermediate Representation，可以理解为资源的中间状态，EG 的 IR 分为两类：Infra IR 和 xDS IR。Infra IR 包含的主要是资源相关的信息，如 EnvoyProxy 的 Deployment、Service 等资源的关键信息。xDS IR 包含主要是下发到 EnvoyProxy 的 xDS 信息。
+ Infra Manager：提供将 Infra IR 转化为实际的 Kubernetes 资源，对集群中的资源 create/update/delete 进行管理。
+ xDS Translator：提供将 xDS IR 转化为 go-control-plane xDS API 的能力。
+ xDS Server：作为实际的 xDS gRPC Server，下发 xDS 信息至数据面的 EnvoyProxy。

## egctl 介绍

egctl 是 Envoy Gateway 提供的一个命令行工具，目的是让 EG 的运维和开发者，方便的对 EG 控制面、数据面实时状态进行监控。

目前 egctl 提供的能力包括：

+ egctl version：获取控制面数据面的版本信息
+ egctl config：获取数据面不同 envoyProxy 示例的 xDS 配置信息，支持对不同 xDS 资源类型过滤：all、bootstrap、cluster、endpoint、listener、route。
+ egctl experimental translate：将 gateway api 资源翻译为 xDS 配置信息，支持按不同 xDS 类型过滤。

未来规划的能力：

+ egctl analyze：分析集群中 EG 配置的状态和校验信息
+ egctl install：提供一键安装 EG 的能力
+ egctl uninstall：提供一键卸载 EG 的能力
+ egctl manifest：提供输出 EG manifests 的能力

## 本地开发指南

### Envoy Gateway 本地开发

EG  目前提供了方便的 install.yaml 以及 helm chart 的安装方式，不过要进行本地开发，当我们对代码进行了修改，我们该如何部署并测试呢？

首先 clone fork Repo 的代码：

```shell
git clone git@github.com:YOURFORK/gateway.git
```

EG 提供了强大的 Make Targets，能够方便的做本地测试，执行 `make help` 可查看提供的命令：

<div align="center">
<img src="https://liuxunzhuo.oss-cn-chengdu.aliyuncs.com/blog/image-20230326145947335.png" alt="help" />
</div>

当你修改代码后，需要一个集群去部署和测试 EG，如果你没有现有的集群，可以执行：

```shell
make create-cluster
```

会为你创建一个 kind 集群并部署 metallb。 

当集群准备好之后，首先你需要 构建 Dev 镜像并推送到镜像仓库，通过执行以下命令：

```shell
IMAGE=docker.io/bitliu/gateway-dev BINS=envoy-gateway TAG=latest make push-multiarch
```

`IMAGE` 需要替换为你自己的镜像仓库名称，`TAG`  是此构建镜像的 TAG，此时最新的镜像 `docker.io/bitliu/gateway-dev:latest` 就会被构建并推送到镜像仓库中。

接着，你需要指定镜像为此构建镜像对 EG 进行部署，执行以下命令：

> `IMAGE`  和 `TAG` 需和上一步保持一致

```shell
IMAGE=docker.io/bitliu/gateway-dev TAG=latest make kube-deploy 
```

EG 也提供了一个命令快速部署一个 demo 到集群中：

```shell
make kube-demo
```

你可以基于此 demo 做快速验证。

当测试结束后，可以清理测试资源：

清理 Demos：

```shell
make kube-demo-undeploy
```

清理 EG：

```shell
make kube-undeploy
```

此时，基于你的代码修改的 EG 就会部署在集群中了，然后你就可以基于此部署对修改进行验证。

当你的修改涉及 Envoy Gateway Types 时（CRD API），请记得执行以下，更新 deepcopy 和 manifests：

```shell
make manifests generate
```

### egctl 本地开发

egctl 的测试相比 envoy gateway 就简单很多，当你的代码修改是基于 egctl 时，可以执行以下命令去构建 egctl 最新的 binary 做测试：

```shell
 make build BINS="egctl"
```

生成的二进制文件会在 `bin/{GOOS}/{GOARCH}/egctl`。

## GitHub Workflows 介绍

了解 Workflow 可以让贡献者更清楚提交 PR 之后，如果遇到报错，该如何找到问题并解决它。

Envoy Gateway 提供了多个不同的 GitHub Workflows 对代码进行 测试、验证、发布，主要有这几个 Workflows：

+ [Build and Test](https://github.com/envoyproxy/gateway/blob/main/.github/workflows/build_and_test.yaml) Workflow：此 Workflow 在每次提交 PR 和 合并 PR 时执行，执行了以下任务：
  + 静态校验：
    + linter：包含 golint、yamllint、whitenoise lint and codespell。
    + gen-check：检验是否 go mod tidy 以及 manifest、deepcopy 是否更新。
    + license check：检验是否代码包含 license 信息。
    + coverage test：单元覆盖率测试。
  + 多架构 EG / egctl binary 构建
  + 动态测试：对 v1.24.0、v1.25.3、v1.26.0 三个版本 k8s 的  Gateway API 一致性测试
  + 发布最新镜像和 Helm Chart（当代码被合并后执行）
+ [Docs](https://github.com/envoyproxy/gateway/blob/main/.github/workflows/docs.yaml) Workflow：此 Workflow 在每次提交 PR 和 合并 PR 时执行，EG 文档是基于 GitHub Pages，它执行以下任务：
  + docs-lint：文档校验
  + docs-build：文档构建
  + docs-publish：文档发布（当代码合并之后执行）
+ [Latest Release](https://github.com/envoyproxy/gateway/blob/main/.github/workflows/latest_release.yaml) Workflow：此 Workflow 在每次代码合并之后执行，发布最新的版本，此版本 tag 和 release 都为 `latest` ，方便在版本周期内，用户想使用最新的 EG 特性。
+ [Release](https://github.com/envoyproxy/gateway/blob/main/.github/workflows/release.yaml) Workflow：此 Workflow 在每次版本发布后执行，release manager push 最新的 TAG 之后自动执行构建和发布。

## 一些建议

新参与到贡献的同学，如果不知道从哪里开始，可以优先从 issue 中选择 [Help Wanted](https://github.com/envoyproxy/gateway/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22) 的 label issue，这些 issue 首先是由 maintainer 认可社区需要的。

当然贡献包含提交 issue 本身，如果你发现在使用 EG 的时候有什么优化点，可以提交 issue 和社区讨论。

在提交 PR 之前，请确定是否此 PR 有相应的 issue 介绍，一个缺少描述和 issue 的 PR 往往会让 reviewer 和 maintainer 头疼，会减慢代码合并的速度。

同时推荐对 [EG 设计](https://gateway.envoyproxy.io/v0.3.0/design_docs.html) 进行阅读，了解 EG 的设计思路，如果有什么大的 proposal，建议创建对应的 issue，并提交对应的 Design PR。

社区不仅重视的是代码本身，同时也看重社区氛围，开源不仅是提升自己技术以及影响力的途径之一，也是对协作能力的提升。

参与开源本身是一件很有趣的事，笔者也是因为兴趣和热爱而参加开源，它不是我的工作，我也不会因为参与开源而获得收益，但鼓励大家参与开源，去感受开源背后的精神和魅力。

笔者目前是 Envoy Gateway 的指导委员会成员以及维护者之一，同时是 Gateway API 的 Reviewer，如果你对 Envoy Gateway 感兴趣或者有什么问题，欢迎与我交流，提供宝贵建议。

**Let us have fun！**
