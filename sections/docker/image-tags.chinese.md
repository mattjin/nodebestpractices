# 理解镜像标签vs摘要，谨慎使用`:latest`标签 Understand image tags vs digests and use the `:latest` tag with caution

### 一段解释

如果这是在一个生产环境下，安全和稳定很重要，那么仅仅是“方便”可能不是最好的决定因素。此外，`:latest`标签是Docker的默认标签。这意味着，忘记添加显式标记的开发人员将意外地将镜像的新版本推送到`latest`，如果将`latest`标记作为最新的产品镜像使用，则可能会产生非常意外的结果。

If this is a production situation and security and stability are important then just "convenience" is likely not the best deciding factor. In addition the `:latest` tag is Docker's default tag. This means that a developer who forgets to add an explicit tag will accidentally push a new version of an image as `latest`, which might end in very unintended results if the `latest` tag is being relied upon as the latest production image.

### 代码示例:

```bash
$ docker build -t company/image_name:0.1 .
# :latest image is not updated
$ docker build -t company/image_name
# :latest image is updated
$ docker build -t company/image_name:0.2 .
# :latest image is not updated
$ docker build -t company/image_name:latest .
# :latest image is updated
```

### 其他博主的说法
摘自博客[Vladislav Supalov](https://vsupalov.com/docker-latest-tag/):
> 有些人认为:latest总是指向最近推出（most-recently-pushed）的镜像版本。那是不对的。

摘自[Docker success center](https://success.docker.com/article/images-tagging-vs-digests)
> 

<br/>
