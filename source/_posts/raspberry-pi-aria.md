---
title: 在树莓派上配置下载机
date: 2018-08-25 22:16:14
tags: 
- Raspberry
- Aria2
- Docker
categories: 
- 折腾
---

最近看美剧，为了专心享受美剧的快乐避免烦人的下载保存，

想到了把手头的一个树莓派和一个移动硬盘配置成下载机。

这样，客厅的小米盒子就可以通过局域网直接播放下载完的视频。

<!-- more -->

## 硬件准备

1. 把移动硬盘插到树莓派上, 在树莓派中是`/dev/sda1`

2. 通过网线将树莓派连接到路由器上

3. 在路由器的DHCP配置中将树莓派的IP配置为固定IP:**192.168.31.121**


## 安装Docker

在shell中执行

```sh
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh --mirror Aliyun
```

## 配置存储目录

### 建立下载目录

```
cd ~
mkdir nas
cd nas 
mkdir part0
```

### 挂载移动硬盘

在`/etc/fstable`中添加下面的一行

```
/dev/sda1   /home/pi/nas/part0/     ext4    defaults    0   2
```

在shell中执行

```
mount -a
```

## 配置`Aria2`

### 安装镜像

建立配置文件目录

```
cd ~
mkdir aria2-conf
```

启动镜像

```shell
docker run -d --name aria2-ariang-pi \
-p 6800:6800 -p 80:80 -p 8080:8080 \
-v /home/pi/nas/part0:/aria2/downloads \
-v /home/pi/aria2-conf:/aria2/conf \
-e SECRET=123 huangzulin/aria2-ariang-pi #RPC服务认证密码
```

执行完毕后，在浏览器输入

http://192.168.31.121 访问`Aria-Ng`的前端页面

http://192.168.31.121:8080 访问文件管理

> 如果提示连接错误，到`AriaNg Setting` - `RPC`选项卡中将`Aria2 RPC Secret Token`设置为上面的`123`


## 配置SMB服务器

```
sudo docker run -it -p 139:139 -p 445:445 \
       -p 137:137/udp \
       -p 138:138/udp \
       -v  /home/pi/nas/part0:/downloads \
       -d --name="samba" dperson/samba:rpi \
       -u "pi;pi" \ #用户名:密码
       -s "downloads;/downloads;yes;no;no;all;pi"  # 显示名称;映射路径,跟 第四行后面的/downloads对应
       -g "extensions = no" \
       -n \
       -W \
       -S 
```

执行完后可以通过smb://192.168.31.121 访问文件服务， 并且客厅的小米盒子也可以发现并访问