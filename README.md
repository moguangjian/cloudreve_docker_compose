# cloudreve_docker_compose

Use docker compse building cloudreve, support ssl(使用 docker compose 构建 cloudreve, 支持 ssl)

在网上没有找到具体的使用`openssl`构建`cloudreve`的教程，参考官方的教程自己写了一个使用`docker compose`来构建安全访问的`cloudreve`。

虽然有一个公网 ip，但是 80 端口和 443 端口无法访问，无法申请到免费的 ssl 证书，所以使用了 openssl 生成自己使用的 ssl 证书（使用年限自己定），本教程记录完整过程，以备忘。

## 一、准备要工作的目录及文件

```shell
cd ~
mkdir -p cloudreve_docker_compose/aria2/config
mkdir -p cloudreve_docker_compose/cloudreve/{avatar,uploads}
mkdir -p cloudreve_docker_compose/{data,temp_data}
mkdir -p cloudreve_docker_compose/nginx/{conf.d,ssl}
touch cloudreve_docker_compose/cloudreve/conf.ini
touch cloudreve_docker_compose/cloudreve/cloudreve.db
```

## 二、生成 ssl 证书

```shell
# 生成的证书将位于cloudreve_docker_compose/nginx/ssl
docker run --name ubuntu_ssl -it -v cloudreve_docker_compose/nginx/ssl:/certs ubuntu:22.04 bash
```

```shell
# 进入ubuntu_ssl 容器中执行
apt update
apt-get install openssl
# 查看openssl 版本
openssl version -a
mkdir /certs
openssl req -newkey rsa:4096 -nodes -sha256 -keyout /certs/server.key -x509 -days 36500 -out /certs/server.crt
```

## 三、docker-compose.yml

```yaml
version: "3.8"
services:
  cloudreve:
    container_name: cloudreve
    image: cloudreve/cloudreve:latest
    restart: unless-stopped
    # ports:
    # - 5212:5212
    volumes:
      - temp_data:/data
      - ./cloudreve/uploads:/cloudreve/uploads
      - ./cloudreve/conf.ini:/cloudreve/conf.ini
      - ./cloudreve/cloudreve.db:/cloudreve/cloudreve.db
      - ./cloudreve/avatar:/cloudreve/avatar
    depends_on:
      - aria2
    networks:
      - cloudreve-network
  aria2:
    container_name: aria2
    image: p3terx/aria2-pro
    restart: unless-stopped
    environment:
      - RPC_SECRET=your_aria_rpc_token
      - RPC_PORT=6800
    volumes:
      - ./aria2/config:/config
      - temp_data:/data
    networks:
      - cloudreve-network
  nginx:
    container_name: cloudreve_nginx
    image: nginx:latest
    ports:
      - 18080:80
      - 8443:443
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
      - ./nginx/logs:/var/log/nginx
      # - ./nginx/html:/usr/share/nginx/html
    networks:
      - cloudreve-network
    restart: unless-stopped
    depends_on:
      - cloudreve
volumes:
  temp_data:
    driver: local
    driver_opts:
      type: none
      device: $PWD/data
      o: bind

networks:
  cloudreve-network:
```

## 四、编写 cloudreve.conf

```shell
vi cloudreve_docker_compose/nginx/conf.d/cloudreve.conf
```

输入如下内容：

```conf
server{
  listen  443 ssl  default_server;
  server_name _;
  root  /usr/share/nginx/html;
  ssl_certificate "/etc/nginx/ssl/server.crt";
  ssl_certificate_key "/etc/nginx/ssl/server.key";

  #  ssl_session_cache shared:SSL:1m;
  #  ssl_session_timeout 1om;
  #  ssl_ciphers PROFILE=SYSTEM;
  #  ssl_prefer_server_ciphers on;
  #  Load configuration files for the default server block.include/etc/nginx/default.d/*.conf;

  location / {
    proxy_pass http://cloudreve:5212;
    proxy_set_header Host $host;
  }

  error_page 404 /404.html;
    location = /40x.html{
  }

  error_page 500  502 503 504 /50x.html;

  location = /50x.html  {
  }
}
```

## 五、运行

```shell
docker compose up -d
```

## 六、通过 https 访问

```url
https://yourip:8433
```
