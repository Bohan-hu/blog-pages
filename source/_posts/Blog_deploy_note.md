---
title: 使用Hexo搭建个人博客过程记录
date: 2020/8/21 20:00:00
tag: 
	- 技术
	- 环境配置
category:
	- 笔记
---

本文主要记录使用Hexo+Next主题搭建博客，并部署在Github Pages，使之能够通过个人域名访问的全过程。

## 准备

### 环境配置

首先需要安装node环境，安装完毕后，在命令行中输入：
```
npm install hexo-cli -g
```
安装完毕后，进入某个目录，在命令行中输入
```
hexo init blog
cd blog
npm install
```
然后安装Next主题：
```
npm install hexo-theme-next
```
以上命令会将next主题依赖添加到`package.json`文件中，方便后续在Github CI上部署。


### 修改Hexo配置文件

此后，修改`_config.yml`文件，找到`theme`这一行：

```
theme: next
```

根据hexo文档中的指导，修改其他的内容。

https://hexo.io/zh-cn/docs/configuration

### 修改Next配置文件

将配置文件复制到根目录：

```
cp node_modules/hexo-theme-next/_config.yml _config.next.yml
```

具体设置参考Next文档。

https://theme-next.js.org/docs/theme-settings/

根据文档，添加about, tags, categories页面。

## 部署到Github

### 建立Github仓库

在Github中建立一个公开仓库，命名为`<name>.github.io`。

在Hexo配置文件中，修改最后三行：

```
deploy:
  type: git
  repository: git@github.com:Bohan-hu/Bohan-hu.github.io.git
  branch: master
```

### 建立源代码仓库

在本地刚刚建立的Blog文件夹中，新建`.github\workflows\deploy.yml`，用于CI自动部署。

![image-20200823133939523](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823133939523.png)

```
name: CI

on:
  push:
    branches:
      - master

env:
  GIT_USER: Bohan-Hu
  GIT_EMAIL: bohan.hu@ieee.org
  DEPLOY_REPO: Bohan-hu/Bohan-hu.github.io
  DEPLOY_BRANCH: master

jobs:
  build:
    name: Build on node ${{ matrix.node_version }} and ${{ matrix.os }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        os: [ubuntu-latest]
        node_version: [12.x]

    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Checkout deploy repo
        uses: actions/checkout@v2
        with:
          repository: ${{ env.DEPLOY_REPO }}
          ref: ${{ env.DEPLOY_BRANCH }}
          path: .deploy_git

      - name: Use Node.js ${{ matrix.node_version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node_version }}

      - name: Configuration environment
        env:
          HEXO_DEPLOY_PRI: ${{secrets.HEXO_DEPLOY_PRI}}
        run: |
          sudo timedatectl set-timezone "Asia/Shanghai"
          mkdir -p ~/.ssh/
          echo "$HEXO_DEPLOY_PRI" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          git config --global user.name $GIT_USER
          git config --global user.email $GIT_EMAIL

      - name: Install dependencies
        run: |
          npm install

      - name: Deploy hexo
        run: |
          npm run deploy
```

### 配置仓库权限

此后，在本地生成一对公私密钥。

```
ssh-keygen -t rsa -f
```

在Github中，对于网站源代码仓库，将`private key`在源文件仓库中加入到`HEXO_DEPLOY_PRI`变量中，将`public key`放入网站的文件仓库中的`Deploy keys`中。

在Github新建一个私有仓库，将网站源文件（即Hexo刚刚建立的文件夹）推上去，观察CI自动部署的情况。

## 使用个人域名访问

域名的服务商此处选择Godaddy，并使用Cloudflare提供的解析服务。

### 设置Cloudflare

进入Cloudflare，添加刚刚注册的域名，并进入DNS选项卡，添加A记录。

![image-20200823134133941](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823134133941.png)

设置完毕后，需要一定时间生效，此时，进入SSL/TTS-边缘证书-始终使用HTTPS：

![image-20200823134157565](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823134157565.png)

### 设置Godaddy

在Godaddy注册域名后，在域名解析的选项中，设置为Cloudflare的域名服务器。

![image-20200823134210525](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823134210525.png)

### 添加CNAME

在博客文件夹的`source`子目录中，新建`CNAME`文件，用大写写入自己绑定到Github的域名。

![image-20200823134225476](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823134225476.png)

至此，可以通过域名访问自己的博客主页了。

## 设置图床

由于Hexo添加图片较为麻烦，建议使用图床，国内使用Gitee作为图床是比较好的选择。

8.23更新：由于Gitee对于微信浏览器的UA做了限制，所以现在采用Github+JsDelivr的方案。

### 安装Picgo并建立Github仓库

在Picgo官网下载安装Picgo后，在Github中建立一个新的公开仓库，同时需要使用Readme初始化，以新建Master分支。

### 配置插件

注意链接格式，前缀为`https://cdn.jsdelivr.net/gh/用户名/仓库名`

![image-20200823134358958](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823134358958.png)

### 配置Typora

![image-20200823134508984](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823134508984.png)

如果出现Fetch Error，更改监听端口为36677.

![image-20200823134519328](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823134519328.png)