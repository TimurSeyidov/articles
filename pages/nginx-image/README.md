# Обработка изображений на лету с помощью Nginx]
[назад к оглавлению](/README.md)

 > Данное решение опробовано на Linux Debian 12/11/10
 > **У вас должен быть root доступ к серверу**

По истечению времени многие сталкиваются с задачей, что необходимо на сайте обрабатывать изображения:
- изменение размера
- обрезка
- наложение "водяного знака"

Привычным решением для многих становится использовать скрипты на PHP/JS/Python/etc. Но у такого способа есть недостатки: необходимо заботиться о кэше, времени жизни. А решение на PHP для каждой задачи вообще создает отдельный процесс.

В данной статье я опишу процесс: как это можно автоматизировать с помощью Nginx и возложить на него все задачи.

## 1. Подготовка
За обработку изображений в Nginx отвечает модуль **ngx_http_image_filter_module**. В последних версиях он включен по умолчанию. Если у Вас уже установлен Nginx, вы можете проверить его наличие с помощью команды:
```bash
nginx -V 2>&1 | xargs -n1 | grep 'with-http_image_filter_module' | wc -l
```
Если установлен, то в ответе Вы увидите 1. Если Вам не нужно поддержки "водяного знака", можете пропускать следующий шаг

## 2. Сборка Nginx из исходников
Я обычно устанавливаю Nginx из исходников, так как данный вариант дает полный контроль над процессом.
### 2.1 Скачиваем Nginx
Скачать исходники для nginx можно на [официальном сайте](https://nginx.org/en/download.html). Предлагаю сделать это на стороне нашего сервера.

Подключаемся по SSH к серверу и переходим в домашнюю папку
```bash
cd ~
```
В данной папке я создам папку **sources**, в ней будем хранить исходники программ
```bash
mkdir sources; cd sources
```
Скачиваем исходники (в примере используется стабильная версия 1.24.0)
```bash
wget https://nginx.org/download/nginx-1.24.0.tar.gz
```
>Если у вас не установлена утилита **wget**, ее можно установить из репозитория
>```bash
> apt install wget -y
>```
Распаковываем скачанных архив
```bash
tar -zxvf nginx-1.24.0.tar.gz
```

После мы должны увидеть директорию с исходниками:
```bash
$ ls -al
drwxr-xr-x 3 user user    4096 Sep 22 08:40 .
drwx------ 3 user user    4096 Sep 22 08:39 ..
drwxr-xr-x 8 user user    4096 Apr 10 21:45 nginx-1.24.0
-rw-r--r-- 1 user user 1112471 Apr 11 12:04 nginx-1.24.0.tar.gz
```
Переходим в папку **nginx-1.24.0**
```bash
cd nginx-1.24.0
```

### 2.2 Добавление модуля наложения водяного знака
Если Вас не нужно добавление водяного знака на изображение, пропустите этот шаг.
Дело в том, что стандартный модуль Nginx не умеет накладывать водяной знак на изображение. Благо существует пропатченая версия этого модуля, которая добавляет такую возможность.
Нам необходимо скачать из [репозитория](https://github.com/intaro/nginx-image-filter-watermark) пропатченную версию файла **ngx_http_image_filter_module.c** и заменить им стандартный модуль в исходниках в папке **src/http/modules/**

### 2.3 Сборка
Теперь нам необходимо собрать nginx из исходников
```bash
./configure --with-http_image_filter_module=dynamic --with-http_ssl_module --with-http_v2_module --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --with-pcre --pid-path=/var/run/nginx.pid --with-http_ssl_module
```
Немного поясню:
- **--with-http_image_filter_module=dynamic** - в сборку включается модуль обработки изображений (динамическое подключение)
- **--with-http_ssl_module**, **--with-http_ssl_module** - в сборку включается модуль работы в SSL сертификатами
- **--with-http_v2_module** - в сборку включается модуль поддержки протокола HTTP2
- **--prefix** - путь установки Nginx
- **--sbin-path** - путь для symbol link
- **--conf-path** - файл базовой конфигурации Nginx
- **--error-log** - папка для хранения error логов
- **--http-log-path** - папка для хранения access логов
- **--with-pcre** - поддержка PCRE
- **--pid-path** - файл процесса Nginx

Во время сборки мы можем получить ошибки:
```bash
./configure: error: C compiler cc is not found
```
Устанавливаем компилятор C
```bash
apt install gcc -y
```

```bash
./configure: error: the HTTP rewrite module requires the PCRE library.
```
Отсутствует библиотека для регулярных выражений (Perl Compatible Regular Expressions). Устанавливаем
```bash
apt install libpcre3 libpcre3-dev -y
```

```bash
./configure: error: SSL modules require the OpenSSL library.
```
 Отсутствует библиотека ssl. Устанавливаем
```bash
apt install libssl-dev -y
```

```bash
./configure: error: the HTTP gzip module requires the zlib library.
```
Отсутствует библиотека для компрессии. Устанавливаем
```bash
apt install zlib1g zlib1g-dev -y
```

```bash
./configure: error: the HTTP image filter module requires the GD library.
```
Отсутствует библиотека обработки изображений. Устанавливаем
```bash
apt install libgd-dev -y
```

Если конфигурация прошла успешно, мы должны получить в консоли текст:
```bash
...
Configuration summary
  + using system PCRE library
  + using system OpenSSL library
  + using system zlib library
...
```
Компилируем
```bash
make
```
Если вы получили ошибку
```bash
make: command not found
```
то установите утилиту
```bash
apt install make -y
```

После компиляции посмотрите в логе, нет ли ошибок.
Устанавливаем
```bash
make install
```

Поздравляю! Вы установили Nginx из исходников. Проверим версию Nginx
```bash
/usr/sbin/nginx -V

nginx version: nginx/1.24.0
built by gcc 12.2.0 (Debian 12.2.0-14)
built with OpenSSL 3.0.9 30 May 2023
TLS SNI support enabled
configure arguments: --with-http_image_filter_module=dynamic --with-http_ssl_module --with-http_v2_module --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --with-pcre --pid-path=/var/run/nginx.pid --with-http_ssl_module
```

Запускаем Nginx
```bash
/usr/sbin/nginx 
```

Проверяем, запущен ли процесс:
```bash
root       14413  0.0  0.0  43348  3404 ?        Ss   09:00   0:00 nginx: master process /usr/sbin/nginx
nobody     14414  0.0  0.1  44284  7232 ?        S    09:00   0:00 nginx: worker process
user        14471  0.0  0.0   6060  1856 pts/0    S+   09:00   0:00 grep nginx
```

Проверим его работу
```
curl http://localhost
```
Если Вы получили ошибку
```bash
bash: curl: command not found
```
то установите утилиту cURL:
```bash
apt install curl -y
```
и повторите проверку.

В результате мы получим:
```curl
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Теперь нам надо "убить" процессы Nginx
```bash
kill -9 $(sudo lsof -t -i:80)
```

### 2.4 Создаем системный сервис
Сервис позволит не только запускать и останавливать nginx, но так же стартовать nginx при загрузке/перезагрузке системы.

Создаем/редактируем файл в любом тексотовом редакторе на сервере (я использую **nano**)
```bash
nano /lib/systemd/system/nginx.service
```
И пишем конфигурацию сервиса
```ini
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Запускаем nginx:
```
systemctl start nginx
```
И проверяем есть ли процесс:
```bash
ps aux | grep nginx

root       15393  0.0  0.0  43348  3396 ?        Ss   09:07   0:00 nginx: master process /usr/sbin/nginx
nobody     15394  0.0  0.1  44284  7240 ?        S    09:07   0:00 nginx: worker process
user        15450  0.0  0.0   6060  1844 pts/0    S+   09:08   0:00 grep nginx
```

Судя по ответу всё в порядке. Поскольку теперь у нас есть сервис, мы можем посмотреть статус nginx используя команду:
```bash
systemctl status nginx

● nginx.service - The NGINX HTTP and reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; disabled; preset: enabled)
     Active: active (running) since Fri 2023-09-22 09:07:39 EDT; 53s ago
    Process: 15391 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 15392 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 15393 (nginx)
      Tasks: 2 (limit: 4583)
     Memory: 3.0M
        CPU: 37ms
     CGroup: /system.slice/nginx.service
             ├─15393 "nginx: master process /usr/sbin/nginx"
             └─15394 "nginx: worker process"
```

На данный момент, после перезагрузки машины, Nginx не будет запущен, что, очевидно, плохо для веб-сервера. Поэтому сделаем так, чтобы Nginx "заводился" после ребута.

Остановим сервис:
```bash
systemctl stop nginx
```
И выполним:
```bash
systemctl enable nginx
```
и получим сообщение:
```
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /lib/systemd/system/nginx.service.
```

### 2.5 Базовая настройка Nginx
Открываем файл **/etc/nginx/nginx.conf**
У Вас там может быть портянка из настроек. Я приведу пример своей конфигураци
```nginx
user  www-data;
worker_processes  4;
load_module modules/ngx_http_image_filter_module.so;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
    use epoll;
    multi_accept on;
}


http {
    include       mime.types;
    default_type  application/octet-stream;
    include /home/user/configurations/nginx/*.conf;
}
```
Поясним:
- **user** -  пользователь, под кем будет запускаться процесс
- **worker_processes** - количество одновременных процессов. По одному на каждое ядро
- **pid** - PID процесса
- **include** - папка с конфигурациями доменов

Сохраняемся и проверяем конфигурацию
```bash
nginx -t

nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 3. Настройка хоста для обработки изображения
Обычно я храню настройки для хостов отдельно от папки Nginx.
Создадим в домашней директории папку для хранения конфигураций
```bash
cd ~; mkdir -p configurations/nginx; cd configurations/nginx
```
Создаем файл с расширением **.conf**. Для примера я буду использовать домен **img.site.local**
Создаем файл:
```bash
nano img.site.local.conf
```
И прописываем конфигурацию
```nginx
proxy_cache_path /web/cache/images levels=1:2 keys_zone=thumbs:10m inactive=24h max_size=5G;

server {
    listen 80;
    listen [::]:80;
    server_name img.site.local;
    server_tokens off;
    location / {
        proxy_pass http://localhost:8081;
        proxy_cache thumbs;
        proxy_cache_valid  200      24h;
        proxy_cache_valid  404 415  1m;
    }
}

server {
    listen 8081;

    root /web/mediacontent;

    set $sharpen 0;
    set $quality 100;

    image_filter_buffer 20M;

    if ($uri ~ ^/(crop|resize)/(\d+|-)x(\d+|-)/){
         set $w $2;
         set $h $3;
    }

    error_page 415 = /empty;

    location ~ ^/crop/(?:\d+|-)x(?:\d+|-)/.*\.(?:jpg|gif|png)$ {
        rewrite ^/crop/[\w\d-]+/(.*)$ /$1;
        if (!-f $request_filename) {
            rewrite ^.*$ /notfound last;
        }
        include /home/<user>/configurations/nginx/inc/watermark.conf;
        image_filter crop $w $h;
        break;
    }
    location ~ ^/resize/(?:\d+|-)x(?:\d+|-)/.*\.(?:jpg|gif|png)$ {
        rewrite ^/resize/[\w\d-]+/(.*)$ /$1;
        if (!-f $request_filename) {
            rewrite ^.*$ /notfound last;
        }
        include /home/<user>/configurations/nginx/inc/watermark.conf;
        image_filter resize $w $h;
        break;
    }
    location ~ ^/original/.*\.(?:jpg|png)$ {
        rewrite ^/original/(.*)$ /$1;
        if (!-f $request_filename) {
            rewrite ^.*$ /notfound last;
        }
        break;
    }
    location ~ ^.*\.(?:jpg|png)$ {
        image_filter_sharpen $sharpen;
        image_filter_jpeg_quality $quality;
        image_filter_webp_quality $quality;
        include /home/<user>/configurations/nginx/inc/watermark.conf;
    }

    location = /notfound {
        return 404;
    }

    location = /empty {
        empty_gif;
    }
}
```
Поясним:
- Оригиналы картинок будут храниться в папке **/web/mediacontent**.
Для теста я поместил в нее изображение **picture.jpg**.
![Оригинальная картинка](https://timurseyidov.github.io/articles/pages/nginx-image/assetspicture.jpeg)
- Кэш будет хранится в папке **web/cache/images** сутки с максимальным размером 5Gb
- Если Вам не нужна поддержка водяного знака, удалите строки **include /home/\<user\>/configurations/nginx/inc/watermark.conf;**, иначе:
    - замените **\<user\>** на имя пользователя
    - создайте рядом с текущей конфигурацией папку **inc** и создайте в ней файл **watermark.conf**
    ```bash
    mkdir inc; cd inc; nano watermark.conf
    ```
    и добавьте содержимое
    ```nginx
    image_filter watermark;
    image_filter_watermark /web/watermark.png;
    image_filter_watermark_position center-center;
    image_filter_watermark_width_from 300;
    image_filter_watermark_height_from 150;
    ```
    - **image_filter_watermark** -  путь до картинки с водным знаком
    ![Водный знак](https://timurseyidov.github.io/articles/pages/nginx-image/assetswatermark.png)
    - **image_filter_watermark_position** - позиция
    - **image_filter_watermark_width_from** - минимальная ширина картинки, при которой накладывается знак
    - **image_filter_watermark_height_from** - минимальная высота картинки, при которой накладывается знак
- доступные **URL**
    - http:\/\/img.site.local\/crop\/**\<ширина\>**x**\<высота\>**\/\<путь_до_картинки\>
      обрезает картинку до указаной ширины/высоты, кэширует и отдает результат
    - http:\/\/img.site.local\/resize\/**\<ширина\>**x**\<высота\>**\/\<путь_до_картинки\>
      изменяет размер картинки до указаной ширины/высоты, кэширует и отдает результат 
    - http:\/\/img.site.local\/original\/\<путь_до_картинки\>
      возвращает оригинальную картинку
    - http:\/\/img.site.local\/\<путь_до_картинки\>
      кэширует и возвращает оригинальную картинку с водным знаком
> Изменение размера и обрезка делается с сохранением пропорций

Проверим конфигурацию Nginx:
```bash
nginx -t

nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Если все хорошо
```bash
service nginx start # Если Nginx не запущен
service nginx reload # Если он был запущен ранее
```
# 4. Проверка результата
http://img.site.local/picture.jpg
**Оригинальный размер с водным знаком**
![Оригинальный размер с водяным знаком](https://timurseyidov.github.io/articles/pages/nginx-image/assets/example_1.jpeg)
**Оригинальная картинка**
http://img.site.local/original/picture.jpg
![Оригинальная картинка](https://timurseyidov.github.io/articles/pages/nginx-image/assets/example_2.jpeg)
**Изменение размера 300x300**
http://img.site.local/resize/300x300/picture.jpg
![Изменение размера 300x300](https://timurseyidov.github.io/articles/pages/nginx-image/assets/example_3.jpeg)
**Обрезка 300x300**
http://img.site.local/crop/300x300/picture.jpg
![Обрезка 300x300*](https://timurseyidov.github.io/articles/pages/nginx-image/assets/example_4.jpeg)
**Обрезка 250x140 (без водяного знака)**
http://img.site.local/crop/250x140/picture.jpg
![Обрезка 250x140 без водного знака](https://timurseyidov.github.io/articles/pages/nginx-image/assets/example_5.jpeg)

## Дополнительно
Если Вы используете водяной знак, но хотите добавить возможность убирать знак, можно добавить такую возможность.
Для этого достаточно (к примеру), добавить еще несколько правил:
```nginx
# после проверки url
# ...
if ($uri ~ ^/nowater/(crop|resize)/(\d+|-)x(\d+|-)/){
        set $w $2;
        set $h $3;
}
# ...
location ~ ^/nowater/crop/(?:\d+|-)x(?:\d+|-)/.*\.(?:jpg|gif|png)$ {
    rewrite ^/nowater/crop/[\w\d-]+/(.*)$ /$1;
    if (!-f $request_filename) {
        rewrite ^.*$ /notfound last;
    }
    image_filter crop $w $h;
    break;
}
location ~ ^/nowater/resize/(?:\d+|-)x(?:\d+|-)/.*\.(?:jpg|gif|png)$ {
    rewrite ^/nowater/resize/[\w\d-]+/(.*)$ /$1;
    if (!-f $request_filename) {
        rewrite ^.*$ /notfound last;
    }
    image_filter resize $w $h;
    break;
}
location ~ ^/nowater/(?!crop/|resize/).*\.(?:jpg|png)$ {
    rewrite ^/nowater/(?!crop|resize)/(.*)$ /original/$2;
    break;
}
```

**Обрезка 400x400 без водяного знака**
http://img.site.local/nowater/crop/400x400/picture.jpg
![Обрезка 400x400 без водяного знака](https://timurseyidov.github.io/articles/pages/nginx-image/assets/example_7.jpeg)
**Изменение размера 400x400 без водяного знака**
http://img.site.local/nowater/resize/400x400/picture.jpg
![Изменение размера 400x400 без водяного знака](https://timurseyidov.github.io/articles/pages/nginx-image/assets/example_6.jpeg)
