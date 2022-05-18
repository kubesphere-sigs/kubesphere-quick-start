# KubeSphere 3.2.1 快速入门

本仓库的文档旨在通过多个快速入门的示例帮助您了解 KubeSphere 3.2.1 的基本使用流程，带您快速上手 KubeSphere 3.2.1。

## 贡献指南

本指南为纯静态网站，使用 [mdBook](https://github.com/rust-lang/mdBook) 构建，贡献步骤如下：

### 1. 认领章节

首先到 [Issue 页面](https://github.com/kubesphere-sigs/kubesphere-quick-start/issues?q=is%3Aissue+is%3Aopen+sort%3Aupdated-desc)选择自己感兴趣章节相关的 Issue，在 Issue 下面回复认领。

### 2. Fork 该仓库

Fork 该仓库，然后 Clone 到本地，假设你的用户名是 xxx，那么 Clone 命令就是：`git clone https://github.com/xxx/kubesphere-quick-start`。

### 3. 编译并预览

为了能够在本地即时预览文档，需要先[安装 mdBook](https://rust-lang.github.io/mdBook/guide/installation.html)。更新文档之后便可以执行命令 `mdbook serve --open` 来即时预览文档。

### 4. 提交更新

确认无误后，将更新提交到自己的仓库：

```bash
$ git add .
$ git commit "xxx"
$ git push -u origin main
```

### 5. 提交 PR

在浏览器中打开你的仓库页面，选择『Pull requests』--> 『New pull request』。

![](https://pek3b.qingstor.com/kubesphere-community/images/202205182220637.png)

确认无误后，点击『Create pull request』即可提交 PR。

![](https://pek3b.qingstor.com/kubesphere-community/images/202205182220990.png)
