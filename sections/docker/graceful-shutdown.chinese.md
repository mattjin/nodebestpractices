# 优雅地关闭Shutdown gracefully

<br/><br/>

### 一段解释

In a Dockerized runtime like Kubernetes, containers are born and die frequently. This happens not only when errors are thrown but also for good reasons like relocating containers, replacing them with a newer version and more. It's achieved by sending a notice (SIGTERM signal) to the process with a 30 second grace period. This puts a challenge on the developer to ensure the app is handling the ongoing requests and clean-up resources in a timely fashion. Otherwise thousands of sad users will not get a response. Implementation-wise, the shutdown code should wait until all ongoing requests are flushed out and then clean-up resources. Easier said than done, practically it demands orchestrating several parts: Tell the LoadBalancer that the app is not ready to serve more requests (via health-check), wait for existing requests to be done, avoid handling new requests, clean-up resources and finally log some useful information before dying. If Keep-Alive connections are being used, the clients must also be notified that a new connection should be established - A library like [Stoppable](https://github.com/hunterloftis/stoppable) can greatly help achieving this.

<br/><br/>


### 代码示例 – Placing Node.js as the root process allows passing signals to the code (see [bootstrap using node](./bootstrap-using-node.md))

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

### Example - The shutdown phases

From the blog, [Rising Stack](https://blog.risingstack.com/graceful-shutdown-node-js-kubernetes/)

![alt text](../../assets/images/Kubernetes-graceful-shutdown-flowchart.png "The shutdown phases")
