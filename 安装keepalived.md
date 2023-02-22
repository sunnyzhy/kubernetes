# 安装 keepalived

[keepalived 官网](https://www.keepalived.org/ 'keepalived')

## 下载

```bash
mkdir -p /usr/local/k8s/keepalived

wget --no-check-certificate https://www.keepalived.org/software/keepalived-2.2.7.tar.gz -P /usr/local/k8s/keepalived
```

## 安装

```bash
yum install -y gcc openssl-devel libnl3-devel net-snmp-devel

cd /usr/local/k8s/keepalived

tar -zxvf keepalived-2.2.7.tar.gz

cd keepalived-2.2.7

./configure --prefix=/usr/local/keepalived

make && make install
```

## 设置开机启动

```bash
mkdir -p /etc/keepalived

cp /usr/local/keepalived/etc/sysconfig/keepalived  /etc/sysconfig/keepalived 

cp /usr/local/keepalived/sbin/keepalived /usr/sbin/keepalived

cp /usr/local/keepalived-2.2.7/keepalived/etc/init.d/keepalived  /etc/init.d/keepalived

cp /usr/local/keepalived/etc/keepalived/keepalived.conf.sample /etc/keepalived/keepalived.conf

chkconfig keepalived on

systemctl enable keepalived

systemctl start keepalived
```

## 编辑配置文件

***state 取值:***

- MASTER: 主机标识
- BACKUP: 备机标识

***```keepalived``` 并不会创建端口，所以 ```virtual_server``` 和 ```real_server``` 的端口必须一致且必须是 ```real_server``` 的端口。***

MASTER 主要配置:

- state MASTER
- interface ens33
- priority 100

BACKUP 主要配置:

- state BACKUP
- interface ens33
- priority 99

查看主机网卡:

```bash
# ip addr
...
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:8a:8a:d3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.163/24 brd 192.168.5.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::42de:6cd4:dde4:b13e/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
...
```

配置 VIP:

```bash
vim /etc/keepalived/keepalived.conf
```

```conf
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.5.100
    }
}

virtual_server 192.168.5.100 443 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.5.61 443 {
        weight 1
        SSL_GET {
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3
            retry 3
            delay_before_retry 3
        }
    }
}
```

查看主机网卡(VIP 192.168.5.100 已生效):

```bash
# ip addr
...
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:8a:8a:d3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.163/24 brd 192.168.5.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.5.100/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::42de:6cd4:dde4:b13e/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
...
```

## 启动

```bash
# systemctl start keepalived

# systemctl status keepalived
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running)
```

## FAQ

### systemd[1]: Can't open PID file /run/keepalived.pid (yet?) after start: No such file or directory

```bash
pkill keepalived
```
