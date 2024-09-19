Платформа “Лига вузов” является единым информационно-коммуникационным пространством, доступное всем заинтересованным сотрудникам Компании и вузам-партнерам. Процессы взаимодействия с вузами являются открытыми и прозрачными для всех, кто имеет к ней доступ. Таким образом, использование платформы повышает оперативность и точность управленческих решений и обеспечивает своевременные реакции на общие запросы компании и вузов.
2	Требования к техническому обеспечению
2.1	Требования к аппаратному обеспечению
Для корректного функционирования системы и работы с 200 одновременными подключениями необходим виртуальный сервер со следующими характеристиками:
№	Характеристика	Значение
1.		Процессор	4 CPU
2.		Оперативная память	4096 МБ
3.		Постоянная память	40 ГБ
4.		Операционная система	Ubuntu 18.04
2.2	Требования к программному обеспечению
Для корректного функционирования системы на сервере должно быть установлено следующее программное обеспечение:
№	Наименование	Версия
1.	`	Nginx 	1.21.4
2.		NodeJS	16.13.1
3.		npm	8.1.2
4.		Pm2	5.1.2
5.		PHP	7.4
6.		MySQL	8.0
7.		Git	8.9

3	Алгоритм развертывания системы
3.1	Конфигурация WEB-сервера
3.1.1	Установка и настройка nginx
Для работы с виртуальными хостами необходимо установить WEB-сервер nginx. Для установки необходимо выполнить следующие команды:
sudo apt update
sudo apt install nginx
Для проверки работы службы выполните:
systemctl status nginx
3.1.2	Конфигурация виртуальных хостов
Для создания виртуального хоста выполните следующую команду:
sudo vi /etc/nginx/sites-available/hostname.conf
После создания файлов, с конфигурацией хостов необходимо создать символьные ссылки на файлы конфигурации виртуальных хостов в директории /etc/nginx/sites-enabled и произвести перезапуск nginx:
sudo ln -s /etc/nginx/sites-available/hostname.conf /etc/nginx/sites-enabled/hostname.conf
sudo service nginx restart
3.1.2.1	Websocket
upstream wsocket {
        server localhost:3211;
}

map $http_upgrade $connection upgrade {
        default upgrade;
        ''      close;
}

server {
        listen 80;
        listen [::]:80;
        root /var/www/socket;
        access_log /var/opt/logs/dev-access-socket.log combined;
        error_log /var/opt/logs/dev-error-socket.log error;
        index index.php index.html index.htm;
        server_name socket.ligavuzov.ru;

        location /{
                add_header 'Access-Control-Allow-Origin' '*' always;
                add_header 'Access-Control-Allow-Credentials' 'true' always;
                proxy_pass http://wsocket;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_hide_header 'Access-Control-Allow-Origin';
                proxy_hide_header 'Access-Control-Allow-Credentials';
        }
}
3.1.2.2	Хост приложений
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /var/www/html/prod/dist;
        access_log /var/opt/logs/prod-access.log combined;
        error_log /var/opt/logs/prod-error.log error;
        index index.php index.html index.htm;
        server_name _;

        location / {
          try_files $uri $uri/ /index.html;
        }

        location /api {
        alias /var/www/api/prod/web;
        location ~ \.php$ {
                include /etc/nginx/fastcgi.conf;
                fastcgi_pass unix:/run/php/php7.4-fpm.sock;
                fastcgi_param SCRIPT_FILENAME $request_filename;
                fastcgi_read_timeout 150;
        }
        if (!-e $request_filename){
                rewrite ^/(.*) /api/index.php?r=$1 last;
        }
        }

        client_max_body_size 32m;
}

3.1.2.3	Хост файлового менеджера
server {
        listen 80 files.ligavuzov.ru;

        root /var/www/files;

        access_log /var/opt/logs/prod-access.log combined;
        error_log /var/opt/logs/prod-error.log error;

        index index.php index.html index.htm;

        server_name files.ligavuzov.ru;

        location / {
          try_files $uri $uri/ /index.html;
        }

}

3.2	Установка СУБД
MySQL есть в репозиториях Ubuntu. Он разбит на несколько пакетов. 
Для того чтобы установить MySQL сервер выполните команду: 
sudo apt-get install mysql-server
При установке конфигурационный скрипт запросит пароль для администратора (root) базы данных. 
Для того чтобы установить консольный клиент MySQL выполните команду: 
sudo apt-get install mysql-client
Для того чтобы установить модуль для работы с MySQL в PHP выполните команду: 
sudo apt-get install php8-mysql
Конфигурация сервера MySQL содержится в файле /etc/mysql/my.cnf. 

3.3	Установка Git
Для установки Git с помощью пакетного менеджера. сначала обновим списки пакетов из репозиториев:
sudo apt update
Затем осталось загрузить и установить программу:
sudo apt install git

3.4	Установка PHP
Для установки PHP последней стабильной версии выполните следующую команду:
sudo apt install php
sudo apt install php-fpm
После установки внестие изменения в конфигурационный файл php.ini
sudo vi /etc/php/7.4/fpm/php.ini
Измените значения параметров следующим образом:
php_admin_flag[display_errors] = off
php_admin_value[error_log] = /var/log/fpm-php.losst.log
php_admin_flag[log_errors] = on
php_admin_value[memory_limit] = 32M

3.5	Установка NodeJS
Выполните следующие команды, чтобы обновить индекс пакета и установить Node.js и npm:
sudo apt updatesudo apt install nodejs npm
Приведенная выше команда установит ряд пакетов, включая инструменты, необходимые для компиляции и установки собственных надстроек из npm.
После этого проверьте установку, запустив:
nodejs --version
v16.13.1

3.6	Установка PM2
Для установки pm2 после установки NodeJS выполните следующую команду:
npm install pm2 -g

3.7	Установка приложений
3.7.1	Установка серверной части системы
Создайте каталог серверной части приложения:
mkdir /var/www/api
chown www-data:www-data /var/www/api
chmod 755 /var/www/api
Выполните checkout в директории:
GIT_WORK_TREE=/var/www/api/prod/ git checkout -f master 
После развертывания исходного кода необходимо установить все зависимые пакеты с помощью следующей команды:
composer install
После завершения установки необходимо заполнить переменные окружения, расположенные в файле /*application_directiry*/.env. Пример данных для заполнения расположен в файле /*application_directiry*/.env_sample.
Для развертывания базы данных выполните следующую команду в корневом каталоге приложения:
php yii migrate

3.7.2	Установка клиентской части системы
Создайте каталог клиентской части приложения:
mkdir /var/www/front
chown www-data:www-data /var/www/front
chmod 755 /var/www/front
Для развертывания исходного кода необходимо выполнить следующую команду:
GIT_WORK_TREE=/var/www/front/prod/ git checkout -f master 
После развертывания исходного кода необходимо установить все зависимые пакеты с помощью следующей команды:
npm i
Для выполнения сборки выполните:
npm run build

3.7.3	Развертывание websocket
Создайте каталог размещения websocket:
mkdir /var/www/wsocket
chown www-data:www-data /var/www/wsocket
chmod 755 /var/www/wsocket
Для развертывания исходного кода необходимо выполнить следующую команду:
GIT_WORK_TREE=/var/www/socket/prod/ git checkout -f master 
После развертывания исходного кода необходимо установить все зависимые пакеты с помощью следующей команды:
npm i
Для запуска websocket выполните команду в корневом каталоге приложения:
pm2 start app.js

