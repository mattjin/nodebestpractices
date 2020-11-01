# 清理编译过程的秘钥，避免使用秘钥作为参数（Clean build-time secrets, avoid secrets as args）

<br/><br/>

### 一段解释

Docker 映像不仅仅是一堆文件，而是显示构建期间所发生的一些层级关系。在普通场景中，开发人员需要在构建过程中知道npm令牌（主要是私有registries）- 会通过传递令牌作为构建参数来实现，但这是错误的。这可能看起来是没什么问题并安全的，但此令牌现在可以从开发人员机器中的Docker历史记录、Docker registry和CI中获得。获取到该令牌的攻击者现在就能够拥有写入此组织私有npm registry的权限。还有两个更安全的替代方法：完美无瑕的一个替代方案是使用Docker --secret功能（截至2020年7月还在实验阶段），它只允许在构建期间安装文件。第二种方法是使用带args的多阶段（multi-stage）生成，然后只将必要的文件复制到生产环境。后一种技术不会将秘钥与镜像一起提供，但将出现在本地Docker的历史记录中 - 对大多数组织来说，这通常被认为足够安全。
A Docker image isn't just a bunch of files but rather multiple layers revealing what happened during build-time. In a very common scenario, developers need the npm token during build time (mostly for private registries) - this is falsely achieved by passing the token as a build time args. It might seem innocent and safe, however this token can now be fetched from the developer's machine Docker history, from the Docker registry and the CI. An attacker who gets access to that token is now capable of writing into the organization private npm registry. There are two more secured alternatives: The flawless one is using Docker --secret feature (experimental as of July 2020) which allows mounting a file during build time only. The second approach is using multi-stage build with args, building and then copying only the necessary files to production. The last technique will not ship the secrets with the images but will appear in the local Docker history - This is typically considered as secured enough for most organizations.

<br/><br/>

### 代码示例 – Using Docker mounted secrets (experimental but stable)

<details>

<summary><strong>Dockerfile</strong></summary>

```
# syntax = docker/dockerfile:1.0-experimental

FROM node:12-slim
WORKDIR /usr/src/app
COPY package.json package-lock.json ./
RUN --mount=type=secret,id=npm,target=/root/.npmrc npm ci

# The rest comes here
```

</details>

<br/><br/>

### 代码示例 – Building securely using multi-stage build

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

# The ARG and .npmrc won't appear in the final image but can be found in the Docker daemon un-tagged images list - make sure to delete those
```

</details>

<br/><br/>

### 代码示例 Anti Pattern – Using build time args

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

# Deleting the .npmrc within the same copy command will not save it inside the layer, however it can be found in image history

CMD ["node","index.js"]
```

</details>

<br/><br/>

### Blog Quote: "These secrets aren’t saved in the final Docker"

From the blog, [Alexandra Ulsh](https://www.alexandraulsh.com/2019/02/24/docker-build-secrets-and-npmrc/?fbclid=IwAR0EAr1nr4_QiGzlNQcQKkd9rem19an9atJRO_8-n7oOZXwprToFQ53Y0KQ)

> In November 2018 Docker 18.09 introduced a new --secret flag for docker build. This allows us to pass secrets from a file to our Docker builds. These secrets aren’t saved in the final Docker image, any intermediate images, or the image commit history. With build secrets, you can now securely build Docker images with private npm packages without build arguments and multi-stage builds.

```

```
