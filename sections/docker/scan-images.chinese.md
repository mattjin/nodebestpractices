# 上线前扫描整个镜像 Scan the entire image before production

<br/><br/>

### 一段解释

扫描代码中的漏洞是一项有价值的行为，但它并不能覆盖所有潜在的威胁。为什么？因为操作系统级别上也存在漏洞，应用程序可能会执行那些二进制文件，如Shell、Tarball、OpenSSL。此外，在代码扫描之后可能会注入易受攻击的依赖项（即供应链攻击），因此在生产正常之前扫描最终镜像。这个想法类似于E2E测试 - 在单独测试各个部分之后，最终检查组装的可交付成果是很有价值的。有3个主要的扫描系列：带有缓存漏洞数据库的本地/CI应用、云中的扫描服务以及在docker构建过程中进行扫描的一系列工具。第一组是最流行的，且通常是最快的工具，比如[Trivvy](https://github.com/aquasecurity/trivy)，[Anchore](https://github.com/anchore/anchore)和[Snyk](https://support.snyk.io/hc/en-us/articles/360003946897-Container-security-overview)值得探索。大多数CI供应商提供一个本地插件，方便与这些扫描服务进行交互。应该注意的是，这些扫描服务覆盖了大量的场景，因此在几乎每一次扫描中都会发现结果 - 考虑设置高阈值杆以避免应接不暇。

<br/><br/>

### 代码示例 – 使用Trivvy扫描 Scanning with Trivvy

<details>

<summary><strong>Bash</strong></summary>

```console
$ sudo apt-get install rpm
$ wget https://github.com/aquasecurity/trivy/releases/download/{TRIVY_VERSION}/trivy_{TRIVY_VERSION}_Linux-64bit.deb
$ sudo dpkg -i trivy_{TRIVY_VERSION}_Linux-64bit.deb
$ trivy image [YOUR_IMAGE_NAME]
```

</details>

<br/><br/>

### 示例报告 – Docker扫描结果 (由Anchore提供)

![Report examples](../../assets/images/anchore-report.png "Docker scan report")
