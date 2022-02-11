+++
title="使用静态网站写博客的最佳实践"
description="SSG+Github存储源码+Github Action自动编译+Github Pages发布网站"
date=2022-02-11T20:37:06+08:00

[taxonomies]
categories = ["BestPractices"]
tags = ["best practice", "tip"]

[extra]
toc = true
comments = true
+++

我尝试过hexo，用过hugo，最终转向zola这个完全还没发展起来的SSG。这篇文章记录下使用SSG写博客的最佳实践。

## 主题选择

主题选择驱使我从hugo迁移到zola。我对主题的要求非常苛刻：

1. 积极维护中，并且有庞大的用户群体。
2. 样式简洁美观，我倾向于fluent design。
3. 功能简单但不单调
    1. 支持latex渲染
    2. 支持toc
    3. （可选）支持评论
    4. （可选）支持admonition
4. 我极其讨厌复杂庞大的可选项（没错，我也讨厌kde），所以配置文件越精简越好。

在hugo中，我从Loveit（似乎停止维护了）搬迁到papermod（非常惊艳，但是公式渲染需要html代码配置，并且渲染异常），再尝试了stack（太花哨）、terminal（风格不喜）、MemE（配置项过于庞大），我逃离到了zola。再一番抉择后发现了现在正在使用的主题[DeepThought](https://github.com/RatanShreshtha/DeepThought)，它是zola中几乎最流行的主题，界面简洁优美，配置精简，原生支持渲染，toc可选，支持评论，唯一的缺点就是缺少中文搜索的支持。


## 博客源文本存储

首先，博客的源码肯定是要纳入版本管理的，理所当然需要给它们创建一个仓库。

这里有一些约定：

1. 只有网站本身使用的图片（图标、头像）放到static文件夹中，其他博文图片全部使用图床的方式+markdown链接引用。
2. 画图尽量使用mermaid、plantuml、draw.io这些纯文图图片来记录。draw.io同样存储于图床。

## 博客静态网站存储

想要使用Github Pages，需要创建一个仓库用于存储build出来的静态网站文件。

## 博客编译自动化

现在源码在仓库A中，编译的静态网站在仓库B中，每次写完博文都需要手动build下然后推送仓库B、仓库A，这当然无法忍受。

于是可以利用Github Action配置流水线，每次提交到仓库A都会触发流水线，自动编译发布仓库B。

这里涉及到仓库A的编译发布流水线提交git到仓库B的权限问题，github可以使用token来解决。

但是我更倾向于使用仓库级密钥：

通过以下命令创建一对SSh key：

```bash
ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""
```

将生成的gh-pages.pub公钥配置仓库B，允许写入。并将生成的gh-pages私钥配置仓库A的私有变量，我将它命名为`ZOLA`。

这是我zola的配置，`.github/workflows/deploy-website.yml`：

```yaml
name: CI
on:
  push:
    branches: [ main ]
  workflow_dispatch:
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout the code
        uses: actions/checkout@master
        with:
          submodules: true
      - name: Build Static Website
        uses: shalzz/zola-deploy-action@master
        env:
          BUILD_ONLY: true
          BUILD_DIR: .
      - name: Deploy Static Website
        uses: peaceiris/actions-gh-pages@v3
        with: 
         deploy_key: ${{ secrets.ZOLA }}
         external_repository: oliverdding/oliverdding.github.io
         publish_branch: main
         publish_dir: ./public
         user_name: 'github-actions[bot]'
         user_email: 'github-actions[bot]@users.noreply.github.com'
         full_commit_message: ${{ github.event.head_commit.message }}
```

shalzz/zola-deploy-action也可以直接推送，但是我想K.I.S.S些，就职责分离了。hexo、hugo大体相同，只要改动第二部build即可。
