---
layout: post
title: 'OpenWrt基本安装与配置'
subtitle: 'OpenWrt的初步安装及基本配置'
date: 2020-02-21
categories: 技术
tags: OpenWrt 软路由 安装
---

## 安装OpenWrt

### 准备

官网：[https://openwrt.org](https://openwrt.org)

软路由：一台主机

工具：U盘

微PE工具箱：[http://www.wepe.com.cn](http://www.wepe.com.cn)

写盘工具：[https://m0n0.ch/wall/physdiskwrite.php](https://m0n0.ch/wall/physdiskwrite.php)

### 制作安装工具

- 使用U盘制作微PE启动盘
- 下载写盘工具(physdiskwrite)和OpenWrt安装文件，OpenWrt安装文件根据自己的软路由配置下载合适的版本，以64位x86主机为例，一般下载combined-ext4/combined-squashfs的版本，例如[https://downloads.openwrt.org/releases/19.07.1/targets/x86/64/openwrt-19.07.1-x86-64-combined-ext4.img.gz](https://downloads.openwrt.org/releases/19.07.1/targets/x86/64/openwrt-19.07.1-x86-64-combined-ext4.img.gz)
- 将下载的写盘工具(physdiskwrite)和OpenWrt安装文件存放到制作好的微PE工具箱中，下载的OpenWrt安装文件为压缩文件需先解压为.img镜像文件

### 开始安装

- U盘插入软路由，开机启动后按F11进入BIOS，选择从U盘启动进入PE，不同主板进入的按键不同，大部分从F1-F12尝试即可
- 进入保存有写盘工具和OpenWrt安装文件的目录，右键选择进入控制台，输入如下命令，xxxxxx.img替换为你的OpenWrt安装文件，选择需要安装到的硬盘(0or1)再输入y确认
```
physdiskwrite.exe -u xxxxxx.img
```
- 关机拔出U盘后开机启动顺利进入OpenWrt系统，安装完成

