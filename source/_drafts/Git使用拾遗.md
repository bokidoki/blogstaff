---
title: Git使用拾遗
tags:
categories: git
date: 2019/09/20
img:
---

## 前言

Git是很好用版本管理工具，但是专门花时间去学习，没有必要。在日常工作中通常会遇到各种使用问题，不会就只能去Google了😂，在此记录一下自己平时在使用Git时遇到的小问题，这样也算是间接的去学习了吧。

## 常用网站

- [Git-scm](https://git-scm.com/about)
- [Git中文手册](https://git-reference.readthedocs.io/zh_CN/latest/)
- [廖雪峰的Git教程](https://www.liaoxuefeng.com/wiki/896043488029600)
  
## 日常问题记录

- git 删除Master分支
  - 进入git项目主页---选择右上角的设置---Edit Project
  - 找到Default Branch---选择temp为默认分支
  - 找到Visibility Level，将原来设置为private的修改为public
  - 重新操作步骤3的删除master分支，问题解决

- git删除远程目录上的目录/文件
  - git rm -r -n --cached 文件/目录 预览要删除的文件or目录
  - git rm -r --cached 文件/目录
