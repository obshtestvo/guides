# Подкарване на нов Rails проект

В това ръководство са описани основните стъпки по подкарване на нов Ruby on
Rails проект в production режим.

## Общ преглед

Инсталацията минава през следните основни етапи и следва следните основни
принципи:

1. Създаване на нов контейнер за приложението. Може да се ползва общ контейнер
   за staging и production инсталациите, ако има нужда и от двете.
2. Инсталира се глобално системно Ruby, като се компилира от source. Инсталира
   се под `root`.
3. Приложението се инсталира и се подкарва под `www-data`. Файловете му се
   складират във `/var/www/именаприложението`.
4. Пред приложението има Nginx сървър. Това е в новосъздадения контейнер. До
   контейнера HTTP заявките се препращат от външния Nginx сървър, работещ на
   host машината koi.obshtestvo.bg, чрез proxy_pass.
5. За deployment на приложението се препоръчва да се ползва Capistrano. При
   желание за автоматичен deployment (on GitHub push), може да се ползват
   услугите на DeployHQ, където имаме акауънт.
6. За application server се ползва Puma (в Jungle режим).

## Инсталация

### 1. Нов контейнер

Прави се нов LXC контейнер, като се влезе като root на koi.obshtestvo.bg и
се изпълни `container/create` от `/root`. Нека `някаквоиме` е името на
контейнера. За подробности, вижте [документацията на `container/create`](https://github.com/obshtestvo/create-lxc#readme).

    container/create някаквоиме

Тази операция отнема 3-4 минути. След това влизате като `root` в контейнера
и си добавяте публичния ключ във файла `.ssh/authorized_keys`.

За да влезете в новия "сървър" (реално в контейнера), ви трябва неговото ID.
Скриптът `container/create` предлага такова веднага след името на контейнера.
Ако неговото ID е 7, може да се SSH-нете отвън така:

    ssh root@koi.obshtestvo.bg -p 2207

Ако ID-то е 14, сменяте порта на `2214`.

### 2. Инсталация на Ruby

Ако Ruby on Rails проектът няма специфични изисквания за версия на Ruby, се
препоръчва да се инсталира най-новата стабилна версия. Примерите по-долу са
за последната стабилна версия в момента на писане на това ръководство.

Инсталацията на Ruby става чрез компилация от код, с помощта на проекта
[ruby-build](https://github.com/sstephenson/ruby-build), който се използва в
stand-alone режим. Следват се и препоръките за съответната OS в
[Wiki-то на ruby-build](https://github.com/sstephenson/ruby-build/wiki).

Инсталирайте необходими зависимости (като `root`):

    apt-get install -y autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev
    apt-get install -y curl wget

Като `root`, в `/root`, изпълнете следното:

    git clone https://github.com/sstephenson/ruby-build.git
    cd ruby-build
    ./install.sh

Изберете желаната версия на Ruby. Списъкът с актуалните в момента версии може
да се провери с командата `ruby-build --definitions`. След това я инсталирайте:

    ruby-build 2.1.4 /usr/local

Инсталирайте [Bundler](http://bundler.io):

    gem install bundler


### 3. Подготовка на потребител

Като `root` изпълнете следните команди, за да подготвите потребителя `www-data`:

    mkdir -p /var/www
    cp -r .vimrc .ssh .bashrc .profile /var/www/
    chown -R www-data:www-data /var/www/
    chsh -s /bin/bash www-data

Вече трябва да може да се логнете като `www-data` със `su - www-data`, или
директно по SSH отвън.

### 4. Nginx и HTTP препращане

#### 4.1. Уеб сървър в контейнера инсталация

Първо, на машината (в контейнера) се инсталира Nginx сървър (като `root`):

    apt-get install nginx

Създава се конфигурация на сайта:

    vim /etc/nginx/sites-available/именаприложението.conf

Ето примерна конфигурация, където има нужда да смените поне стойностите на
следните променливи:

- `ИМЕНАПРИЛОЖЕНИЕТО`
- `ОСНОВЕНДОМЕЙН`
- `ДРУГИДОМЕЙНИ`

    upstream puma_ИМЕНАПРИЛОЖЕНИЕТО_production {
      server unix:/var/www/ОСНОВЕНДОМЕЙН/shared/tmp/sockets/puma.sock fail_timeout=0;
    }

    server {
      listen 80;

      client_max_body_size 4G;
      keepalive_timeout 10;

      error_page 500 502 504 /500.html;
      error_page 503 @503;

      server_name ОСНОВЕНДОМЕЙН ДРУГИДОМЕЙНИ;
      root /var/www/ОСНОВЕНДОМЕЙН/current/public;
      try_files $uri/index.html $uri @puma_ИМЕНАПРИЛОЖЕНИЕТО_production;

      location @puma_ИМЕНАПРИЛОЖЕНИЕТО_production {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://puma_ИМЕНАПРИЛОЖЕНИЕТО_production;
        # limit_req zone=one;
        access_log /var/www/ИМЕНАПРИЛОЖЕНИЕТО/shared/log/nginx.access.log;
        error_log /var/www/ИМЕНАПРИЛОЖЕНИЕТО/shared/log/nginx.error.log;
      }

      location ^~ /assets/ {
        gzip_static on;
        expires max;
        add_header Cache-Control public;
      }

      location = /50x.html {
        root html;
      }

      location = /404.html {
        root html;
      }

      location @503 {
        error_page 405 = /system/maintenance.html;
        if (-f $document_root/system/maintenance.html) {
          rewrite ^(.*)$ /system/maintenance.html break;
        }
        rewrite ^(.*)$ /503.html break;
      }

      if ($request_method !~ ^(GET|HEAD|PUT|POST|DELETE|OPTIONS)$ ){
        return 405;
      }

      if (-f $document_root/system/maintenance.html) {
        return 503;
      }

      location ~ \.(php|html)$ {
        return 405;
      }
    }

Конфигурацията се активира:

    ln -s /etc/nginx/sites-available/ИМЕНАПРИЛОЖЕНИЕТО.conf /etc/nginx/sites-enabled

#### 4.2. Препращане от host-машината

Създайте нов конфигурационен файл (като `root`):

    vim /etc/nginx/sites-available/ИМЕНАПРИЛОЖЕНИЕТО

Има нужда да промените поне следните променливи в конфигурацията:

- `ОСНОВЕНДОМЕЙН` - например, `glasatni.bg`.
- `НОМЕРКОНТЕЙНЕР` - това е номерът (идентификаторът) на новосъздадения контейнер.

Примерна конфигурация (без SSL):

    # This first part of the configuration removes "www." from the hostname.
    server {
      listen      94.156.219.164:80;
      server_name www.ОСНОВЕНДОМЕЙН;
      return      301 $scheme://ОСНОВЕНДОМЕЙН$request_uri;
    }

    # This is the main configuration
    server {
      listen      94.156.219.164:80;
      server_name ОСНОВЕНДОМЕЙН *.ОСНОВЕНДОМЕЙН;
      location / {
        proxy_pass http://10.255.0.НОМЕРКОНТЕЙНЕР:80/;
        include /etc/nginx/_proxy_settings.conf;
        client_max_body_size 100M;
      }
    }

Активирайте препращането:

    ln -s /etc/nginx/sites-available/ИМЕНАПРИЛОЖЕНИЕТО /etc/nginx/sites-enabled
    /etc/init.d/nginx reload

### 5. Capistrano deployment

### 6. Puma Jungle
