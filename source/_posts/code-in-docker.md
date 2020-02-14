---
title: 如何用VS Code优雅的在容器中开发
date: 2020-02-11
toc: false
categories:
  - Coding
tags:
  - VS Code
  - Docker
---

最近干了不少零碎事，写论文，搭博客，填表爬虫。~~为了优雅~~，这些杂七乱八的活用到的开发环境自然要容器化，既能互相隔离，又方便一言不合开新容器从头再来。

<!-- more -->

根据 Dockerfile 构建包含开发环境的镜像，利用 VS Code 的 Remote - Container 插件创建容器、挂载源码目录并在容器内安装 VS Code Server，实现用容器内的开发环境处理本地的代码。

![architecture-containers](/images/architecture-containers.png)

:point_up_2:就是这么个意思。

首先在**源码目录**内新建一个 `.devcontainer` 文件夹，里面放一个 `Dockerfile` 和一个 `devcontaniner.json` 。根据源码内容配置文件内容，以我写论文的 TeX Live 环境为例

`Dockerfile`

```Dockerfile
FROM alpine

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
    apk add texlive-full libstdc++ git
```

其中git和libstdc++是为 VS Code 准备的。

`devcontainer.json`

```json
{
  "name": "TeX Live",
  "dockerFile": "Dockerfile",
  "settings": {
    "terminal.integrated.shell.linux": "/bin/ash"
    //settings.json里的其他配置， 太长不写
  },
  "extensions": [james-yu.latex-workshop], // 插件标识
}
```

安装 [Remote - Container](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) 插件，点击左下角绿色图标，在弹出对话框中选择 “ Remote-Containers: Open Folder in Container ...” 选项，打开刚才搞的**源码目录**，VS Code 就会自动构建镜像、新建容器并进行初始化配置，自动化程度很高。

默认情况下，VS Code 将在新容器的文件系统内安装 VS Code Server 和插件，如果经常需要开新容器，如**重建镜像**或**使用同一镜像打开不同文件夹**，则可以将其在数据卷（volume）中，并在每次创建新容器时挂载，虽然会降低性能，但可节约时间。

在 `devcontainer.json` 中添加以下部分：

```json
"mounts": [
    // 二选一
    // 仅保留插件
    "source=unique-vol-name-here,target=/root/.vscode-server/extensions,type=volume",
    // 保留插件和VS Code Server
    "source=unique-vol-name-here,target=/root/.vscode-server,type=volume",
]
```

如果想重新安装 VS Code Server和插件，删掉容器卷即可。

```bash
docker volume rm unique-vol-name-here
```

## 参考

[Developing inside a Container](https://code.visualstudio.com/docs/remote/containers)
[Advanced Container Configuration](https://code.visualstudio.com/docs/remote/containers-advanced)
