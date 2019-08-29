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

# 安装composer
ADD composer.phar /usr/local/bin/composer
RUN chmod 755 /usr/local/bin/composer