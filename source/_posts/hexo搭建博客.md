---
title: hexo搭建博客
date: 2020-11-28 19:43:10
tags:
---
# Hexo搭建个人博客

Hexo是一个快速、简洁且高效的博客框架。Hexo使用[Markdown](https://daringfireball.net/projects/markdown/)（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。本篇文章主要介绍如何使用<b>Hexo + github</b> 搭建个人博客。

## Git仓库配置

### 创建仓库

登录[github](http://github.com),创建仓库。注意：username为你的github昵称
{% asset_img 创建仓库.png %}

### 本地Git配置

- 配置git用户名 
```bash
git config --global user.name "username"
```

- 配置git邮箱
```bash
git config --global user.email "useremail"
```

### 配置SSH Keys

- 生成本地SSH Keys 
  ```bash
  ssh-keygen -t rsa -C "youremail"
  ```
  按照提示，下一步直至提示生成完毕。
- 按照下面的路径去复制id_rsa.pub的内容
  ```bash
  cd /.ssh
  cat id_rsa.pub
  ```
- [去github中配置ssh key](https://github.com/settings/keys),执行 New SSH Key
  {% asset_img 配置sshkey.png %}
  title随便写，然后粘贴刚才复制的id_rsa.pub内容到Key里

## 安装Hexo

### hexo前置

- [NodeJS](https://nodejs.org/en/)(Node版本>=10.13)
- [Git](https://git-scm.com/)

### 安装

- #### 全局安装

```bash
npm install -g hexo-cli
```

- #### 局部安装(作为一个依赖)

``` bash
npm install hexo
```
之后的使用可以采用以下两种方式:
1. `npx hexo <command>`
2. 将 Hexo 所在的目录下的 node_modules 添加到环境变量之中(`echo 'PATH="$PATH:./node_modules/.bin"' >> ~/.profile`)即可直接使用 hexo <command>

## 建站

### 命令

- `hexo init <folder>` 初始化一个项目
- `hexo new <layout> <title>` 创建一篇文章
- `hexo generate` 生成静态部署文件
- `hexo publish <layout> <title>` 把草稿发布为正式文件
- `hexo server` 启动本地服务，默认访问地址：http://localhost:4000
- `hexo deploy` 部署文件到服务器
- `hexo clean` 清除缓存文件
- `hexo list` 列出博客资料
- `hexo version` 展示hexo版本

### 初始化项目

```bash
hexo init blog
cd blog
npm i
```

### 文件结构

- _config.yml 博客的配置信息
- package.json 项目的配置信息
- scaffolds 模版文件
- source 资源文件
- themes 主题文件

## 部署

hexo提供了快速方便的一键部署功能，让你只需要一条命令就能够将文件部署到静态服务器上。
```bash
hexo deploy
```

### 部署前置

部署前我们需要安装[hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)
```
npm install hexo-deployer-git --save
```

### 项目配置

部署前我们需要去配置文件中配置一下，让hexo知道部署到哪里
```bash
deploy:
  type: 'git'
  repo: 'https://github.com/<username>/<username>.github.io.git'
  branch: '<branch>'
```

### git配置

进入项目配置中,`https://github.com/<username>/<username>.github.io/settings`,配置你部署的分支作为展示分支
{% asset_img GitPage配置.png %}

### 进行部署

```bash
hexo clean
hexo generate
hexo deploy
```
部署成功后，过几秒刷新以下`http://<username>.github.io`，就可以看到你的博客了。

## 参考文章
[Websites for you and your projects](https://pages.github.com/)
[hexo](https://hexo.io/zh-cn/docs/)