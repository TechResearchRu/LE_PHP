# LE_PHP_Docker
Контейнер заточен для LE_Framework, выполняет функционал php-fpm
* php 7.4.22
* json, xml, zip, freetype, curl, bcmatch, mysqli
* поддержка RabbitMQ (amqp)
* пропатченная локаль, месяца с маленькой буквы и в родительном падеже
* отправка почты через ssmtp

## Установка

```bash
mkdir php_le_docker;
cd php_le_docker;
wget https://raw.githubusercontent.com/TechResearchRu/LE_PHP_Docker/main/Dockerfile
docker build -t php-le .
```

## Запуск

Если нужно просто поиграться в терминале
```bash
docker run -it --rm php-le /bin/sh
```

Продакшн режим примерно такой (но без поддержки php sendmail), по настройке sendmail из контейнера смотрите раздел ниже
```bash
docker run -d \
--restart=always \
--name project1.ru \
-p 127.0.0.1:9000:9000 \
-v /www:/www \
php-le
```

## Настройка почты
Сперва на локальном компе сделаем конфиг, который прокинем
```
mkdir -p /docker_conf/etc/ssmtp/
nano /docker_conf/etc/ssmtp/ssmtp.conf
```

Вставим примерно такое
```
#see ptr
rewriteDomain=web.mydomain.ru
FromLineOverride=YES
#host ip, see "ip addr|grep docker0"
mailhub=172.17.0.1
UseTLS=YES
UseSTARTTLS=YES
```

> Нужно прописать на ip вашего сервера PTR запись у хостера или провайдера и на эту обратную запись настроить postfix, в примере это **web.mydomain.ru**, 
> проверить ваш обратный адрес можно так `nslookup 8.8.8.8`, 
> в примере выше есть ip **172.17.0.1**, это IP моста в докере, его можно узнать так: `ip addr | grep docker0`

Далее настраиваем Postfix на хосте.

При подобном методе есть вероятность полететь в спам, если у вас несколько разных сайтов с разных доменов будут отправлять, поэтому в Postfix нужно прописать следующее

```sh
#тут хак для подмены отправителя из докеров
sender_canonical_classes = envelope_sender
sender_canonical_maps =  regexp:/etc/postfix/sender_canonical_maps
```

и еще создать файлик `/etc/postfix/sender_canonical_maps` и прописать в него
```
/.+/    postmaster@web.mydomain.ru
```

### Запуск контейнера с настройками sendmail
```
docker run -d \
--restart=always \
--name project1.ru \
-p 127.0.0.1:9000:9000 \
-v /www:/www \
-v /docker_conf/etc/ssmtp/ssmtp.conf:/etc/ssmtp/ssmtp.conf \
php-le

```

#### Тестовый скриптик для отправки почты
```
<?php

$to      = 'mymail@gmail.com';
$subject = 'the subject';
$message = 'hello4';
$headers = 'From: admin2@site.ru' . "\r\n" .
    'X-Mailer: PHP/' . phpversion();

mail($to, $subject, $message, $headers);
```

## Про даты
Обычно PHP берет из локали название месяца и оно там в именительном падеже
> 01 август 2021

Но нам нужно вот так

> 01 августа 2021

В данном контейнере скомпилированы локали с патчами на русский язык.
Проверить можно примерно вот так:

```php
if(setlocale(LC_ALL, 'ru_RU.UTF-8','Russian_Russia.65001')===false) 
        exit('not find UTF-8 LOCALE');
if(setlocale(LC_NUMERIC, 'en_US.UTF-8', 'C.UTF-8','C')===false) 
        exit('not find C NUMERIC LOCALE');

// 01 августа 2021
echo strftime('%d %B %Y', time());
```

