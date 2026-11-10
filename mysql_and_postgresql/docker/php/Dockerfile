FROM php:8.2.35-apache

ARG UID=1000
ARG GID=1000

# Library sistem + ekstensi PHP untuk Laravel
RUN apt-get update && apt-get install -y --no-install-recommends \
        git unzip libzip-dev libicu-dev libpq-dev libonig-dev \
        libpng-dev libjpeg62-turbo-dev libfreetype6-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j"$(nproc)" \
        pdo_mysql pdo_pgsql mbstring bcmath intl zip gd opcache pcntl \
    && rm -rf /var/lib/apt/lists/*

# Composer dari image resmi
COPY --from=composer:2.8 /usr/bin/composer /usr/bin/composer

# Apache: DocumentRoot ke public/ (Laravel) + mod_rewrite
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
        /etc/apache2/sites-available/*.conf /etc/apache2/apache2.conf \
    && a2enmod rewrite headers

# Konfigurasi PHP untuk development
COPY php.ini /usr/local/etc/php/conf.d/zz-app.ini

# Samakan UID/GID www-data dengan user laptop supaya file di ./src bukan milik root
RUN groupmod -o -g "${GID}" www-data && usermod -o -u "${UID}" -g "${GID}" www-data

WORKDIR /var/www/html
