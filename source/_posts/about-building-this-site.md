
---
title: 关于这个博客的搭建
date: 2020-02-08
toc: true
categories:
  - Hexo
tags:
  - Blog
  - Hexo
  - Travis CI
---

目前站名叫 Fizzy - Logbook，打算用来记录一些折腾过程中的闲散杂事。

这是我在这里写的第一篇博客，我决定记录一下它是怎么来的。


<!-- more -->

简单来说，Fizzy - Logbook 由 Hexo 搭建，托管于GitHub ，使用 Travis CI 自动构建。
```mermaid
graph LR
  R(GitHub Repo)
  T(Travis CI)
  P(GitHub Pages)
  R --push?--> T
  T --hexo g & hexo d-->P
```



##  Hexo

[Hexo](https://hexo.io/) 是一个基于 Node.js 的开源博客框架，快速、简洁且高效。

```dockerfile
FROM node:13-alpine
 
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
    apk add --no-cache git python && \
    npm install -g cnpm --registry=https://registry.npm.taobao.org && \
    cnpm install hexo-cli -g

WORKDIR /blog

EXPOSE 4000
```

执行 `docker image build -t hexo:base .` 构建基础镜像

```dockerfile
FROM hexo:base

RUN hexo init
```

执行 `docker image build -t hexo:init .` 构建镜像

执行 ` docker container run --rm -it -p 4000:4000 hexo:init ash` 创建容器。

用 `hexo s` 命令启动 server，在浏览器访问 `localhost:4000` 查看网站效果。

此时可以对源码根目录下的 `_config.yml` 进行编辑，对网站进行初步的定制，也可寻找中意的主题替换初始主题。随时用 `hexo s` 观看效果，如果不知道哪搞坏了，重建一个容器重新弄就好。

~~Fizzy - Logbook 使用了 [polorbear 主题](https://github.com/frostfan/hexo-theme-polarbear)。[调整了文章链接格式](https://github.com/fizzybreezy/fizzybreezy.github.io/commit/4a3b239b5545a61c1892f95867dbcb82f8e1300f)并[启用了分类边栏](https://github.com/fizzybreezy/fizzybreezy.github.io/commit/bf5316c286c1acb2feb8b8069ebf0431eb2cb5b4)。~~

~~值得注意的是，该主题（包括其他一些主题）需要安装包 `hexo-render-sass`，该包的依赖 `node-sass` 在安装过程中下载一个二进制文件，由于墙内网络原因下载经常失败，[解决方案](https://github.com/lmk123/blog/issues/28)在此。~~

Fizzy - Logbook 使用了 [unnamed42](https://unnamed42.github.io/) 修改的 [kunkka](https://github.com/unnamed42/hexo-theme-kunkka)主题，该主题包含了目录和搜索功能，稍加配置即可达到很棒的效果。感谢 unnamed42 和 原作者 [iMuFeng](https://mufeng.me/) 的工作。



## GitHub Pages

[GitHub Pages](https://pages.github.com/) 是一项静态站点托管服务，它直接从 GitHub 上的仓库获取网站源码，通过构建过程运行文件，然后发布网站。

首先在 GitHub 建立一个名为 username.github.io 的仓库，其中 username 是自己的用户名。

然后将上一步的博客源码推入仓库的 hexo 分支。

```bash
cd /blog
git init
git checkout -b hexo
git add --all
git commit -m 'first commit'
git remote add origin https://github.com/username/username.github.io
git push -u origin hexo
```

在 `.gitignore` 中加入以下内容：

```
node_modules/
tmp/
*.log
.idea/
yarn.lock
package-lock.json
.nyc_output/
```



## Travis CI

[Travis CI](https://travis-ci.org/)提供持续集成服务（Continuous Integration, CI），通过绑定 GitHub仓库，在每次代码发生变动时提供环境进行构建和测试。

在这里，每当博客的源码发生变动（调整页面、写新post）时，Travis CI 会根据配置文件 `.travis.yml` 自动生成、部署网页。

使用Travis CI 自动部署 GitHub Pages，需要通过 token 使其拥有对仓库的推送权限。在 GitHub 的 Settings / Developer Settings / Personal Access Tokens 页面中新建 token，赋予 repo 的全部权限即可。注意保护生成的token。

### 配置 Travis CI 服务

接下来使用 GitHub 账户登录 Travis CI，进行以下几步操作：

1. 打开博客源码仓库的同步开关

2. 在设置中关闭 “Build pushed pull requests”

3. 加入变量 “GH_TOKEN"，值为之前生成的token，注意关闭 ”DISPLAY VALUE IN BUILD LOG“ 选项。

至此 Travis CI 服务配置完毕。

### 配置自动构建

回到博客源码目录，

创建 `.travis.yml` ，加入以下内容：

```yaml
language: node_js
sudo: required
node_js: --lts

cache:
  directories:
    - node_modules

branches:
  only:
    - hexo 

before_install:
  - npm install -g hexo-cli

install:
  - npm install
  - npm install hexo-deployer-git --save

script:
  - hexo clean
  - hexo generate

after_script:
  - git config user.name "fizzybreezy"
  - git config user.email "shenxingjian@live.com"
  - sed -i "s/gh_token/${GH_TOKEN}/g" ./_config.yml
  - hexo deploy  
```

这个文件定义了自动构建的流程，使用时注意替换 git 的参数。

在配置文件 `_config.yml` 中加入以下部分：

```yaml
deploy:
  type: git
  repo: https://gh_token@github.com/fizzybreezy/fizzybreezy.github.io.git
  branch: master
```

这部分定义了自动构建时 `hexo deploy` 命令的执行方式，其中的 gh_token 部分会在编译时会被 sed 替换为访问GitHub仓库的 token 。

### 触发第一次构建

将以上修改 `git push` ，即可触发 Travis CI 的自动构建，不出错的话，master 分支会出现自动部署的源码，此时博客也可以通过 `username.github.io` 访问了。



## 对博客进行修改

现在博客已经上线，发布新文章只需三步：

1. `git pull`
2. 用Markdown进行编写
3. `git push`

如果要对博客的主题、样式等进行修改，需要用到 Hexo 环境。

```dockerfile
FROM hexo:base

RUN git clone -b hexo https://github.com/fizzybreezy/fizzybreezy.github.io /blog && \
    cnpm install && \
    hexo clean

EXPOSE 4000
```

对修改后的博客源码 `git push` 即可。



## 参考

[Hexo遇上Travis-CI：可能是最通俗易懂的自动发布博客图文教程](https://juejin.im/post/5a1fa30c6fb9a045263b5d2a)








