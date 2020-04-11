FROM ubuntu:12.04

# PHP 5.3 + Apache で必要なパッケージをインストール
RUN apt-get -qq update
RUN apt-get install -y -qq --no-install-recommends \
  apache2 \
  libapache2-mod-php5 \
  php5

RUN apt-get install -y php5-cli

# Install Composer
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
RUN php composer-setup.php
RUN php -r "unlink('composer-setup.php');"
RUN mv composer.phar /usr/local/bin/composer
ENV COMPOSER_ALLOW_SUPERUSER 1

EXPOSE 80
CMD bash -c "source /etc/apache2/envvars && exec /usr/sbin/apache2 -DFOREGROUND"
VOLUME ["/var/www"]

RUN a2enmod rewrite

RUN apt-get install -y git
RUN apt-get install -y php5-mysql
