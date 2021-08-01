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

Продакшн режим примерно такой
```bash
docker run -d \
--restart=always \
--name project1.ru \
-p 127.0.0.1:9000:9000 \
-v /www:/www \
php-le
```

## Настройка почты
