# my-docker_setup

Environment development lokal berbasis Docker: **Apache + PHP 8.2 + MySQL + phpMyAdmin + PostgreSQL + pgAdmin**.
Semua jalan di container, jadi PHP/Composer tidak perlu di-install di laptop. Container otomatis jalan saat laptop menyala (`restart: always`).

## Isi

| Service | Container | Image | Akses | Login |
|---|---|---|---|---|
| app (Apache + PHP 8.2.35 + Composer) | `my-app` | build dari `php:8.2.35-apache` | http://localhost:8080 | - |
| mysqldb | `my-mysql` | `mysql:8.4` | `localhost:3306` | `my_root` / `my_pass` (root: `my_pass`), DB `mydb` |
| phpmyadmin | `my-phpmyadmin` | `phpmyadmin:5.2` | http://localhost:8081 | user MySQL di atas |
| postgresdb | `my-postgres` | `postgres:16-alpine` | `localhost:5432` | `my_root` / `my_pass`, DB `mydb` |
| pgadmin | `my-pgadmin` | `dpage/pgadmin4:9` | http://localhost:8082 | `admin@admin.com` / `my_pass` |

> Password di atas hanya untuk development lokal. Jangan dipakai di server.

## Struktur folder

```
my-docker_setup/
├── docker-compose.yml
├── docker/
│   └── php/
│       ├── Dockerfile
│       └── php.ini
└── src/                 # kode aplikasi (Laravel), di-mount ke /var/www/html
    └── public/
        └── index.php    # phpinfo() untuk tes awal
```

## Prasyarat

- Linux Ubuntu dengan Docker Engine + Docker Compose plugin
- User masuk grup `docker` (supaya tidak perlu `sudo`)
- Docker aktif saat boot:

```bash
docker --version && docker compose version
sudo systemctl enable docker containerd
```

## Setup

```bash
# 1. Cek UID/GID; kalau bukan 1000, ubah build.args di docker-compose.yml
id -u; id -g

# 2. Build dan jalankan
docker compose up -d --build

# 3. Cek status (semua harus running / healthy)
docker compose ps

# 4. Cek versi lewat container
docker compose exec app php -v       # PHP 8.2.35
docker compose exec app composer -V
```

Lalu buka:
- http://localhost:8080 : halaman `phpinfo()`
- http://localhost:8081 : phpMyAdmin
- http://localhost:8082 : pgAdmin

## Membuat project Laravel

```bash
rm -rf src/* src/.[!.]*   # kosongkan src (hapus file tes)
docker compose exec app composer create-project laravel/laravel .
```

Atur `src/.env` Laravel memakai nama service, bukan `127.0.0.1`:

```env
DB_CONNECTION=mysql
DB_HOST=mysqldb
DB_PORT=3306
DB_DATABASE=mydb
DB_USERNAME=my_root
DB_PASSWORD=my_pass
```

Untuk PostgreSQL: `DB_CONNECTION=pgsql`, `DB_HOST=postgresdb`, `DB_PORT=5432`.

## Perintah harian

```bash
docker compose exec app bash                   # masuk ke container app
docker compose exec app php artisan migrate    # artisan
docker compose exec app composer install       # composer
docker compose logs -f app                     # log Apache/PHP
docker compose restart app                     # restart app
docker compose up -d --build                   # build ulang setelah ubah Dockerfile/php.ini
docker compose down                            # matikan semua (data volume tetap)
```

## Catatan

- `php -v` di terminal laptop menampilkan PHP bawaan laptop (kalau ada). PHP container selalu lewat `docker compose exec app ...`.
- Kalau port 3306, 5432, 8080, 8081, atau 8082 sudah dipakai, ganti angka kiri di `ports`, contoh `"3307:3306"`.
- Dengan `restart: always`, container yang di-`stop` tetap hidup lagi setelah reboot. Pakai `docker compose down` untuk benar-benar mematikan.
- Upgrade PHP cukup ganti baris `FROM` di `docker/php/Dockerfile`, lalu `docker compose up -d --build`. PHP 8.2 EOL 31 Des 2026.

## File konfigurasi

### docker-compose.yml

```yaml
services:
  # Apache + PHP 8.2 + Composer (satu container)
  app:
    build:
      context: ./docker/php
      args:
        UID: 1000 # samakan dengan `id -u` di laptop
        GID: 1000 # samakan dengan `id -g` di laptop
    image: my-app:php8.2
    container_name: my-app
    restart: always
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
    environment:
      TZ: Asia/Makassar
    depends_on:
      mysqldb:
        condition: service_healthy
      postgresdb:
        condition: service_healthy
    networks:
      - mynetwork

  mysqldb:
    image: mysql:8.4
    container_name: my-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: my_pass
      MYSQL_DATABASE: mydb
      MYSQL_USER: my_root
      MYSQL_PASSWORD: my_pass
      TZ: Asia/Makassar
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -pmy_pass --silent"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    networks:
      - mynetwork

  phpmyadmin:
    image: phpmyadmin:5.2
    container_name: my-phpmyadmin
    restart: always
    environment:
      PMA_HOST: mysqldb
      PMA_PORT: 3306
      UPLOAD_LIMIT: 64M
    ports:
      - "8081:80"
    depends_on:
      mysqldb:
        condition: service_healthy
    networks:
      - mynetwork

  postgresdb:
    image: postgres:16-alpine
    container_name: my-postgres
    restart: always
    environment:
      POSTGRES_USER: my_root
      POSTGRES_PASSWORD: my_pass
      POSTGRES_DB: mydb
      TZ: Asia/Makassar
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U my_root -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - mynetwork

  pgadmin:
    image: dpage/pgadmin4:9
    container_name: my-pgadmin
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: my_pass
    ports:
      - "8082:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    depends_on:
      postgresdb:
        condition: service_healthy
    networks:
      - mynetwork

networks:
  mynetwork:
    driver: bridge

volumes:
  mysql_data:
  postgres_data:
  pgadmin_data:
```

### docker/php/Dockerfile

```dockerfile
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
```

### docker/php/php.ini

```ini
date.timezone = Asia/Makassar
memory_limit = 512M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 120
display_errors = On
opcache.enable = 1
opcache.validate_timestamps = 1
```

### src/public/index.php

```php
<?php
// Halaman tes sementara. Hapus setelah project Laravel dibuat di ./src.
phpinfo();
```
