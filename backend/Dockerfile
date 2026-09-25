FROM php:8.4-cli-alpine
RUN apk add --no-cache postgresql-dev git unzip \
    && docker-php-ext-install pdo_pgsql pcntl
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
WORKDIR /var/www/html
EXPOSE 8000
CMD ["sh", "-c", "composer install --no-interaction && php artisan serve --host=0.0.0.0 --port=8000"]
