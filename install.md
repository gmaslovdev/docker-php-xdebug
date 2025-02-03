Чтобы включить **Xdebug** для PHP в вашем Docker-окружении, нужно выполнить несколько шагов. Мы добавим Xdebug в ваш PHP-контейнер и настроим его для работы с IDE (например, PHPStorm или VSCode).

---

### 1. **Добавление Xdebug в Dockerfile для PHP**

Предположим, ваш Dockerfile для PHP находится в `./docker/php/Dockerfile`. Добавьте установку Xdebug:

```Dockerfile
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
```

---

### 2. **Создание конфигурации Xdebug**

Создайте файл `./docker/php/xdebug.ini` для настройки Xdebug:

```ini
[xdebug]
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_host=host.docker.internal  # Для Windows/Mac
; xdebug.client_host=172.17.0.1  # Для Linux (IP хоста в Docker-сети)
xdebug.client_port=9003
xdebug.log=/var/log/xdebug.log
xdebug.idekey=PHPSTORM  # Или VSCODE, если используете VSCode
```

- **`xdebug.client_host`:** Указывает, куда Xdebug будет отправлять отладочные данные. Для Windows/Mac используйте `host.docker.internal`. Для Linux укажите IP хоста в Docker-сети (обычно `172.17.0.1`).
- **`xdebug.client_port`:** Порт, который слушает ваш IDE (по умолчанию 9003 для Xdebug 3).
- **`xdebug.idekey`:** Ключ IDE, который вы укажете в настройках вашего редактора.

---

### 3. **Обновление `docker-compose.yml`**

Добавьте переменные окружения для Xdebug в сервис `php`:

```yaml
services:
  php:
    build: ./docker/php
    volumes:
      - './src:/var/www/html'
    environment:
      PHP_IDE_CONFIG: "serverName=docker-cli"
      XDEBUG_MODE: debug  # Включаем режим отладки
      XDEBUG_CONFIG: "client_host=host.docker.internal client_port=9003"  # Конфигурация Xdebug
    networks:
      - app-network
```

---

### 4. **Настройка IDE**

#### Для **PHPStorm**:
1. Откройте настройки (`File > Settings`).
2. Перейдите в `Languages & Frameworks > PHP > Servers`.
3. Добавьте новый сервер:
   - Name: `docker-cli` (должно совпадать с `serverName` в `PHP_IDE_CONFIG`).
   - Host: `localhost`.
   - Port: `80`.
   - Используйте path mappings: `/var/www/html` → `./src` (укажите путь к вашему проекту на хосте).
4. Перейдите в `Languages & Frameworks > PHP > Debug`.
   - Убедитесь, что порт Xdebug установлен на `9003`.
5. Включите "Start Listening for PHP Debug Connections" (иконка телефона в верхнем правом углу).

#### Для **VSCode**:
1. Установите расширение [PHP Debug](https://marketplace.visualstudio.com/items?itemName=felixfbecker.php-debug).
2. Создайте файл `launch.json` в директории `.vscode`:
   ```json
   {
     "version": "0.2.0",
     "configurations": [
       {
         "name": "Listen for Xdebug",
         "type": "php",
         "request": "launch",
         "port": 9003,
         "pathMappings": {
           "/var/www/html": "${workspaceFolder}/src"
         }
       }
     ]
   }
   ```
3. Запустите отладку (F5).

---

### 5. **Пересборка и запуск контейнеров**

1. Пересоберите PHP-контейнер:
   ```bash
   docker-compose build php
   ```
2. Запустите контейнеры:
   ```bash
   docker-compose up -d
   ```

---

### 6. **Проверка Xdebug**

1. Установите точку останова в вашем PHP-коде.
2. Откройте проект в браузере (например, `http://localhost`).
3. Если все настроено правильно, IDE перехватит выполнение кода в точке останова.

---

### 7. **Дополнительные советы**

- **Логи Xdebug:** Если отладка не работает, проверьте логи Xdebug:
  ```bash
  docker-compose exec php tail -f /var/log/xdebug.log
  ```
- **Профилирование:** Если нужно профилировать код, измените `xdebug.mode` на `debug,profile` и настройте вывод профилирования.

---

Теперь Xdebug настроен и готов к использованию! 🎉

Для работы PHP с Xdebug через Nginx, нужно правильно настроить Nginx, чтобы он передавал запросы PHP-FPM (FastCGI Process Manager). Вот пример конфигурации Nginx, который будет работать с вашим текущим `docker-compose.yml`.

---

### 1. **Конфигурация Nginx**

Создайте файл `./docker/nginx/conf.d/app.conf` (этот путь указан в `docker-compose.yml` для маунта конфигураций Nginx):

```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/html;
    index index.php index.html index.htm;

    # Логирование
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Обработка PHP-файлов
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;  # Имя сервиса PHP из docker-compose.yml
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PHP_VALUE "error_log=/var/log/php_errors.log";
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }

    # Запрет доступа к скрытым файлам
    location ~ /\.ht {
        deny all;
    }

    # Запрет доступа к логам
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
}
```

---

### 2. **Объяснение конфигурации**

- **`fastcgi_pass php:9000;`:** Указывает, что запросы к PHP должны передаваться на сервис `php` (из `docker-compose.yml`) на порт 9000 (стандартный порт для PHP-FPM).
- **`fastcgi_param SCRIPT_FILENAME`:** Указывает путь к исполняемому PHP-файлу.
- **`try_files $uri $uri/ /index.php?$query_string;`:** Позволяет использовать "красивые" URL (например, для Laravel или других фреймворков).
- **Логирование:** Логи Nginx и PHP будут доступны в контейнере Nginx.

---

### 3. **Обновление `docker-compose.yml`**

Убедитесь, что ваш `docker-compose.yml` правильно маунтит конфигурацию Nginx:

```yaml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - 80:80
    volumes:
      - './src:/var/www/html'  # Исходный код
      - './docker/nginx/conf.d:/etc/nginx/conf.d'  # Конфигурация Nginx
      - './docker/nginx/logs:/var/log/nginx'  # Логи Nginx (опционально)
    depends_on:
      php:
        condition: service_started
    networks:
      - app-network
```

---

### 4. **Структура проекта**

После добавления конфигурации Nginx, структура вашего проекта должна выглядеть так:

```
.
├── docker/
│   ├── nginx/
│   │   ├── conf.d/
│   │   │   └── app.conf  # Конфигурация Nginx
│   │   └── logs/  # Логи Nginx (опционально)
│   ├── php/
│   │   ├── Dockerfile  # Dockerfile для PHP
│   │   └── xdebug.ini  # Конфигурация Xdebug
│   └── volume/
│       └── mysql/  # Том для MySQL
├── src/  # Исходный код вашего приложения
│   └── index.php  # Пример PHP-файла
└── docker-compose.yml
```

---

### 5. **Пример `index.php`**

Для тестирования создайте файл `./src/index.php`:

```php
<?php
phpinfo();
```

---

### 6. **Запуск и проверка**

1. Пересоберите контейнеры:
   ```bash
   docker-compose build
   ```
2. Запустите контейнеры:
   ```bash
   docker-compose up -d
   ```
3. Откройте браузер и перейдите по адресу `http://localhost`. Вы должны увидеть вывод `phpinfo()`.

---

### 7. **Проверка Xdebug через Nginx**

1. Убедитесь, что Xdebug работает:
   - Установите точку останова в `index.php`.
   - Откройте `http://localhost` в браузере.
   - Если все настроено правильно, IDE перехватит выполнение кода.

2. Проверьте логи Nginx и PHP, если что-то не работает:
   ```bash
   docker-compose logs nginx
   docker-compose logs php
   ```

---

Теперь у вас есть полностью настроенный стек PHP + Nginx + Xdebug, готовый для разработки! 🚀