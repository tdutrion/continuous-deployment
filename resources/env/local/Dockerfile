FROM php:7.2-apache

RUN docker-php-ext-install pdo pdo_mysql \
 && a2enmod rewrite \
 && sed -i 's!/var/www/html!/var/www/html/public!g' /etc/apache2/sites-available/*.conf

WORKDIR /var/www/html
