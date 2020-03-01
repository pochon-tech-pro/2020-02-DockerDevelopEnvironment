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

<details>
<summary>PHPの公式Dockerfile</summary>

### 公式のDockerfileを解読
```Dockerfile
#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "update.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM debian:stretch-slim

# prevent Debian's PHP packages from being installed
# https://github.com/docker-library/php/pull/542
RUN set -eux; \
	{ \
		echo 'Package: php*'; \
		echo 'Pin: release *'; \
		echo 'Pin-Priority: -1'; \
	} > /etc/apt/preferences.d/no-debian-php

# dependencies required for running "phpize"
# (see persistent deps below)
ENV PHPIZE_DEPS \
		autoconf \
		dpkg-dev \
		file \
		g++ \
		gcc \
		libc-dev \
		make \
		pkg-config \
		re2c

# persistent / runtime deps
RUN apt-get update && apt-get install -y \
		$PHPIZE_DEPS \
		ca-certificates \
		curl \
		xz-utils \
	--no-install-recommends && rm -r /var/lib/apt/lists/*

ENV PHP_INI_DIR /usr/local/etc/php
RUN mkdir -p $PHP_INI_DIR/conf.d

##<autogenerated>##
RUN apt-get update \
	&& apt-get install -y --no-install-recommends \
		apache2 \
	&& rm -rf /var/lib/apt/lists/*

ENV APACHE_CONFDIR /etc/apache2
ENV APACHE_ENVVARS $APACHE_CONFDIR/envvars

RUN set -eux; \
	\
# generically convert lines like
#   export APACHE_RUN_USER=www-data
# into
#   : ${APACHE_RUN_USER:=www-data}
#   export APACHE_RUN_USER
# so that they can be overridden at runtime ("-e APACHE_RUN_USER=...")
	sed -ri 's/^export ([^=]+)=(.*)$/: ${\1:=\2}\nexport \1/' "$APACHE_ENVVARS"; \
	\
# setup directories and permissions
	. "$APACHE_ENVVARS"; \
	for dir in \
		"$APACHE_LOCK_DIR" \
		"$APACHE_RUN_DIR" \
		"$APACHE_LOG_DIR" \
		/var/www/html \
	; do \
		rm -rvf "$dir"; \
		mkdir -p "$dir"; \
		chown "$APACHE_RUN_USER:$APACHE_RUN_GROUP" "$dir"; \
# allow running as an arbitrary user (https://github.com/docker-library/php/issues/743)
		chmod 777 "$dir"; \
	done; \
	\
# logs should go to stdout / stderr
	ln -sfT /dev/stderr "$APACHE_LOG_DIR/error.log"; \
	ln -sfT /dev/stdout "$APACHE_LOG_DIR/access.log"; \
	ln -sfT /dev/stdout "$APACHE_LOG_DIR/other_vhosts_access.log"; \
	chown -R --no-dereference "$APACHE_RUN_USER:$APACHE_RUN_GROUP" "$APACHE_LOG_DIR"

# Apache + PHP requires preforking Apache for best results
RUN a2dismod mpm_event && a2enmod mpm_prefork

# PHP files should be handled by PHP, and should be preferred over any other file type
RUN { \
		echo '<FilesMatch \.php$>'; \
		echo '\tSetHandler application/x-httpd-php'; \
		echo '</FilesMatch>'; \
		echo; \
		echo 'DirectoryIndex disabled'; \
		echo 'DirectoryIndex index.php index.html'; \
		echo; \
		echo '<Directory /var/www/>'; \
		echo '\tOptions -Indexes'; \
		echo '\tAllowOverride All'; \
		echo '</Directory>'; \
	} | tee "$APACHE_CONFDIR/conf-available/docker-php.conf" \
	&& a2enconf docker-php

ENV PHP_EXTRA_BUILD_DEPS apache2-dev
ENV PHP_EXTRA_CONFIGURE_ARGS --with-apxs2 --disable-cgi
##</autogenerated>##

# Apply stack smash protection to functions using local buffers and alloca()
# Make PHP's main executable position-independent (improves ASLR security mechanism, and has no performance impact on x86_64)
# Enable optimization (-O2)
# Enable linker optimization (this sorts the hash buckets to improve cache locality, and is non-default)
# Adds GNU HASH segments to generated executables (this is used if present, and is much faster than sysv hash; in this configuration, sysv hash is also generated)
# https://github.com/docker-library/php/issues/272
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie"

ENV GPG_KEYS CBAF69F173A0FEA4B537F470D66C9593118BCCB6 F38252826ACD957EF380D39F2F7956BC5DA04B5D

ENV PHP_VERSION 7.3.1
ENV PHP_URL="https://secure.php.net/get/php-7.3.1.tar.xz/from/this/mirror" PHP_ASC_URL="https://secure.php.net/get/php-7.3.1.tar.xz.asc/from/this/mirror"
ENV PHP_SHA256="cfe93e40be0350cd53c4a579f52fe5d8faf9c6db047f650a4566a2276bf33362" PHP_MD5=""

RUN set -xe; \
	\
	fetchDeps=' \
		wget \
	'; \
	if ! command -v gpg > /dev/null; then \
		fetchDeps="$fetchDeps \
			dirmngr \
			gnupg \
		"; \
	fi; \
	apt-get update; \
	apt-get install -y --no-install-recommends $fetchDeps; \
	rm -rf /var/lib/apt/lists/*; \
	\
	mkdir -p /usr/src; \
	cd /usr/src; \
	\
	wget -O php.tar.xz "$PHP_URL"; \
	\
	if [ -n "$PHP_SHA256" ]; then \
		echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c -; \
	fi; \
	if [ -n "$PHP_MD5" ]; then \
		echo "$PHP_MD5 *php.tar.xz" | md5sum -c -; \
	fi; \
	\
	if [ -n "$PHP_ASC_URL" ]; then \
		wget -O php.tar.xz.asc "$PHP_ASC_URL"; \
		export GNUPGHOME="$(mktemp -d)"; \
		for key in $GPG_KEYS; do \
			gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
		done; \
		gpg --batch --verify php.tar.xz.asc php.tar.xz; \
		command -v gpgconf > /dev/null && gpgconf --kill all; \
		rm -rf "$GNUPGHOME"; \
	fi; \
	\
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false $fetchDeps

COPY docker-php-source /usr/local/bin/

RUN set -eux; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libcurl4-openssl-dev \
		libedit-dev \
		libsodium-dev \
		libsqlite3-dev \
		libssl-dev \
		libxml2-dev \
		zlib1g-dev \
		${PHP_EXTRA_BUILD_DEPS:-} \
	; \
##<argon2>##
	sed -e 's/stretch/buster/g' /etc/apt/sources.list > /etc/apt/sources.list.d/buster.list; \
	{ \
		echo 'Package: *'; \
		echo 'Pin: release n=buster'; \
		echo 'Pin-Priority: -10'; \
		echo; \
		echo 'Package: libargon2*'; \
		echo 'Pin: release n=buster'; \
		echo 'Pin-Priority: 990'; \
	} > /etc/apt/preferences.d/argon2-buster; \
	apt-get update; \
	apt-get install -y --no-install-recommends libargon2-dev; \
##</argon2>##
	rm -rf /var/lib/apt/lists/*; \
	\
	export \
		CFLAGS="$PHP_CFLAGS" \
		CPPFLAGS="$PHP_CPPFLAGS" \
		LDFLAGS="$PHP_LDFLAGS" \
	; \
	docker-php-source extract; \
	cd /usr/src/php; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
# https://bugs.php.net/bug.php?id=74125
	if [ ! -d /usr/include/curl ]; then \
		ln -sT "/usr/include/$debMultiarch/curl" /usr/local/include/curl; \
	fi; \
	./configure \
		--build="$gnuArch" \
		--with-config-file-path="$PHP_INI_DIR" \
		--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
		\
# make sure invalid --configure-flags are fatal errors intead of just warnings
		--enable-option-checking=fatal \
		\
# https://github.com/docker-library/php/issues/439
		--with-mhash \
		\
# --enable-ftp is included here because ftp_ssl_connect() needs ftp to be compiled statically (see https://github.com/docker-library/php/issues/236)
		--enable-ftp \
# --enable-mbstring is included here because otherwise there's no way to get pecl to use it properly (see https://github.com/docker-library/php/issues/195)
		--enable-mbstring \
# --enable-mysqlnd is included here because it's harder to compile after the fact than extensions are (since it's a plugin for several extensions, not an extension in itself)
		--enable-mysqlnd \
# https://wiki.php.net/rfc/argon2_password_hash (7.2+)
		--with-password-argon2 \
# https://wiki.php.net/rfc/libsodium
		--with-sodium=shared \
		\
		--with-curl \
		--with-libedit \
		--with-openssl \
		--with-zlib \
		\
# bundled pcre does not support JIT on s390x
# https://manpages.debian.org/stretch/libpcre3-dev/pcrejit.3.en.html#AVAILABILITY_OF_JIT_SUPPORT
		$(test "$gnuArch" = 's390x-linux-gnu' && echo '--without-pcre-jit') \
		--with-libdir="lib/$debMultiarch" \
		\
		${PHP_EXTRA_CONFIGURE_ARGS:-} \
	; \
	make -j "$(nproc)"; \
	make install; \
	find /usr/local/bin /usr/local/sbin -type f -executable -exec strip --strip-all '{}' + || true; \
	make clean; \
	\
# https://github.com/docker-library/php/issues/692 (copy default example "php.ini" files somewhere easily discoverable)
	cp -v php.ini-* "$PHP_INI_DIR/"; \
	\
	cd /; \
	docker-php-source delete; \
	\
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
	apt-mark auto '.*' > /dev/null; \
	[ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
	find /usr/local -type f -executable -exec ldd '{}' ';' \
		| awk '/=>/ { print $(NF-1) }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -r apt-mark manual \
	; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	\
	php --version; \
	\
# https://github.com/docker-library/php/issues/443
	pecl update-channels; \
	rm -rf /tmp/pear ~/.pearrc

COPY docker-php-ext-* docker-php-entrypoint /usr/local/bin/

# sodium was built as a shared module (so that it can be replaced later if so desired), so let's enable it too (https://github.com/docker-library/php/issues/598)
RUN docker-php-ext-enable sodium

ENTRYPOINT ["docker-php-entrypoint"]
##<autogenerated>##
COPY apache2-foreground /usr/local/bin/
WORKDIR /var/www/html

EXPOSE 80
CMD ["apache2-foreground"]
##</autogenerated>##
```

### ベースイメージの指定
```Dockerfile:
FROM debian:stretch-slim
```
- Dockerfileは全てFROMから始まり、ベースのイメージを設定する
- stretchとは、Debianの各メジャーバージョンに付けられるコードネームのこと (ここでは Debian v9(stretch) を指す)
  - 他にも前のバージョン`jessie`や次のバージョン`buster`などがある
  - https://wiki.debian.org/DebianReleases
**Linuxディストリビューションのトレンド**
- https://w3techs.com/technologies/history_details/os-linux
  - UbuntuとDebianが大きくシェアを占めている
  - UbuntuはDebianを元にしており、Debianとほぼ同じように扱える
  - debianの方が軽量でサーバー向けという位置付け

### パッケージの制御（/etc/apt/preferences.d）
```Dockerfile
# prevent Debian's PHP packages from being installed
# https://github.com/docker-library/php/pull/542
RUN set -eux; \
    { \
        echo 'Package: php*'; \
        echo 'Pin: release *'; \
        echo 'Pin-Priority: -1'; \
    } > /etc/apt/preferences.d/no-debian-php
```
- Debianではパッケージ管理システムにAPTを使い、**APTのコマンドとしてapt-get**が使用される
  - 例えば、apt-getコマンドと使用した場合、以下のようなイメージで自動的に最新バージョンへ更新することができる
  ```sh:
  $ apt-get update
  $ apt-get upgrade
  ```
  - しかし、中にはソースからビルドしたものを使い、APTからはインストールして欲しくない時がある
  - その際に`/etc/apt/preferences.d`で、特定のパッケージのインストールを制御することができる
  - 上記のケースだと、**パッケージ名がphpで始まるパッケージは絶対にインストールしない**という設定になっている
- また、最初に`set`コマンドによってシェルに関する設定を行っている
  - -e オプションによって実行したコマンドが1つでもエラーになれば直ちに終了する
  - -u オプションで未定義の変数などを使おうとすればエラーにするようにしている
  - -x オプションでは実行コマンドとその引数をトレースとして出力するようにしている
  - **setコマンドはDockerfileで頻出するコマンド**

  <details>
  <summary>Linuxコマンドにおける;や&&の意味</summary>
    
    **コマンド1が終了したらコマンド2を実行する（実行結果に関わらず）**
    ```sh
      コマンド1 ; コマンド2　
    ```
    - 使用例1.　5分後にdateコマンドを実行する
    ```sh:
      sleep 5m ; date
    ```

    **コマンド1を実行しつつコマンド2も実行する**
    ```sh
      コマンド1 & コマンド2　
    ```
    - 使用例1.　/home/test/test.shを実行しログを出力しつつ、viでtest.txtを編集する
    ```sh:
      sh /home/test/test.sh >> /var/log/test.log & vi /home/test/test.txt
    ```

    **コマンド1が正常終了したらコマンド2を実行する**
    ```sh
      コマンド1 && コマンド2　
    ```
    - 使用例1.　/home/testにディレクトリ移動ができたら、test.txtを作成する
    ```sh:
      cd /home/test/ && touch test.txt
    ```
    - 使用例2.　ダウンロードしてきたtar.gzを解凍後、ディレクトリへ移動
    ```sh:
      tar zxf xxx-2.x.tar.gz && cd xxx-2.x
    ```
    - 使用例3. 何かのパッケージをソースからインストールする
    ```sh:
      ./configure && make && make install
    ```

    **コマンド1の結果をコマンド2に渡して実行**
    ```sh
      コマンド1 | コマンド2
    ```
    - 使用例1.　ps auxで実行中のプロセスを出力し（ターミナルには出力されない）、その中からキーワードhttpdにマッチする行を出力する
    ```sh:
      ps aux | grep httpd
    ```
    - 使用例2. アクセス数の集計例
    ```log:acces_log.txt
      xxx.xx.xx.xxx - - [19/Dec/2019:06:38:49 +0900] "GET /test/aaaa.html?id=123 HTTP/1.1" 200 13 "https://www.xxx.com/mypage/index.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.102 Safari/537.36 Edge/18.18362"
    ```
    ```sh:
      sudo cat /var/log/httpd/access_log | grep '\[19/Dec/2019:1'| grep -E '(\.html|\.php|/)(\?[^\\s]*)? HTTP' | awk '{print $4}' | cut -b 2-18 | sort | uniq -c
      # 25 19/Dec/2019:06:38
    ```

    **コマンド1が異常終了したらコマンド2が実行される**
    ```sh
      コマンド1 || コマンド2
    ```
    - 使用例1.　/home/test/abcにディレクトリ移動ができなかったら、/home/test/abcディレクトリを作成する
    ```sh:
      cd /home/egawa/abc || mkdir /home/egawa/abc
    ```
  </details>

### phpizeに必要なパッケージ（$PHPIZE_DEPS）
```sh:
# dependencies required for running "phpize"
# (see persistent deps below)
ENV PHPIZE_DEPS \
        autoconf \
        dpkg-dev \
        file \
        g++ \
        gcc \
        libc-dev \
        make \
        pkg-config \
        re2c
```
- PHPの拡張モジュールのビルドツールであるphpizeで必要なパッケージリストを環境変数に設定している
- この環境変数は後ほどapt-getで使われる
  - autoconf: configureというパッケージインストールスクリプトを作成するためのパッケージ
  - dpkg-dev: APTより低水準なdebianのパッケージ管理システム
  - file: ファイルの形式などを調べるfileコマンド
  - g++: C++ コンパイラ
  - gcc: C コンパイラ
  - libc-dev: C言語 標準ライブラリ
  - make: Makefileというファイルを基にコンパイルを行うツール
  - pkg-config: コンパイルする際に必要なライブラリの情報を取得するツール
  - re2c: CとC++のための字句解析ツール

### パッケージのインストール（apt-get install）
```Dockerfile:
# persistent / runtime deps
RUN apt-get update && apt-get install -y \
        $PHPIZE_DEPS \
        ca-certificates \
        curl \
        xz-utils \
    --no-install-recommends && rm -r /var/lib/apt/lists/*
```
- 先ほどの$PHPIZE_DEPSとその他のパッケージをインストールしている
- **apt-get updateはインストール可能なパッケージリストを更新するコマンド**
  - updateは、`/etc/apt/sources.list`に書かれているURLからインストール可能なパッケージを`/var/lib/apt/lists`に保存する
  - `/etc/apt/sources.list`
  ```sh:/etc/apt/sources.list
    deb http://deb.debian.org/debian stretch main
    deb http://security.debian.org/debian-security stretch/updates main
    deb http://deb.debian.org/debian stretch-updates main
  ```
- **apt-get install -yは列挙するパッケージをインストールするコマンド**
  - -yオプションが無い場合はインストール前に「インストールしていいですか」という類の確認が出る
  - ただし、Dockerfileの場合はこういったインタラクティブなコマンド処理は行えないため、-yオプションによってスキップしている
  - `--no-install-recommends`オプションは、**おすすめの関連パッケージをインストールさせない**ためのオプション 
  - 余計なパッケージを入れられるとDockerのイメージサイズが大きくなるため、基本的にインストールしない方が良い
- インストール後は、`rm -r /var/lib/apt/lists/*`を行い、`apt-get update`で取得したパッケージのソースを削除している
  - これらのソースはインストール後に使うことは無いためである
- ca-certificates: 認証局（CA：Certification Authority）の証明書などを含んだパッケージ
- curl: httpを始めとした様々なプロトコルで通信を行うためのツール
- xz-utils: xzという圧縮フォーマットのファイルを作成・展開するためのツール

### PHP_INI_DIR
```Dockerfile:
ENV PHP_INI_DIR /usr/local/etc/php
RUN set -eux; \
    mkdir -p "$PHP_INI_DIR/conf.d"; \
```
- PHPでは.iniファイルを設定ファイルとして扱う
- これらの保存ディレクトリを作成する

### Apache root directory
```Dockerfile:
# allow running as an arbitrary user (https://github.com/docker-library/php/issues/743)
    [ ! -d /var/www/html ]; \
    mkdir -p /var/www/html; \
    chown www-data:www-data /var/www/html; \
    chmod 777 /var/www/html
```
- Apacheをwww-dataユーザーから実行できるようにするための処理を行っている
- www-dataはApacheを実行する際のデフォルトの実行ユーザーである
- `[ ! -d /var/www/html ]`は、**もし/var/www/htmlがディレクトリではなかったらという意味**であり、真であれば続くコマンドが実行される
- chownによって`/var/www/html`の所有者をwww-dataにしている
- chmodによってどのユーザーでも`/var/www/html`を読み書きや実行しても良いという権限にしている

### Apache2のインストール
```Dockerfile:
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends apache2; \
    rm -rf /var/lib/apt/lists/*; \
```
- apache2をapt-getによってインストールしている
```Dockerfile:
ENV APACHE_CONFDIR /etc/apache2
ENV APACHE_ENVVARS $APACHE_CONFDIR/envvars
```
- Apacheの設定ファイルのディレクトリなどを環境変数に設定する
```Dockerfile:
RUN set -eux; \
	\
# generically convert lines like
#   export APACHE_RUN_USER=www-data
# into
#   : ${APACHE_RUN_USER:=www-data}
#   export APACHE_RUN_USER
# so that they can be overridden at runtime ("-e APACHE_RUN_USER=...")
	sed -ri 's/^export ([^=]+)=(.*)$/: ${\1:=\2}\nexport \1/' "$APACHE_ENVVARS"; \
	\
```
- APACHE_RUN_USERが指定されていればwww-dataではなく、そちらを優先する
  ```sh:
  `export APACHE_RUN_USER=www-data`
  ```
  を
  ```sh:
  : ${APACHE_RUN_USER:=www-data}
  export APACHE_RUN_USER
  ```
  - に変換していて、APACHE_RUN_USER が指定されていた場合そちらを優先するようにしている
```Dockerfile:
# setup directories and permissions
    . "$APACHE_ENVVARS"; \
    for dir in \
        "$APACHE_LOCK_DIR" \
        "$APACHE_RUN_DIR" \
        "$APACHE_LOG_DIR" \
    ; do \
        rm -rvf "$dir"; \
        mkdir -p "$dir"; \
        chown "$APACHE_RUN_USER:$APACHE_RUN_GROUP" "$dir"; \
# allow running as an arbitrary user (https://github.com/docker-library/php/issues/743)
        chmod 777 "$dir"; \
    done; \
```
- Apacheで利用する各ディレクトリを再作成し、適切なパーミッションに設定している
```Dockerfile:
# delete the "index.html" that installing Apache drops in here
    rm -rvf /var/www/html/*; \
```
- Apacheインストール時に作成されるindex.htmlを削除している
```Dockerfile:
# logs should go to stdout / stderr
    ln -sfT /dev/stderr "$APACHE_LOG_DIR/error.log"; \
    ln -sfT /dev/stdout "$APACHE_LOG_DIR/access.log"; \
    ln -sfT /dev/stdout "$APACHE_LOG_DIR/other_vhosts_access.log"; \
    chown -R --no-dereference "$APACHE_RUN_USER:$APACHE_RUN_GROUP" "$APACHE_LOG_DIR"
```
- Apacheのログを標準出力・標準エラー出力に出力するように設定している
  - lnコマンドによってリンクを作成することができる
  - -s オプションでハードリンクではなくシンボリックリンクにしている
  - -f オプションで同じファイルがあった場合でも強制的に上書きしている
  - -T オプションでリンク先をディレクトリではなく通常ファイルとして扱うようにしている
- chownの-Rオプションで再帰的に、そして--no-dereferenceオプションでシンボリックリンク自体の所有者を変更するようにしている
```Dockerfile:
# Apache + PHP requires preforking Apache for best results
RUN a2dismod mpm_event && a2enmod mpm_prefork
```
- Apacheでは`a2dismod`コマンドでモジュールを無効化したり、`a2enmod`コマンドで有効化したりできる
- 今回の2つのモジュールは、ApacheのMPM (Multi Processing Module)というApacheの並行処理方法に関するモジュールのこと
- **PHPはスレッドセーフな言語ではないため、マルチプロセスで動かさなければならない**
- そのためmpm_preforkをモジュールを利用して、リクエストを処理するApacheのプロセスをあらかじめforkするようなマルチプロセス処理形態にする
```Dockerfile:
# PHP files should be handled by PHP, and should be preferred over any other file type
RUN { \
        echo '<FilesMatch \.php$>'; \
        echo '\tSetHandler application/x-httpd-php'; \
        echo '</FilesMatch>'; \
        echo; \
        echo 'DirectoryIndex disabled'; \
        echo 'DirectoryIndex index.php index.html'; \
        echo; \
        echo '<Directory /var/www/>'; \
        echo '\tOptions -Indexes'; \
        echo '\tAllowOverride All'; \
        echo '</Directory>'; \
    } | tee "$APACHE_CONFDIR/conf-available/docker-php.conf" \
    && a2enconf docker-php
```
- Apacheが.phpファイルを処理するための設定ファイルを作成している
- FilesMatchディレクティブで、リクエストファイルがphpファイルであれば`application/x-httpd-php`というハンドラで処理するようにしている
- 設定ファイルを愚直にechoしているが、別ファイルに切り出してDockerのCOPYコマンドを使う方が簡潔になるが、まあ良い
```Dockerfile:
ENV PHP_EXTRA_BUILD_DEPS apache2-dev
ENV PHP_EXTRA_CONFIGURE_ARGS --with-apxs2 --disable-cgi
```

<details>
<summary>Apache MPMとは</summary>

## プロセスとスレッドの違い
- プロセスとはCPU上で実行されるもので、タスクを完了するために、Linuxのカーネルが制御するあらゆるリソースを使うことができる
- スレッドとは1つのプロセスから生成される実行単位であり、同じプロセスから並行でスレッドを起動させることができる
- スレッドはメモリや、オープン中のファイルなどのリソースを共有することができる
- 同じアプリケーションのデータにアクセスすることができる
- プロセスはリソースを共有することができない、そのため、プロセスを起動させるには、リソースをコピーすることが必要
- 言い換えると、スレッドは同じタイミングで、共有しているリソースに変更をかけるべきではない
- そのために、ロックをかけたり、シリアルに動かしたりという制御をするのはアプリケーションの責任ということになる
- 性能の観点からはスレッドを起動するほうが、効率的

## Webサーバの基本的な並行処理のモデル
- Webサーバに接続するクライアントが1人だけであれば、並行処理について考えることは少ない
- しかし、多くの場合は同時に複数人のクライアントに対応しなければならない
- リクエストを並行処理するためのWebサーバの実装モデルがいくつかある

### マルチプロセスモデル

> クライアントからのリクエストごとに fork をして子プロセスを生成し、その子プロセスに処理を委ねる方式
- プロセスの fork では、メモリ上の親プロセスのアドレス空間を、生成した子プロセスのアドレス空間にコピーする
- したがって、その分のコストが発生し、低速と言われている
- また、リクエストが増えれば増えるほど、子プロセスの数とそれに伴うメモリ消費量も増えてしまう (すべての子プロセスがPHPインタプリタおよび関連ライブラリをロードする)
- この方式の利点
  - **メモリ空間がプロセスごとに独立しているためスクリプト言語などを組み込みやすい**
  - 後述のマルチスレッドモデルと違い資源の競合について考慮しなくてよい

### マルチスレッドモデル

> クライアントからのリクエストごとにスレッドを生成する方式と、あらかじめスレッドを生成しておくモデル
- マルチプロセスモデルとは違い、プロセスではなくスレッドを使用する
- このため、**プロセスの fork の際に発生するコピー作業が発生しない** (各スレッドはメモリ空間を共有する)
- プロセスの生成よりもオーバヘッドが小さいと言われている (メモリ消費量についても同様)
- この方式の利点
  - メモリ空間を共有するので、コンテキストスイッチの際に発生するメモリ空間の切り替えや、それに伴うキャッシュの削除を省略できる
  - コンテキストスイッチ: OSや処理系などがコンテキスト（状態）を保存して、プロセスやスレッドなどを切り替えること
- この方式の難点
  - スレッド間での資源の競合を考慮したプログラムを書く必要がある
  - 実装が難しくなりコードも複雑なものになりやすい

### イベント駆動モデル

> 1つのプロセスで複数のリクエストを処理する方式
- 上記2つのモデルでは、**クライアントからの要求を受けてレスポンスを返すという一連の流れに対してを1つのプロセス or スレッドが割り当てる**ことでそれぞれのリクエストに対応
- イベント駆動モデルでは、**リクエスト数に関係なくイベント発生のタイミングで処理を切り替え、1つのプロセスがすべてのリクエストを処理**
- プロセスが1つしか無いということは、CPUコアを1つしか活用できないことを意味する
- Nginx などでは、イベント駆動のプロセスをCPUコアそれぞれに起動しておくなどして、この欠点に対応している
- この方式の利点
  - リクエスト数が増えてもプロセスやスレッドの数が増えることがない
  - メモリ消費量やコンテキスト切り替えのオーバヘッドなどのコストを抑える事ができる

## MPM (Multi Processing Module)
- Webブラウザからのリクエストを Apache がどのように並行処理するか、という部分の処理をモジュール化したもの
- Apache では上記のようなモデルの中からどの実装を使用するかをこの MPM によって選択することができる
- Apache そのものにこれらの処理が組み込まれずにモジュール化されていることによって、各々のWebサイト向けにカスタマイズすることが容易
- **Apache2.2までは MPM は静的にリンクしなければならない**
- **Apache2.3 からは LoadModule ディレクティブで動的に選択することが可能になった**

## MPMの種類

### prefork
- **マルチプロセスモデル**
- prefork という名の通り、クライアントからリクエストが来る前にあらかじめ一定数の子プロセスを fork して待機させておく
- これにより、fork の回数を減らしてパフォーマンス向上を図る
- リクエストごとにプロセスが分かれているため、あるプロセスの障害が他のプロセスに影響を及ぼすことがない
- たがって、安定した通信をすることが可能

### worker
- **マルチスレッドモデルとマルチプロセスモデルのハイブリッドモデル**
- 制御用の親プロセスがいくつかの子プロセスを作成し、その子プロセスそれぞれがマルチスレッドモデルでリクエストを捌く
- スレッド1つが1つのクライアントの処理を担当
- prefork に比べ生成されるプロセスの数を抑えることができるので、資源の節約が可能
- しかし、スレッドを使用するモデルなので、**mod_phpなどの非スレッドセーフなモジュールを利用する際には使用できない**

### event
- **worker をベースとしたマルチスレッドモデルとマルチプロセスモデルとイベント駆動モデルのハイブリッドモデル**
- KeepAlive の処理を別のスレッドに割り振って通信を処理することによって、パフォーマンスの向上を図る
  - KeepAlive: https://milestone-of-se.nesuke.com/nw-basic/as-nw-engineer/keepalive-tcp-http/
- また、クライアントとのネットワークI/Oのみイベント駆動モデルで実装されている
- こちらもスレッドを使用するモデルなので、非スレッドセーフなモジュールを利用する際には使用できない
- `Apache2.4 + prefork`よりも`Apache2.4 + event + mod_proxy_fcgi + php-fpm`のほうが省メモリとのこと
  - https://norikone.hatenablog.com/entry/2016/02/07/Apache2_4_prefork%2Bmod_php%E3%81%8B%E3%82%89event%2Bphp-fpm%2Bmod_proxy_fcgi%E3%81%B8

</details>

### PHPのインストール
```Dockerfile:
# Apply stack smash protection to functions using local buffers and alloca()
# Make PHP's main executable position-independent (improves ASLR security mechanism, and has no performance impact on x86_64)
# Enable optimization (-O2)
# Enable linker optimization (this sorts the hash buckets to improve cache locality, and is non-default)
# Adds GNU HASH segments to generated executables (this is used if present, and is much faster than sysv hash; in this configuration, sysv hash is also generated)
# https://github.com/docker-library/php/issues/272
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie"
```
- コンパイルで用いられる最適化のオプション...らしい
```Dockerfile:
ENV GPG_KEYS CBAF69F173A0FEA4B537F470D66C9593118BCCB6 F38252826ACD957EF380D39F2F7956BC5DA04B5D
```
- ダウンロードしたPHPのソースが改ざんされていないかをチェックする
- `gpg (GNU Privacy Guard)`と呼ばれる暗号化ソフトウェアが使われる
- この$GPG_KEYSは後ほどこのgpgによって使うフィンガープリント（ハッシュ関数で算出したハッシュ値）である
```Dockerfile:
ENV PHP_VERSION 7.3.1
ENV PHP_URL="https://secure.php.net/get/php-7.3.1.tar.xz/from/this/mirror" PHP_ASC_URL="https://secure.php.net/get/php-7.3.1.tar.xz.asc/from/this/mirror"
ENV PHP_SHA256="cfe93e40be0350cd53c4a579f52fe5d8faf9c6db047f650a4566a2276bf33362" PHP_MD5=""
```
- PHPのバージョンと、PHPのダウンロードURL、そしてソースのハッシュ値を定義している
  - https://www.php.net/downloads.php に記載されている
```Dockerfile:
RUN set -xe; \
    \
    fetchDeps=' \
        wget \
    '; \
    if ! command -v gpg > /dev/null; then \
        fetchDeps="$fetchDeps \
            dirmngr \
            gnupg \
        "; \
    fi; \
    apt-get update; \
    apt-get install -y --no-install-recommends $fetchDeps; \
    rm -rf /var/lib/apt/lists/*; \
```
- fetchDepsという変数に`apt-get install`するものを格納している
- wgetはファイルをダウンロードする際に用いるツール
- gpgコマンドが存在しなければfetchDepsに`gnupg`と`dirmngr（証明書の管理ツール）`を追加している
```Dockerfile:
    mkdir -p /usr/src; \
    cd /usr/src; \
    \
    wget -O php.tar.xz "$PHP_URL"; \
    \
    if [ -n "$PHP_SHA256" ]; then \
        echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c -; \
    fi; \
    if [ -n "$PHP_MD5" ]; then \
        echo "$PHP_MD5 *php.tar.xz" | md5sum -c -; \
    fi; \
```
- `/usr/srcディレクトリ`を作成し移動している
- このディレクトリは慣用的にソースを置く場所となってい
- 次にwgetコマンドで$PHP_URLからPHPのソースを`php.tar.xz`として保存している
- $PHP_SHA256が設定されていれば、`sha256sum -c`によってソースのハッシュ値と比較する
- もし違っていれば改ざんされている可能性があるためエラーが返ってくる
- $PHP_MD5でも同様
```Dockerfile:
    if [ -n "$PHP_ASC_URL" ]; then \
        wget -O php.tar.xz.asc "$PHP_ASC_URL"; \
        export GNUPGHOME="$(mktemp -d)"; \
        for key in $GPG_KEYS; do \
            gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
        done; \
        gpg --batch --verify php.tar.xz.asc php.tar.xz; \
        command -v gpgconf > /dev/null && gpgconf --kill all; \
        rm -rf "$GNUPGHOME"; \
    fi; \
```
- `gpg`すなわち電子署名を用いてソースが改ざんされていないかを確認する
- 基本的にハッシュ値の比較だけでもセキュリティ的に十分
- PHPの公式である (https://www.php.net/downloads.php) まで改竄されている事を危惧して厳重に電子署名でチェックしている
```Dockerfile:
apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false $fetchDeps
```
- 先ほどインストールした$fetchDepsはもう使わないため、`apt-get purge`によりアンインストールしている 
- `APT::AutoRemove::RecommendsImportantオプション`をfalseにすることで、$fetchDepsの依存関係にあるパッケージを削除している

### PHPのビルド
```Dockerfile:
COPY docker-php-source /usr/local/bin/
```
- `docker-php-source`というシェルスクリプトをコピーしている
  - このシェルスクリプトは、Dockerfileと同じディレクトリに存在する
  - PHPのソースを解凍したり削除したりする処理をまとめている
```Dockerfile:
RUN set -eux; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libcurl4-openssl-dev \
        libedit-dev \
        libsodium-dev \
        libsqlite3-dev \
        libssl-dev \
        libxml2-dev \
        zlib1g-dev \
        ${PHP_EXTRA_BUILD_DEPS:-} \
    ; \
```
- `apt-mark showmanual`はAPTによって手動でインストールしたパッケージリストを取得するコマンドである
- これをsavedAptMarkとして保持しておき、これからインストールするビルドにしか使わないパッケージ群だけを削除する際に用いる
- libcurl4-openssl-dev: SSL/TLS通信に必要なパッケージ
- libedit-dev: 改行処理や履歴に関するパッケージ?
- libsodium-dev: 暗号化やハッシュ計算などを提供するパッケージ
- libsqlite3-dev: SQLiteという組み込み型DBのパッケージ
- libssl-dev: SSL/TLSの暗号化プロトコルに必要なパッケージ
- libxml2-dev: XMLを使うためのパッケージ
- zlib1g-dev: deflateと呼ばれる圧縮法を実装したライブラリ
- $PHP_EXTRA_BUILD_DEPSにはApacheのインストール時にapache2-devを設定している
- apache2-devはApache上でPHPを動かすために用いられる
```Dockerfile:
sed -e 's/stretch/buster/g' /etc/apt/sources.list > /etc/apt/sources.list.d/buster.list;
```
- **sed は置換処理を行うコマンド**
- `'s/stretch/buster/g'`と書くことで、stretchという文字列を全てbusterに置換することができる
- 置換後のテキストを`/etc/apt/sources.list.d/buster.list`にファイルとして保存することで、新たにパッケージのインストールを制御している
  - busterとはstretchの次バージョン
  - stretchに存在しない新しいパッケージをインストールしたい場合、このようにパッケージのダウンロード先URLを追加する
```Dockerfile:
    { \
        echo 'Package: *'; \
        echo 'Pin: release n=buster'; \
        echo 'Pin-Priority: -10'; \
        echo; \
        echo 'Package: libargon2*'; \
        echo 'Pin: release n=buster'; \
        echo 'Pin-Priority: 990'; \
    } > /etc/apt/preferences.d/argon2-buster; \
    apt-get update; \
    apt-get install -y --no-install-recommends libargon2-dev; \
```
- `/etc/apt/preferences.d`は、特定のパッケージのインストールを制御することができる
- 今回だと**コードネームがbusterのパッケージはlibargon2*以外インストールしない**
- argon2はパスワードのハッシュ関数であり、PHP7.2から導入されている
```Dockerfile:
    rm -rf /var/lib/apt/lists/*; \
    \
    export \
        CFLAGS="$PHP_CFLAGS" \
        CPPFLAGS="$PHP_CPPFLAGS" \
        LDFLAGS="$PHP_LDFLAGS" \
    ; \
    docker-php-source extract; \
    cd /usr/src/php; \
    gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
    debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
# https://bugs.php.net/bug.php?id=74125
    if [ ! -d /usr/include/curl ]; then \
        ln -sT "/usr/include/$debMultiarch/curl" /usr/local/include/curl; \
    fi; \
```
- ビルド時のオプションを環境変数や変数に設定したり、PHPのソースを解凍したりしている
```Dockerfile:
    ./configure \
        --build="$gnuArch" \
        --with-config-file-path="$PHP_INI_DIR" \
        --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
        \
# make sure invalid --configure-flags are fatal errors intead of just warnings
        --enable-option-checking=fatal \
        \
# https://github.com/docker-library/php/issues/439
        --with-mhash \
        \
# --enable-ftp is included here because ftp_ssl_connect() needs ftp to be compiled statically (see https://github.com/docker-library/php/issues/236)
        --enable-ftp \
# --enable-mbstring is included here because otherwise there's no way to get pecl to use it properly (see https://github.com/docker-library/php/issues/195)
        --enable-mbstring \
# --enable-mysqlnd is included here because it's harder to compile after the fact than extensions are (since it's a plugin for several extensions, not an extension in itself)
        --enable-mysqlnd \
# https://wiki.php.net/rfc/argon2_password_hash (7.2+)
        --with-password-argon2 \
# https://wiki.php.net/rfc/libsodium
        --with-sodium=shared \
        \
        --with-curl \
        --with-libedit \
        --with-openssl \
        --with-zlib \
        \
# bundled pcre does not support JIT on s390x
# https://manpages.debian.org/stretch/libpcre3-dev/pcrejit.3.en.html#AVAILABILITY_OF_JIT_SUPPORT
        $(test "$gnuArch" = 's390x-linux-gnu' && echo '--without-pcre-jit') \
        --with-libdir="lib/$debMultiarch" \
        \
        ${PHP_EXTRA_CONFIGURE_ARGS:-} \
    ; \
```
- configureというシェルスクリプトを実行しているだけ
- このconfigureスクリプトはPHPのソースに含まれており、**インストールに必要なライブラリのチェックとMakefileの生成**を行う
- Makefileファイルはmakeというコンパイルを行うコマンドで用いる
- configureスクリプトの実行時に様々なオプションを指定している
  - --with-config-file-path: PHPの設定ファイルディレクトリを指定
  - --with-openssl: PHPがOpenSSLをサポートできるようにしたりしている
```Dockerfile:
    make -j "$(nproc)"; \
    make install; \
    find /usr/local/bin /usr/local/sbin -type f -executable -exec strip --strip-all '{}' + || true; \
    make clean; \
```
- makeでPHPのコンパイルを行う
- -j オプションでコンパイルを実行するジョブ数を指定することができる
- `nproc`コマンドで取得できるCPU数をそのまま渡している
- `make install`はmakeでビルドしたバイナリなどを規定のディレクトリに移動させるコマンドである
- `find`コマンドはファイルやディレクトリを検索するコマンド
  - 今回は実行可能なバイナリファイルを列挙して、それを`strip`コマンドの引数として渡している。
  - `strip --strip-all`コマンドは渡されたファイルのシンボルテーブル（デバッグ用に使われるデータ）を全て削除し、実行ファイルのサイズを軽量化している
- `make clean`コマンドは、コンパイル時に生成したもう使わないファイルなどを削除する
```Dockerfile:
# https://github.com/docker-library/php/issues/692 (copy default example "php.ini" files somewhere easily discoverable)
    cp -v php.ini-* "$PHP_INI_DIR/"; \
    \
    cd /; \
    docker-php-source delete; \
```
- PHPのソースに[php.ini-development](https://github.com/php/php-src/blob/master/php.ini-development)は[php.ini-production](https://github.com/php/php-src/blob/master/php.ini-production)やといった初期iniファイルがあるので、それをコピーしている
- `docker-php-source delete`でPHPのソースを削除する
```Dockerfile:
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
    apt-mark auto '.*' > /dev/null; \
    [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
    find /usr/local -type f -executable -exec ldd '{}' ';' \
        | awk '/=>/ { print $(NF-1) }' \
        | sort -u \
        | xargs -r dpkg-query --search \
        | cut -d: -f1 \
        | sort -u \
        | xargs -r apt-mark manual \
    ; \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
```
- ビルド時のみ必要だったパッケージをapt-get purgeで削除している (軽量化)
```Dockerfile:
    php --version; \
    \
# https://github.com/docker-library/php/issues/443
    pecl update-channels; \
    rm -rf /tmp/pear ~/.pearrc
```
- phpのバージョンを表示し、PHPのインストールが正常にできているかを確認している
- `pecl update-channels`でpeclのリポジトリを更新している
```Dockerfile:
COPY docker-php-ext-* docker-php-entrypoint /usr/local/bin/
```
- `docker-php-ext-*`はPHPの拡張機能をインストールするためのスクリプトである
- Dockerfileと同じディレクトリに存在している
- 自分たちが使うのは基本的に`docker-php-ext-install`というスクリプト
- これはDockerfile内で`RUN docker-php-ext-install curl`のように使えば、PHPのcurl拡張機能がインストールされる
```Dockerfile:
# sodium was built as a shared module (so that it can be replaced later if so desired), so let's enable it too (https://github.com/docker-library/php/issues/598)
RUN docker-php-ext-enable sodium
```
- sodiumというPHPにおける暗号ライブラリを有効化している
- PHP7.2では標準で組み込まれているが、有効化するためにenableスクリプトを実行している
```Dockerfile:
ENTRYPOINT ["docker-php-entrypoint"]
##<autogenerated>##
COPY apache2-foreground /usr/local/bin/
WORKDIR /var/www/html

EXPOSE 80
CMD ["apache2-foreground"]
##</autogenerated>##
```
- Apache2をフォアグラウンドで実行するスクリプトを実行している

</details>