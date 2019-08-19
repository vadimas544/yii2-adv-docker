# yii2-adv-docker

Install ubuntu or centos
apt-get update
install docker docker-compose php composer

Create a folder with our project

mkdir beesweet
cd beesweet && mkdir docker

In folder docker create Dockerfile

# Базовый образ с nginx и php
FROM richarvey/nginx-php-fpm

# Добавляем наше веб приложение
ADD app /var/www/app

# Удаляем конфиги сайтов которые там есть
RUN rm -Rf /etc/nginx/sites-enabled/*

# Добавляем наш конфиг
ADD docker/conf/nginx/site.conf /etc/nginx/sites-available/site.conf
# Включаем его
RUN ln -s /etc/nginx/sites-available/site.conf /etc/nginx/sites-enabled/site.conf

In folder docker create docker-compose.yml

# Последняя версия docker-compose
version: '3'

# Создаем общую сеть deafult для всех контейнеров
networks:
  default:
    driver: bridge

# Создаем отдельные контейнеры
services:
  # Контейнер с веб-приложением
  app:
    # Собираем из Dockerfile 
    build: 
      # Корнем указываем корень основного проекта
      context: ../
      dockerfile: ./docker/Dockerfile
    # Показываем наружу 80 порт
    ports: 
      - "80:80"
    # Подключаем к общей сети с другими контейнерами
    networks: 
      - default
    # Запускаем только после db
    depends_on: 
      - db    
    # Линкуем внешнюю папку с исходниками внутрь
    volumes:
      - "../app:/var/www/app"
      # Так же линкуем конфиг для nginx
      - "./conf/nginx:/etc/nginx/sites-available"      
  # Контейнер с базой данных
  db:
    image: mariadb
    # Подключаем к общей сети с другими контейнерами
    networks: 
      - default
    # Показываем наружу порт
    ports:
      - "3336:3306"
    # Задаем параметры для инициализации БД
    environment:
      # Пароль к БД
      MYSQL_ROOT_PASSWORD: root
      # Создаваемая по умолчанию бд
      MYSQL_DATABASE: yii-template-db
    # Линкуем внешнюю папку для хранения БД
    volumes:
      - "./database:/var/lib/mysql"

For nginx create a virtual host for frontend

server {
    charset utf-8;
    client_max_body_size 128M;

    listen 80; ## listen for ipv4
	server_name beesweet.ua;
    root        /var/www/app/frontend/web/;
    index       index.php;

    #access_log  /var/www/app/log/frontend-access.log;
    #error_log   /var/www/app/log/frontend-error.log;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    # uncomment to avoid processing of calls to non-existing static files by Yii
    #location ~ \.(js|css|png|jpg|gif|swf|ico|pdf|mov|fla|zip|rar)$ {
    #    try_files $uri =404;
    #}
    #error_page 404 /404.html;

    # deny accessing php files for the /assets directory
    location ~ ^/assets/.*\.php$ {
        deny all;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/var/run/php-fpm.sock;
        try_files $uri =404;
    }

    location ~* /\. {
        deny all;
    }
}

И также для backend конфига!!!

Go to /var/www/beesweet and execute:

composer create-project --prefer-dist yiisoft/yii2-app-advanced app
composer install

docker-compose -f docker/docker-compose.yml up -d
docker ps

Initialize

app/init --env=Development --overwrite=All

nano app/common/config/main-local.php and change root — root, хост БД — db, имя БД — yii-template-db.

Подключаемся к контейнеру docker exec -it docker_app_1 bash

Подключаемся к контейнеру docker exec -it docker_db_1 bash, заходим в mysql и создаем БД - beesweet.ua 

Выполняем команду миграции БД php /var/www/app/yii migrate

И выходим exit
Тормозим сервис docker-compose -f docker/docker-compose.yml down
Запускаем его заново docker-compose -f docker/docker-compose.yml up -d

Прописываем пути в /etc/hosts
Открываем localhost в браузере и смотрим на новый сайт.
