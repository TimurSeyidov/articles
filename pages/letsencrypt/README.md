# Установка и настройка LetsEncrypt на вашем сервере
[назад к оглавлению](../../README.md)

 > Данное решение опробовано на Linux Debian 12/11/10
 > **У вас должен быть root доступ к серверу**
 > Основано на [данное статье](https://g-soft.info/articles/10873/kak-zaschitit-nginx-s-pomoschyu-let-s-encrypt-v-debian/)

## Задача
Защитить Nginx с помощью бесплатного сертификата Let's Encrypt.

## Особенности Let's Encrypt
- **Нулевая стоимость, бесконечная ценность**: Let's Encrypt выпускает SSL-сертификаты бесплатно. Это выгодно как для предприятий, так и для частных лиц, уравнивая условия игры и обеспечивая надежную защиту.
- **Автоматизация**: Let's Encrypt автоматизирует утомительную задачу получения и обновления SSL-сертификатов. Эта автоматизация не только исключает человеческий фактор, но и обеспечивает постоянную актуальность ваших сертификатов.
- **Безопасность и доверие**: Внедрение SSL-сертификатов от Let's Encrypt шифрует обмен данными между вашим сервером и клиентами. Это шифрование очень важно для защиты конфиденциальных данных и укрепления доверия между пользователями.
- **Повсеместная совместимость**: Большинство современных браузеров распознают сертификаты Let's Encrypt. Это гарантирует беспроблемную и безопасную работу ваших пользователей.
- **Защита от угроз**: Бдительный процесс обновления и строгие правила использования сертификатов делают Let's Encrypt грозным защитником от множества киберугроз.

## Решение
Для работы с сертификатами нам необходимо установить утилиту **Certbot** на ваш сервер.
**Certbot** - это мощный инструмент, который упрощает процесс получения и настройки SSL-сертификатов от Let's Encrypt.
Можно установить его как отдельное приложение (смотрите в оригинальной статье). Я же установлю его в качествет **snap**  пакета

### 1. Обновление репозиториев пакетов Debian
Перед установкой Certbot очень важно убедиться, что репозитории пакетов и существующие пакеты в вашей системе Debian обновлены. 
```bash
sudo apt update
sudo apt upgrade
```

### 2. Установка поддержки snap
Установим **snapd**
```bash
sudo apt install snapd
```
Обновим ядро **snap**
```bash
snap install core; snap refresh core
```

### 3. Установка Certbot
Выполним команду для установки **Certbot** в качестве snap пакета
```
snap install --classic certbot
```
В ответ получим:
```bash
root@vps:~# snap install --classic certbot
certbot 2.6.0 from Certbot Project (certbot-eff✓) installed
root@vps:~# 
```

### 4. Подготовка Nginx
Мы предпологаем, что у Вас уже конфигурация для вашего домена в Nginx. К примеру возьмем какой-нибудь адрес **site.local**
Отрываем файл конфигурации домена и в секцию **server** добавим настройку:
```nginx
location ~ /.well-known {
    root /etc/nginx/html;
    allow all;
}
```
Папка может быть любой (я указал на ту, которую nginx создал автоматически). Главное, чтобы у этой папки был доступ на запись.
>**Для чего это нужно?** При запросе сертификата Let's Encrypt должен удостовериться, что сайт доступен и принадлежит Вам. Certbot создает в этой папке файл **id** и отправляем **id** на сервис проверки. Let's Encrypt попытается проверить доступ по адресу **http:\/\/site.local\/.well-known\/...\/\<id\>.html**
Проверяем конфигурацию Nginx
```bash
nginx -t
```
Если все хорошо, перезагружаем конфигурацию:
```
service nginx reload
```

### 5. Выпуск сертификата
Для выпуска сертификата нужно выполнить команду:
```bash
sudo certbot --nginx --agree-tos --redirect --hsts --staple-ocsp --email <your email> -d site.local
```
Обратите внимание, что необходимо указать **email**
Флаг **--nginx** подскажет Certbot'у, что мы используем nginx. Certbot найдет файл конфигурации и пропишет все пути к файлам, а также пропишет дополнительные настройки:
- ssl_certificate
- ssl_certificate_key
- ssl_session_timeout
- ssl_session_cache
- ssl_session_tickets
- параметры Диффи-Хеллмана
- ssl_protocols
- ssl_ciphers
- ssl_prefer_server_ciphers
- заголовки HSTS
- ssl_stapling
- ssl_stapling_verify
- ssl_trusted_certificate

В случае успеха мы получим:
```bash
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/site.local/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/site.local/privkey.pem
   Your cert will expire on 2023-12-05. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

### 6. Настройка расписания обновления сертификатов
Так как сертификат выпускаем сроком на 3 месяца, необходимо его обновлять. Для автоматического обновления можно добавить задачу в **crontab**.
```bash
sudo crontab -e
```
Затем добавьте следующую строку в нижнюю часть файла. Эта строка устанавливает ежедневную проверку обновления в 3:00 утра:

```crontab
0 3 * * * certbot renew --quiet
```