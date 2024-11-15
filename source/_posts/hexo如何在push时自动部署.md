---
title: hexo如何在push时自动部署到github
date: 2024-11-15 15:58:36
tags:
    - hexo
    - github
    - 自动部署
    - 博客
category:
    - 博客
---

# 介绍

传统的 hexo 部署到 github 时，需要一个专门的仓库存放静态文件，但是现在 github 提供了 actions 功能，可以直接用来编译、存放静态文件，只需要一个仓库存放 hexo 代码库就可以了

# 不足

因为 github 要求个人版如果想要使用 pages 就必须将对应的仓库公开，传统的 hexo 部署时只需要公开静态文件就可以了，但是这种方法源码也要公开，否则不能使用 pages

# 好处

写完直接 push 就可以了，然后就等 github 为你编译、上传文件了

# 创建 hexo 代码库

# 新建 actions


