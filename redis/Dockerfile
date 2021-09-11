FROM wordpress:php7.4-fpm

RUN apt-get update && apt-get upgrade -y && pecl install redis \
    && docker-php-ext-enable redis
