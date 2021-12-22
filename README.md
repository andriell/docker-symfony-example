## Установка Docker
Все, что вам нужно, это зарегистрироваться на docker.com и скачать приложение под свою ОС. Скачали, авторизовались локально, поехали дальше.

Структура приложения
Структура нашего приложения будет выглядеть следующим образом:


## Nginx

    server {
        listen 80;
        index index.php index.html;
        error_log /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;
        root /symfony/public;
    
        client_max_body_size 128m;
    
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        add_header 'Access-Control-Allow-Methods' 'GET,POST,PUT,DELETE,HEAD,OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Origin,Content-Type,Accept,Authorization' always;
    
        location / {
            try_files $uri $uri/ /index.php?$args;
        }
    
        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass php-fpm:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
        }
    }

На первый взгляд, ничего необычного, но прошу вас обратить внимание на директивы root /symfony/public; и fastcgi_pass php-fpm:9000;. Чуть позже я объясню, что в них необычного.

Dockerfile для образа с nginx выглядит достаточно просто:

    FROM nginx:1.17
    ADD ./default.conf /etc/nginx/conf.d/default.conf
    WORKDIR /symfony

Команда FROM указывает Docker, что мы хотим использовать официальный образ nginx 1.17-й версии, скачанный с docker hub. Команда ADD добавляет в контейнер с nginx файл default.conf с локальной машины (т.е. тот, что выше), а WORKDIR устанавливает рабочую директорию внутри контейнера.

## php-cli

Dockerfile для php-cli будет выглядеть следующим образом:

    FROM php:7.4-cli
    
    RUN apt-get update && apt-get install -y \
        libpq-dev \
        wget \
        zlib1g-dev \
        libmcrypt-dev \
        libzip-dev
    
    RUN docker-php-ext-install pdo pdo_mysql zip
    
    RUN wget https://getcomposer.org/installer -O - -q | php -- --install-dir=/bin --filename=composer --quiet
    
    WORKDIR /symfony

Кое-что в этом докерфайле вам должно быть знакомо, это команды apt-get update/apt-get install. Так как мы находимся в контейнере, где установлена linux система, мы можем запускать команды через RUN. Итак, мы устанавливаем необходимые утилиты для самой ОС, а с помощью команды docker-php-ext-install устанавливаем расширения для работы с базами данных, файлами и так далее. Те, кто хотя бы когда-то устанавливали расширения для php, понимают, на сколько это проще того, что было раньше. Дальше мы скачиваем установщик композера, с помощью специальных флагов указываем имя, папку, где будет лежать наш композер, а потом устанавливаем текущую директорию.
    
    php-fpm
    FROM php:7.4-fpm
    
    RUN apt-get update && apt-get install -y \
        libpq-dev \
        wget \
        zlib1g-dev \
        libmcrypt-dev \
        libzip-dev
    
    RUN docker-php-ext-install pdo pdo_mysql
    
    WORKDIR /symfony

Для php-fpm Dockerfile почти такой же, за исключением того, что мы не устанавливаем composer, так как он нам не нужен, когда есть php-cli.

## docker-compose

Утилита docker-compose идет в составе Docker и нужна для управления несколькими контейнерами одновременно. Она поможет нам быстро развернуть приложение и начать разработку. Я приведу весь файл конфигурации, а потом объясню, что каждая команда значит.

    version: '3.0'
    
    services:
      nginx:
        build:
          context: ./docker/nginx
        volumes:
          - ./app:/symfony
        container_name: ${PROJECT_NAME}-nginx
        restart: always
        ports:
          - "8081:80"
    
      php-fpm:
        build:
          context: ./docker/php-fpm
        volumes:
          - ./app:/symfony
        container_name: ${PROJECT_NAME}-php-fpm
        depends_on:
          - mysql
    
      php-cli:
        build:
          context: ./docker/php-cli
        volumes:
          - ./app:/symfony
        command: sleep 10000
        container_name: ${PROJECT_NAME}-php-cli
    
      mysql:
        image: mysql:8.0
        command: --default-authentication-plugin=mysql_native_password
        volumes:
          - mysql:/var/lib/mysql
        container_name: ${PROJECT_NAME}-mysql
        restart: always
        environment:
          - "MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}"
          - "MYSQL_DATABASE=${MYSQL_DATABASE}"
          - "MYSQL_USER=${MYSQL_USER}"
          - "MYSQL_PASSWORD=${MYSQL_PASSWORD}"
    
    volumes:
      mysql:

В начале файла мы должны указать версию docker-compose, чтобы пользоваться или не пользоваться командами определенной версии. Далее, в директиве services, мы описываем наши контейнеры. Для этого указываем имя (nginx, php-cli, php-fpm, mysql), в build -> context мы должны указать, откуда брать Dockerfile, на основании которого мы будем собирать наш контейнер. Чтобы все ваши данные после завершения работы контейнера не были потеряны, используют волюмы. Том (volume) - это папка хоста, примонтированная к файловой системе контейнера. В данном случае мы монтируем папку app с локальной машины в папку symfony, которая находится в контейнере (слева направо).

Дальше мы можем указать container_name, иначе докер сгенерирует его сам и вам будет тяжело его запомнить, чтобы войти в контейнер. ${PROJECT_NAME} - это переменная, которую докер ищет в енв, в нашем случае - это .env файл, там вы можете указать любое произвольное имя. Таким образом, имя контейнера может быть таким: docker-symfony-nginx. Директива restart:always, думаю, более чем очевидна и не требует объяснений. ports пробрасывает порты из контейнера (которые правее) наружу на нашу машину (которые левее). Таким образом, сайт будет открываться по localhost:8081.

То же самое мы проделываем с контейнером php-fpm. Сюда еще добавляется команда depends_on, где мы указываем, что php-fpm зависит от контейнера с mysql (который мы укажем ниже), а значит, пока не поднимется mysql, не поднимется php-fpm.

В php-cli добавляется директива command, в которой мы просто указали юниксовую команду, заставляющую терминал заснуть на 10 тысяч секунд. Зачем это нам нужно, затем, что в php-cli по умолчанию нет демона, который бы работал в фоне, поэтому наш контейнер после запуска сразу же завершится и вам не получится зайти в него.

Для контейнера с mysql мы не писали свой Dockerfile, а используем образ mysql:8.0. Без этой команды --default-authentication-plugin=mysql_native_password контейнер запустить не удастся. Что она делает и зачем нужна, можете прочитать тут. environment позволяет прокинуть какие-то данные внутрь контейнера. Таким образом, мы прокидываем наши пароли, название базы и имя юзера в контейнер.

Теперь самое интересное, запись mysql:/var/lib/mysql немного отличается от того, что вы видели ранее. Дело в том, что каждый раз при создании и удалении контейнера генерируется новый id для него, и, соответственно, каждый раз генерировалась бы новая папка с данными базы, т.е. база каждый раз была бы новой. Выход есть - использовать именованный том. Нам необходимо указать его в инструкции volumes, где mysql - это имя контейнера, а у контейнера примонтировать его следующим образом: mysql:/var/lib/mysql. Путь для хранения данных вашего именованного тома будет вычисляться по формуле <DOCKER_PATH>/volumes/<VOLUME_NAME>/_data. Теперь при удалении и создании контейнера Docker будет цеплять папку с вашим томом, зная имя контейнера. Чтобы проверить это, выполните в терминале следующую команду: docker volume ls. Она покажет все существующие волюмы. Выберите тот, который появится после того, как мы запустим наш docker-compose и выполните следующую команду: docker volume inspect <container_name>. Ответ будет примерно следующим:

    [
        {
            "CreatedAt": "2020-02-15T19:33:19Z",
            "Driver": "local",
            "Labels": {
                "com.docker.compose.project": "docker-symfony",
                "com.docker.compose.version": "1.23.2",
                "com.docker.compose.volume": "mysql"
            },
            "Mountpoint": "/var/lib/docker/volumes/docker-symfony_mysql/_data",
            "Name": "docker-symfony_mysql",
            "Options": null,
            "Scope": "local"
        }
    ]
    

Еще один хороший вариант хранить базу данных mysql не в volume, а в локальной папке.

    mysql:
        image: mysql:8.0
        ...
          - ./mysql-db:/var/lib/mysql
        ...

Прежде чем мы запустим сборку и убедимся, что все хорошо, я расскажу про то, что обещал выше, а именно про root /symfony/public; и fastcgi_pass php-fpm:9000;, хотя, вероятно, вы уже и сами догадались. В root мы указываем имя рабочей директории из контейнера, которую примонтировали к нашей локальной папке, а именно symfony/public из контейнера будет смотреть в app/public нашего проекта. В fastcgi_pass, где мы должны указать адрес FastCGI-сервера, мы указываем имя контейнера с php-fpm и порт, который по умолчанию всегда 9000. Поскольку мы пользуемся утилитой docker-compose, контейнеры по умолчанию друг друга видят, поэтому нам не нужно их связывать через links или networks (о них читайте подробнее в документации).

## Первый запуск
Давайте попробуем запустить то, что получилось. Для этого в корне проекта (там, где ваш docker-compose) выполните следующую команду 

    docker-compose up --build -d

Эта команда поднимает ваши контейнеры, но перед этим запускает сборку (--build) и запускает ваши контейнеры в режиме демона (-d). Первая сборка будет долгой, поэтому придется подождать. Когда все выполнилось, вы должны будете увидеть примерно следующее:

    Creating docker-symfony-nginx   ... done
    Creating docker-symfony-php-cli ... done
    Creating docker-symfony-php-fpm ... done
    Creating docker-symfony-mysql ... done

Это означает, что все прошло хорошо. Чтобы убедиться, что сайт работает, создайте в папке app папку public и положите туда файл index.php с таким содержимым:

    <?php phpinfo();

Вы должны будете увидеть знакомую вам страницу. Если все прошло хорошо, давайте установим Symfony в нашу папку app. Заходим в наш контейнер с php-cli: 

    docker exec -it docker-symfony-php-cli bash

В контейнере выполняем следующую команду, чтобы установить Symfony: 

    composer create-project symfony/website-skeleton app

После установки выполняем несколько команд, чтобы избавиться от вложенности папок:

    rm -Rf public
    mv /symfony/app/* /symfony
    mv /symfony/app/.* /symfony
    rm -Rf app

Теперь пробуйте открыть localhost:8081, вас должна приветствовать свежая версия Symfony:


Напоследок покажу, как подключиться к базе данных. Для этого необходимо указать те пароль и имя, что мы прокинули в контейнер ранее, а в качестве хоста надо указать имя контейнера, т.е. mysql:

Файл app/.env

    DATABASE_URL=mysql://root:ppp1234567890@mysql/symfony?serverVersion=8.0

Чтобы остановить контейнеры, выполните команду 

    docker-compose down
