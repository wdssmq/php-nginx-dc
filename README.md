# php-nginx-dc

用于部署多站点 PHP + Nginx 环境的 Docker Compose 配置，使用 webdevops/php-nginx；

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

4、{{SSL 配置占位}}

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
