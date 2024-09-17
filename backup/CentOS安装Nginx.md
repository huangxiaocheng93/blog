# 命令整合

## 安装

```bash
yum -y install epel-release
yum -y update
yum -y install nginx
systemctl enable nginx
systemctl start nginx

```

# yum命令

yum相当于是centos版的软件管家，里面已经收录了很多软件，方便对软件进行管理。因为yum官方源里面没有nginx，直接安装会提示找不到软件，所以我们要先安装epel的源（以下操作均默认使用root权限）。

```bash
yum -y install epel-release
```

然后更新

```bash
yum -y update
```

# nginx安装

```bash
yum -y install nginx
```

# 设置

## 开机自启动

```bash
systemctl enable nginx
```

## 启动

```bash
systemctl start nginx
```

## 测试配置文件正确性

```bash
nginx -t
```

## 重新加载配置文件

```bash
nginx -s reload
```