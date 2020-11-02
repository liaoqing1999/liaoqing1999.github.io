---
layout: post
title: createNewFile方法java.io.IOException问题
date: 2020-11-02 10:46:09
tags: java
categories: java
---

# createNewFile方法引起的java.io.IOException问题

## 报错java.io.IOException: 系统找不到指定的路径

原因：

**createNewFile这个方法只能在一层目录下创建文件，不能跳级创建**

mkdir(s)可以创建多层不存在的目录，但无法直接创建一个file文件  最终会创建和文件名一样的文件夹

解决办法：

先获取文件的父级，再创建文件夹，最后创建文件

```
 	File parent = new File(file.getParent());
    if(!parent.exists()) {
       parent.mkdirs();
    }
    file.createNewFile();
```

## 报错java.io.IOException: 没有权限访问

原因：

1. 可能是访问的文件所在的磁盘没有权限

打开属性，属性--->安全---->编辑，然后把除完全控制的其他权限全部勾选，如图

![](java-io-IOException问题/权限控制.png) 

如果是C盘，就算设置了，也有可能报错java.io.IOException: 客户端没有所需的特权

那么考虑将文件路径存在其他盘符

2.检查访问的文件类型是否为文件（而不是目录），（如上述用mkdirs创建的就是目录）

