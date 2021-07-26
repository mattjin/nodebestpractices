# 利用缓存减少构建时间

## 一段解释

Docker镜像是层的组合，Dockerfile中的每个指令都创建一个层。如果指令相同，或者使用`COPY`或`ADD`的文件相同，docker守护进程可以在构建之间重用这些层。⚠️如果缓存不能用于特定的层，那么所有后续层也将失效。这就是为什么秩序很重要。正确布局Dockerfile以减少构建中移动部件的数量是至关重要的；更新较少的指令应该放在顶部，不断变化的指令（如应用程序代码）应该放在底部。同样重要的是，触发长时间操作的指令应该接近顶部，以确保它们只在真正必要的时候发生（除非每次构建docker映像时都会发生更改）。如果操作正确，从缓存重建整个docker映像几乎可以是瞬时的。

![Docker layers](../../assets/images/docker_layers_schema.png)

* 图片来自jessgreb01的[Digging into Docker layers](https://medium.com/@jessgreb01/digging-into-docker-layers-c22f948ed612)*

### 规则

#### 避免一直改变的LABEL 

如果Dockerfile顶部有一个包含构建版本号的标签，则每次构建时缓存都将失效

```Dockerfile
# 文件起始
FROM node:10.22.0-alpine3.11 as builder

# 这里不要这么做！
LABEL build_number="483"

#... Dockerfile剩余部分
```

#### Have a good .dockerignore file

[**See: On the importance of docker ignore**](./docker-ignore.md)

The docker ignore avoids copying files that could bust our cache logic, like tests results reports, logs or temporary files.

#### Install "system" packages first

It is recommended to create a base docker image that has all the system packages you use. If you **really** need to install packages using `apt`,`yum`,`apk` or the likes, this should be one of the first instructions. You don't want to reinstall make,gcc or g++ every time you build your node app.
**Do not install package only for convenience, this is a production app.**

#### First, only ADD your package.json and your lockfile

```Dockerfile
COPY "package.json" "package-lock.json" "./"
RUN npm ci
```

The lockfile and the package.json change less often. Copying them first will keep the `npm install` step in the cache, this saves precious time. 

### Then copy your files and run build step (if needed) 

```Dockerfile
COPY . .
RUN npm run build
```

## 示例

### Basic Example with node_modules needing OS dependencies
```Dockerfile
#Create node image version alias
FROM node:10.22.0-alpine3.11 as builder

RUN apk add --no-cache \
    build-base \
    gcc \
    g++ \
    make

USER node
WORKDIR /app
COPY "package.json" "package-lock.json" "./"
RUN npm ci --production
COPY . "./"


FROM node as app

USER node
WORKDIR /app
COPY --from=builder /app/ "./"
RUN npm prune --production

CMD ["node", "dist/server.js"]
```


### Example with a build step (when using typescript for example)
```Dockerfile
#Create node image version alias
FROM node:10.22.0-alpine3.11 as builder

RUN apk add --no-cache \
    build-base \
    gcc \
    g++ \
    make

USER node
WORKDIR /app
COPY "package.json" "package-lock.json" "./"
RUN npm ci
COPY . .
RUN npm run build


FROM node as app

USER node
WORKDIR /app
# Only copying the files that we need
COPY --from=builder /app/node_modules node_modules
COPY --from=builder /app/package.json .
COPY --from=builder /app/dist dist
RUN npm prune --production

CMD ["node", "dist/server.js"]
```

## Useful links

Docker docs: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache
