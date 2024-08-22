
## 基于Nginx RTMP模块构建直播系统(Demo)

> OS: Oracle Linux 8

#### 1. 安装必要的依赖

更新系统并安装必需的依赖项：

```
sudo dnf update -y
sudo dnf install -y gcc make pcre-devel openssl-devel zlib-devel wget unzip
```

#### 2. 下载和编译 Nginx 及 RTMP 模块

下载 Nginx 和 RTMP 模块

```
cd /usr/local/src
wget http://nginx.org/download/nginx-1.25.0.tar.gz  # 确保使用最新版本
wget https://github.com/arut/nginx-rtmp-module/archive/refs/heads/master.zip
```

解压下载的文件
```
tar -zxvf nginx-1.25.0.tar.gz
unzip master.zip
```

编译并安装 Nginx

```
cd nginx-1.25.0
./configure --with-http_ssl_module --add-module=../nginx-rtmp-module-master
make
sudo make install
```
#### 3. 配置 Nginx

编辑 Nginx 配置文件，通常在 /usr/local/nginx/conf/nginx.conf。

```
sudo vi /usr/local/nginx/conf/nginx.conf
```
添加 RTMP 和 HLS 配置

在 nginx.conf 中添加以下内容：

```
worker_processes auto;
events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       8080;
        server_name  _;

        location / {
            root   html;
            index  index.html index.htm;
        }

        # HLS 设置
        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            alias /tmp/hls/;
            expires -1;
            add_header Cache-Control no-cache;
        }
    }
}

rtmp {
    server {
        listen 1935;  # RTMP 监听端口
        chunk_size 4096;

        application live {
            live on;
            record off;

            # HLS 配置
            hls on;
            hls_path /tmp/hls;
            hls_fragment 3;
            hls_playlist_length 10;
            hls_cleanup off;
        }
    }
}
```

#### 4. 启动 Nginx

使用以下命令启动 Nginx 服务：

```
sudo /usr/local/nginx/sbin/nginx
```

#### 5. 配置防火墙

确保防火墙允许 RTMP 和 HTTP 流量通过。您可以使用 firewalld 来配置防火墙。

启用防火墙端口

```
sudo firewall-cmd --permanent --add-port=1935/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

> 在OCI上，需要在虚拟机所在subnet的security list中开放以上对应的端口

#### 6. 使用 OBS 推送视频流

在 OBS 中进行以下设置：

```
流类型：自定义流服务器
URL：rtmp://<your-server-ip>/live
流密钥：例如 mystream
```
启动推流后，OBS 将视频流推送到 Nginx 服务器。

#### 7. 通过 Web 端访问视频流
打开浏览器并输入以下地址：
```
http://<your-server-ip>:8080/hls/mystream.m3u8
```
