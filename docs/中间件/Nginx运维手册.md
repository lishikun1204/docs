# Nginx 运维手册

> 版本：Nginx 1.24+  
> 适用对象：运维工程师、开发工程师  
> 最后更新：2026-03-08

---

## 目录

1. [Nginx 安装部署](#一-nginx-安装部署)
2. [配置管理](#二-配置管理)
3. [日常运维操作](#三-日常运维操作)
4. [性能监控与优化](#四-性能监控与优化)
5. [安全加固](#五-安全加固)
6. [高可用方案实施](#六-高可用方案实施)
7. [常见故障处理](#七-常见故障处理)

---

## 一、Nginx 安装部署

### 1.1 环境准备

#### 1.1.1 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | Linux (CentOS 7/8, Ubuntu 18.04/20.04/22.04) |
| CPU | 建议 2 核以上 |
| 内存 | 建议 2GB 以上 |
| 磁盘 | 建议 10GB 以上 |

#### 1.1.2 依赖安装

```bash
# CentOS/RHEL
yum install -y gcc gcc-c++ make pcre pcre-devel zlib zlib-devel openssl openssl-devel wget

# Ubuntu/Debian
apt-get update
apt-get install -y build-essential libpcre3 libpcre3-dev zlib1g zlib1g-dev libssl-dev wget
```

### 1.2 源码安装

#### 1.2.1 下载与编译

```bash
# 创建安装目录
mkdir -p /opt/nginx
cd /opt/nginx

# 下载 Nginx 1.24.0（稳定版）
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar xzf nginx-1.24.0.tar.gz
cd nginx-1.24.0

# 配置编译选项
./configure \
  --prefix=/usr/local/nginx \
  --user=nginx \
  --group=nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_realip_module \
  --with-http_stub_status_module \
  --with-http_gzip_static_module \
  --with-pcre \
  --with-stream \
  --with-stream_ssl_module

# 编译安装
make
make install

# 创建 nginx 用户
useradd -r -s /sbin/nologin nginx
```

#### 1.2.2 创建系统服务

```bash
# 创建 systemd 服务文件
cat > /etc/systemd/system/nginx.service << 'EOF'
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

# 启动服务
systemctl daemon-reload
systemctl enable nginx
systemctl start nginx
systemctl status nginx
```

### 1.3 包管理器安装

#### 1.3.1 CentOS/RHEL

```bash
# 添加 EPEL 和 Nginx 官方源
yum install -y epel-release
rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm

# 安装 Nginx
yum install -y nginx

# 启动服务
systemctl enable nginx
systemctl start nginx
```

#### 1.3.2 Ubuntu/Debian

```bash
# 添加 Nginx 官方源
curl -fsSL https://nginx.org/keys/nginx_signing.key | apt-key add -
echo "deb http://nginx.org/packages/ubuntu/ $(lsb_release -cs) nginx" > /etc/apt/sources.list.d/nginx.list

# 安装 Nginx
apt-get update
apt-get install -y nginx

# 启动服务
systemctl enable nginx
systemctl start nginx
```

### 1.4 Docker 安装

#### 1.4.1 单容器部署

```bash
# 拉取镜像
docker pull nginx:1.24-alpine

# 运行容器
docker run -d \
  --name nginx \
  --restart always \
  -p 80:80 \
  -p 443:443 \
  -v /data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /data/nginx/conf.d:/etc/nginx/conf.d:ro \
  -v /data/nginx/html:/usr/share/nginx/html:ro \
  -v /data/nginx/logs:/var/log/nginx \
  -v /data/nginx/ssl:/etc/nginx/ssl:ro \
  nginx:1.24-alpine
```

#### 1.4.2 Docker Compose 部署

```yaml
version: '3.8'
services:
  nginx:
    image: nginx:1.24-alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./html:/usr/share/nginx/html:ro
      - ./logs:/var/log/nginx
      - ./ssl:/etc/nginx/ssl:ro
    networks:
      - webnet
networks:
  webnet:
    driver: bridge
```

### 1.5 安装验证

```bash
# 检查版本
nginx -v
nginx -V

# 检查配置语法
nginx -t

# 检查进程
ps aux | grep nginx

# 检查端口
netstat -tlnp | grep nginx
ss -tlnp | grep nginx

# 访问测试
curl -I http://localhost
```

---

## 二、配置管理

### 2.1 配置文件结构

```
/usr/local/nginx/
├── conf/
│   ├── nginx.conf          # 主配置文件
│   ├── mime.types          # MIME 类型定义
│   ├── fastcgi.conf        # FastCGI 配置
│   └── conf.d/             # 额外配置目录
│       ├── default.conf
│       └── ssl.conf
├── logs/
│   ├── access.log          # 访问日志
│   ├── error.log           # 错误日志
│   └── nginx.pid           # PID 文件
├── html/
│   └── index.html          # 默认首页
└── sbin/
    └── nginx               # 可执行文件
```

### 2.2 主配置文件详解

```nginx
# /usr/local/nginx/conf/nginx.conf

# 全局配置
user  nginx;                          # 运行用户
worker_processes  auto;               # 工作进程数，auto = CPU核心数
worker_rlimit_nofile 65535;           # 每个进程最大打开文件数
error_log  /var/log/nginx/error.log warn;  # 错误日志级别
pid        /var/run/nginx.pid;        # PID 文件位置

# 事件模块
events {
    use epoll;                        # 事件驱动模型
    worker_connections  10240;        # 每个进程最大连接数
    multi_accept on;                  # 同时接受多个连接
}

# HTTP 模块
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 日志格式定义
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      '$request_time $upstream_response_time';

    access_log  /var/log/nginx/access.log  main;

    # 性能优化
    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    keepalive_timeout  65;
    types_hash_max_size 2048;

    # Gzip 压缩
    gzip  on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript 
               application/xml application/xml+rss text/javascript;

    # 请求体大小限制
    client_max_body_size 50m;
    client_body_buffer_size 128k;

    # 代理缓冲
    proxy_buffer_size 4k;
    proxy_buffers 4 32k;
    proxy_busy_buffers_size 64k;

    # 上游服务器定义
    upstream backend {
        server 192.168.1.101:8080 weight=3;
        server 192.168.1.102:8080 weight=2;
        server 192.168.1.103:8080 backup;
        keepalive 32;
    }

    # 包含额外配置
    include /etc/nginx/conf.d/*.conf;
}
```

### 2.3 虚拟主机配置

#### 2.3.1 静态网站

```nginx
server {
    listen 80;
    server_name www.example.com example.com;
    root /var/www/example;
    index index.html index.htm;

    # 访问日志
    access_log /var/log/nginx/example.access.log main;
    error_log /var/log/nginx/example.error.log;

    # 静态文件缓存
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|txt)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # 自定义错误页面
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

#### 2.3.2 反向代理

```nginx
server {
    listen 80;
    server_name api.example.com;

    # 反向代理到后端服务
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";

        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # 错误处理
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_next_upstream_tries 3;
    }

    # API 文档
    location /swagger/ {
        proxy_pass http://backend/swagger-ui.html;
    }
}
```

#### 2.3.3 HTTPS 配置

```nginx
server {
    listen 443 ssl http2;
    server_name www.example.com;

    # SSL 证书配置
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;

    # SSL 协议和加密套件
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/ssl/chain.crt;
    resolver 8.8.8.8 8.8.4.4 valid=300s;

    root /var/www/example;
    index index.html;
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name www.example.com;
    return 301 https://$server_name$request_uri;
}
```

### 2.4 负载均衡配置

#### 2.4.1 负载均衡算法

```nginx
# 轮询（默认）
upstream backend_round {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

# 加权轮询
upstream backend_weight {
    server 192.168.1.101:8080 weight=3;
    server 192.168.1.102:8080 weight=2;
    server 192.168.1.103:8080 weight=1;
}

# IP 哈希（会话保持）
upstream backend_iphash {
    ip_hash;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

# 最少连接
upstream backend_least {
    least_conn;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

# 一致性哈希
upstream backend_hash {
    hash $request_uri consistent;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

#### 2.4.2 健康检查

```nginx
upstream backend {
    zone backend 64k;
    
    server 192.168.1.101:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.102:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 192.168.1.103:8080 backup;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=5s fails=3 passes=2;
    }
}
```

### 2.5 缓存配置

#### 2.5.1 代理缓存

```nginx
# 缓存路径配置
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m 
                 max_size=10g inactive=60m use_temp_path=off;

server {
    listen 80;
    server_name cache.example.com;

    location / {
        proxy_cache my_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_key $scheme$request_method$host$request_uri;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        add_header X-Cache-Status $upstream_cache_status;
        
        proxy_pass http://backend;
    }
}
```

#### 2.5.2 静态文件缓存

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;

    # 图片缓存
    location ~* \.(jpg|jpeg|png|gif|webp|svg)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # CSS/JS 缓存
    location ~* \.(css|js)$ {
        expires 7d;
        add_header Cache-Control "public";
    }

    # 字体文件
    location ~* \.(woff|woff2|ttf|otf|eot)$ {
        expires 90d;
        add_header Cache-Control "public, immutable";
        add_header Access-Control-Allow-Origin "*";
    }
}
```

### 2.6 配置热加载

```bash
# 测试配置语法
nginx -t

# 平滑重载配置（不中断服务）
nginx -s reload

# 或使用 systemd
systemctl reload nginx
```

---

## 三、日常运维操作

### 3.1 服务管理

```bash
# 启动服务
systemctl start nginx
/usr/local/nginx/sbin/nginx

# 停止服务
systemctl stop nginx
nginx -s stop       # 快速停止
nginx -s quit       # 优雅停止（处理完当前请求）

# 重启服务
systemctl restart nginx

# 重载配置
systemctl reload nginx
nginx -s reload

# 查看状态
systemctl status nginx

# 查看进程
ps aux | grep nginx
pstree -p | grep nginx
```

### 3.2 日志管理

#### 3.2.1 日志格式

```nginx
# 主格式
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';

# JSON 格式
log_format json escape=json '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"request":"$request",'
    '"status":"$status",'
    '"body_bytes_sent":"$body_bytes_sent",'
    '"request_time":"$request_time",'
    '"upstream_response_time":"$upstream_response_time",'
    '"http_referrer":"$http_referer",'
    '"http_user_agent":"$http_user_agent"'
'}';
```

#### 3.2.2 日志操作

```bash
# 查看访问日志
tail -f /var/log/nginx/access.log

# 查看错误日志
tail -f /var/log/nginx/error.log

# 实时监控访问量
tail -f /var/log/nginx/access.log | grep -o '^[0-9.]*' | sort | uniq -c

# 统计状态码
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 统计访问量前10的IP
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 统计访问量前10的URL
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 日志切割脚本
#!/bin/bash
LOG_DIR="/var/log/nginx"
DATE=$(date +%Y%m%d)
mv $LOG_DIR/access.log $LOG_DIR/access-$DATE.log
mv $LOG_DIR/error.log $LOG_DIR/error-$DATE.log
nginx -s reopen
find $LOG_DIR -name "*.log" -mtime +30 -delete
```

### 3.3 流量控制

#### 3.3.1 限流配置

```nginx
# 定义限流区域
http {
    # 基于IP的请求限制（每秒10个请求）
    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;
    
    # 基于IP的连接限制
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    
    # 基于URI的请求限制
    limit_req_zone $server_name zone=server_limit:10m rate=100r/s;

    server {
        location /api/ {
            # 请求限流，允许突发5个请求
            limit_req zone=req_limit burst=5 nodelay;
            
            # 连接限流，每个IP最多10个连接
            limit_conn conn_limit 10;
            
            proxy_pass http://backend;
        }
        
        # 限流后的响应
        limit_req_status 429;
        limit_conn_status 429;
    }
}
```

#### 3.3.2 带宽限制

```nginx
server {
    location /download/ {
        # 限制下载速度为 100KB/s
        limit_rate 100k;
        
        # 传输 1MB 后开始限速
        limit_rate_after 1m;
        
        alias /var/www/download/;
    }
}
```

### 3.4 访问控制

#### 3.4.1 IP 访问控制

```nginx
server {
    location /admin/ {
        # 允许特定IP
        allow 192.168.1.0/24;
        allow 10.0.0.0/8;
        
        # 拒绝其他所有
        deny all;
        
        proxy_pass http://backend;
    }

    location /internal/ {
        # 仅允许内网访问
        allow 127.0.0.1;
        allow 192.168.0.0/16;
        allow 10.0.0.0/8;
        deny all;
        
        root /var/www/internal;
    }
}
```

#### 3.4.2 基本认证

```bash
# 创建密码文件
yum install -y httpd-tools
htpasswd -c /etc/nginx/.htpasswd admin
htpasswd /etc/nginx/.htpasswd user1
```

```nginx
server {
    location /admin/ {
        auth_basic "Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        
        proxy_pass http://backend;
    }
}
```

### 3.5 URL 重写

```nginx
server {
    # 域名重定向
    server_name old.example.com;
    return 301 https://new.example.com$request_uri;

    location / {
        # 将 /article/123 重写为 /article?id=123
        rewrite ^/article/(\d+)$ /article?id=$1 last;
        
        # 将 /user/profile 重写为 /user.php?action=profile
        rewrite ^/user/(\w+)$ /user.php?action=$1 last;
        
        # 将旧路径重定向到新路径
        rewrite ^/old-path/(.*)$ /new-path/$1 permanent;
        
        # 防盗链
        location ~* \.(jpg|jpeg|png|gif)$ {
            valid_referers none blocked server_names *.example.com;
            if ($invalid_referer) {
                return 403;
            }
        }
    }
}
```

---

## 四、性能监控与优化

### 4.1 状态监控

#### 4.1.1 开启状态页面

```nginx
server {
    listen 80;
    server_name localhost;

    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        allow 192.168.0.0/16;
        deny all;
    }
}
```

#### 4.1.2 状态指标说明

```bash
# 访问状态页面
curl http://localhost/nginx_status

# 输出示例：
# Active connections: 291 
# server accepts handled requests
#   16630948 16630948 31070465 
# Reading: 6 Writing: 179 Waiting: 106
```

| 指标 | 说明 |
|------|------|
| Active connections | 当前活跃连接数 |
| accepts | 已接受的连接总数 |
| handled | 已处理的连接总数 |
| requests | 请求总数 |
| Reading | 正在读取请求头的连接数 |
| Writing | 正在写入响应的连接数 |
| Waiting | 等待下一个请求的连接数 |

### 4.2 性能优化

#### 4.2.1 内核参数优化

```bash
# /etc/sysctl.conf
# 最大文件打开数
fs.file-max = 655350

# TCP 连接复用
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1

# TCP 连接队列
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# TCP 超时
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200

# 应用配置
sysctl -p
```

#### 4.2.2 Nginx 配置优化

```nginx
# 工作进程优化
worker_processes auto;               # 自动检测CPU核心数
worker_cpu_affinity auto;            # 自动绑定CPU
worker_rlimit_nofile 65535;          # 最大文件描述符

events {
    use epoll;                       # Linux 高性能事件模型
    worker_connections 65535;        # 每个进程最大连接数
    multi_accept on;                 # 同时接受多个连接
    accept_mutex off;                # 高负载时关闭互斥锁
}

http {
    # 连接优化
    keepalive_timeout 65;            # 长连接超时
    keepalive_requests 10000;        # 长连接最大请求数
    
    # 文件传输优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # 缓冲优化
    client_body_buffer_size 16k;
    client_header_buffer_size 1k;
    client_max_body_size 50m;
    large_client_header_buffers 4 8k;
    
    # 输出缓冲
    output_buffers 1 32k;
    postpone_output 1460;
    
    # 文件缓存
    open_file_cache max=65535 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
}
```

#### 4.2.3 Gzip 压缩优化

```nginx
http {
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1k;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types 
        text/plain 
        text/css 
        text/xml 
        text/javascript 
        application/json 
        application/javascript 
        application/xml 
        application/xml+rss 
        application/x-javascript
        image/svg+xml;
    gzip_disable "msie6";
}
```

### 4.3 性能测试

```bash
# 使用 ab 测试
ab -n 10000 -c 100 http://localhost/

# 使用 wrk 测试
wrk -t12 -c400 -d30s http://localhost/

# 使用 hey 测试
hey -n 10000 -c 100 http://localhost/

# 使用 siege 测试
siege -c 100 -r 100 http://localhost/
```

---

## 五、安全加固

### 5.1 隐藏版本信息

```nginx
http {
    server_tokens off;
    more_clear_headers 'Server';
    more_clear_headers 'X-Powered-By';
}
```

### 5.2 安全头配置

```nginx
server {
    # 防止点击劫持
    add_header X-Frame-Options "SAMEORIGIN" always;
    
    # 防止 MIME 类型嗅探
    add_header X-Content-Type-Options "nosniff" always;
    
    # XSS 防护
    add_header X-XSS-Protection "1; mode=block" always;
    
    # 内容安全策略
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" always;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # 引用策略
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # 权限策略
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
}
```

### 5.3 SSL/TLS 加固

```nginx
server {
    listen 443 ssl http2;
    
    # 仅使用 TLS 1.2 和 1.3
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # 强加密套件
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    
    # DH 参数
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;
    
    # 会话缓存
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
}
```

### 5.4 防护配置

#### 5.4.1 防止 SQL 注入

```nginx
server {
    location ~* "(union.*select.*\(|select.*from.*information_schema|insert.*into.*values|delete.*from|drop.*table|exec.*xp_cmdshell)" {
        return 403;
    }
}
```

#### 5.4.2 防止目录遍历

```nginx
server {
    # 禁止访问敏感目录
    location ~* ^/(config|data|logs|tmp|backup|admin)/ {
        deny all;
        return 404;
    }
    
    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

#### 5.4.3 防止恶意 User-Agent

```nginx
server {
    if ($http_user_agent ~* (bot|crawl|spider|scan|nikto|sqlmap|nmap|masscan)) {
        return 403;
    }
    
    # 空User-Agent拒绝
    if ($http_user_agent = "") {
        return 403;
    }
}
```

### 5.5 WAF 配置

```nginx
# 使用 ModSecurity
load_module modules/ngx_http_modsecurity_module.so;

http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsecurity/modsecurity.conf;
}

# 或使用 Naxsi
location / {
    DeniedUrl "/RequestDenied";
    CheckRule "$SQL >= 8" BLOCK;
    CheckRule "$RFI >= 8" BLOCK;
    CheckRule "$TRAVERSAL >= 4" BLOCK;
    CheckRule "$EVADE >= 4" BLOCK;
    CheckRule "$XSS >= 8" BLOCK;
    
    proxy_pass http://backend;
}

location /RequestDenied {
    return 403;
}
```

---

## 六、高可用方案实施

### 6.1 Keepalived + Nginx

#### 6.1.1 架构说明

```
                    VIP: 192.168.1.100
                           |
            +--------------+--------------+
            |                             |
    Master Nginx                   Backup Nginx
    192.168.1.101                  192.168.1.102
            |                             |
            +--------------+--------------+
                           |
                    后端服务器集群
```

#### 6.1.2 Keepalived 配置

**Master 节点配置：**

```bash
# /etc/keepalived/keepalived.conf
global_defs {
    router_id nginx_master
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    
    virtual_ipaddress {
        192.168.1.100
    }
    
    track_script {
        check_nginx
    }
}
```

**Backup 节点配置：**

```bash
# /etc/keepalived/keepalived.conf
global_defs {
    router_id nginx_backup
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    
    virtual_ipaddress {
        192.168.1.100
    }
    
    track_script {
        check_nginx
    }
}
```

**健康检查脚本：**

```bash
#!/bin/bash
# /etc/keepalived/check_nginx.sh

if [ $(ps -C nginx --no-header | wc -l) -eq 0 ]; then
    systemctl start nginx
    sleep 2
fi

if [ $(ps -C nginx --no-header | wc -l) -eq 0 ]; then
    systemctl stop keepalived
fi
```

### 6.2 动态负载均衡

#### 6.2.1 使用 Consul 服务发现

```nginx
# 安装 nginx-upsync-module
# 编译时添加 --add-module=/path/to/nginx-upsync-module

upstream backend {
    server 127.0.0.1:11111;
    
    upsync 127.0.0.1:8500/v1/kv/upstreams/backend upsync_timeout=6m upsync_interval=500ms upsync_type=consul strong_dependency=off;
    
    upsync_dump_path /var/log/nginx/backend.conf;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

#### 6.2.2 动态添加/删除后端

```bash
# 添加后端服务器
curl -X PUT -d '{"server":"192.168.1.103:8080"}' http://127.0.0.1:8500/v1/kv/upstreams/backend/192.168.1.103:8080

# 删除后端服务器
curl -X DELETE http://127.0.0.1:8500/v1/kv/upstreams/backend/192.168.1.103:8080

# 查看所有后端
curl http://127.0.0.1:8500/v1/kv/upstreams/backend?recurse
```

---

## 七、常见故障处理

### 7.1 常见错误码

| 错误码 | 说明 | 可能原因 |
|--------|------|----------|
| 400 | Bad Request | 请求语法错误 |
| 403 | Forbidden | 权限不足、目录索引禁用 |
| 404 | Not Found | 文件不存在、路径错误 |
| 429 | Too Many Requests | 触发限流 |
| 500 | Internal Server Error | 后端程序错误 |
| 502 | Bad Gateway | 后端服务不可用 |
| 503 | Service Unavailable | 后端服务过载 |
| 504 | Gateway Timeout | 后端服务超时 |

### 7.2 故障排查流程

#### 7.2.1 502 Bad Gateway

```bash
# 1. 检查后端服务状态
systemctl status backend
netstat -tlnp | grep 8080

# 2. 检查 Nginx 错误日志
tail -f /var/log/nginx/error.log

# 3. 检查 SELinux
getenforce
setenforce 0

# 4. 检查防火墙
firewall-cmd --list-all

# 5. 测试后端连通性
curl http://127.0.0.1:8080
telnet 127.0.0.1 8080
```

**解决方案：**

```nginx
# 增加超时时间
proxy_connect_timeout 300s;
proxy_send_timeout 300s;
proxy_read_timeout 300s;

# 增加缓冲区
proxy_buffer_size 16k;
proxy_buffers 4 64k;
proxy_busy_buffers_size 128k;

# 检查 upstream 配置
upstream backend {
    server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
}
```

#### 7.2.2 504 Gateway Timeout

```bash
# 1. 检查后端处理时间
# 2. 检查网络延迟
ping backend_server
traceroute backend_server

# 3. 查看后端日志
```

**解决方案：**

```nginx
# 增加超时配置
location /api/ {
    proxy_pass http://backend;
    proxy_connect_timeout 300;
    proxy_send_timeout 300;
    proxy_read_timeout 300;
    
    # 快速失败
    proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
}
```

#### 7.2.3 连接数过多

```bash
# 查看当前连接数
netstat -an | grep :80 | wc -l

# 查看 Nginx 状态
curl http://localhost/nginx_status

# 查看系统连接状态
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

**解决方案：**

```nginx
# 增加连接数限制
events {
    worker_connections 65535;
}

# 优化 keepalive
http {
    keepalive_timeout 65;
    keepalive_requests 10000;
}

# 调整系统参数
# /etc/sysctl.conf
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.core.somaxconn = 65535
```

### 7.3 性能问题排查

#### 7.3.1 CPU 使用率高

```bash
# 查看 Nginx 进程 CPU 使用
top -p $(pgrep nginx | tr '\n' ',')

# 查看进程详情
pidstat -p $(pgrep nginx | head -1) 1 5

# 分析调用栈
perf top -p $(pgrep nginx | head -1)
```

**优化方案：**

```nginx
# 减少 worker 进程数
worker_processes auto;

# 关闭不必要的日志
access_log off;

# 优化正则表达式
# 避免复杂的正则匹配
```

#### 7.3.2 内存使用高

```bash
# 查看内存使用
ps aux | grep nginx
pmap -x $(pgrep nginx | head -1)

# 查看连接内存
ss -n | awk '{print $1}' | sort | uniq -c
```

**优化方案：**

```nginx
# 限制缓冲区大小
client_body_buffer_size 16k;
client_header_buffer_size 1k;
large_client_header_buffers 4 8k;

# 限制请求体大小
client_max_body_size 10m;

# 优化代理缓冲
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 4 32k;
```

### 7.4 日志分析

```bash
# 统计各状态码数量
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 统计访问量前10的IP
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 统计请求时间最长的10个请求
awk '{print $NF, $7}' access.log | sort -rn | head -10

# 统计某时间段的访问量
awk '$4 >= "[08/Mar/2026:00:00:00" && $4 <= "[08/Mar/2026:23:59:59"' access.log | wc -l

# 统计被攻击的IP
awk '{print $1}' access.log | sort | uniq -c | sort -rn | awk '$1 > 1000 {print}'
```

---

**修订记录**

| 版本 | 日期 | 作者 | 修改内容 |
|------|------|------|----------|
| v1.0 | 2026-03-08 | 运维团队 | 初始版本 |
