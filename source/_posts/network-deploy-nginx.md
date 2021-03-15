---
title: 企业网络之部署NGINX
cover:  /img/network-deploy/nginx_title.png
subtitle: 企业网络之部署NGINX
categories: "企业网络"
tags: "企业网络"
author:
  nick: 袁苏东
  link: https://github.com/yuansudong
typora-root-url: network-deploy-nginx

---
NGINX是一款轻量级的WEB服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器，在BSD-LIKE 协议下发行.



其特点是占有内存少，并发能力强，事实上nginx的并发能力在同类型的网页服务器中表现较好.



NGINX的安装方式有两种.第一种是通过YUM源安装,第二种是编译安装.



此处采用编译安装的方式.之所以采用编译安装的方式,是因为个人所从事的工作中会频繁的用到一些NGINX的三方模块.



通过维护一个NGINX工程,在需要用到三方模块的时候,在将三方模块通过GIT CLONE 进NGINX工程之后,配合CI/CD,可以很方便的对线上或者内网的NGINX做更新.



#### 01.通过YUM安装相关依赖



```
dnf  install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
dnf  config-manager --set-enabled powertools
dnf  install -y wget curl gcc-c++ pcre pcre-devel zlib zlib-devel 
dnf  install -y libxslt-devel
dnf  install -y libuuid-devel libblkid-devel libudev-devel 
dnf  install -y fuse-devel libedit-devel libatomic_ops-devel
dnf  install -y openssl openssl-devel perl gd-devel
```



#### 02.下载NGINX的源码并解压



```
wget http://nginx.org/download/nginx-1.19.6.tar.gztar -zxvf nginx-1.19.6.tar.gzmv nginx-1.19.6 nginx
```



#### 03.配置工程



在配置工程之前,需要使用下列命令查看下NGINX有哪些模块.根据自身目前需求可以删减相关的模块



```
[root@localhost nginx]# ./configure --help
  --help                             print this message
  --prefix=PATH                      set installation prefix
  --sbin-path=PATH                   set nginx binary pathname
  --modules-path=PATH                set modules path
  --conf-path=PATH                   set nginx.conf pathname
  --error-log-path=PATH              set error log pathname
  --pid-path=PATH                    set nginx.pid pathname
  --lock-path=PATH                   set nginx.lock pathname

  --user=USER                        set non-privileged user for
                                     worker processes
  --group=GROUP                      set non-privileged group for
                                     worker processes

  --build=NAME                       set build name
  --builddir=DIR                     set build directory

  --with-select_module               enable select module
  --without-select_module            disable select module
  --with-poll_module                 enable poll module
  --without-poll_module              disable poll module

  --with-threads                     enable thread pool support

  --with-file-aio                    enable file AIO support

  --with-http_ssl_module             enable ngx_http_ssl_module
  --with-http_v2_module              enable ngx_http_v2_module
  --with-http_realip_module          enable ngx_http_realip_module
  --with-http_addition_module        enable ngx_http_addition_module
  --with-http_xslt_module            enable ngx_http_xslt_module
  --with-http_xslt_module=dynamic    enable dynamic ngx_http_xslt_module
  --with-http_image_filter_module    enable ngx_http_image_filter_module
  --with-http_image_filter_module=dynamic
                                     enable dynamic ngx_http_image_filter_module
  --with-http_geoip_module           enable ngx_http_geoip_module
  --with-http_geoip_module=dynamic   enable dynamic ngx_http_geoip_module
  --with-http_sub_module             enable ngx_http_sub_module
  --with-http_dav_module             enable ngx_http_dav_module
  --with-http_flv_module             enable ngx_http_flv_module
  --with-http_mp4_module             enable ngx_http_mp4_module
  --with-http_gunzip_module          enable ngx_http_gunzip_module
  --with-http_gzip_static_module     enable ngx_http_gzip_static_module
  --with-http_auth_request_module    enable ngx_http_auth_request_module
  --with-http_random_index_module    enable ngx_http_random_index_module
  --with-http_secure_link_module     enable ngx_http_secure_link_module
  --with-http_degradation_module     enable ngx_http_degradation_module
  --with-http_slice_module           enable ngx_http_slice_module
  --with-http_stub_status_module     enable ngx_http_stub_status_module

  --without-http_charset_module      disable ngx_http_charset_module
  --without-http_gzip_module         disable ngx_http_gzip_module
  --without-http_ssi_module          disable ngx_http_ssi_module
  --without-http_userid_module       disable ngx_http_userid_module
  --without-http_access_module       disable ngx_http_access_module
  --without-http_auth_basic_module   disable ngx_http_auth_basic_module
  --without-http_mirror_module       disable ngx_http_mirror_module
  --without-http_autoindex_module    disable ngx_http_autoindex_module
  --without-http_geo_module          disable ngx_http_geo_module
  --without-http_map_module          disable ngx_http_map_module
  --without-http_split_clients_module disable ngx_http_split_clients_module
  --without-http_referer_module      disable ngx_http_referer_module
  --without-http_rewrite_module      disable ngx_http_rewrite_module
  --without-http_proxy_module        disable ngx_http_proxy_module
  --without-http_fastcgi_module      disable ngx_http_fastcgi_module
  --without-http_uwsgi_module        disable ngx_http_uwsgi_module
  --without-http_scgi_module         disable ngx_http_scgi_module
  --without-http_grpc_module         disable ngx_http_grpc_module
  --without-http_memcached_module    disable ngx_http_memcached_module
  --without-http_limit_conn_module   disable ngx_http_limit_conn_module
  --without-http_limit_req_module    disable ngx_http_limit_req_module
  --without-http_empty_gif_module    disable ngx_http_empty_gif_module
  --without-http_browser_module      disable ngx_http_browser_module
  --without-http_upstream_hash_module
                                     disable ngx_http_upstream_hash_module
  --without-http_upstream_ip_hash_module
                                     disable ngx_http_upstream_ip_hash_module
  --without-http_upstream_least_conn_module
                                     disable ngx_http_upstream_least_conn_module
  --without-http_upstream_random_module
                                     disable ngx_http_upstream_random_module
  --without-http_upstream_keepalive_module
                                     disable ngx_http_upstream_keepalive_module
  --without-http_upstream_zone_module
                                     disable ngx_http_upstream_zone_module

  --with-http_perl_module            enable ngx_http_perl_module
  --with-http_perl_module=dynamic    enable dynamic ngx_http_perl_module
  --with-perl_modules_path=PATH      set Perl modules path
  --with-perl=PATH                   set perl binary pathname

  --http-log-path=PATH               set http access log pathname
  --http-client-body-temp-path=PATH  set path to store
                                     http client request body temporary files
  --http-proxy-temp-path=PATH        set path to store
                                     http proxy temporary files
  --http-fastcgi-temp-path=PATH      set path to store
                                     http fastcgi temporary files
  --http-uwsgi-temp-path=PATH        set path to store
                                     http uwsgi temporary files
  --http-scgi-temp-path=PATH         set path to store
                                     http scgi temporary files

  --without-http                     disable HTTP server
  --without-http-cache               disable HTTP cache

  --with-mail                        enable POP3/IMAP4/SMTP proxy module
  --with-mail=dynamic                enable dynamic POP3/IMAP4/SMTP proxy module
  --with-mail_ssl_module             enable ngx_mail_ssl_module
  --without-mail_pop3_module         disable ngx_mail_pop3_module
  --without-mail_imap_module         disable ngx_mail_imap_module
  --without-mail_smtp_module         disable ngx_mail_smtp_module

  --with-stream                      enable TCP/UDP proxy module
  --with-stream=dynamic              enable dynamic TCP/UDP proxy module
  --with-stream_ssl_module           enable ngx_stream_ssl_module
  --with-stream_realip_module        enable ngx_stream_realip_module
  --with-stream_geoip_module         enable ngx_stream_geoip_module
  --with-stream_geoip_module=dynamic enable dynamic ngx_stream_geoip_module
  --with-stream_ssl_preread_module   enable ngx_stream_ssl_preread_module
  --without-stream_limit_conn_module disable ngx_stream_limit_conn_module
  --without-stream_access_module     disable ngx_stream_access_module
  --without-stream_geo_module        disable ngx_stream_geo_module
  --without-stream_map_module        disable ngx_stream_map_module
  --without-stream_split_clients_module
                                     disable ngx_stream_split_clients_module
  --without-stream_return_module     disable ngx_stream_return_module
  --without-stream_set_module        disable ngx_stream_set_module
  --without-stream_upstream_hash_module
                                     disable ngx_stream_upstream_hash_module
  --without-stream_upstream_least_conn_module
                                     disable ngx_stream_upstream_least_conn_module
  --without-stream_upstream_random_module
                                     disable ngx_stream_upstream_random_module
  --without-stream_upstream_zone_module
                                     disable ngx_stream_upstream_zone_module

  --with-google_perftools_module     enable ngx_google_perftools_module
  --with-cpp_test_module             enable ngx_cpp_test_module

  --add-module=PATH                  enable external module
  --add-dynamic-module=PATH          enable dynamic external module

  --with-compat                      dynamic modules compatibility

  --with-cc=PATH                     set C compiler pathname
  --with-cpp=PATH                    set C preprocessor pathname
  --with-cc-opt=OPTIONS              set additional C compiler options
  --with-ld-opt=OPTIONS              set additional linker options
  --with-cpu-opt=CPU                 build for the specified CPU, valid values:
                                     pentium, pentiumpro, pentium3, pentium4,
                                     athlon, opteron, sparc32, sparc64, ppc64

  --without-pcre                     disable PCRE library usage
  --with-pcre                        force PCRE library usage
  --with-pcre=DIR                    set path to PCRE library sources
  --with-pcre-opt=OPTIONS            set additional build options for PCRE
  --with-pcre-jit                    build PCRE with JIT compilation support

  --with-zlib=DIR                    set path to zlib library sources
  --with-zlib-opt=OPTIONS            set additional build options for zlib
  --with-zlib-asm=CPU                use zlib assembler sources optimized
                                     for the specified CPU, valid values:
                                     pentium, pentiumpro

  --with-libatomic                   force libatomic_ops library usage
  --with-libatomic=DIR               set path to libatomic_ops library sources

  --with-openssl=DIR                 set path to OpenSSL library sources
  --with-openssl-opt=OPTIONS         set additional build options for OpenSSL

  --with-debug                       enable debug logging
```



下面是个人需要的配置



```
./configure --with-libatomic --with-pcre --with-compat \
  --with-stream --with-stream=dynamic --with-http_sub_module --with-http_dav_module --with-http_flv_module   \
  --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module \
  --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module \
  --with-http_auth_request_module    --with-http_random_index_module   --with-http_secure_link_module \
  --with-http_degradation_module     --with-http_slice_module --with-http_stub_status_module \
  --with-threads --with-file-aio --with-http_ssl_module --with-http_v2_module --with-http_realip_module \
  --with-http_addition_module --with-http_xslt_module --with-http_xslt_module=dynamic --with-http_image_filter_module --with-http_image_filter_module=dynamic\
  --prefix=/etc/nginx \
  --sbin-path=/etc/nginx/nginx \
  --modules-path=/etc/nginx/modules \
  --conf-path=/etc/nginx/nginx.conf \
  --error-log-path=/etc/nginx/log \
  --pid-path=/etc/nginx/run/nginx.pid \
  --lock-path=/etc/nginx/nginx.lock
```



#### 04.编译&安装



```
make make install
```



#### 05.加入systemctl



进入 /usr/lib/systemd/system 目录下，编辑文件 nginx.service



```
cat > /usr/lib/systemd/system/nginx.service <<EOF
[Unit]
Description=nginx
After=network.target

[Service]
Type=forking
ExecStart=/etc/nginx/nginx
ExecReload=/etc/nginx/nginx -s reload
ExecStop=/etc/nginx/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target

EOF
systemctl daemon-reload 
systemctl start nginx
systemctl enable nginx
```



#### 06.使用方式



```
systemctl start nginx # 启动nginx
systemctl status nginx # 查看nginx的命令
systemctl restart nginx # 重启nginx服务
systemctl stop nginx # 停止nginx服务
systemctl enable nginx # 将nginx设为开机启动
systemctl reload nginx # 重新加载配置
```