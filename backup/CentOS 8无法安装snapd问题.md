使用certbot安装证书时需要通过snapd来安装，centos8下安装snapd报错：

```bash
上次元数据过期检查：0:07:16 前，执行于 2023年07月03日 星期一 03时03分49秒。
错误：
 问题: 软件包 snapd-2.58.3-1.el8.x86_64 需要 snapd-selinux = 2.58.3-1.el8，但没有提供者可以被安装
  - 冲突的请求
  - 没有东西可提供 selinux-policy >= 3.14.3-108.el8_7.1（snapd-selinux-2.58.3-1.el8.noarch 需要）
  - 没有东西可提供 selinux-policy-base >= 3.14.3-108.el8_7.1（snapd-selinux-2.58.3-1.el8.noarch 需要）
(尝试添加 '--skip-broken' 来跳过无法安装的软件包 或 '--nobest' 来不只使用软件包的最佳候选)
```

原因是因为snapd-selinux-2.58.3-1.el8.noarch这个包下载不到，可以手动下载更新进去

```bash
yum -y install yum-utils
yumdownloader snapd-selinux-2.58.3-1.el8.noarch
rpm --force --nodeps -ivh snapd-selinux-2.58.3-1.el8.noarch.rpm

```

然后重新安装snapd

```bash
yum -y install snapd
```