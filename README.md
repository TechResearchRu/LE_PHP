# LE_PHP_Docker
Контейнер заточен для LE_Framework, выполняет функционал php-fpm
* php 7.4.22
* json, xml, zip, freetype, curl, bcmatch, ssl и другие полезные модули
* mysqli и sqlite
* поддержка png, jpeg, webp, gif
* поддержка RabbitMQ (amqp)
* пропатченная локаль, месяца с маленькой буквы и в родительном падеже
* отправка почты через ssmtp (транспорт smtp)
* корректная работа iconv на конструкциях //TRASLITE //IGNORE и других, в базовом докере PHP такого нет
* uid пользователя www-data изменен на 33, специально для Debian хостов, но при желании вы можете поменять его на свой, отредактировав Dockerfile. Созданные через php-fpm файлы внутри контейнера будут корректно читаться из папки на хосте.

## Сборка из DockerFile

```bash
mkdir php_le_docker;
cd php_le_docker;
wget https://raw.githubusercontent.com/TechResearchRu/LE_PHP_Docker/main/Docker/Dockerfile
docker build -t php-le .
```

## Запуск FPM

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

## Выполнение CLI скриптов
```bash
docker exec -it project1.ru php -v
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

### Postfix

> Вместо локального Postfix можно прямо к почтовой учетке подключаться типа gmail, yandex etc... Но там будьте готовы к ограничениям, скорей всего придется платный аккаунт для домена покупать или использовать ящик вашего хостера.

1. Настраиваем Postfix на хосте.

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

## NGINX Example
С конфигам пока не было времени вникнуть до конца, вероятно он не очень оптимален, если так, то сообщите мне

```
#project1
server 
{
	listen 80;
	server_name project1.loc;
	root /www/project1.loc/web;
	index index.php;


    location ~* ^/pub/(.+\.(?:gif|jpe?g|png|js|css|woff|ttf|svg|eot|html|htm|txt))$
    {
         alias /mp/v1.8/PUB/$1;
         access_log off;
         expires 10d;
    }


	location ~* ^.+\.(txt|jpe?g|gif|png|ico|css|txt|bmp|rtf|js|svg|eot|ttf|woff|html?)$
	{
         access_log off;
         add_header Cache-Control "public, max-age=31536000, immutable";
	}
	
	#все запросы направить на index.php
	location / {
		rewrite ^/(.*)$ /index.php;
	}


	location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass localhost:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

}
```


## Про даты
Обычно PHP берет из локали название месяца и оно там в именительном падеже
> 01 август 2021

Но нам нужно вот так

> 01 августа 2021

В данном контейнере скомпилированы локали с патчами на русский язык.
Проверить можно примерно вот так:

```php
<?php
if(setlocale(LC_ALL, 'ru_RU.UTF-8','Russian_Russia.65001')===false) 
        exit('not find UTF-8 LOCALE');
if(setlocale(LC_NUMERIC, 'en_US.UTF-8', 'C.UTF-8','C')===false) 
        exit('not find C NUMERIC LOCALE');

// 01 августа 2021
echo strftime('%d %B %Y', time());
```



## Обратная связь
Буду рад любой критике в пользу оптимизации.

Если у вас возникли сложности, то тоже пишите, вероятно помогу.
Моя почта pavelbbb@gmail.com , меня зовут Павел Беляев.

