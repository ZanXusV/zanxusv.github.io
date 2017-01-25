---
layout:     post
title:      "Install shadowsocks on CentOS7"
subtitle:   "Shadowsocks-libev"
date:       2017-01-19 22:17:00+0800
author:     "ZanXus"
header-img: "img/blog/header/post-bg-06.jpg"
thumbnail: /img/blog/thumbs/thumb06.png
tags: [shadowsocks,centos7]
category: [linux]
comments: true
share: true
---


## Firstly,get the latest source code:

```sh
git clone https://github.com/shadowsocks/shadowsocks-libev.git
cd shadowsocks-libev
git submodule update --init --recursive
```

## Then install prequirement to build source code:

```sh
yum install epel-release -y
yum install gcc autoconf libtool automake make openssl-devel pcre-devel asciidoc xmlto zlib-devel openssl-devel libsodium-devel udns-devel libev-devel  -y
yum -y install dnf
```

## Add yum repo `epel` for CentOS7:

```sh
cd /etc/yum.repos.d/
wget https://copr.fedorainfracloud.org/coprs/librehat/shadowsocks/repo/epel-7/librehat-shadowsocks-epel-7.repo
```

## Install `shadowsocks-libev` via `dnf`:

```sh
su -c 'dnf update'
su -c 'dnf install shadowsocks-libev'
```

## Now the ss server side installation completed.Next need to config server.

```sh
vim /etc/shadowsocks-libev/config.json
```

## The default configuration is:
```sh
{
    "server":"127.0.0.1",
    "server_port":8388,
    "local_port":1080,
    "password":"barfoo!",
    "timeout":60,
    "method":null,
}
```

## Change `server` value to true server ip,change `server_port`  and `password` as you want,but `server_port` is recommended a large number otherwise you need to start ss service with root user.`method` can be `aes-128-cfb`, `aes-192-cfb`, `aes-256-cfb`, `bf-cfb`, `cast5-cfb`, `des-cfb`, `rc4-md5`, `chacha20`, `salsa20`, `rc4`, `table`,but `aes-256-cfb` is better.

## In the `shadowsocks-libev` source folder:

```sh
cp ./rpm/SOURCES/etc/init.d/shadowsocks-libev /etc/init.d/shadowsocks-libev
chmod +x /etc/init.d/shadowsocks-libev
```

## Config ss service start on server startup:

```sh
chkconfig --add shadowsocks-libev
chkconfig shadowsocks-libev on
```

## Start the ss service:

```sh
service shadowsocks-libev start
```

## Check the service using `systemctl status shadowsocks-libev -l`,the result will like below if  ss service running well.
```sh
● shadowsocks-libev.service - Shadowsocks-libev Default Server Service
   Loaded: loaded (/usr/lib/systemd/system/shadowsocks-libev.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2017-01-19 15:19:47 UTC; 24s ago
     Docs: man:shadowsocks-libev(8)
 Main PID: 27409 (ss-server)
   CGroup: /system.slice/shadowsocks-libev.service
           └─27409 /usr/bin/ss-server -a root -c /etc/shadowsocks-libev/config.json -u

Jan 19 15:19:47 zanxus systemd[1]: Started Shadowsocks-libev Default Server Service.
Jan 19 15:19:47 zanxus systemd[1]: Starting Shadowsocks-libev Default Server Service...
Jan 19 15:19:48 zanxus ss-server[27409]: 2017-01-19 15:19:48 INFO: UDP relay enabled
Jan 19 15:19:48 zanxus ss-server[27409]: 2017-01-19 15:19:48 INFO: initializing ciphers... aes-256-cfb
Jan 19 15:19:48 zanxus ss-server[27409]: 2017-01-19 15:19:48 INFO: tcp port reuse enabled
Jan 19 15:19:48 zanxus ss-server[27409]: 2017-01-19 15:19:48 INFO: udp port reuse enabled
Jan 19 15:19:48 zanxus ss-server[27409]: 2017-01-19 15:19:48 INFO: listening at ${SERVER_IP}:${SERVER_PORT}
```

## After we configuring client we found that the ss vpn is still invalid ,that's because the ss server port was  prevented by firewall by default,CentOS7's firewall updated `iptables` to `firewall`,so we use:

```sh
firewall-cmd --permanent --add-port=${SERVER_PORT}/tcp ##change the ${SERVER_PORT} to real port
firewall-cmd --permanent --add-port=${SERVER_PORT}/udp
firewall-cmd --reload
```

## Now we can use shadowsocks  client to breakthrough the network limitation.

## Reference: 
   [https://github.com/shadowsocks/shadowsocks-libev#install-from-repository-1](https://github.com/shadowsocks/shadowsocks-libev#install-from-repository-1)
   [https://shadowsocks.org/en/config/quick-guide.html](https://shadowsocks.org/en/config/quick-guide.html)
   [http://morning.work/page/2015-12/install-shadowsocks-on-centos-7.html](http://morning.work/page/2015-12/install-shadowsocks-on-centos-7.html)
   [https://gist.github.com/aa65535/ea090063496b0d3a1748](https://gist.github.com/aa65535/ea090063496b0d3a1748)











