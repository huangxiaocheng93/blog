# certbot官网地址

[[Certbot](https://certbot.eff.org/)](https://certbot.eff.org/)

# 安装snapd

snapd是一个包管理工具（类似应用商店），certbot的工具是发布在这里。

## 安装epel源

如果已经安装过就不需要了

```bash
yum install epel-release
```

## 安装snapd

```bash
yum -y install snapd
```

## snapd启用通信socket

```bash
systemctl enable --now snapd.socket
```

## 创建符号链接

```bash
ln -s /var/lib/snapd/snap /snap
```

```bash
snap install core
snap refresh core

```

## 安装certbot

```bash
snap install --classic certbot
```

```bash
ln -s /snap/bin/certbot /usr/bin/certbot
```

## 安装证书

```bash
certbot --nginx
```

or

```bash
certbot certonly --nginx
```

```bash
certbot install --cert-name ttk.npex.top
```

## 证书续期

```bash
certbot renew
```