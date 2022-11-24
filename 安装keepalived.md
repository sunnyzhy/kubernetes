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

cp /usr/local/keepalived/etc/keepalived/keepalived.conf.sample /usr/local/keepalived/etc/keepalived/keepalived.conf

cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived

cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig

cp /usr/local/keepalived/sbin/keepalived /usr/sbin

chkconfig keepalived on

systemctl enable keepalived

systemctl start keepalived
```

## 编辑配置文件

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
```

***state 取值:***

- MASTER: 主机标识
- BACKUP: 备机标识

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
