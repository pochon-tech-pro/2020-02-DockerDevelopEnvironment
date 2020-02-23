## 2020-02-DockerDevelopEnvironment

### 概要

- 2020/02時点でのDocker開発環境群 (Docker for Mac)

### 2020-NodeJS

- NodeJS開発環境用

- Dockerfile

```Dockerfile
FROM node:10-alpine
  
ENV APP_PORT 8001
ENV APP_ROOT /app
EXPOSE $APP_PORT
WORKDIR $APP_ROOT
CMD [ "sh" ]

RUN apk update && \
    apk add git openssh curl jq && \
    curl -o- -L https://yarnpkg.com/install.sh | sh

RUN apk add python make g++
```

- docker-compose.yml

```yaml:docker-compose.yml
version: '3.4'
x-logging:
  &default-logging
  driver: "json-file"
  options:
    max-size: "100k"
    max-file: "3"
services:

  app:
    build: ./
    logging: *default-logging
    volumes:
    - ./app/:/app/
    command: sh -c "yarn start"
    ports: 
      - "8001:8001"
```

### 2020-PHP-Laravel

- Dockerfile
```Dockerfile
FROM php:7.3-apache

RUN apt update && apt-get install -y git libzip-dev
RUN docker-php-ext-install pdo_mysql zip

# Install Composer
# RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
# RUN php composer-setup.php
# RUN php -r "unlink('composer-setup.php');"
# RUN mv composer.phar /usr/local/bin/composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
ENV COMPOSER_ALLOW_SUPERUSER 1

RUN a2enmod rewrite

WORKDIR /var/www

# Install opcache apcu
RUN docker-php-ext-install opcache
RUN pecl install apcu
RUN docker-php-ext-enable apcu

#COPY php.ini /usr/local/etc/php/php.ini
#COPY . /var/www
```

- docker-compose.yml
```yaml:docker-compose.yml
version: '3.4'
x-logging:
  &default-logging
  driver: "json-file"
  options:
    max-size: "100k"
    max-file: "3"
volumes:
  mysql_data: { driver: local }
services:

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: pass
      MYSQL_DATABASE: db
      MYSQL_USER: user
      MYSQL_PASSWORD: pass
      TZ: 'Asia/Tokyo'
    volumes:
    - mysql_data:/var/lib/mysql

  www:
    image: php-app
    build: ./
    logging: *default-logging
    volumes:
    - ./www:/var/www
    - ./www/php.ini:/usr/local/etc/php/php.ini
    environment:
    - DB_CONNECTION=mysql
    - DB_HOST=mysql
    - DB_DATABASE=db
    - DB_USERNAME=user
    - DB_PASSWORD=pass
    ports: [ 80:80 ]
```

**Laravelプロジェクトの立ち上げ**

```sh:
docker-compose up -d
```
- localhostをブラウザで開き`not found`の確認(Apacheが動いているか確認)

```sh:
$ docker-compose exec www bash
root@eb3201f46687:/var/www# composer create-project --prefer-dist laravel/laravel laravel
Installing laravel/laravel (v6.12.0)
As there is no 'unzip' command installed zip files are being unpacked using the PHP zip extension.
This may cause invalid reports of corrupted archives. Besides, any UNIX permissions (e.g. executable) defined in the archives will be lost.
Installing 'unzip' may remediate them.
  - Installing laravel/laravel (v6.12.0): Downloading (100%)         
Created project in laravel
> @php -r "file_exists('.env') || copy('.env.example', '.env');"
Loading composer repositories with package information
Updating dependencies (including require-dev)
                                                                                                                                     
  [Composer\Downloader\TransportException]                                                                                                                 
  The "http://repo.packagist.org/p/sebastian/version%24b03db57dbfd1edd8c8da676b1c55d30da12dabd410c11d18bc0235a7faa43fc3.json" file could not be downloade  
  d: failed to open stream: Cannot assign requested address  

# エラーが起きたので設定の確認をするも正常そう？
root@eb3201f46687:/var/www# composer diag
Checking platform settings: OK
Checking git settings: OK
Checking http connectivity to packagist: OK
Checking https connectivity to packagist: OK
Checking github.com rate limit: OK
Checking disk free space: OK
Checking pubkeys: 
Tags Public Key Fingerprint: 57815BA2 7E54DC31 7ECC7CC5 573090D0  87719BA6 8F3BB723 4E5D42D0 84A14642
Dev Public Key Fingerprint: 4AC45767 E5EC2265 2F0C1167 CBBB8A2B  0C708369 153E328C AD90147D AFE50952
OK
Checking composer version: OK
Composer version: 1.9.3
PHP version: 7.3.15
PHP binary path: /usr/local/bin/php 
```
- ネットワークのエラーのようだが、下記を参考にパブリックDNSの設定してみたがだめだった
  - https://github.com/shipping-docker/php-app/issues/36
- 原因は、httpsを強制する必要があるみたいで、下記を参考にコンテナ内のConfigを変えることで解決した
　- https://stackoverflow.com/questions/53986012/getting-error-while-installing-laravel-installer-in-window-10

```sh:
root@eb3201f46687:/var/www# composer config -g repo.packagist composer https://packagist.org
root@eb3201f46687:/var/www# composer create-project --prefer-dist laravel/laravel laravel
root@eb3201f46687:/var/www# ln -s laravel/public html
```
- localhostをブラウザで開き、Laravelのトップページが描画されていることを確認


