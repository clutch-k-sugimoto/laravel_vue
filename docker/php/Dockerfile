FROM php:8.0.9-fpm-alpine

# CORY php.ini
COPY ./docker/php/php.ini /usr/local/etc/php/php.ini

# Composer install
COPY --from=composer:2.1.5 /usr/bin/composer /usr/bin/composer

# install Node.js
RUN apk add --no-cache --virtual g++
RUN apk add --no-cache libstdc++ && apk add --no-cache libgcc
COPY --from=node:16.7.0-alpine /usr/local/bin /usr/local/bin
COPY --from=node:16.7.0-alpine /usr/local/lib /usr/local/lib

RUN apk update \
    && apk add \
    git \
    zip \
    unzip \
    vim \
    && docker-php-ext-install pdo_mysql bcmath

WORKDIR /var/www/html
