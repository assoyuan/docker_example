# 简单易懂的docker教程

## 基本概念

> 镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例一样，镜像是静态的定义，容器是镜像运行时的实体，容器可以被创建、启动、停止、删除、暂停等。

## 常用命令

```
docker pull pkg:version     # 拉取 image
docker images               # 查看所有 image
docker rmi myphp            # 删除 image
docker ps                   # 查看运行中的 container
docker ps -a                # 查看所有 container
docker inspect myphp        # 查看 container 配置
docker logs myphp           # 查看 container 日志
docker restart myphp        # 重启 container
docker stop myphp           # 停止 container
docker rm myphp             # 删除 container
docker exec -it myphp bash  # 进入到 container 
docker build -t repository_name[:tag_name] Dockerfile_dir # 构建 image
docker volume ls            # 查看数据卷
docker volume prune         # 删除所有数据卷！！！！！！超级危险，生产环境不要执行！！！！！！
```

## Mac使用docker搭建PHP环境

### （一阶）使用docker逐一构建

####  首先创建对应的目录结构

```
lnmp -|
     -| nginx   -|
                -| conf.d -|
                -| nginx.conf
                -| fastcgi_params
                -| mime.types
              
     -| logs    -|
                -| nginx -|
             
     -| mysql   -|
     
     -| php     -|
     
     -| www     -|
                -| index.php

```

#### 0. 在网上自己找教程安装docker

#### 1. 下载镜像

```
docker pull php:7.2-fpm     # 冒号后选择版本

docker pull nginx

docker pull mysql:5.7       # 不需要本地数据库可忽略

docker pull redis:3.2       # 不需要本地redis可忽略

docker images               # 查看已下载的所有镜像
```

#### 2. 创建容器

==要注意容器的启动顺序==

参数解释：

```
注：

-i 表示允许我们对容器进行操作

-t 表示在新容器内指定一个为终端

-d 表示容器在后台执行

/bin/bash 这将在容器内启动bash shell

-p 为容器和宿主机创建端口映射

--name 为容器指定一个名字

-v 将容器内路径挂载到宿主机路径

--privileged=true 给容器特权,在挂载目录后容器可以访问目录以下的文件或者目录

--link可以用来链接2个容器，使得源容器（被链接的容器）和接收容器（主动去链接的容器）之间可以互相通信，解除了容器之间通信对容器IP的依赖 
格式：原容器名:设置的别名
在nginx配置文件中，监听php的ip，就可以使用别名，而不使用ip
```

```
###### mysql容器

docker run --name mydb -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7

# 注：-MYSQL_ROOT_PASSWORD=123456 给mysql设置初始密码，如果不需要搭建本地数据库直接下一步

###### redis容器

docker run --name myredis -p 6379:6379 -d redis:3.2

# 注: 如果不需要搭建本地redis直接下一步


###### php容器容器

docker run --name myphp -d -p 9000:9000 -e TZ=Asia/Shanghai -v ~/Documents/lnmp/www:/var/www/html -v ~/Documents/lnmp/php:/usr/local/etc/php --link mydb:mydb --link myredis:myredis --privileged=true php:7.2-fpm

# 注： 如果不需要搭建本地数据库或者redis可以省去--link mydb:mydb --link myredis:myredis
#注意-v 挂载一个空文件夹是会覆盖容器中的内容,所以配置文件要事先准备好


###### nginx容器

docker run --name mynginx -d -p 80:80 -e TZ=Asia/Shanghai -v ~/Documents/lnmp/www:/usr/share/nginx/html -v ~/Documents/lnmp/nginx:/etc/nginx -v ~/Documents/lnmp/logs/nginx:/var/log/nginx --link myphp:myphp --privileged=true nginx

#注：
#-v语句冒号后是容器内的路径 我将nginx的网页项目目录 配置目录 日志目录分别挂载到了我事先准备好的~/Documents/lnmp目录下
#--link myphp:myphp 将nginx容器和php容器连接 通过别名myphp就不再#需要去指定myphp容器的ip了 
```

`docker ps -a` 查看所有容器运行成功 这里环境也就基本搭建完成了

==此时访问localhost，应该存在很多问题需要排坑，大多是nginx配置问题，接着往下看==

如果有容器创建失败，则可以使用docker logs name 查看日志，按照日志报错解决问题

挂载目录后就可以不用进入容器中修改配置，直接在对应挂载目录下改配置文件 修改nginx配置到 ~/Documents/lnmp/nginx/conf.d/xxx.conf

nginx配置示例

```
#user  nobody;
worker_processes  1;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root /usr/share/nginx/html;
            # 这里是工作目录指定，已经映射到自定义的文件夹
            # 如果入口还在下层，可以在后面加目录，比如：/usr/share/nginx/html/public
            index index.html index.htm index.php;
        }
        
        location ~ \.php$ {
           fastcgi_pass   myphp:9000;
           # 容器与容器之间建立连接必须指定对方的ip，使用命令docker inspect myphp可以看到最现在IPAdress参数就是该容器ip
           # 我们在创建容器时，已经通过 --link 的方式创建容器，我们可以使用被link容器的别名进行访问，而不是通过ip，解除了对ip的依赖
           fastcgi_index  index.php;
           fastcgi_param  SCRIPT_FILENAME /var/www/html/$fastcgi_script_name;
           # myphp和mynginx的工作目录不同，nginx的是/usr/share/nginx/html
           # php的是/var/www/html 所以在创建容器时我们已经将两个目录都挂载到宿主主机相同的目录上了~/Documents/lnmp/www，介在这里不能使用宿主主机公共挂载目录，要使用php的工作目录
           include        fastcgi_params;
        }
    }

    include /etc/nginx/conf.d/*.conf;
}
```

#### 3. php扩展安装

```
docker exec -ti myphp /bin/bash      # 首先进入容器

docker-php-ext-install pdo pdo_mysql # 安装pdo_mysql扩展

docker-php-ext-install redis
# 注: 此时报错提示redis.so 因为一些扩展并不包含在 PHP 源码文件中
```

==解决办法：==

1. 通过源码安装

```
tar zxvf /server/php_lib/redis-4.1.0.tgz # 解压已经下载好的redis扩展包

docker cp /server/php_lib/redis-4.1.0 myphp:/usr/src/php/ext/redis # 将扩展放到容器中 再执行安装
```

```
注：
直接将扩展包放到容器ext目录里可能会报错Error: No such container:path: myphp:/usr/src/php/ext
你可以多开一个服务器窗口 进入php容器中执行docker-php-ext-install  redis此时报错error: /usr/src/php/ext/redis does not exist
保持这个状态然后在你的第一个服务器窗口执行上条命令就成功了 
(具体原因未知但确实要执行一次docker-php-ext-install命令 容器中才会开放/usr/src/php/ext这个目录)
```

2. 通过pecl安装 ==（推荐）==

```
# 注: 
# 官方推荐使用 PECL（PHP 的扩展库仓库，通过 PEAR 打包）。
# 用 pecl install 安装扩展，然后再用官方提供的 docker-php-ext-enable 
# 快捷脚本来启用扩展

pecl install redis && docker-php-ext-enable redis

docker restart myphp        # 装完扩展 退出容器 重启容器
```

#### 4. 构建自己的目录结构(直接摘抄)

```
你也可以构建自己所要的server目录结构 首先要知道挂载一个空文件夹会清空容器中文件夹下所有内容 所以应该先拷贝再挂载
例如: 创建一个临时容器 sudo docker run --name mynginx -p 80:80 -it -d nginx
进入到容器中查自己所要的配置文件目录地址 例如: /etc/nginx 退出容器 
拷贝容器中所要的目录结构到宿主机 例如: docker cp mydb:/etc/nginx /server/nginx
删除容器 创建新容器时就可以挂载该目录了 此后对nginx的配置文件的修改就可以直接在宿主机上快捷操作
docker run --name mynginx -d -p 80:80 -v /server/nginx:/etc/nginx --link myphp:myphp --privileged=true  nginx
```

### （二阶）使用docker-compose自动化构建

```
完成以上步骤你就已经初步了解了docker的基本容器操作
docker-compose是编排容器的。例如，你有一个php镜像，一个mysql镜像，一个nginx镜像。如果没有docker-compose，
那么每次启动的时候，你需要敲各个容器的启动参数，环境变量，容器命名，指定不同容器的链接参数等等一系列的操作，
相当繁琐。而用了docker-composer之后，你就可以把这些命令一次性写在docker-composer.yml文件中，以后每次启动
这一整个环境（含3个容器）的时候，你只要敲一个docker-composer up命令就ok了
```

#### 1. 安装docker-compose

> mac中不用执行这一步，安装docker默认已经安装完docker-compose

```
curl -L https://github.com/docker/compose/releases/download/1.8.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose

docker-compose --version
```

#### 2. 一键部署环境

进入到docker-compose.yml目录下，执行`docker-compose up -d`，后台开启容器

#### 3. docker-compose.yml配置参考

==docker-compose的语法建议查看官方文档==

```
version: "2"
services:
  mydb:
    image: mysql:5.7    # 容器的引用镜像
    container_name: "mydb" # 容器命名
    restart: always
    ports:
      - "3306:3306"
    volumes:    # 挂载目录写这里(也可以写相对路径)
      - ~/Documents/lnmp/mysql:/var/lib/mysql
    environment:    # 自定义环境变量
      MYSQL_ROOT_PASSWORD: 123456
      TZ: Asia/Shanghai   # 时区

  myredis:
    image: redis:3.2
    container_name: "myredis"
    restart: always
    ports:
      - "6379:6379"
    volumes:
      - ~/Documents/lnmp/redis:/data
    environment:
        TZ: Asia/Shanghai

  myphp:
    image: php:7.2-fpm
    container_name: "myphp"
    restart: alwaysr
    ports:
      - "9000:9000"
    volumes:
      - ~/Documents/lnmp/www:/var/www/html
      - ~/Documents/lnmp/php:/usr/local/etc/php
    links:
      - "mydb"
      - "myredis"
    environment:
        TZ: Asia/Shanghai

  mynginx:
    image: nginx:latest
    container_name: "mynginx"
    restart: always
    ports:
      - "80:80"
    volumes:
      - ~/Documents/lnmp/www:/usr/share/nginx/html
      - ~/Documents/lnmp/nginx:/etc/nginx
      - ~/Documents/lnmp/logs/nginx:/var/log/nginx
    links:
      - "myphp"
    environment:
        TZ: Asia/Shanghai
```

对比上面运行容器命令来看docker_yml的配置结构和语义就一目了然了

### (三阶)dokcer-compose和dockerfile 完整构建

用了docker-compose实现一键式操作，但问题是PHP的扩展库还是得自己单独装 所以这里需要用到Dockerfile来构建自定义容器镜像实现真正的一键完成


#### 1. 完善目录结构

```
docker  -|
        -| dockerfiles  -|
                        -| docker-compose.yml
                        
                        -| mount -|
                                 -| conf -|
                                         -| php.ini
                                         -| nginx.conf
                                         -| my.cnf
                                         -| vhost.conf
                                         -| mime.types
                                         
                                 -| data -|
                                         -| mysql -|
                                         -| redis -|
                                 
                                 -| logs -|
                                         -| nginx -|
                                 
                        -| mysql -|
                                 -| Dockerfile
                                 
                        -| nginx -|
                                 -| Dockerfile
                                 
                        -| php   -|
                                 -| Dockerfile
                                 
                        -| redis -|
                                 -| Dockerfile
                        
        -| website  -|
                -| index.php
                -| index.html
```

#### 2. php Dockerfile 示例

> 在这里只展示php的Dockerfile，因为最复杂的就是PHP的镜像，其他配置详情可以在github中查看

```
FROM php:7.2-fpm

# 按需安装php扩展
# 1.0.2 增加 bcmath, calendar, exif, gettext, sockets, dba, 
# mysqli, pcntl, pdo_mysql, shmop, sysvmsg, sysvsem, sysvshm 扩展
RUN docker-php-ext-install -j$(nproc) bcmath calendar exif gettext \
sockets dba mysqli pcntl pdo_mysql shmop sysvmsg sysvsem sysvshm

# 1.0.7 增加 soap 扩展
RUN apt-get update && \
	apt-get install -y --no-install-recommends \ 
	libxml2-dev libtidy-dev libxslt1-dev zlib1g-dev \ 
	&& rm -r /var/lib/apt/lists/* \ 
	&& docker-php-ext-install -j$(nproc) soap
    
# 使用pecl安装扩展，记得在php.ini中添加配置项(grpc依赖于zlib1g-dev扩展)
RUN pecl install grpc-1.22.1 \
	&& pecl install swoole-4.4.4 \
	&& docker-php-ext-enable grpc swoole

# 如果扩展下载时间太长，可以先下载到本地，再copy到指定目录，再用pecl安装
COPY ./redis-4.0.1.tgz /home/redis.tgz
COPY ./xdebug-2.6.0.tgz /home/xdebug.tgz
RUN pecl install /home/redis.tgz \
    && pecl install /home/xdebug.tgz \
    && docker-php-ext-enable redis xdebug

# 安装composer(compoaer.phar需要先下载到本地)
ADD composer.phar /usr/local/bin/composer
RUN chmod 755 /usr/local/bin/composer
```

**注意：**

自定义php的dockerfile构建自定义镜像同时安装扩展
完成了所有dockerfile配置后 docker-compose.yml文件就不需要
再用官方镜像image:php-fpm:7.2 而是直接==build：./php== 直接引用目录配置好的Dockerfile

**最后提示:** 镜像一旦创建了下次docker-compose会直接取已有镜像而不会build创建 若你修改了Dockerfile配置请记得删除之前镜像并重新构建

#### 3. docker-compose.yml 示例

```
version: "2"
services:
  mysql:
    build: ./mysql # 这里换成了从Dockerfile文件构建
    image: dev-mysql # 为构建的image命名 repository_name:tag_name(可选)
    container_name: dev-mysql # 创建容器时的名称
    ports:
      - "3306:3306"
    volumes: # 挂载目录写这里
      - ./mount/data/mysql:/var/lib/mysql
      - ./mount/conf/my.cnf:/etc/mysql/conf.d/my.cnf
    environment: # 自定义环境变量
      MYSQL_ROOT_PASSWORD: 123456
      TZ: Asia/Shanghai

  redis:
    build: ./redis
    image: dev-redis
    container_name: dev-redis
    ports:
      - "6379:6379"
    volumes:
      - ./mount/data/redis:/data
    environment:
        TZ: Asia/Shanghai

  php:
    build: ./php
    image: dev-php
    container_name: dev-php
    ports:
      - "9000:9000"
    volumes:
      - ../website:/var/www/html
      - ./mount/conf/php.ini:/usr/local/etc/php/php.ini
    links:
      - "mysql" # service名称
      - "redis"
    environment:
        TZ: Asia/Shanghai

  nginx:
    build: ./nginx
    image: dev-nginx
    container_name: dev-nginx
    ports:
      - "80:80"
    volumes:
      - ../website:/usr/share/nginx/html
      - ./mount/conf/nginx.conf:/etc/nginx/nginx.conf
      - ./mount/conf/vhosts.conf:/etc/nginx/conf.d/default.conf
      - ./mount/logs/nginx:/var/log/nginx
    links:
      - "php"
    environment:
        TZ: Asia/Shanghai

```

Dockerfile 和 docker-compose.yml配置好之后，执行
`docker-compose up --build`
构建并启动

也可以执行 `docker-compose build` 先构建，再使用下面的命令启动

首次构建之后，以后启动可以使用 `docker-compose up -d` 后台启动

### 常见问题

#### 1. 在宿主主机执行php容器cli命令

> docker exec -it dev-php bash -c 'composer --version'

也可以直接使用 `docker exec -it dev-php bash` 进入容器，再执行命令

#### 2. 同步本地hosts，在docker中连接外网

我本地项目需要连接google服务器，使用的代理需要添加本地hosts才可以请求，换成doker之后无法请求，解决办法如下：

把本地hosts同步到docker中，在docker-compose.yml中的php service中，添加以下代码

```
extra_hosts:
  - "googleads.googleapis.com: 172.217.160.74"
  - "oauth2.googleapis.com: 216.58.200.234"
```

#### 3. Laravel框架 配置.dev文件连接docker容器

`.dev`文件需要把对应的IP修改成容器的container_name

```
# before
DB_HOST=127.0.0.1

# after
DB_HOST=dev-mysql
```

#### 4. Mysql修改初始密码之后，重新构建容器不生效

删除mysql挂载的数据卷下的数据，再重新构建


#### 5. 使用客户端连接Mysql容器

```
docker exec -it dev-mysql bash      # 进入到mysql容器
mysql -u root -p                    # 连接mysql
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';      #修改root 可以通过任何客户端连接
exit                                # 退出容器
```

#### 6. docker for mac Laravel请求响应慢

可以在挂载目录后面加`:cached`解决，但会牺牲一致性

```
volumes:
      - /project/path:/var/www/html:cached
```

加完之后性能还是比不上原生环境，这和mac中的文件系统有关系，不止mac，在windows也有同样的问题，需要等docker团队去解决

参考自：

* [文件系统共享](https://docs.docker.com/docker-for-mac/osxfs/)
* [卷安装的性能调整（共享文件系统）](https://docs.docker.com/docker-for-mac/osxfs-caching/)
* [Docker for Mac中的用户指导缓存](https://blog.docker.com/2017/05/user-guided-caching-in-docker-for-mac/)

### 参考链接

> docker灵活的构建php环境
> https://github.com/ydtg1993/server

> docker-compose.yml 语法解析 
> https://deepzz.com/post/docker-compose-file.html

> php:7.2-fpm 扩展配置
> https://www.jianshu.com/p/20fcca06e27e

> Docker在PHP项目开发环境中的应用
> http://avnpc.com/pages/build-php-develop-env-by-docker

> Docker — 从入门到实践
> https://yeasy.gitbooks.io/docker_practice/content/

> Docker Reference
> https://docs.docker.com/reference/

> Docker Store
> https://store.docker.com/