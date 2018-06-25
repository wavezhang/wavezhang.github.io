
---
title: LVS环境搭建
date: 2018-03-14 21:16:25
tags: LVS
---

# 网络配置
```            
                    +----------+
                    |  Client  |
                    +-----+----+
                      eth0|192.168.43.63
                          | 
---------------------------------------------net 192.168.43.0/24             
                          |
                      eth0|192.168.43.225
                    +----------+
                    |    LVS   |
                    +-----+----+
                      eth1|192.168.2.128
                          |
---------------------------------------------net 192.168.2.0/24
                          |
                 ------------------
    192.168.2.131|eth0        eth0|192.168.2.132
          +----------+        +----------+
          |RealServer|        |RealServer|
          +-----+----+        +-----+----+

```

# 环境安装

## 配置Server
1. 客户端路由指向LB
```bash
route add -net 192.168.43.0 netmask 255.255.255.0 gw 192.168.2.128
```

2. 配置http server
```bash
systemctl start nginx
echo "192.168.2.131" > /usr/share/nginx/html/index.html
```

## 配置LB
系统使用centos 7
1. 安装ipvsadm
```bash
yum -y install ipvsadm
# enable IP forward
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```

2. 配置ipvs

```bash
export VIP=192.168.43.225
export RS="192.168.2.131 192.168.2.132"
ipvsadm -A -t $VIP:80 -s lc 
for server in $RS; do
  ipvsadm -a -t $VIP:80 -r $server:80 -m
done
```

其中lc表示采用基于最小连接数的负载均衡算法


# 测试

在客户端上用浏览器或者curl 访问
```
http://192.168.43.225
```
这时候会返回Real Server响应的内容
```bash
# curl http://192.168.43.225
192.168.2.131
```
