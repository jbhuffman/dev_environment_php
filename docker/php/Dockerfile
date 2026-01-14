FROM php:8.4-fpm-alpine

RUN apk add --no-cache \
    bash \
    icu-dev \
    oniguruma-dev \
    git \
    curl \
    libpng-dev \
    libjpeg-turbo-dev \
    libwebp-dev \
    freetype-dev \
    libzip-dev \
    zip \
    unzip \
    freetype \
    libpng \
    libjpeg-turbo \
    libwebp \
    mysql-client \
    fcgi \
    && docker-php-ext-install \
    pdo \
    pdo_mysql \
    intl \
    mbstring \
    zip \
    opcache

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html
COPY php.ini /usr/local/etc/php/conf.d/php.ini
COPY php-fpm-healthcheck /usr/local/bin/php-fpm-healthcheck
RUN chmod +x /usr/local/bin/php-fpm-healthcheck