FROM ubuntu:18.04

# ppa追加してphp5.6を参照できるようにする
RUN apt-get -qq update
RUN apt-get install -y software-properties-common
RUN add-apt-repository ppa:ondrej/php

# tzdateの選択で止まらないようにする
ENV DEBIAN_FRONTEND=noninteractive

# PHP 5.6 + Apache で必要なパッケージをインストール
RUN apt-get -qq update
RUN apt-get install -y -qq --no-install-recommends \
  apache2 \
  php5.6 \
  libapache2-mod-php5.6 \
  php5.6-cli

RUN a2enmod rewrite
RUN a2enmod php5.6

# Install Composer
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
RUN php composer-setup.php
RUN php -r "unlink('composer-setup.php');"
RUN mv composer.phar /usr/local/bin/composer
ENV COMPOSER_ALLOW_SUPERUSER 1

EXPOSE 80
CMD bash -c "source /etc/apache2/envvars && exec /usr/sbin/apache2 -DFOREGROUND"
VOLUME ["/var/www"]
WORKDIR "/var/www"

RUN apt-get install -y git
RUN apt-get install -y php5.6-mbstring php5.6-mysql php5.6-gd
RUN apt-get install -y php5.6-curl php5.6-dom
