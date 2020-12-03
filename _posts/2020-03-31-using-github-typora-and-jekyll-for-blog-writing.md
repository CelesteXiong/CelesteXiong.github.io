---
layout: post
title: Build gitpage with github, jekyll and typora
date: 2020-03-31 20:09 +0800
categories: 博客
tags: 媒体
---

# 命令行

进入 Gemfile 文件所在的目录
```shell
bundle exe jekyll build
git add .
git commit -m "blah"
git push
```

# 添加图片

1. 在media中建立文件夹存储文件d: media/title/
2. md中图片地址: "/../../media/title/img.png"
3. 将图片放入d即可

