# 优雅地关闭

<br/><br/>

### 一段解释

在一个容器化的运行时，比如Kubernetes，容器频繁地被创建和删除。这种情况不仅发生在抛出错误时，也发生在可接受的场景，比如重新定位容器、用更新的版本替换它们等等。它是通过向进程发送一个通知（SIGTERM信号）来实现的，该通知具有30秒的宽限期。这给开发者带来了一个挑战，即如何确保应用程序能够及时处理正在进行的请求并清理资源。否则成千上万受影响的用户将得不到回应。在实现方面，关闭程序应该等待所有正在进行的请求被清除，然后清理资源。说起来容易做起来难，实际上它需要协调好几个部分：告诉负载均衡器应用程序不准备提供更多的请求（通过健康检查），等待现有的请求完成，避免处理新的请求，清理资源，最后在关闭前记录一些有用的信息。如果正在使用Keep-Alive连接，还必须通知客户机应该建立一个新的连接 — 类似于[Stoppable]的库(https://github.com/hunterloftis/stoppable)可以极大地帮助实现这一点。

<br/><br/>


### 代码示例 – 让Node.js作为根进程使发送信号给程序成为可能(见 [bootstrap using node](./bootstrap-using-node.chinese.md))

<details>

<summary><strong>Dockerfile</strong></summary>

```dockerfile
FROM node:12-slim

# 构建逻辑在这里

CMD ["node", "index.js"]
# 上一行将会让Node.js作为根进程（PID1）

```

</details>

<br/><br/>

### 代码示例 – 使用Tiny进程管理器发送信号给Node

<details>

<summary><strong>Dockerfile</strong></summary>

```dockerfile
FROM node:12-slim

# 构建逻辑在这里

ENV TINI_VERSION v0.19.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini
ENTRYPOINT ["/tini", "--"]

CMD ["node", "index.js"]
# 现在Node将作为TINI的一个子进程，而TINI扮演PID1的角色

```

</details>

<br/><br/>

### 反模式的代码示例 - 使用npm脚步初始化进程

<details>

<summary><strong>Dockerfile</strong></summary>

```dockerfile
FROM node:12-slim

# 构建逻辑在这里

CMD ["npm", "start"]
# 现在Node将作为npm的一个子进程，且将不会接收到信号

```

</details>

<br/><br/>

### 示例 - 关闭阶段

从博客，[Rising Stack](https://blog.risingstack.com/graceful-shutdown-node-js-kubernetes/)

![alt text](../../assets/images/Kubernetes-graceful-shutdown-flowchart.png "关闭阶段")
