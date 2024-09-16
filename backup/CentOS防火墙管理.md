# 命令整合

## 安装

```bash
yum -y install firewalld
systemctl start firewalld
systemctl enable firewalld

```

## 开放80和443端口

```bash
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=80/udp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --zone=public --add-port=443/udp --permanent
firewall-cmd --reload

```

# 安装firewalld

```bash
yum -y install firewalld
```

# 常用命令

## 启动

```bash
systemctl start firewalld
```

## 关闭

```bash
systemctl stop firewalld
```

## 查看状态

```bash
systemctl status firewalld
```

## 开机启动

```
systemctl enable firewalld
```

## 开机禁用

```
systemctl disable firewalld
```

## 安装并配置开启启动

```bash
yum -y install firewalld
systemctl start firewalld
systemctl enable firewalld

```

# 配置firewalld-cmd

查看版本

```bash
firewall-cmd --version
```

查看帮助

```bash
firewall-cmd --help
```

显示状态

```bash
firewall-cmd --state
```

查看所有打开的端口

```bash
firewall-cmd --zone=public --list-ports
```

更新防火墙规则

```bash
firewall-cmd --reload
```

查看区域信息

```bash
firewall-cmd --get-active-zones
```

查看指定接口所属区域

```bash
firewall-cmd --get-zone-of-interface=eth0
```

拒绝所有包

```bash
firewall-cmd --panic-on
```

取消拒绝状态

```bash
firewall-cmd --panic-off
```

查看是否拒绝

```bash
firewall-cmd --query-panic
```

## 开启一个端口

### 添加

```bash
firewall-cmd --zone=public --add-port=80/tcp --permanent
```

（--permanent永久生效，没有此参数重启后失效）

### 重新载入

开启或者关闭了端口后需要reload才能生效

```bash
firewall-cmd --reload
```

### 查看

yes即为开启

```bash
firewall-cmd --zone=public --query-port=80/tcp
```

### 删除

```bash
firewall-cmd --zone=public --remove-port=80/tcp --permanent
```

### 备注

systemctl是CentOS7的服务管理工具中主要的工具，它融合之前service和chkconfig的功能于一体。
启动服务

```bash
systemctl start firewalld.service
```

关闭服务

```bash
systemctl stop firewalld.service
```

重启服务

```bash
systemctl restart firewalld.service
```

显示服务的状态

```bash
systemctl status firewalld.service
```

在开机时启用服务

```bash
systemctl enable firewalld.service
```

在开机时禁用服务

```bash
systemctl disable firewalld.service
```

查看服务是否开机启动

```bash
systemctl is-enabled firewalld.service
```

查看已启动的服务列表

```bash
systemctl list-unit-files|grep enabled
```

查看启动失败的服务列表

```bash
systemctl --failed
```