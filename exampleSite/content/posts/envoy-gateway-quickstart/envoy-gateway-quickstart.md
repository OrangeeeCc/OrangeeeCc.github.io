---
weight: 1
title: "Envoy Gateway 指南：快速开始"
date: 2023-03-20T21:40:32+08:00
lastmod: 2023-03-20T21:40:32+08:00
draft: false
author: "Xunzhuo"
authorLink: "https://github.com/xunzhuo"
description: "本文介绍如何快速部署 Envoy Gateway，以及通过一个简单的例子，来展示如何通过 Envoy Gateway 访问 kubernetes 集群中的服务。"

tags: ["Envoy Gateway", "Open Source", "API Gateway"]
categories: ["Envoy Gateway"]

toc:
  auto: false
---

## 前言

Envoy Gateway 是由 EnvoyProxy 社区发起的，联合 Contour、Emissary 等基于 Envoy 的 API 网关开源项目及社区，共同参与并维护的 API 网关项目。

<div align="center">
<img src="https://liuxunzhuo.oss-cn-chengdu.aliyuncs.com/blog/architecture.png" alt="architecture" style="zoom:20%;" />
</div>

Envoy Gateway 可独立或作为 Kubernetes 中的 API 网关，并以 **Gateway API** 为其配置语言。

<div align="center">
<img src="https://liuxunzhuo.oss-cn-chengdu.aliyuncs.com/blog/image-20230321190933818.png" alt="image-20230321190933818" style="zoom:60%;" align="center"/>
</div>

Gateway API 它提供了相比 Ingress 更先进的 API，它基于面向角色思想而设计，有着更合理的权限控制模型，相比 Ingress Annotation 也更统一、清晰的 API 语义，大大增加了 API 的可移植性，并提供了更合理的扩展方式，以及不同协议的支持，这些都是 EG 采用 Gateway API 的原因。

笔者为 Envoy Gateway 项目的核心维护者以及 Envoy Gateway 指导委员会成员，计划开始持续输出关于 Envoy Gateway 的设计、使用、开源贡献、最新特性等内容的分享，欢迎大家参与到 Envoy Gateway 项目的贡献中来，欢迎 issue 和 PR！

本文介绍如何快速部署 Envoy Gateway，以及通过一个简单的例子，来展示如何通过 Envoy Gateway 访问 kubernetes 集群中的服务。

## 前置条件

部署环境需要一个 k8s 集群，本文测试环境为：腾讯云 TKE 集群：

```shell
❯ kubectl version --short

Client Version: v1.25.4
Kustomize Version: v4.5.7
Server Version: v1.24.4-tke.5
```

## 安装

Envoy Gateway 目前已经支持基于 DockerHub OCI 的 Helm Chart 安装，只需一条命令即可快速安装 Envoy Gateway，包含 EG 本身的 CRD 以及最新的 Gateway API，同时 Envoy Gateway 将 **envoy-gateway-system** 作为其 Root Namespace：

``` shell
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v0.0.0-latest -n envoy-gateway-system --create-namespace
```

执行后可以看到以下日志：

``` shell
❯ helm install eg oci://docker.io/envoyproxy/gateway-helm --version v0.0.0-latest -n envoy-gateway-system --create-namespace
Pulled: docker.io/envoyproxy/gateway-helm:v0.0.0-latest
Digest: sha256:51d2ec8064e6e97ebbae2744891bdcdc33037ae7b58f4f9c03f1ff0e4d3ca8c2
NAME: eg
LAST DEPLOYED: Tue Mar 21 16:52:47 2023
NAMESPACE: envoy-gateway-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
**************************************************************************
*** PLEASE BE PATIENT: Envoy Gateway may take a few minutes to install ***
**************************************************************************

Envoy Gateway is an open source project for managing Envoy Proxy as a standalone or Kubernetes-based application gateway.

Thank you for installing Envoy Gateway! 🎉

Your release is named: eg. 🎉

Your release is in namespace: envoy-gateway-system. 🎉

To learn more about the release, try:

  $ helm status eg -n envoy-gateway-system
  $ helm get all eg -n envoy-gateway-system

To have a quickstart of Envoy Gateway, please refer to https://gateway.envoyproxy.io/latest/user/quickstart.html.

To get more details, please visit https://gateway.envoyproxy.io and https://github.com/envoyproxy/gateway.
```

执行以上命令后，等待 Envoy Gateway 可用：

```shell
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

通过以上简单的步骤，集群里就已经有可用的 Envoy Gateway 了，接下来通过一个示例来演示如何快速访问集群中的服务:

```shell
❯ kubectl get pod -n envoy-gateway-system
NAME                             READY   STATUS    RESTARTS   AGE
envoy-gateway-7d98b48ddd-zjtn9   2/2     Running   0          84s
```

## Demo

安装 GatewayClass、Gateway 、HTTPRoute 以及演示 App：

```shell
kubectl apply -f https://github.com/envoyproxy/gateway/releases/download/latest/quickstart.yaml -n default
```

演示 APP 是部署在 Default Namespace 的一个后端服务，基于 `k8s-staging-ingressconformance/echoserver`：

```shell
❯ kubectl get pod -n default
NAME                       READY   STATUS    RESTARTS   AGE
backend-6fdd4b9bd8-fln68   1/1     Running   0          22m
❯ kubectl get svc -n default
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
backend      ClusterIP   xxx.xxx.xxx.xxx   <none>        3000/TCP   23m
```

获取 Gateway 关联的 Service：

```shell
export ENVOY_SERVICE=$(kubectl get svc -n envoy-gateway-system --selector=gateway.envoyproxy.io/owning-gateway-namespace=default,gateway.envoyproxy.io/owning-gateway-name=eg -o jsonpath='{.items[0].metadata.name}')
```

测试是否获取正确 Service：

```shell
echo $ENVOY_SERVICE
```

Port-Forward service 以快速访问：

```shell
kubectl -n envoy-gateway-system port-forward service/${ENVOY_SERVICE} 8888:80 &
```

通过 Envoy Gateway 请求目标 App：

```shell
curl --verbose --header "Host: www.example.com" http://localhost:8888/get
```

会得来自目标 App 的以下响应：

```shell
❯ curl --verbose --header "Host: www.example.com" http://localhost:8888/get
*   Trying 127.0.0.1:8888...
* Connected to localhost (127.0.0.1) port 8888 (#0)
> GET /get HTTP/1.1
> Host: www.example.com
> User-Agent: curl/7.86.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: application/json
< x-content-type-options: nosniff
< date: Tue, 21 Mar 2023 09:01:21 GMT
< content-length: 510
< x-envoy-upstream-service-time: 0
< server: envoy
< 
{
 "path": "/get",
 "host": "www.example.com",
 "method": "GET",
 "proto": "HTTP/1.1",
 "headers": {
  "Accept": [
   "*/*"
  ],
  "User-Agent": [
   "curl/7.86.0"
  ],
  "X-Envoy-Expected-Rq-Timeout-Ms": [
   "15000"
  ],
  "X-Envoy-Internal": [
   "true"
  ],
  "X-Forwarded-For": [
   "xx.xx.xx.xx"
  ],
  "X-Forwarded-Proto": [
   "http"
  ],
  "X-Request-Id": [
   "de3ffad0-62ce-4fbb-a67d-ad0c8773bf19"
  ]
 },
 "namespace": "default",
 "ingress": "",
 "service": "",
 "pod": "backend-6fdd4b9bd8-pg78f"
}                                                                                                                    
```

同时你也可以直接通过访问 LB 的 IP 以访问后端服务：

```shell
export GATEWAY_HOST=$(kubectl get svc/${ENVOY_SERVICE} -n envoy-gateway-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

测试是否获取正确 LB IP：

```shell
echo $GATEWAY_HOST
```

发送请求：

```shell
curl --verbose --header "Host: www.example.com" http://$GATEWAY_HOST/get
```

你会得到类似上面的响应结果，至此你已经通过 Envoy Gateway 访问到了 k8s 中的服务了。

## 清理集群

删除 GatewayClass、Gateway、HTTPRoute 以及 示例 App：

```shell
kubectl delete -f https://github.com/envoyproxy/gateway/releases/download/latest/quickstart.yaml --ignore-not-found=true
```

删除 Envoy Gateway：

```shell
helm uninstall eg -n envoy-gateway-system
```

