# 清理编译过程的秘钥，避免使用秘钥作为参数（Clean build-time secrets, avoid secrets as args）

<br/><br/>

### 一段解释

Docker映像不仅仅是一堆文件，而是展示构建期间所发生的层级关系。在普通场景中，开发人员在构建过程中需要知道npm令牌（主要是对于私有registry）- 会通过把令牌作为构建参数来传递，但这种实现是错误的。这可能看起来是没什么问题，并且安全，但此令牌可以从开发人员机器中的Docker历史记录、Docker registry和CI中获得。获取到该令牌的攻击者就能够拥有写入此组织私有npm registry的权限。还有两个更安全的替代方法：完美无瑕的一个替代方案是使用Docker --secret功能（截至2020年7月还在实验阶段），它只允许在构建期间挂载(mount)文件。第二种方法是使用带args的多阶段（multi-stage）构建，然后只将必要的文件复制到生产环境。后一种技术不会将秘钥与镜像一起提供，但将出现在本地Docker的历史记录中 - 对大多数组织来说，这通常被认为足够安全。
A Docker image isn't just a bunch of files but rather multiple layers revealing what happened during build-time. In a very common scenario, developers need the npm token during build time (mostly for private registries) - this is falsely achieved by passing the token as a build time args. It might seem innocent and safe, however this token can now be fetched from the developer's machine Docker history, from the Docker registry and the CI. An attacker who gets access to that token is now capable of writing into the organization private npm registry. There are two more secured alternatives: The flawless one is using Docker --secret feature (experimental as of July 2020) which allows mounting a file during build time only. The second approach is using multi-stage build with args, building and then copying only the necessary files to production. The last technique will not ship the secrets with the images but will appear in the local Docker history - This is typically considered as secured enough for most organizations.

<br/><br/>

### 代码示例 – 使用Docker挂载秘钥（实验功能但稳定）Using Docker mounted secrets (experimental but stable)

<details>

<summary><strong>Dockerfile</strong></summary>

```
# syntax = docker/dockerfile:1.0-experimental

FROM node:12-slim
WORKDIR /usr/src/app
COPY package.json package-lock.json ./
RUN --mount=type=secret,id=npm,target=/root/.npmrc npm ci

# 剩余部分
```

</details>

<br/><br/>

### 代码示例 – 使用多阶段（multi-stage build）安全构建Building securely using multi-stage build

<details>

<summary><strong>Dockerfile</strong></summary>

```

FROM node:12-slim AS build
ARG NPM_TOKEN
WORKDIR /usr/src/app
COPY . /dist
RUN echo "//registry.npmjs.org/:\_authToken=\$NPM_TOKEN" > .npmrc && \
 npm ci --production && \
 rm -f .npmrc

FROM build as prod
COPY --from=build /dist /dist
CMD ["node","index.js"]

# ARG和.npmrc在最终的镜像中不会出现，但会在Docker daemon un-tagged images list中找到 - 确保删除他们 The ARG and .npmrc won't appear in the final image but can be found in the Docker daemon un-tagged images list - make sure to delete those
```

</details>

<br/><br/>

### 代码示例 反模式 - 使用构建阶段参数 Anti Pattern – Using build time args

<details>

<summary><strong>Dockerfile</strong></summary>

```

FROM node:12-slim
ARG NPM_TOKEN
WORKDIR /usr/src/app
COPY . /dist
RUN echo "//registry.npmjs.org/:\_authToken=\$NPM_TOKEN" > .npmrc && \
 npm ci --production && \
 rm -f .npmrc

# 在拷贝命令的同时删除.npmrc文件不会在layer里面保存它, 但在镜像历史里面还是会找到它 Deleting the .npmrc within the same copy command will not save it inside the layer, however it can be found in image history

CMD ["node","index.js"]
```

</details>

<br/><br/>

### 博客引用: "秘钥不会保存在最终的Docker中These secrets aren’t saved in the final Docker"

摘自博客, [Alexandra Ulsh](https://www.alexandraulsh.com/2019/02/24/docker-build-secrets-and-npmrc/?fbclid=IwAR0EAr1nr4_QiGzlNQcQKkd9rem19an9atJRO_8-n7oOZXwprToFQ53Y0KQ)

> 在2018年11月，Docker18.09在docker构建过程中引入一个新的标志（flag）--secret。它允许用户从一个文件传递秘钥到Docker包（build）。这些秘钥不会保存在最终的Docker镜像，中间镜像和镜像提交历史里面。借助构建参数secret，您现在可以使用私有npm包安全地构建Docker映像，而无需构建参数和多阶段（multi-stage）构建。

In November 2018 Docker 18.09 introduced a new --secret flag for docker build. This allows us to pass secrets from a file to our Docker builds. These secrets aren’t saved in the final Docker image, any intermediate images, or the image commit history. With build secrets, you can now securely build Docker images with private npm packages without build arguments and multi-stage builds.

```

```
