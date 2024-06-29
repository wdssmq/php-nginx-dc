# php-nginx-dc

用于部署多站点 PHP + Nginx 环境的 Docker Compose 配置，使用 webdevops/php-nginx；

带 SSL 证书申请和续期方案

带 MySQL 数据库；

## 镜像说明

webdevops/php-nginx

链接：[https://hub.docker.com/r/webdevops/php-nginx](https://hub.docker.com/r/webdevops/php-nginx "webdevops/php-nginx - Docker Image | Docker Hub")

文档：[webdevops/php-nginx](https://dockerfile.readthedocs.io/en/latest/content/DockerImages/dockerfiles/php-nginx.html "webdevops/php-nginx")

mysql/mysql-server

链接：[https://hub.docker.com/r/mysql/mysql-server](https://hub.docker.com/r/mysql/mysql-server "mysql/mysql-server - Docker Image | Docker Hub")

文档：[MySQL :: MySQL 8.0 Reference Manual :: 2.5.6 Deploying MySQL on Linux with Docker Containers](https://dev.mysql.com/doc/refman/8.0/en/linux-installation-docker.html "MySQL :: MySQL 8.0 Reference Manual :: 2.5.6 Deploying MySQL on Linux with Docker Containers")

## 部署运行

1、环境变量设置：

项目根目录下创建 `.env` 文件，参考 `.env.sample` 文件设置环境变量；

2、启动容器：

```bash
# 启动
sudo docker-compose up -d

# 停止
sudo docker-compose stop

# 完全移除容器
sudo docker-compose down

# 查看配置
sudo docker-compose config

```

## 添加子站点并配置域名

1、在 `data-www` 目录下新建站点目录，例如 `site1`，用于存放站点文件；

2、在 `data-nginx/vhost` 目录下新建站点配置文件 `site1.conf`；

3、参考下方示例配置文件，修改 `server_name` 和 `root`；

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name site1.com www.site1.com;

    root "/app/site1";
    index index.php index.html index.htm;

    include /opt/docker/etc/nginx/vhost.common.d/*.conf;
}

```

4、SSL 申请及续期：

配置中`acme.sh` 容器并以守护进程方式运行，用于申请和续期 SSL 证书；

「[说明 · acmesh-official/acme.sh Wiki](https://github.com/acmesh-official/acme.sh/wiki/%E8%AF%B4%E6%98%8E "说明 · acmesh-official/acme.sh Wiki")」

「[Run acme.sh in docker](https://github.com/acmesh-official/acme.sh/wiki/Run-acme.sh-in-docker#3-run-acmesh-as-a-docker-daemon "Run acme.sh in docker")」

```bash
# 邮箱注册
sudo docker exec -it acme.sh -register-account -m mail@example.com

# 申请证书 - 方式一 - 文件验证
sudo docker exec -it acme.sh -issue -d site1.com --webroot /app/site1

# 申请证书 - 方式二 - DNS 验证 - 支持泛域名
# https://dash.cloudflare.com/profile/api-tokens
sudo docker exec \
    -e CF_Key="CF_KeyCF_KeyCF_KeyCF_KeyCF_Key" \
    -e CF_Email="mail@example.com" \
    acme.sh --issue -d site1.com -d *.site1.com --dns dns_cf

# ↑ 两个 -d 参数，一个是主域名，一个是泛域名
# 更多 DNS API 配置见：https://github.com/acmesh-official/acme.sh/wiki/dnsapi

# 部署证书到实际路径，替换 site1.com 为实际域名后执行
sudo docker exec \
    -e DEPLOY_DOCKER_CONTAINER_LABEL=for_deploy_ssl \
    -e DEPLOY_DOCKER_CONTAINER_KEY_FILE=/opt/docker/etc/nginx/ssl/site1.com/key.pem \
    -e DEPLOY_DOCKER_CONTAINER_CERT_FILE="/opt/docker/etc/nginx/ssl/site1.com/cert.pem" \
    -e DEPLOY_DOCKER_CONTAINER_CA_FILE="/opt/docker/etc/nginx/ssl/site1.com/ca.pem" \
    -e DEPLOY_DOCKER_CONTAINER_FULLCHAIN_FILE="/opt/docker/etc/nginx/ssl/site1.com/full.pem" \
    -e DEPLOY_DOCKER_CONTAINER_RELOAD_CMD="nginx -s reload" \
    acme.sh --deploy -d site1.com --deploy-hook docker

# 证书续期：上述操作的参数会记录在 ./data-nginx/acme.sh 内，可直接执行下方命令续期
sudo docker exec -it acme.sh --cron
# ↑ 定时执行配置 crontab -e —— `0 0 * * * docker exec -it acme.sh --cron`


```

「[deploy to docker containers](https://github.com/acmesh-official/acme.sh/wiki/deploy-to-docker-containers "deploy to docker containers")」

```nginx

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name site1.com;

    root "/app/site1";
    index index.php index.html index.htm;

    include /opt/docker/etc/nginx/vhost.common.d/*.conf;


    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers "TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5";

    ssl_prefer_server_ciphers on;

    ssl_certificate     /opt/docker/etc/nginx/ssl/site1.com/full.pem;
    ssl_certificate_key /opt/docker/etc/nginx/ssl/site1.com/key.pem;
}

```

5、伪静态：`vhost.common.d/10-location-root.conf` 已经包含了一个通用的规则；

6、站点文件权限，数据库创建管理，Nginx 重启等操作参考下方命令；

## 命令管理

```sh
# 修改 data-www 内权限
sudo chown -R 1000:1000 data-www


# 命令行创建数据库，db_password docker_site1 等自行替换
sudo docker exec -it MySQL mysql \
    -uroot -pdb_password \
    -e "CREATE DATABASE docker_site1 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
# Z-BlogPHP 可以直接在安装时创建


# 重启容器内的 Nginx
docker exec -it www_app nginx -s reload


# 进入容器
docker exec -it www_app /bin/bash

```


## phpMyAdmin 连接管理数据库

如果需要 phpMyAdmin 可单独配置：

```bash
# 强制删除容器
docker rm --force phpMyAdmin
docker run --name phpMyAdmin \
  --network=php-nginx-dc_net_web \
  -p 9100:80 \
  -e PMA_HOST=MySQL \
  -e UPLOAD_LIMIT=20M \
  -d phpmyadmin/phpmyadmin

# 关闭（但不删除）
docker stop phpMyAdmin
# 启用
docker start phpMyAdmin

```


## 其他

**以下命令用于从原始容器复制初始配置，不理解什么用的话无视就好：**

```sh
# 容器外执行
NGINX_DIR=~/24-docker/php-nginx-dc/nginx
rm -rf "${NGINX_DIR}"
# mkdir -p "${NGINX_DIR}"
# docker cp www_app:/etc/nginx/nginx.conf "${NGINX_DIR}/"
# ↑ 这里会复制到软链接，真实文件下边的文件夹包含了。Orz
docker cp www_app:/opt/docker/etc/nginx "${NGINX_DIR}"

```

_吐槽：a1 的真实文件是 b1，b1 里引用了 a2，但是 a2 的真实文件是 b2。。_


## 投喂支持

爱发电：[https://afdian.net/a/wdssmq](https://afdian.net/a/wdssmq "沉冰浮水正在创作和 z-blog 相关或无关的各种有用或没用的代码 | 爱发电")

哔哩哔哩：[https://space.bilibili.com/44744006](https://space.bilibili.com/44744006 "沉冰浮水的个人空间\_哔哩哔哩\_bilibili")「投币或充电」「[大会员卡券领取 - bilibili](https://account.bilibili.com/account/big/myPackage "大会员卡券领取 - bilibili")」

RSS 订阅：[https://feed.wdssmq.com](https://feed.wdssmq.com "沉冰浮水博客的 RSS 订阅地址") 「[「言说」RSS 是一种态度！！](https://www.wdssmq.com/post/20201231613.html "「言说」RSS 是一种态度！！")」

在更多平台关注我：[https://www.wdssmq.com/guestbook.html#其他出没站点和信息](https://www.wdssmq.com/guestbook.html#%E5%85%B6%E4%BB%96%E5%87%BA%E6%B2%A1%E5%9C%B0%E7%82%B9%E5%92%8C%E4%BF%A1%E6%81%AF "在更多平台关注我")

更多「小代码」：[https://cn.bing.com/search?q=小代码+沉冰浮水](https://cn.bing.com/search?q=%E5%B0%8F%E4%BB%A3%E7%A0%81+%E6%B2%89%E5%86%B0%E6%B5%AE%E6%B0%B4 "小代码 沉冰浮水 - 必应搜索")

<!-- ##################################### -->
