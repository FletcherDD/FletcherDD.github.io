---
layout: post
title: 'OpenWrt安装V2Ray'
subtitle: '在OpenWrt系统安装V2Ray，并实现透明代理'
date: 2020-02-22
categories: 技术
tags: OpenWrt V2Ray 软路由 透明代理 代理
---

官方编译的OpenWrt没有V2Ray的安装包，OpenWrt也可以视为一个Linux系统，故可以参考V2Ray的Linux安装脚本

## 安装V2Ray

### 下载安装脚本

```
$ wget https://install.direct/go.sh
```
然而OpenWrt的wget不支持SSL需要安装相关模块才能下载https链接，也可以直接打开[https://install.direct/go.sh](https://install.direct/go.sh)，将此脚本复制编辑到OpenWrt本地
```
$ vim go.sh

#!/bin/bash

# This file is accessible as https://install.direct/go.sh
# Original source is located at github.com/v2ray/v2ray-core/release/install-release.sh

# If not specify, default meaning of return value:
# 0: Success
# 1: System error
# 2: Application error
# 3: Network error

# CLI arguments
PROXY=''
HELP=''
FORCE=''

......
......

```

### 运行安装脚本

安装脚本使用bash运行，而OpenWrt使用的是ash，虽然ash和bash兼容但实测ash无法运行此shell脚本，所以需要先安装bash，以及其他一些模块
```
$ opkg update
$ opkg install bash
$ opkg install curl
$ opkg install unzip
```
安装脚本会从github下载安装包，如果你的OpenWrt无法访问github，找到go.sh中
```
DIST_SRC='github'
```
修改为
```
DIST_SRC='jsdelivr'
```
开始安装
```
$ /bin/bash go.sh
Installing V2Ray v4.22.1 on x86_64
Donwloading V2Ray: https://cdn.jsdelivr.net/gh/v2ray/dist/v2ray-linux-64.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 11.6M  100 11.6M    0     0  24.5M      0 --:--:-- --:--:-- --:--:-- 25.2M
Archive:  /tmp/v2ray/v2ray.zip
  inflating: /usr/bin/v2ray/geoip.dat
  inflating: /usr/bin/v2ray/geosite.dat
  inflating: /usr/bin/v2ray/v2ctl
  inflating: /usr/bin/v2ray/v2ray
PORT:19732
UUID:330e0c4a-36ca-4fe6-a071-3eaa3fcd1a92
V2Ray v4.22.1 is installed.
```

## config.json配置

根据自己的V2Ray代理服务器配置来配置客户端
```
$ vim /etc/v2ray/config.json
```
下面是一个WebSocket+TLS+Web的VMess协议的客户端配置，设置了一个sock入口代理和一个http入口代理。默认直连，非国内域名则走代理，bt流量直连
```
{
	"inbounds": [
		{
			"port": 10808,
			"protocol": "socks",
			"settings": {
				"auth": "noauth",
				"udp": true
			}
		},
		{
			"port":20808,
			"protocol":"http",
			"settings":{
				"accounts": [
					{
						"user": "your-username",
						"pass": "your-password"
					}
				],
				"allowTransparent": false
			}
		}
	],
	"outbounds": [
		{
			"tag": "direct",
			"protocol": "freedom"
		},
		{
			"tag": "proxy",
			"protocol": "vmess",
			"settings": {
				"vnext": [
					{
						"address": "your-v2ray-server-domain",
						"port": 443,
						"users": [
							{
								"id": "your-v2ray-server-vmess-user-id",
								"alterId": 4,
								"security": "auto"
							}
						]
					}
				]
			},
			"streamSettings": {
				"network": "ws",
				"security": "tls",
				"wsSettings": {
					"path": "your-v2ray-server-path"
				}
			},
			"mux": {
				"enabled": true
			}
		}
	],
	"routing": {
		"domainStrategy": "IPOnDemand",
		"rules": [
			{
				"type": "field",
				"outboundTag": "proxy",
				"domain": [
					"geosite:geolocation-!cn"
				]
			},
			{
				"type": "field",
				"outboundTag": "direct",
				"protocol": [
					"bittorrent"
				]
			}
		]
	}
}
```
测试配置文件
```
$ /usr/bin/v2ray/v2ray -test -config=/etc/v2ray/config.json
V2Ray 4.22.1 (V2Fly, a community-driven edition of V2Ray.) Custom (go1.13.5 linux/amd64)
A unified platform for anti-censorship.
Configuration OK.
```

## 启动服务

### 运行V2Ray

目前运行此V2Ray安装脚本后，并没有给OpenWrt系统添加V2Ray的启动服务，自己手动写启动脚本也是一种方法

我们也可以在控制台运行V2Ray
```
$ /usr/bin/v2ray/v2ray -config=/etc/v2ray/config.json
```
此命令为前台运行，我们需要将其转为后台运行，可以使用nohup/setsid/&或者screen等方法

### nohup后台启动

- 安装nohup模块
```
opkg update
opkg install coreutils-nohup
```
- 运行脚本
```
$ nohup /usr/bin/v2ray/v2ray -config=/etc/v2ray/config.json > /dev/null 2>&1 &
```
- 查看V2Ray进程
```
$ ps |grep v2ray
16263 root      117m S    /usr/bin/v2ray/v2ray -config=/etc/v2ray/config.json
16865 root      1080 S    grep v2ray
```

### 开机启动

设置开机启动，将上述命令写入/etc/rc.local即可
```
$ vim /etc/rc.local 

nohup /usr/bin/v2ray/v2ray -config=/etc/v2ray/config.json > /dev/null 2>&1 &
exit 0
```

### 代理测试

- OpenWrt路由器端测试

```
$ curl -x socks5://127.0.0.1:10808 google.com

<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

```
$ curl -x http://your-http-proxy-username:your-http-proxy-password@127.0.0.1:20808 google.com

<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

- 客户端测试

  客户端连接OpenWrt路由器，浏览器配置sock或者http代理，代理服务器为路由器域名或IP地址


## 透明代理

客户端仍然需要设置OpenWrt为代理服务器才能走V2Ray，使用透明代理可以实现客户端无需任何配置即可通过V2Ray代理上网

以下通过iptables(OpenWrt) + dns(V2Ray)使用TPROXY方式来实现透明代理

### V2Ray配置

- inbounds添加任意门，标记transparent，开放端口12345（举个栗子）

```
{
	"tag":"transparent",
	"port": 12345,
	"protocol": "dokodemo-door",
	"settings": {
		"network": "tcp,udp",
		"followRedirect": true
	},
	"sniffing": {
		"enabled": true,
		"destOverride": [
			"http",
			"tls"
		]
	},
	"streamSettings": {
		"sockopt": {
			"tproxy": "tproxy"
		}
	}
}
```

- outbounds直连自由门设置domainStrategy为UseIP

```
{
	"tag": "direct",
	"protocol": "freedom",
	"settings": {
		"domainStrategy": "UseIP"
	},
	"streamSettings": {
		"sockopt": {
			"mark": 255
		}
	}
}
```

- 配置V2Ray内置DNS

```
"dns": {
	"servers": [
		{
			"address": "223.5.5.5", 
			"port": 53,
			"domains": [
				"geosite:cn",
				"ntp.org",   
				"your-v2ray-server-domain" 
			]
		},
		"114.114.114.114",
		{
			"address": "8.8.8.8", 
			"port": 53,
			"domains": [
				"geosite:geolocation-!cn",
				"ntp.org",   
				"your-v2ray-server-domain" 
			]
		},
		"1.1.1.1", 
		"localhost"
	]
}
```

- 路由设置domainStrategy为IPIfNonMatch/IPOnDemand，并配置DNS路由规则

```
"routing": {
	"domainStrategy": "IPOnDemand",
	"rules": [
		{
			"type": "field",
			"inboundTag": [
				"transparent"
			],
			"port": 123,
			"network": "udp",
			"outboundTag": "direct" 
		},    
		{
			"type": "field", 
			"ip": [ 
				"223.5.5.5",
				"114.114.114.114"
			],
			"outboundTag": "direct"
		},
		{
			"type": "field",
			"ip": [ 
				"8.8.8.8",
				"1.1.1.1"
			],
			"outboundTag": "proxy"
		}
	]
}
```

### OpenWrt配置

- 可能需要安装一些必要的模块（未验证）

```
opkg update
opkg install iptables-mod-tproxy
```

- 设置防火墙规则，编辑文件/etc/firewall.user，假设路由器本机IP段为192.168.0.0/16

```
ip rule add fwmark 1 table 100
ip route add local default dev lo table 100

iptables -t mangle -N V2RAY
iptables -t mangle -A V2RAY -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A V2RAY -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A V2RAY -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY -d 240.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY -d 192.168.0.0/16 -p tcp -j RETURN
iptables -t mangle -A V2RAY -d 192.168.0.0/16 -p udp ! --dport 53 -j RETURN
iptables -t mangle -A V2RAY -p udp -j TPROXY --on-port 12345 --tproxy-mark 1
iptables -t mangle -A V2RAY -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1
iptables -t mangle -A PREROUTING -j V2RAY
```

### 一些问题

- 只能代理连接到路由器的终端，而本机无法实现透明代理（所以设置了http和socks入口）

- 路由器重启后需要再手动重启下防火墙，否则终端连接不到外网，原因未知

- error日志中大量出现以下警告和错误

```
[Error] v2ray.com/core/app/dns: UDP:223.5.5.5:53 cannot find the pending request
[Warning] v2ray.com/core/transport/internet/tcp: failed to accepted raw connections > accept tcp [::]:12345: accept4: too many open files
```

扩大文件句柄限制，之后需要重启V2Ray，cannot find the pending request的错误仍然无法解决

```
ulimit -n 65535
```



- 参考文档
  - [白话文教程](https://toutyrater.github.io)
  - [新白话文教程](https://guide.v2fly.org)
  - [用户手册](https://www.v2ray.com)
  - [项目地址](https://github.com/v2ray/v2ray-core)
