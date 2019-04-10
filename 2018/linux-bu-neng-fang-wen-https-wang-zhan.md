# linux 不能访问 https 网站

## 背景

linux 上的 python 脚本一直正常运行，最近几天连续出错，查看日志，发现如下错误信息

```bash
ssl.SSLError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:777)
```

看上去是脚本访问某个 https 网页失败导致的。而且这个错误并不是必现的，这个 https 网页有时候又可以正常访问

## 安装 ssl 库

印象里这不是第一次了，我找了下以前做的记录，如下

```bash
# 解决 linux 无法访问 https 的问题
yum install openssl-devel
yum install zlib-devel bzip2-devel sqlite sqlite-devel openssl-devel
```

一通操作猛如虎，感觉可以搞定收工了，然而......

```bash
curl https://www.baidu.com

curl: (60) Peer certificate cannot be authenticated with known CA certificates
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
```

尼玛这是什么操作？百度的 ssl 证书不能被已知的 CA 验证？用浏览器打开百度网站很正常，这样看来 linux 系统自带的证书库有点过时了吧

## 更新证书库

经过一番研究，发现 linux 系统自带的证书库是  `/etc/pki/tls/certs/ca-bundle.crt`，估摸着是这个证书库自身的问题，至于为什么时灵时不灵，还真没有搞明白——总之不管那么多，更新一下本地证书库吧，如下操作一番

安装 ca-certificates

```bash
yum install ca-certificates
```

更新本地证书库

```bash
update-ca-trust -h
usage: /usr/bin/update-ca-trust [extract | check | enable | disable | force-enable | force-disable ]

update-ca-trust check
PEM/JAVA Status: DISABLED.
   (Legacy setup with static files.)
PKCS#11 module Status, see symbolic links reported below:
lrwxrwxrwx 1 root root 28 Jul 16 10:08 /etc/alternatives/libnssckbi.so.x86_64 -> /usr/lib64/nss/libnssckbi.so
    (link resolving to NSS: using legacy static list)
    (link resolving to p11-kit: using the new source configuration)

update-ca-trust enable
```

现在再试一下

```bash
 curl https://www.baidu.com
<!DOCTYPE html>
<!--STATUS OK--><html> <head>......</html>
```

搞定，收工

