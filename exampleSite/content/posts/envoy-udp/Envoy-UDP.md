---
weight: 1
title: "EnvoyProxy 中 UDP 协议概览"
date: 2023-04-24T21:40:32+08:00
lastmod: 2023-04-24T21:40:32+08:00
draft: false
author: "Xunzhuo"
authorLink: "https://github.com/xunzhuo"
description: "本文结合一些基础的网络知识，介绍了 EnvoyProxy 中 UDP 协议的介绍与使用。"

tags: ["EnvoyProxy", "Networking"]
categories: ["EnvoyProxy"]

toc:
  auto: false
---

# Envoy 中 UDP 协议概览

本文结合一些基础的网络知识，介绍了 EnvoyProxy 中 UDP 协议的介绍与使用。

## 背景知识

### TCP 与 UDP 对比

UDP（用户数据报协议）和TCP（传输控制协议）都是网络通信中常用的传输层协议，它们之间的主要区别如下：

1. 连接模式：

TCP 是面向连接的协议，它需要在通信前先进行三次握手建立连接，然后才能进行数据传输。而 UDP 是面向无连接的协议，它不需要建立连接，每个数据报都是相互独立的。

2. 可靠性：

TCP 提供可靠的数据传输，它保证数据的传输顺序和完整性，如果发生丢包或错误，TCP 会进行重传和错误检测等机制来保证数据的正确性。而 UDP 不提供可靠的数据传输保证，如果发生丢包或错误，UDP 不会进行重传和错误检测，因此有可能导致数据的丢失或不完整。

3. 传输效率：

由于TCP需要进行连接建立、错误检测、重传等操作，因此在传输效率方面不如 UDP。而 UDP 具有较高的传输效率，因为它不需要进行连接建立和错误检测等操作，同时也没有 TCP 的拥塞控制机制，因此可以实现更快的数据传输。

4. 应用场景：

TCP 通常用于对数据传输可靠性要求较高的场景，如文件传输、网页浏览、邮件传输等，因为 TCP 可以保证数据的正确性和完整性。而 UDP 通常用于对数据传输效率要求较高的场景，如音视频传输、游戏数据传输等，因为 UDP 具有较高的传输效率和低延迟优势。

综上，TCP 和 UDP 各有优势，具体选择哪种协议要根据应用场景和需求来决定。如果要求数据传输的可靠性和完整性，则选择 TCP；如果要求数据传输效率和低延迟，则选择 UDP。

不过 Google 开源了 `QUIC` 协议，它是一套基于 `UDP` 的开源协议，它汇集了 `TCP`和 `UDP` 的优点，传输高效并且可靠。

### TCP 与 Session

在 TCP 协议中，Session 通常是指一对主机之间建立的一种持续的、可靠的通信连接。TCP协议采用三次握手来建立连接，通过四次挥手来终止连接，从而确保了连接的可靠性和数据的完整性。

在建立连接时，TCP 协议使用源IP地址、目标IP地址、源端口号、目标端口号这四个元素来标识一条连接。这个标识可以看作是 TCP Session 的一个唯一标识符，也被称为 Session ID。

通过这个四元组，TCP 协议可以识别出每个连接的唯一标识符，从而实现对连接的管理和控制。 TCP 协议使用 Session ID 来区分不同的连接，确保每个连接都是独立的、互不干扰的。

Session ID 在 TCP 协议中起着非常重要的作用，它可以用于连接的管理、控制、跟踪和调试等。

总之，在 TCP 协议中，Session 是指一对主机之间建立的一种持续的、可靠的通信连接，Session ID 则是用于标识和管理这种连接的唯一标识符，它通常由源IP地址、目标IP地址、源端口号和目标端口号这四个元素组成。

### UDP 与 Session

在 UDP（用户数据报协议）中，不存在像 TCP（传输控制协议）中那样的会话（Session）概念。UDP 是一种面向无连接的协议，每个 UDP 数据报都是一个独立的包，它们之间没有先后顺序或关联性，也没有对数据报是否到达或重复的确认机制。

UDP 通信是 Stateless 的，每次数据包都是相互独立的，没有上下文关系，因此 UDP 的数据包是没有 Session ID 的。每次通信都是从头开始，通信结束后就断开连接，不会保持长久的连接状态。

因此，使用 UDP 协议时，需要在应用层自行实现数据报的编号、确认、重传等功能，以保证数据的可靠传输。

当然，在实际应用中，为了更好的管理和控制 UDP 的通信，可以通过一些手段来模拟"会话"的概念，比如在应用层协议中定义一些约定好的标识符来标识通信的双方，或者通过网络地址和端口号来识别通信的进程等。但这些方法并不是 UDP 协议本身提供的功能。

## Envoy 中 UDP 的支持

Envoy 是一种开源的代理服务器，它支持多种协议和网络模型，包括 UDP 协议。在 Envoy 中，对于 UDP 协议的通信，可以使用 UDP listener 和 UDP proxy 来进行管理和控制。

UDP 代理监听过滤器允许 Envoy 在 UDP 客户端和服务器之间作为`非透明代理`运行。非透明性意味着上游服务器将看到 Envoy 实例的源 IP 和端口，而不是客户端的 IP 和端口。所有数据报从客户端流向 Envoy，再流向上游服务器，然后再返回 Envoy 和客户端。

由于 UDP 不是面向连接的协议，Envoy 必须跟踪客户端的会话，以便将来自上游服务器的响应数据报路由回正确的客户端。

在 Envoy 中，每个 UDP 会话都会被赋予一个唯一的 Session ID，它由 Envoy 根据 UDP 数据包的**源地址、目标地址、源端口号和目标端口号**等信息自动生成，并在整个会话中保持不变。这个 Session ID 可以用于在 Envoy 中跟踪和管理 UDP 通信的状态，包括限流、监控、安全策略等方面，会话持续到达到空闲超时时间（**`idle_timeout`**）为止。

可以通过将 `use_per_packet_load_balancing` 设置为 `true` 来禁用上述会话粘滞性。在这种情况下，将启用每个数据块的负载均衡。这意味着使用当前使用的负载均衡策略在 UDP 代理接收到每个数据块时选择上游主机。


如果将 `use_original_src_ip` 字段设置为 `true`，UDP 代理监听器过滤器还可以作为透明代理运行。但请注意，它不会将端口转发给上游，只会将 IP地址 转发给上游。

### UDP 负载均衡策略与健康检查

Envoy 在负载均衡 UDP 数据报时，将充分利用配置的负载均衡器来配置上游集群。

默认情况下，当创建新会话时，Envoy 将使用配置的负载均衡器选择一个上游主机并将该会话与之关联。**将来属于该会话的所有数据报将路由到同一上游主机**。

但是，如果将 `use_per_packet_load_balancing` 字段设置为 `true`，则 Envoy 使用配置的负载均衡器会在下一个数据报上选择另一个上游主机，并在不存在此类主机时创建一个新会话。因此，如果有多个上游主机可用于负载均衡器，则每个数据块都将转发到不同的主机。

当上游主机变得不健康（由于主动健康检查），当收到下一个数据报时，Envoy 将尝试创建到健康主机的新会话。每个上游集群可以创建的会话数受到集群的最大连接熔断器的限制，默认情况下，此值为 `1024`。

### UDP 路由策略

在 UDP 路由中，可以根据 [UDP network input](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/matching/matching_api#extension-category-envoy-matching-network-input)：

- [Destination IP](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/matching/common_inputs/network/v3/network_inputs.proto#extension-envoy-matching-inputs-destination-ip). 目标 IP
- [Destination port](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/matching/common_inputs/network/v3/network_inputs.proto#extension-envoy-matching-inputs-destination-port). 目标端口
- [Source IP](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/matching/common_inputs/network/v3/network_inputs.proto#extension-envoy-matching-inputs-source-ip).  来源 IP
- [Source port](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/matching/common_inputs/network/v3/network_inputs.proto#extension-envoy-matching-inputs-source-port).  来源端口

对 UDP 流量进行路由分发，并且可以基于强大的 [Matching API](https://www.envoyproxy.io/docs/envoy/latest/xds/type/matcher/v3/matcher.proto#envoy-v3-api-msg-xds-type-matcher-v3-matcher)，对 UDP 路由进行描述。而 UDP 路由(extensions.filters.udp.udp_proxy.v3.Route) 配置很简单，只有目标 Cluster：

```json
{
  "cluster": ...
}
```

通过以下例子来做一个示例，以下 matcher 配置将 Envoy 根据源 IP 路由 UDP 数据报，忽略`源 IP` 为 `127.0.0.1` 以外的数据报，并且根据不同的`源端口`将其余数据报过滤到不同的 Cluster：

```yaml
        matcher:
          # The outer matcher list matches source IP.
          matcher_list:
            matchers:
            - predicate:
                single_predicate:
                  input:
                    name: envoy.matching.inputs.source_ip
                    typed_config:
                      '@type': type.googleapis.com/envoy.extensions.matching.common_inputs.network.v3.SourceIPInput
                  value_match:
                    exact: 127.0.0.1
              on_match:
                matcher:
                  # The inner matcher tree matches source port.
                  matcher_tree:
                    input:
                      name: envoy.matching.inputs.source_port
                      typed_config:
                        '@type': type.googleapis.com/envoy.extensions.matching.common_inputs.network.v3.SourcePortInput
                    exact_match_map:
                      map:
                        "80":
                          action:
                            name: route
                            typed_config:
                              '@type': type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.Route
                              cluster: udp_service
                        "443":
                          action:
                            name: route
                            typed_config:
                              '@type': type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.Route
                              cluster: udp_service2
                  on_no_match:
                    action:
                      name: route
                      typed_config:
                        '@type': type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.Route
                        cluster: udp_service3
```

再举个完整的例子，以下示例配置将使 Envoy 监听 UDP 端口 1234 ，并配置 [UDP Listener 配置(udp_listener_config)](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/udp_listener_config.proto#envoy-v3-api-msg-config-listener-v3-udplistenerconfig)，并代理到监听 1235 端口的 UDP 上游服务器，允许双向 9000 字节的数据包：

```yaml
admin:
  address:
    socket_address:
      protocol: TCP
      address: 127.0.0.1
      port_value: 9901
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        protocol: UDP
        address: 127.0.0.1
        port_value: 1234
    udp_listener_config:
      downstream_socket_config:
        max_rx_datagram_size: 9000
    listener_filters:
    - name: envoy.filters.udp_listener.udp_proxy
      typed_config:
        '@type': type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
        stat_prefix: service
        matcher:
          on_no_match:
            action:
              name: route
              typed_config:
                '@type': type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.Route
                cluster: service_udp
        upstream_socket_config:
          max_rx_datagram_size: 9000
  clusters:
  - name: service_udp
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: service_udp
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 1235
```

### Envoy UDP 相关配置

Envoy 中 UDP 代理的监听器配置在 [config.listener.v3.UdpListenerConfig](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/udp_listener_config.proto#envoy-v3-api-msg-config-listener-v3-udplistenerconfig):

```json
{
  "downstream_socket_config": {...},
  "quic_options": {...},
  "udp_packet_packet_writer_config": {...}
}
```

**downstream_socket_config**：下游监听器 UDP socket 配置，可设置最大接受 UDP 数据报的大小等。

**quic_options**：QUIC 协议的配置。如果为空，则不会在此侦听器上启用 QUIC。

**udp_packet_packet_writer_config**：UDP 数据包 Write 配置。

Envoy 中 UDP 代理的核心配置在 [extensions.filters.udp.udp_proxy.v3.UdpProxyConfig](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/udp/udp_proxy/v3/udp_proxy.proto#extensions-filters-udp-udp-proxy-v3-udpproxyconfig)：

```json
{
  "stat_prefix": ...,
  "cluster": ...,
  "matcher": {...},
  "idle_timeout": {...},
  "use_original_src_ip": ...,
  "hash_policies": [],
  "upstream_socket_config": {...},
  "use_per_packet_load_balancing": ...,
  "access_log": [],
  "proxy_access_log": []
}
```

**stat_prefix**：UDP Proxy 的生成的 stats 前缀。

**cluster**：转发给的 Upstream Cluster 名称，已经 Deprecated，使用 matcher 替代。

**matcher**：matching API，用与描述 UDP 路由规则。

**idle_timeout**：session 保持时间，默认为一分钟。

**use_original_src_ip**：向上游发送数据包时，是否使用下游 IP 地址作为发送方 IP 地址。此选项开启后不会传递源端口信息，并且需要 Linux 内核必须开启 IPv6。

**hash_policies**：指定 UDP 哈希策略，数据包可以通过哈希策略进行路由。如果没有设置 hash_policies，基于哈希的负载均衡算法将随机选择一个主机。目前哈希策略的数量限制为 1。

**use_per_packet_load_balancing**：对每个收到的数据块进行每个数据包的负载均衡。如果没有指定，默认为 false，这意味着每个数据块被转发到该 "会话"（由源IP/端口和本地IP/端口识别）第一次接收数据块时选择的上游主机。

**access_log**、**proxy_access_log**： access log 相关配置。

