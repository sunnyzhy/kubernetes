# 安装 keepalived

## 安装

```bash
yum install -y gcc openssl-devel libnl3-devel net-snmp-devel

yum -y install keepalived
```

## 设置开机启动

```bash
systemctl enable keepalived

systemctl start keepalived
```

## 编辑配置文件

```bash
vim /etc/keepalived/keepalived.conf
```
