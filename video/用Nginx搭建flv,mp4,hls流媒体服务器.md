## 用Nginx搭建flv,mp4,hls流媒体服务器

> 转自：http://blog.sina.com.cn/s/blog_15b2d69500102wc5r.html

  模块:nginx_mod_h264_streaming（支持h264编码MP4格式的视频）
  模块:http_flv_module （支持flv）
  模块:http_mp4_module （支持mp4）
  模块:nginx-rtmp-module （支持rtmp协议，也支持HLS）

1. 模块下载和解压

  wget http://nginx.org/download/nginx-1.6.0.tar.gz
  wget http://h264.code-shop.com/download/nginx_mod_h264_streaming-2.2.7.tar.gz
  wget http://sourceforge.net/projects/pcre/files/pcre/8.35/pcre-8.35.tar.gz
  wget http://zlib.net/zlib-1.2.8.tar.gz
  wget http://www.openssl.org/source/openssl-1.0.1g.tar.gz
  wget -O nginx-rtmp-module.zip https://github.com/arut/nginx-rtmp-module/archive/master.zip

  unzip nginx-rtmp-module.zip
  tar -zxvf nginx-1.6.0.tar.gz
  tar -zxvf nginx_mod_h264_streaming-2.2.7.tar.gz
  tar -zxvf pcre-8.35.tar.gz
  tar -zxvf zlib-1.2.8.tar.gz
  tar -zxvf openssl-1.0.1g.tar.gz

2. 配置命令，会生成makefile文件

  ./configure \
    --prefix=/usr/local/nginx \
    --add-module=../nginx_mod_h264_streaming-2.2.7 \
    --add-module=../nginx-rtmp-module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_stub_status_module \
    --with-http_ssl_module \
    --with-pcre=../pcre-8.35 \
    --with-zlib=../zlib-1.2.8 \
    --with-debug

3. 编译和安装

  make
  make install

4. 问题解决

  4.1 如果在configure过程中出现以下错误：
    /root/nginx_mod_h264_streaming-2.2.7/src/ngx_http_streaming_module.c: In function ‘ngx_streaming_handler’:
    /root/nginx_mod_h264_streaming-2.2.7/src/ngx_http_streaming_module.c:158: error: ‘ngx_http_request_t’ has no member named ‘zero_in_uri’
    make[1]: *** [objs/addon/src/ngx_http_h264_streaming_module.o] Error 1
    make[1]: Leaving directory `/root/nginx-0.8.54'
    make: *** [build] Error 2
    
    那么将src/ngx_http_streaming_module.c文件中以下代码删除或者是注释掉就可以了：
    
    if (r->zero_in_uri)
    {
      return NGX_DECLINED;
    }

    如果你没有对这个文件做个更改，那么应该在源码的第157-161行。这个问题是由于版本原因引起，在此不再讨论。
    修改完之后，记得先执行make clean，然后再进行重新执行configure、make，最后make install。
  4.2 如果在编译过程中出现以下错误：
    cc1: warnings being treated as errors
    那么修改/nginx-1.6.0/objs/Makefile文件
    CFLAGS =  -pipe  -O -W -Wall -Wpointer-arith -Wno-unused-parameter -Werror -g  -D_LARGEFILE_SOURCE -DBUILDING_NGINX -I../nginx-rtmp-module-master
    把上面的 -Werror去掉，不把warnning当作error处理

5. Nginx的配置

  #user  nobody;
  worker_processes  1;
  #error_log  logs/error.log;
  #error_log  logs/error.log  notice;
  #error_log  logs/error.log  info;
  #pid        logs/nginx.pid;
  events {
    worker_connections  1024;
  }

  rtmp {
    server {
        listen 1935;
        chunk_size 4000;
        # video on demand for flv files
        application vod {
            play /usr/local/nginx/html/flv;
        }
        # video on demand for mp4 files
        application vod2 {
            play /usr/local/nginx/html/mp4;
        }
        application hls {
            live on;
            hls on;
            hls_path /tmp/hls;
        }
        # MPEG-DASH is similar to HLS
        application dash {
            live on;
            dash on;
            dash_path /tmp/dash;
        }
    }
  }
  http {
    include       mime.types;
    default_type  application/octet-stream;
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    #access_log  logs/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    #keepalive_timeout  0;
    keepalive_timeout  65; 
    #gzip  on;
    server {
        # in case we have another web server on port 80
        listen       8080;
        server_name  localhost;
        #charset koi8-r;
        #access_log  logs/host.access.log  main; 
        location / {
            root   html;
            index  index.html index.htm;
        }
        #error_page  404              /404.html;
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
         location ~ \.mp4$ {
            mp4;
        }
        location ~ \.flv$ {
            flv;
         }
        # This URL provides RTMP statistics in XML
        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            # XML stylesheet to view RTMP stats.
            # Copy stat.xsl wherever you want
            # and put the full directory path here
            root /var/www/;
        }
        location /hls {
            # Serve HLS fragments
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            #where the m3u8 and ts files are
            alias /usr/local/nginx/html/hls;           
            #live streaming setting           
            #root /tmp;
            #add_header Cache-Control no-cache;
        }
        location /dash {
            # Serve DASH fragments
            root /tmp;
            add_header Cache-Control no-cache;
        }
    }
}

6. 用ffmpeg生成测试序列

* 对于mp4文件，生成moov信息前移的mp4格式,适合流媒体播放。
  ffmpeg -i /home/administrator/Videos/Amelia_720p.mp4 -c:v libx264 -c:a libvo_aacenc -f mp4 -movflags faststart /home/administrator/Videos/moovfront.mp4

* 对于flv文件，用flvmeta工具在metadata中注入关键帧的信息，支持随意拖动播放。
  ffmpeg -i/home/administrator/Videos/Baroness.mp4 -vcodec libx264 -acodec libvo_aacenc -b:a 128k -ar 44100 -ac 2 -f flv /home/administrator/Videos/Baroness.flv
  
  flvmeta -U -m -k /home/administrator/Videos/Baroness.flv /home/administrator/Videos/Baroness_meta.flv

* 对于flv的播放，或者直接生成f4v格式的文件。
  ffmpeg -i /home/administrator/Videos/sample/vc1_1080p.mkv -acodec libfdk_aac -ac 2 -b:a 128k -ar 48000 -vcodec libx264 -pix_fmt yuv420p -profile:v main -level 32 -b:v 1000K -r 29.97 -g 30 -s 960x540 -f f4v /home/administrator/Videos/hddvd_1000k.f4v

7. Nginx启动，重启，关闭命令

  start nginx 开启  

  nginx -s stop 快速关闭   

  nginx -s quit 完全关闭   

  nginx -s reload 修改过配置文件，快速关闭旧的，开启新服务    

  nginx -s reopen 重新打开日志文件  

[停止操作]

停止操作是通过向nginx进程发送信号来进行的

步骤1：查询nginx主进程号

ps -ef | grep nginx

在进程列表里 面找master进程，它的编号就是主进程号了。

步骤2：发送信号

从容停止Nginx：

kill -QUIT 主进程号

快速停止Nginx：

kill -TERM 主进程号

强制停止Nginx：

pkill -9 nginx

另外， 若在nginx.conf配置了pid文件存放路径则该文件存放的就是Nginx主进程号，如果没指定则放在nginx的logs目录下。有了pid文 件，我们就不用先查询Nginx的主进程号，而直接向Nginx发送信号了，命令如下：

kill -信号类型 '/usr/nginx/logs/nginx.pid'

 

[平滑重启]

如果更改了配置就要重启Nginx，要先关闭Nginx再打开？不是的，可以向Nginx 发送信号，平滑重启。

平滑重启命令：

kill -HUP 主进程号或进程号文件路径

或者使用 

/usr/nginx/sbin/nginx -s reload

注意，修改了配置文件后最好先检查一下修改过的配置文件是否正 确，以免重启后Nginx出现错误影响服务器稳定运行。判断Nginx配置是否正确命令如下：

nginx -t -c /usr/nginx/conf/nginx.conf

或者

/usr/nginx/sbin/nginx -t

 

8. 播放测试

启动nginx后测试：

       http://192.168.1.105/player.swf?type=http&file=test1.flv

说明： #我的ip是192.168.1.105

         #player.swf是我的JW Player播放器

         #http是表示居于http分发方式

         #test1.flv是我的flv视频文件

 

[flash]

http://localhost:8080/mediaplayer/player.swf?type=http&file=../mp4/HaroldKumar.mp4

http://localhost:8080/mediaplayer/player.swf?type=http&file=../flv/Baroness.flv

[hls --> flash]

http://localhost:8080/jwplayer/HLSprovider/test/jwplayer6/index2.html

[hls]

http://10.240.155.183:8080/hls/movie.m3u8

[rtmp --> http]

http://10.240.155.183:8080/flowplayer/index2.html

[live stream]

./ffmpeg -loglevel verbose -re -i /home/administrator/Videos/sample/h264_720p_hp_5.1_6mbps_ac3_unstyled_subs_planet.mkv  -vcodec libx264 -vprofile baseline -acodec libmp3lame -ar 44100 -ac 2 -f flv rtmp://localhost:1935/hls/movie

参考资料：
http://blog.csdn.net/cjsafty/article/details/9108587
http://www.codesky.net/article/201105/118326.html
http://blog.csdn.net/kl222/article/details/12968815
http://www.leaseweblabs.com/2013/11/streaming-video-demand-nginx-rtmp-module/
http://razvantudorica.com/12/hls-video-on-demand-streaming
http://www.razvantudorica.com/12/how-to-play-hls-with-jwplayer/
https://github.com/arut/nginx-rtmp-module
https://github.com/mangui/HLSprovider
http://nginx.org/en/docs/http/ngx_http_hls_module.html
https://flowplayer.org/pricing/#downloads
http://flash.flowplayer.org/plugins/streaming/rtmp.html
