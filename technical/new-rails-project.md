# Подкарване на нов Rails проект

В това ръководство са описани основните стъпки по подкарване на нов Ruby on
Rails проект в production режим.

Написано е за проектите на [Общество.бг](https://www.obshtestvo.bg) и е
нагодено за архитектурата, поддържана на нашите сървъри. Там всеки проект е в
отделна виртуална машина (LXC контейнер). Въпреки това, ръководството може да
се използва и за други проекти, като приемете, че контейнерът е просто
машината, на която ще инсталирате. Няма да има нужда и от препращането по HTTP
от host-машината към контейнера.

## Променливи

В примерите по-долу са използвани някои "променливи", които трябва да замените
със съответните им стойности. Някои от променливите са:

- `ИМЕНАПРИЛОЖЕНИЕТО` - например, `glasatni`. За предпочитане само alphanumeric.
- `ОСНОВЕНДОМЕЙН` - например, `glasatni.bg`.
- `НОМЕРКОНТЕЙНЕР` - това е номерът (идентификаторът) на новосъздадения
  контейнер. Например, `14`.
- `ПАРОЛАЗАБАЗАТА` - произволен низ с порядъчна дължина, измислен от вас.
- `ДРУГИДОМЕЙНИ` - други домейни, на които трябва да отговаря приложението.
  Например, `*.glasatni.bg`.

## Общ преглед

Инсталацията минава през следните основни етапи и следва следните основни
принципи:

1. Създаване на нов контейнер за приложението. Може да се ползва общ контейнер
   за staging и production инсталациите, ако има нужда и от двете.
2. Инсталира се глобално системно Ruby, като се компилира от source. Инсталира
   се под `root`.
3. Приложението се инсталира и се подкарва под `www-data`. Файловете му ще се
   складират във `/var/www/ИМЕНАПРИЛОЖЕНИЕТО`.
4. За база данни се препоръчва PostgreSQL. Инсталира се базата.
5. Настройва се достъпът до базата. Създава се потребител и празна база данни.
6. За deployment на приложението се препоръчва да се ползва Capistrano. При
   желание за автоматичен deployment (on GitHub push), може да се ползват
   услугите на DeployHQ, където имаме акаунт. Прави се първият deploy. За
   application server се ползва Puma (в Jungle режим). Настройва се като част
   от първия deploy.
7. Пред приложението има Nginx сървър. Това е в новосъздадения контейнер. До
   контейнера HTTP заявките се препращат от външния Nginx сървър, работещ на
   host машината koi.obshtestvo.bg, чрез proxy_pass.

## Инсталация

### 1. Нов контейнер

Прави се нов LXC контейнер, като се влезе като root на koi.obshtestvo.bg и
се изпълни `container/create` от `/root`. Нека `ИМЕНАПРИЛОЖЕНИЕТО` е името на
контейнера. За подробности, вижте [документацията на `container/create`](https://github.com/obshtestvo/create-lxc#readme).

    container/create ИМЕНАПРИЛОЖЕНИЕТО

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

Инсталирайте и [Puma](http://puma.io) (необходим е за Puma Jungle init.d
скриптовете):

    gem install puma

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

### 5. Създаване на потребител и база данни

Изберете парола и потребителско име за връзка с базата.

#### 5.1. PostgreSQL

Изпълнете следното като `root`, за да създадете потребител и база данни:

    su - postgresql
    createuser ИМЕНАПРИЛОЖЕНИЕТО --createdb --no-superuser --no-createrole --pwprompt
    createdb ИМЕНАПРИЛОЖЕНИЕТО_production --owner=ИМЕНАПРИЛОЖЕНИЕТО

#### 5.2. MySQL

Следвайте аналогични стъпки като PostgreSQL.

### 6. Първоначален deployment с Capistrano

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

Необходимо е да имате и някакъв JavaScript runtime на сървъра, който ще се
ползва за компилация на asset-ите (CSS и JavaScript файлове). Може да добавите
такъв със следния ред в `Gemfile`:

```ruby
gem 'therubyracer'
```

Добавете и `Capfile` със следното съдържание:

```ruby
require 'capistrano/setup'
require 'capistrano/deploy'

require 'capistrano/bundler'
require 'capistrano/rails/migrations'
require 'capistrano/rails/assets'
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
set :nginx_server_name, 'ОСНОВЕНДОМЕЙН *.ОСНОВЕНДОМЕЙН'

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

За връзка с базата, ако е PostgreSQL, използвайте следните данни:

    # shared/config/database.yml
    production:
      adapter: postgresql
      database: ИМЕНАПРИЛОЖЕНИЕТО_production
      username: ИМЕНАПРИЛОЖЕНИЕТО
      password: ПАРОЛАЗАБАЗАТА
      host: 127.0.0.1
      encoding: unicode
      pool: 15

#### Локално

Отново изпълнете следната команда локално:

    bundle exec cap production deploy:check

Този път трябва да мине без грешки. След това изпълнете:

    bundle exec cap production deploy

Това би трябвало да качи работещо копие на кода и да пусне миграциите.

### 7. Nginx и HTTP препращане

#### 7.1. Уеб сървър в контейнера инсталация

Първо, на машината (в контейнера) се инсталира Nginx сървър (като `root`):

    apt-get install nginx

След това трябва да се активира Nginx конфигурацията (изпълнете като `root`):

    ln -s /var/www/ОСНОВЕНДОМЕЙН/shared/ИМЕНАПРИЛОЖЕНИЕТО_production_nginx.conf /etc/nginx/sites-enabled/

Уверете се, че има файл `/etc/lsb-release`, който е необходим за задачата
`puma:jungle:setup`, която ще бъде изпълнена след малко (при някои
Debian-базирани дистрибуции този файл липсва):

    touch /etc/lsb-release

**Временно** добавете следното в `/etc/sudoers` на сървъра:

    # Allow www-data to execute all commands without a password
    # Should not be left here permanently
    www-data ALL=(ALL:ALL) NOPASSWD: ALL

##### Локално

Изпълнете следните команди от локалното ви копие:

    bundle exec cap production puma:jungle:setup

##### На сървъра

**Важно!** Премахнете/коментирайте реда `www-data ALL=(ALL:ALL) NOPASSWD: ALL`
от файла `/etc/sudoers` на сървъра.

След това:

    /etc/init.d/puma start
    /etc/init.d/nginx restart

#### 7.2. Препращане от host-машината

Тъй като виртуалните машини (LXC-контейнерите) реално се явяват като сървъри в
NAT, зад host-машината, е необходимо да се настрои препращане от Nginx-а на
host-машината към "вътрешния" Nginx, на съответния контейнер.

##### На host-машината

Създайте нов конфигурационен файл (като `root@koi.obshtestvo.bg` на
host-машината):

    vim /etc/nginx/sites-available/ИМЕНАПРИЛОЖЕНИЕТО

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
    rm -f /etc/nginx/sites-enabled/default # remove the default host
    /etc/init.d/nginx reload

Тук всичко би трябвало вече да работи. Тествайте с ръчно добавен запис за
домейна в `/etc/hosts` на вашата машина и ако работи, (пре)насочете DNS-ите.

Последващи deployments се правят от локалното ви копие, със следната команда:

    bundle exec cap production deploy
