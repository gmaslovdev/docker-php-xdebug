# Используем базовый образ PHP с Apache (или другой, если нужно)
FROM php:8.2-fpm

# Устанавливаем необходимые зависимости
RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libzip-dev \
    libicu-dev \
    && docker-php-ext-install zip pdo_mysql intl

# Устанавливаем Xdebug
RUN pecl install xdebug && docker-php-ext-enable xdebug

# Копируем конфигурацию Xdebug
COPY ./xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini

# Устанавливаем рабочую директорию
WORKDIR /var/www/html