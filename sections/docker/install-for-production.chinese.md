# 删除开发依赖 Remove development dependencies

<br/><br/>

### 一段解释

开发依赖大大增加了容器攻击面（例如，潜在的安全弱点）和容器大小。例如，一些最有影响的npm安全漏洞源于devDependencies，比如[eslint scope](https://eslint.org/blog/2018/07/postmortem-for-malicious-package-publishes)或受影响的开发包，如[nodemon使用的事件流](https://snyk.io/blog/a-post-mortem-of-the-malicious-event-stream-backdoor/)。由于这些原因，最终交付生产的映像应该是安全的和最小的。使用`--production`运行npm install是一个很好的开始，但是运行`npm ci`更安全，它可以确保新的安装和锁文件的存在。删除本地缓存可以节省额外的几十MB。通常需要使用devDependencies在容器内进行测试或调试 - 在这种情况下，[多阶段构建](./multi_stage_builds.md)可以帮助拥有不同的依赖集，最后只为生产提供依赖集。

<br/><br/>

### 代码示例 – 对生产环境进行设置

<details>

<summary><strong>Dockerfile</strong></summary>

```dockerfile
FROM node:12-slim AS build

WORKDIR /usr/src/app
COPY package.json package-lock.json ./
RUN npm ci --production && npm cache clean --force

# 剩余部分
```

</details>

<br/><br/>

### 代码示例 – 使用多阶段构建（multi-stage）对生产环境进行设置

<details>

<summary><strong>Dockerfile</strong></summary>

```dockerfile
FROM node:14.8.0-alpine AS build

COPY --chown=node:node package.json package-lock.json ./
# ✅ Safe install
RUN npm ci
COPY --chown=node:node src ./src
RUN npm run build


# Run-time stage
FROM node:14.8.0-alpine

COPY --chown=node:node --from=build package.json package-lock.json ./
COPY --chown=node:node --from=build node_modules ./node_modules
COPY --chown=node:node --from=build dist ./dist

# ✅ Clean dev packages
RUN npm prune --production

CMD [ "node", "dist/app.js" ]
```

</details>


<br/><br/>

### 代码示例 反模式 – 在一个阶段dockerfile里安装所有依赖

<details>

<summary><strong>Dockerfile</strong></summary>

```dockerfile
FROM node:12-slim AS build

WORKDIR /usr/src/app
COPY package.json package-lock.json ./
# 下面存在两个问题：安装了所有开发依赖，在npm install后并没有删除缓存 Two mistakes below: Installing dev dependencies, not deleting the cache after npm install
RUN npm install

# 剩余部分
```

</details>

<br/><br/>

### 博客引用: "npm ci比普通的install更严格"

摘自[npm documentation](https://docs.npmjs.com/cli/ci.html)

> 此命令类似于npm install，只是它用于自动化环境，如测试平台、持续集成和部署，或者任何您希望确保您的依赖项被干净安装的情况。通过跳过某些面向用户的特性，它可以比常规npm install快得多。它也比常规安装更严格，这有助于捕获大多数npm用户增量安装的本地环境所导致的错误或不一致。
