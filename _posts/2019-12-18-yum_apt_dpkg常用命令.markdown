---
layout: post
title:  "yum/apt/dpkg 常用命令"
date:   2019-12-18 15:02:00 +0800

---

1、centos/redhat下查看某个文件或命令属于哪个rpm包：
```
$ yum provides /etc/passwd
或者
$ rpm -qf /etc/passwd
```

2、ubuntu及衍生版：
```
sudo dpkg -S whereis
或
sudo dpkg-query -S /usr/bin/whereis
```