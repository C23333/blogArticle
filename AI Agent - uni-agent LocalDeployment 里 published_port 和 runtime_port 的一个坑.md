---
title: uni-agent LocalDeployment 里 published_port 和 runtime_port 的一个坑
date: 2026-05-26 15:30:00
tags:
  - uni-agent
  - Docker
  - AI Agent
keywords: uni-agent,LocalDeployment,Docker,published_port,runtime_port,AgentEnv
categories: AI Agent
---



# uni-agent LocalDeployment 里 published_port 和 runtime_port 的一个坑

> 本文只讨论 `uni-agent` local deployment 中 `published_port` 和 `runtime_port` 的区别。
>
> Agent 的 tool calling、parser、训练流程暂时不展开，不影响理解本文主体问题。

## 铺垫

`uni-agent` 的 `AgentEnv` 可以先理解成一个 Agent 执行任务时使用的运行环境。它不是只负责保存几行配置，而是要准备一个可以执行命令、读写文件、安装依赖的 sandbox。

local deployment 下，这个 sandbox 通常由 Docker、Podman 或 Apptainer 启动。sandbox 里面会运行一个 `swerex.server`，外部的 `RemoteRuntime` 再通过 HTTP 请求调用它。也就是说，`execute_bash`、`read_file` 这些工具动作，最终不是直接在当前 Python 进程里执行，而是发给 sandbox 里的 runtime server。

大体流程如下：

```text
AgentEnv
  -> LocalDeployment
  -> docker / podman / apptainer 启动 sandbox
  -> sandbox 内运行 swerex.server
  -> RemoteRuntime 通过 HTTP 访问 swerex.server
  -> execute_bash / read_file / upload 等工具动作
```

画成图大概是这样：

```text
+----------------------+          HTTP          +-------------------------+
|  uni-agent 进程       |  ------------------->  |  sandbox 容器            |
|                      |                        |                         |
|  AgentEnv            |                        |  swerex.server           |
|  LocalDeployment     |                        |  execute_bash/read_file  |
|  RemoteRuntime       |                        |                         |
+----------------------+                        +-------------------------+
```

所以 local deployment 能否启动成功，不只取决于容器有没有起来，还取决于外部的 `RemoteRuntime` 能不能访问到 sandbox 里的 `swerex.server`。

这里就会碰到两个端口：

```text
runtime_port
published_port
```

这两个字段如果只看名字，很容易混在一起。

## Docker 的端口映射

先看一个最普通的 Docker 命令：

```bash
docker run -p 4567:8000 xxx
```

`-p 4567:8000` 表示：

```text
宿主机 4567  ->  容器 8000
```

左边的 `4567` 是宿主机端口，右边的 `8000` 是容器内端口。容器里的服务监听 `8000`，但是宿主机上的程序访问它时，应该访问 `4567`。

也就是：

```bash
curl http://127.0.0.1:4567
```

而不是：

```bash
curl http://127.0.0.1:8000
```

后者是在访问宿主机自己的 `8000` 端口。如果宿主机上没有服务监听这个端口，请求自然不会通。

## 换成 Java 服务看

如果用 Spring Boot 举例，会更直观一点。

项目配置：

```properties
server.port=8080
```

Docker 启动：

```bash
docker run -p 18080:8080 my-spring-app
```

这时候含义是：

* Spring Boot 在容器内监听 `8080`
* 宿主机暴露 `18080`
* 浏览器或其他外部 client 访问 `18080`
* Docker 把宿主机 `18080` 转发到容器 `8080`

流程如下：

```text
浏览器
  |
  | http://127.0.0.1:18080
  v
宿主机 18080
  |
  | Docker 端口映射
  v
容器 8080
  |
  v
Spring Boot
```

所以 `8080` 和 `18080` 都是正确端口，只是视角不同。服务端关心自己监听哪个端口，client 关心自己应该访问哪个端口。

## 回到 `uni-agent`

`LocalDeployment` 里的两个字段，可以先按这个方式理解：

| 字段 | 类比 Spring Boot | 含义 |
| --- | --- | --- |
| `runtime_port` | 容器内 `server.port=8080` | `swerex.server` 在 sandbox 内监听的端口 |
| `published_port` | `docker run -p 18080:8080` 左边的 `18080` | Docker/Podman 暴露给宿主机访问的端口 |

假设配置是：

```text
runtime_port = 8000
published_port = 4567
```

如果 `RemoteRuntime` 在宿主机上访问 sandbox，那么访问链路应该是：

```text
RemoteRuntime -> 127.0.0.1:4567 -> sandbox:8000 -> swerex.server
```

这时 `RemoteRuntimeConfig.port` 应该是 `4567`。如果这里误用了 `8000`，就会变成访问：

```text
http://127.0.0.1:8000
```

这相当于让宿主机上的 client 直接访问容器内部端口，通常是不通的。

## 不能简单认为 Docker/Podman 永远用 published_port

上面的场景是 `RemoteRuntime` 在宿主机上访问 sandbox。但还有另一种情况：`uni-agent` 自己也在容器里运行，并且和 sandbox 容器处在同一个 Docker network。

这时候访问链路可能是：

```text
+-------------------------+        Docker network        +-------------------------+
|  uni-agent 容器          |  ------------------------->  |  sandbox 容器            |
|                         |                              |                         |
|  RemoteRuntime          |                              |  swerex.server:8000      |
+-------------------------+                              +-------------------------+
```

`RemoteRuntime` 可能直接访问 sandbox 容器 IP：

```text
http://172.18.0.9:8000
```

这个访问没有经过宿主机的 `4567`，而是在容器网络里直连 `swerex.server`。这种情况下，`RemoteRuntimeConfig.port` 应该使用 `runtime_port`，也就是 `8000`。

所以这里不能按 container runtime 类型判断端口。不是看到 Docker/Podman 就一律使用 `published_port`，还要看 `RemoteRuntime` 实际访问的 host 是什么。

## 判断方式

可以按访问地址来判断：

* 如果 `RemoteRuntimeConfig.host` 是 `http://127.0.0.1`、`http://localhost`，或者用户显式配置的 Docker host，通常应该使用 `published_port`
* 如果 `RemoteRuntimeConfig.host` 是 sandbox 容器 IP，例如 `http://172.18.0.9`，通常应该使用 `runtime_port`

对应两种链路：

```text
情况一：宿主机访问

RemoteRuntime
  |
  | 127.0.0.1:4567
  v
Docker publish
  |
  | sandbox:8000
  v
swerex.server


情况二：容器网络访问

RemoteRuntime 容器
  |
  | 172.18.0.9:8000
  v
sandbox 容器里的 swerex.server
```

关键不在 Docker 本身，而在 client 从哪里访问 sandbox。

## 测试里覆盖的两个场景

这里的测试不需要真正启动 Docker。要验证的是 `LocalDeployment` 生成的 `RemoteRuntimeConfig` 是否正确，而不是 Docker 的端口映射功能。

宿主机访问场景：

```text
runtime_port = 8000
published_port = 4567
host = http://127.0.0.1
```

期望结果：

```text
RemoteRuntimeConfig.host = http://127.0.0.1
RemoteRuntimeConfig.port = 4567
```

如果 port 变成 `8000`，后续 health check 就会访问：

```text
http://127.0.0.1:8000
```

这就和 Docker 的 `4567:8000` 映射对不上。

容器网络访问场景：

```text
host = http://172.18.0.9
runtime_port = 8000
published_port = 4567
```

期望结果：

```text
RemoteRuntimeConfig.port = 8000
```

这两个例子放在一起看，边界就比较清楚了：`published_port` 和 `runtime_port` 没有谁天然更对，关键是 `RemoteRuntime` 实际从哪条路径访问 sandbox。宿主机入口走 published port，容器网络直连就走 runtime port。

## 和 Agent 的关系

如果只看表面，这只是 Docker 端口映射问题。但放在 AgentEnv 里，它影响的是工具执行链路。

一个能执行命令的 Agent，大体会经过这些步骤：

```text
模型输出 tool call
    |
    v
tool parser 解析
    |
    v
runtime client 发请求
    |
    v
sandbox 执行命令
    |
    v
拿到 observation
    |
    v
塞回模型上下文
```

如果 runtime client 连接不到 sandbox，后面的 tool parser、observation、训练数据都不会进入正题。它表现出来可能像 Agent 工具不可用，但实际问题在 deployment/runtime 这一层。

这和业务系统里排接口问题很像。Controller、Service 写得没问题，不代表请求一定能到应用。Nginx、网关、端口映射、服务注册，其中任何一层配置错了，接口都会失败。

## 以后看到这种端口字段，先问三个问题

```text
服务在哪监听？
client 从哪访问？
中间有没有端口映射？
```

套到这次：

```text
服务在哪监听？
  -> sandbox 容器里的 8000

client 从哪访问？
  -> 如果是宿主机，就是 127.0.0.1
  -> 如果是同网络容器，就是 172.x.x.x

中间有没有端口映射？
  -> 宿主机访问有 4567:8000
  -> 容器网络直连没有这层映射
```

这样就能判断 `RemoteRuntimeConfig.port` 应该取 `published_port` 还是 `runtime_port`。

## 先记到这

`runtime_port` 更接近服务自身监听端口，`published_port` 更接近外部访问入口端口。哪个字段进入 `RemoteRuntimeConfig.port`，取决于 `RemoteRuntimeConfig.host` 是谁。

后面继续看 `RemoteRuntime`，它负责把 `execute_bash` 这类工具调用包装成 HTTP 请求。等这条链路看明白，再看 tool parser 和训练，会更容易接上。

春风十里扬州路，卷上珠帘总不如。