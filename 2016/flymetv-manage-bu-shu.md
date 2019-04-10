# FlymeTV manage 部署

## 安装 jetty

下载 jetty 最新 zip 版本

```bash
wget http://wiki.meizu.com/images/5/50/Jetty-7.6.9-updated20151211.zip
```

 解压缩到 /data/jetty 目录

```bash
unzip Jetty-7.6.9-updated20151211.zip
```

启动 jetty

```bash
/data/jetty/bin/jetty.sh start
```

启动不成功，提示信息为

```bash
Starting Jetty: chown: invalid user: `jetty' 
su: user jetty does not exist
```

## 创建 jetty 用户

jetty 需要 jetty 用户来启动，那么就创建一个 jetty 用户，密码为 jetty

```bash
useradd -d /home/jetty -m jetty 
passwd jetty
```

为了方便，直接把 jetty 赋予 root 权限

修改 /etc/sudoers 文件，找到下面一行，在 root 下面添加一行，如下第三行所示

```bash
 ## Allow root to run any commands anywhere 
 root ALL=(ALL) ALL 
 jetty ALL=(ALL) ALL
```

 修改完毕，现在可以用 jetty 帐号登录，然后用命令 `sudo –` 即可获得 root 权限进行操作

为 jetty 用户设置目录权限

```bash
chown jetty:jetty -R /data/log/jetty 
mkdir -p /data/yard /data/conf /data/logs/jetty 
chown jetty:jetty -R /data/conf /data/yard /data/logs/jetty
```

## 启动 jetty

拉起 jetty 

```bash
/data/jetty/bin/jetty.sh start
```

 查看 jetty 是否成功启动

```bash
ps aux | grep jetty
```

### jetty 的自启动

将脚本 jetty.sh 拷贝到 init.d 目录下

```bash
cp /data/jetty/bin/jetty.sh /etc/init.d/jetty
```

配置 /etc/default/jetty

```bash
# vim /etc/default/jetty 
--------------------------------------------------- 
JETTY_HOME=/data/jetty 
export JETTY_HOME 
PATH=$PATH:$JETTY_HOME/bin 
export PATH
```

注册 jetty 为自启动

```bash
chkconfig --add jetty
chkconfig jetty on
```

done

## 部署 manage

下载 manage 的 war 包到 jetty 的 webapps 目录

```bash
wget -P/data/jetty/webapps http://maven.meizu.com:8081/artifactory/libs-release-local/com/meizu/flymetv-video-manage/1.0.15/flymetv-video-manage-1.0.15.war
```

  然后重启 jetty

```bash
/data/jetty/bin/jetty.sh restart
```

