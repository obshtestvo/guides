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

**Проверка за успех:** Успешно SSH-ване до новия контейнер.

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
    apt-get install -y curl wget sudo

Като `root`, в `/root`, изпълнете следното:

    git clone https://github.com/sstephenson/ruby-build.git
    cd ruby-build
    ./install.sh

Изберете желаната версия на Ruby. Списъкът с актуалните в момента версии може
да се провери с командата `ruby-build --definitions`. След това я инсталирайте:

    ruby-build 2.1.4 /usr/local

Инсталирайте [Bundler](http://bundler.io):

    gem install bundler

**Проверка за успех:** Командата `ruby -v` трябва да покаже последната
инсталирана версия. Например:

    ruby -v
    ruby 2.1.4p265 (2014-10-27 revision 48166) [x86_64-linux]

### 3. Подготовка на потребител

Като `root` изпълнете следните команди, за да подготвите потребителя `www-data`:

    mkdir -p /var/www
    cp -r .vimrc .ssh .bashrc .profile /var/www/
    echo 'export RACK_ENV=production' >> /var/www/.bashrc
    chown -R www-data:www-data /var/www/
    chsh -s /bin/bash www-data

Вече трябва да може да се логнете като `www-data` със `su - www-data`, или
директно по SSH отвън.

**Проверка за успех:** `su - www-data` трябва да ви вкара като `www-data`, като
SHELL-ът там и prompt-ът трябва да приличат на root-ските. Следната команда:

    echo $RACK_ENV

Трябва да изведе "production", когато се изпълни от името на `www-data`.

### 4. Външни зависимости

Инсталирайте други зависимости, от които приложението има нужда, например
бази от данни.

#### PostgreSQL

Инсталацията става по стандартния начин, като `root`:

    apt-get install postgresql libpq-dev

Пакетът `libpq-dev` е необходим, за да може да се компилира Ruby драйвера за
PostgreSQL.

#### MySQL

Подобно на PostgreSQL.

### 5. Capistrano deployment

Препоръчва се употребата на [Capistrano](http://capistranorb.com/).

#### Локално

По-долу се добавят файлове в локалното копие на проекта.

Добавете следните Gem-ове в `Gemfile`-а на проекта, за предпочитане в групата
`:development`:

```ruby
group :development do
  gem 'capistrano'
  gem 'capistrano-rails'
  gem 'capistrano-bundler'
  gem 'capistrano3-puma'
end
```

Добавете следното извън групата `:development`:

```ruby
gem 'puma'
```

Добавете и `Capfile` със следното съдържание:

```ruby
require 'capistrano/setup'
require 'capistrano/deploy'

require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
require 'capistrano/puma'
require 'capistrano/puma/jungle'
require 'capistrano/puma/nginx'

# Loads custom tasks from `lib/capistrano/tasks' if you have any defined.
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

Добавете `config/deploy.rb` със следното примерно съдържание:

```ruby
lock '3.2.1'

set :application,     'ИМЕНАПРИЛОЖЕНИЕТО'
set :repo_url,        'https://github.com/obshtestvo/ОСНОВЕНДОМЕЙН.git'
set :linked_files,    %w(config/database.yml config/secrets.yml config/application.yml)
set :linked_dirs,     %w(bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system)
set :keep_releases,   20
set :rails_env,       'production'
set :puma_init_active_record, true

namespace :deploy do
  namespace :nginx do
    desc 'Generate an Nginx configuration file'
    task :config do
      on roles(:web) do |role|
        template_puma('nginx_conf', "#{shared_path}/#{fetch(:nginx_config_name)}_nginx.conf", role)
      end
    end
  end
end

after 'deploy:check', 'deploy:nginx:config'
after 'deploy:check', 'puma:config'
```

Добавете `config/deploy/production.rb` със следното примерно съдържание:

```ruby
server  'koi.obshtestvo.bg:22НОМЕРКОНТЕЙНЕР',
        user: 'www-data',
        roles: %w(app web db)

set :deploy_to,       '/var/www/ОСНОВЕНДОМЕЙН'
set :puma_threads,    [15, 15]
set :puma_workers,    4
```

Генерирайте шаблони по подразбиране на конфигурациите на Nginx и на Puma, като
изпълните във вашето локално копие на проекта следното:

    bundle install
    rails g capistrano:nginx_puma:config

Commit-нете промените.

Изпълнете локално следната команда (би трябвало да даде грешка заради липсващи
конфигурационни файлове, което е нормално):

    bundle exec cap production deploy:check

Резултатът от командата ще бъде, че ще създаде необходимата структура от
файлове и папки на сървъра.

#### На сървъра

Създайте следните файлове (това са файловете от опцията `:linked_files`,
изредени в `config/deploy.rb`):

    /var/www/ОСНОВЕНДОМЕЙН/shared/config/secrets.yml
    /var/www/ОСНОВЕНДОМЕЙН/shared/config/database.yml
    /var/www/ОСНОВЕНДОМЕЙН/shared/config/application.yml

#### Локално

Отново изпълнете следната команда локално:

    bundle exec cap production deploy:check

Този път трябва да мине без грешки.

#### На сървъра

Активирайте Nginx конфигурацията (изпълнете като `root`):

    ln -s /var/www/ОСНОВЕНДОМЕЙН/shared/ИМЕНАПРИЛОЖЕНИЕТО_production_nginx.conf /etc/nginx/sites-enabled/
    /etc/init.d/nginx restart

Уверете се, че има файл `/etc/lsb-release`, който е необходим за задачата
`puma:jungle:install`, която ще бъде изпълнена след малко (при някои
Debian-базирани дистрибуции този файл липсва):

    touch /etc/lsb-release

**Временно** добавете следното в `/etc/sudoers` на сървъра:

    # Allow www-data to execute all commands without a password
    # Should not be left here permanently
    www-data ALL=(ALL:ALL) NOPASSWD: ALL

#### Локално

Изпълнете следните команди от локалното ви копие:

    bundle exec cap production puma:jungle:install

#### На сървъра

**Важно!** Премахнете/коментирайте реда `www-data ALL=(ALL:ALL) NOPASSWD: ALL`
от файла `/etc/sudoers` на сървъра.

След това:

    /etc/init.d/puma start
    /etc/init.d/nginx restart

TODO

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

### 6. Puma Jungle
