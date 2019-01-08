### nginx操作配置文件进行日志分割

注意：在项目根目录下存在一个配置文件模板，我们可以在这个模板中进行日志分割处理操作。(nginx.conf文件)


警告：对于特定的nginx版本需要nginx的版本，因为配置文件支持动态时间格式日志保存，必须nginx版本大于1.12.*以上。

所以我们这里选择nginx：1.13.7  版本作为测试镜像。

安装nginx版本的方式有很多，官方版本安装，源码安装，或者docker镜像部署安装，我们这里选择使用docker的镜像进行安装测试部署。


配置留意点：

1   用户权限设置


    #user  nginx;
    worker_processes  1;
    
    error_log  /var/log/nginx/error.log warn;
    pid/var/run/nginx.pid;
    user root root;

注：将user  nginx；注释掉，换成user root root;


2 进行http的动态日志加载

    http {
    include   /etc/nginx/mime.types;
    default_type  application/octet-stream;
    
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
      '$status $body_bytes_sent "$http_referer" '
      '"$http_user_agent" "$http_x_forwarded_for"';
    
     map $time_iso8601 $logdate{
    "~^(?<ymd>\d{4}-\d{2}-\d{2})"  $ymd;
    default  'nodate';
     }
    
     access_log  /var/log/nginx/accessvpc-$logdate.log;
    #access_log  /var/log/nginx/access.log  main;
    
    sendfileon;
    #tcp_nopush on;
    
    keepalive_timeout  65;
    
    #gzip  on;
    
    include /etc/nginx/conf.d/*.conf;
    }

其中 map 对应我们希望的时间格式，并不是所有nginx版本都支持map函数的。nginx版本要在1.13.7以上。

我们要理解http和tcp属于不同的传输协议层。

http属于七层网络协议，tcp属于四层，我们再访问应用进行区分。

http应用访问请求很常见，但是tcp进行路由负载均衡同样值得我们重视。


关于tcp配置日志分割按照天进行保存。

    stream {
      log_format   log_stream '$time_local || $remote_addr || $upstream_addr || $status || $protocol || $bytes_received || $session_time || $upstream_bytes_received || $upstream_connect_time';
      map $time_iso8601 $logdate{
    "~^(?<ymd>\d{4}-\d{2}-\d{2})"  $ymd;
    default  'nodate';
      }
      access_log /var/log/nginx/tcp-access-$logdate.log log_stream;
      error_log  /var/log/nginx/tcp-error.log;
    
    
      upstream test {
    server192.168.131.130:80;
      }
      server {
    listen  33010;
    listen  [::]:33010;
    proxy_timeout  600s;
    proxy_pass  test;
      }
    
    }


1   我们需要定义tcp保存的日志格式，还要定义存储的时间格式

    #定义的日志数据格式
	log_format   log_stream '$time_local || $remote_addr || $upstream_addr || $status || $protocol || $bytes_received || $session_time || $upstream_bytes_received || $upstream_connect_time';
     
     #取出来的时间格式
	 map $time_iso8601 $logdate{
    "~^(?<ymd>\d{4}-\d{2}-\d{2})"  $ymd;
    default  'nodate';
      }


2  指定动态变化的日志存储名称

	  
	  access_log /var/log/nginx/tcp-access-$logdate.log log_stream;
      error_log  /var/log/nginx/tcp-error.log;

注意nginx流量转发要注意的问题。
上流和下流分发。