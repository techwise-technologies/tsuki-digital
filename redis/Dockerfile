FROM wordpress:5.8.1-php7.4-fpm

RUN apt-get update && apt-get upgrade -y && pecl install redis \
    && docker-php-ext-enable redis
