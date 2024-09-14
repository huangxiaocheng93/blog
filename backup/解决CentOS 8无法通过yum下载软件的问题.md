原因是CentOS 8已于2021年12月31日停止维护，在2022年1月31日，CentOS团队终于从官方镜像中移除CentOS 8的所有包。

可以把源替换为阿里云的源解决此问题：

```bash
sed -i -e "s|mirrorlist=|#mirrorlist=|g" /etc/yum.repos.d/CentOS-*
sed -i -e "s|#baseurl=http://mirror.centos.org|baseurl=http://mirrors.aliyun.com|g" /etc/yum.repos.d/CentOS-*
```