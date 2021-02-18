---
title: "[自我介绍] 《First Paper》 "
date: 2020-07-09T10:09:36+08:00
lastmod: 2020-07-09T10:09:36+08:00
draft: false
tags: ["personal"]
categories: ["personal"]
author: "Jeremy-boo"

contentCopyright: '<a rel="license noopener" href="https://en.wikipedia.org/wiki/Wikipedia:Text_of_Creative_Commons_Attribution-ShareAlike_3.0_Unported_License" target="_blank">Creative Commons Attribution-ShareAlike License</a>'

---

> 个人博客今天成立啦。

# 涉及技术
    hugo, github and even主题

# 搭建步骤

### 生成博客站点
    // 生成站点
    $ hugo new site path
    // 创建文章
    $ hugo new first.md
    // 指定目录创建文章
    $ hugo new post/about.md
### 安装皮肤
> 皮肤路径在site 的themes路径下，可以在网站下载相关主题，放在themes目录下，然后再config.toml中指定主题
### 部署
1. 本地运行
> hugo server

2. 发布
> hugo -t even

3. hugo+github pages

        git init
        git remote add origin https://github.com/Jeremy-boo/Jeremy-boo.github.io.git
        git add .
        git commit -m "first commit"
        git push -u origin master

### 其他(解决wsl2中sudoders出现问题)
        wsl -u root
        chown root:root /etc/sudoers







