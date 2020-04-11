FROM php:7.3-apache

RUN apt update && apt-get install -y git libzip-dev
RUN docker-php-ext-install pdo_mysql zip

# Install Composer
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
RUN php composer-setup.php
RUN php -r "unlink('composer-setup.php');"
RUN mv composer.phar /usr/local/bin/composer
ENV COMPOSER_ALLOW_SUPERUSER 1

RUN a2enmod rewrite

WORKDIR /var/www

# Install opcache apcu
RUN docker-php-ext-install opcache
RUN pecl install apcu
RUN docker-php-ext-enable apcu

#COPY php.ini /usr/local/etc/php/php.ini
#COPY . /var/www
