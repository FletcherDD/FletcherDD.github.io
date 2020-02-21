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

- 官网：[https://openwrt.org](https://openwrt.org)
- 软路由：一台主机
- 工具：U盘
- 微PE工具箱：[http://www.wepe.com.cn](http://www.wepe.com.cn)
- 写盘工具：[https://m0n0.ch/wall/physdiskwrite.php](https://m0n0.ch/wall/physdiskwrite.php)

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

## 基本配置

### 网络配置与安装语言包

- 使用官方安装包安装的OpenWrt默认域名为openwrt.lan默认IP为192.168.1.1默认开启ssh用户为root无密码，使用以下命令查看wan和lan的配置网口
```
cat /etc/config/network
```
- 若上层路由IP已经是192.168.1.1则需要修改IP
```
vim /etc/config/network
```
- 将软路由wan口与可上网的路由器lan口连接，确认软路由已联网
- 系统web界面为英文，可安装中文语言包：
```
opkg update
opkg install luci-i18n-base-zh-cn
```

### 系统空间扩容

- 系统初始空间仅256M，如果安装的硬盘空间足够大，则仍有大量空闲空间未使用，而一般分区工具可能无法对此系统空间扩容，官方使用OverlayFS来实现扩展软件安装空间，使用时最好了解OverlayFS的特性再决定是否使用此方式来扩容
- [官方文档](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)
- 安装必要的软件：
```
opkg update
opkg install fdisk
opkg install block-mount
```
- 使用fdisk磁盘
```
fdisk -l
```
- 使用fdisk新建分区，输入n回车新建分区，按提示新建合适的分区，最后输入w回车保存
```
fdisk /dev/sda
```
- 格式化分区（这里新建了分区/dev/sda3，系统安装的文件格式为ext4）
```
mkfs.ext4 /dev/sda3
```
- 挂载overlayfs
```
DEVICE="/dev/sda3"
eval $(block info "${DEVICE}" | grep -o -e "UUID=\S*")
uci -q delete fstab.overlay
uci set fstab.overlay="mount"
uci set fstab.overlay.uuid="${UUID}"
uci set fstab.overlay.target="/overlay"
uci commit fstab
```
- 重启
```
reboot
```
- 扩容前后对比

![扩容前](/assets/img/mount_before.png)

![扩容后](/assets/img/mount_after.png)

### 网络配置

#### 管理密码

- 将电脑网口与软路由lan口连接，浏览器打开http://openwrt.lan ，无需密码直接登录。[LuCI](http://openwrt.lan) → 系统 → 管理权，修改软路由登录密码

#### 多LAN口配置

- [LuCI](http://openwrt.lan) → 网络 → 接口，点击编辑LAN → 物理设置 → 接口，选择需要设置为LAN口的网络接口 → 保存 → 保存并应用

#### 拨号上网

- 默认设置桥接上级路由，如果需要用此软路由来拨号上网则需要设置
- [LuCI](http://openwrt.lan) → 网络 → 接口，点击编辑WAN → 基本设置 → 协议PPPOE → 切换协议 → 输入用户名密码 → 保存 → 保存并应用

