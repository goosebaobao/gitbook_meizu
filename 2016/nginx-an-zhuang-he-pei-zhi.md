# Nginx 安装和配置

## 准备工作

创建 nginx 用户

```bash
useradd nginx -d /sbin/nologin
```

安装开发依赖包

```bash
yum -y groupinstall "Development tools" "Server Platform Libraries" 
yum -y install gd gd-devel pcre-devel 
yum -y install openssl openssl-devel
```

## 下载

 进入 /root/download 目录，下载，解压安装包

### nginx 1.2.9

```bash
wget http://nginx.org/download/nginx-1.2.9.tar.gz
tar -zxvf nginx-1.2.9.tar.gz
```

### pcre 8.33

```bash
wget http://downloads.sourceforge.net/project/pcre/pcre/8.33/pcre-8.33.zip
unzip pcre-8.33.zip
```

## 安装

###  安装 pcre 到 /usr/local

```bash
cd pcre-8.33
./configure --prefix=/usr/local/pcre-8.33
make && make install
```

### 安装 nginx 到 /data/nginx

```bash
cd /root/download/nginx-1.2.9

./configure \ --prefix=/data/nginx \
--error-log-path=/data/log/nginx/error.log \
--http-log-path=/data/log/nginx/access.log \
--pid-path=/var/run/nginx/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_flv_module \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/tmp/nginx/client/ \
--http-proxy-temp-path=/var/tmp/nginx/proxy/ \
--http-fastcgi-temp-path=/var/tmp/nginx/fcgi/ \
--http-uwsgi-temp-path=/var/tmp/nginx/uwsgi \
--http-scgi-temp-path=/var/tmp/nginx/scgi \
--with-pcre=/root/download/pcre-8.33 \
--with-file-aio \
--with-http_image_filter_module

make && make install
```

## 配置

默认配置：/data/nginx/conf/nginx.conf

```bash
user nginx nginx;
worker_processes 8;
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;

error_log  /data/log/nginx/nginx_error.log  crit;
#pid        /usr/local/nginx/nginx.pid;


#Specifies the value for maximum file descriptors that can be opened by this process.
worker_rlimit_nofile 50000;

events
{
     use epoll;
     worker_connections 50000;
}


http
{
     include       mime.types;
     default_type  application/octet-stream;
     underscores_in_headers on;
     access_log off;
     error_log off;
    #charset  utf-8;

     server_names_hash_bucket_size 128;

     sendfile on;
     tcp_nopush     on;
     tcp_nodelay on;


        # 日志格式(std log) 注：各字段间以\t(tab字符)分隔。不能简单复制，否则会以空格作为分隔符了
        log_format      main    '$time_iso8601  $status $connection     $connection_requests    $remote_addr    $http_x_forwarded_for   $remote_user
$request_length $request_time   $request_method $server_protocol        $http_host      $server_port    $uri    $args   $http_referer   $body_bytes_sent
        $http_user_agent        $ssl_protocol   $ssl_cipher     $upstream_addr  $upstream_status        $upstream_response_time';


     log_format  time  '$remote_addr $time_local $request_time $http_host $request $status';


     # 处理时间
     keepalive_timeout 60;
     # 用户请求头的超时时间
     client_header_timeout 1m;
     # 用户请求体的超时时间
     client_body_timeout 1m;
     # 用户请求体最大字节数
     client_max_body_size 10m;

     # proxy 用
     #send_timeout 3m;



     # 这几个值不能太小    太小会影响商城购物车的数据
     connection_pool_size 256;
     client_header_buffer_size 64k;
     large_client_header_buffers 4 64k;
     request_pool_size 64k;
     output_buffers 4 64k;
     postpone_output 1460;
     client_body_buffer_size 256k;


     fastcgi_connect_timeout 60;
     fastcgi_send_timeout 60;
     fastcgi_read_timeout 60;

     ## 这个不能大小   太小会常出502错误
     fastcgi_buffer_size 256k;
     fastcgi_buffers 8 256k;
     fastcgi_busy_buffers_size 256k;
     fastcgi_temp_file_write_size 256k;
#     fastcgi_temp_path /dev/shm;
#     fastcgi_intercept_errors on;

#     open_file_cache max=50000 inactive=20s;
#     # 多长时间检查一次缓存的有效信息
#     open_file_cache_min_uses 1;
#     open_file_cache_valid 30s;




     gzip on;
     gzip_min_length  4k;
     gzip_buffers     4 16k;
     gzip_http_version 1.1;
     gzip_comp_level 2;
     gzip_types       text/plain application/x-javascript text/css application/xml;
     gzip_vary on;

#     proxy_cache_path /dev/shm/nginx_cache levels=1:2 keys_zone=cache_one:200m inactive=1d max_size=200m;
#     limit_zone   one $binary_remote_addr 10m;

#     server
#     {
#        error_page 404 /error.html;
#
#        error_page 500 502 503 504 /notic.html;
#        location = /notic.html {
#              root html;
#        }
#     }
#
     include vhosts/*.com;
     include vhosts/*.conf;
     include vhosts/*.cn;

#隐藏nginx版本信息
server_tokens off;

#禁用空主机头访问
server {
        listen 80 default;
        return 403;
        }

}
```

虚拟主机配置：/data/nginx/conf/vhosts/flymetv.conf

```bash
upstream flymetv-server {
    server localhost:8080;
    server 172.17.49.69:8080;
    keepalive 4;
}


 server {
        listen       80;

        server_name tvvideo.meizu.com;
        access_log  /data/log/nginx/tv_access.log;

        location ~* .*\.svn.* {

        return 404;
        }

#        location /fileserver {
#                alias /mnt/www/fileserver;
#        }

        location / {
            proxy_store off;
            proxy_redirect off;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $http_host;
            proxy_pass http://flymetv-server/;

        }
  }

server {
        listen 443;
        access_log  /data/log/nginx/tv_access_ssl.log;
        server_name tvvideo.meizu.com;
        ssl on;
        ssl_certificate meizu.com.crt;
        ssl_certificate_key meizu.com.key.unsecure;
        ssl_session_timeout 5m;
        ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-RSA-RC4-SHA:AES128-GCM-SHA256:AES128-SHA256:AES128-SHA:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:AES256-GCM-SHA384:AES256-SHA256:AES256-SHA:ECDHE-RSA-AES128-SHA256:RC4-SHA:!aNULL:!eNULL:!EXPORT:!DES:!3DES:!MD5:!DSS:!PKS;
        ssl_prefer_server_ciphers on;


        location / {
             proxy_store off;
             proxy_redirect off;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header Host $http_host;
             proxy_set_header x-scheme true;
             proxy_set_header MEIZU_UA MEIZU;
             proxy_pass http://flymetv-server/;
        }
    }
```

## 运行

 执行 `/data/nginx/sbin/nginx`，提示

```bash
nginx: [emerg] mkdir() "/var/tmp/nginx/client/" failed (2: No such file or directory)
```

创建临时目录 

```bash
mkdir -p /var/tmp/nginx/client
```

## 日志切割

/data/nginx/sbin/cut\_nginx\_log.sh

```bash
#!/bin/bash
log_path=/data/log/nginx/
date=`date -d "yesterday" +"%Y%m%d"`
access_file=${log_path}access_${date}.log
error_file=${log_path}error_${date}.log
mv ${log_path}access.log ${access_file}
mv ${log_path}error.log ${error_file}
kill -USR1 `cat ${log_path}nginx.pid`
```

 使用 cron 调度，crontab 设置：`0 0 * * * /data/nginx/sbin/cut_nginx_log.sh`

## 自启动

 创建 /etc/init.d/nginx 文件，内容如下

```bash
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15
# description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid
# Source function library.
. /etc/rc.d/init.d/functions
# Source networking configuration.
. /etc/sysconfig/network
# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0
nginx="/data/nginx/sbin/nginx"
prog=$(basename $nginx)
NGINX_CONF_FILE="/data/nginx/conf/nginx.conf"
[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
lockfile=/var/lock/subsys/nginx
make_dirs() {
   # make required directories
   user=`$nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   if [ -z "`grep $user /etc/passwd`" ]; then
       useradd -M -s /bin/nologin $user
   fi
   options=`$nginx -V 2>&1 | grep 'configure arguments:'`
   for opt in $options; do
       if [ `echo $opt | grep '.*-temp-path'` ]; then
           value=`echo $opt | cut -d "=" -f 2`
           if [ ! -d "$value" ]; then
               # echo "creating" $value
               mkdir -p $value && chown -R $user $value
           fi
       fi
   done
}
start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}
stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}
restart() {
    configtest || return $?
    stop
    sleep 1
    start
}
reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}
force_reload() {
    restart
}
configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}
rh_status() {
    status $prog
}
rh_status_q() {
    rh_status >/dev/null 2>&1
}
case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```

{% hint style="info" %}
来源于 [https://www.nginx.com/resources/wiki/start/topics/examples/redhatnginxinit/](https://www.nginx.com/resources/wiki/start/topics/examples/redhatnginxinit/)
{% endhint %}

接下来要给这个脚本执行权限 

```bash
chmod a+x /etc/init.d/nginx
```

然后配置为自启动

```bash
chkconfig --add nginx
chkconfig nginx on
```



