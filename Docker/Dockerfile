FROM alpine:3.14
ENV PHP_INI_DIR /usr/local/etc/php
ENV PHP_URL="https://www.php.net/distributions/php-7.4.22.tar.xz"
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -pie"

RUN export CFLAGS="$PHP_CFLAGS" CPPFLAGS="$PHP_CPPFLAGS" LDFLAGS="$PHP_LDFLAGS";
#RUN addgroup -g 82 -S www-data; 
RUN adduser -u 82 -D -S -G www-data www-data;
#
#need packages
RUN apk add --no-cache ca-certificates curl tar xz openssl rabbitmq-c icu libintl ssmtp;
#
# MAKE locales
ENV MUSL_LOCPATH="/usr/share/i18n/locales/musl"
RUN apk --no-cache --virtual .locale_build add cmake make musl-dev gcc gettext-dev git && cd /tmp/ && \
	git clone https://gitlab.com/rilian-la-te/musl-locales && \
	cd musl-locales && \
sed -i 's!Январь!января!g; s|Февраль|февраля|g;\
s|Март|марта|g; s|Апрель|апреля|g;\
s|Май|мая|g; s|Июнь|июня|g;\
s|Июль|июля|g; s|Август|августа|g;\
s|Сентябрь|сентября|g; s|Октябрь|октября|g;\
s|Ноябрь|ноября|g; s|Декабрь|декабря|g;' musl-po/ru_RU.po && \
sed -i 's!Янв!янв!g; s|Фев|фев|g;\
s|Мар|мар|g; s|Апр|апр|g;\
s|Май|мая|g; s|Июн|июн|g;\
s|Июл|июл|g; s|Авг|авг|g;\
s|Сен|сен|g; s|Окт|окт|g;\
s|Ноя|ноя|g; s|Дек|дек|g;' musl-po/ru_RU.po && \
cmake -DLOCALE_PROFILE=OFF -DCMAKE_INSTALL_PREFIX:PATH=/usr . && \
make && make install && \
cd .. && rm -r musl-locales && apk del .locale_build; \
export LC_ALL=ru_RU.UTF-8

ENV LANG=ru_RU.UTF-8 LANGUAGE=ru_RU.UTF-8


###dev packages add
RUN apk add --no-cache --virtual .build-deps \
autoconf dpkg-dev dpkg file g++ gcc libc-dev make \
pkgconf re2c argon2-dev coreutils curl-dev \
libedit-dev libsodium-dev libxml2-dev linux-headers \
oniguruma-dev openssl-dev sqlite-dev krb5-dev imap-dev \
libpng-dev libwebp-dev libjpeg-turbo-dev freetype-dev \
rabbitmq-c-dev icu-dev libtool; \
#
#Make iconv 1.16
cd /tmp; \
rm /usr/bin/iconv \
&& curl -SL http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.16.tar.gz | tar -xz -C . \
&& cd libiconv-1.16 \
&& ./configure --prefix=/usr/local \
&& make && make install \
&& libtool --finish /usr/local/lib; \
cd ../; rm -r libiconv-1.16;\
#
#php make
mkdir -p "$PHP_INI_DIR/conf.d" /var/www/html /usr/src/php; \
chown www-data:www-data /var/www/html; chmod 777 /var/www/html;\
cd /usr/src; \
curl -fSL -o php.tar.xz "$PHP_URL"; \
#распакуем
tar -Jxvf /usr/src/php.tar.xz -C /usr/src/php --strip-components=1;\
rm php.tar.xz; \
#
#компилируем
cd /usr/src/php; \
gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
./configure --build="$gnuArch" --with-config-file-path="$PHP_INI_DIR" --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" --enable-option-checking=fatal --with-mhash --enable-ftp --enable-mbstring --enable-mysqlnd --with-password-argon2 --with-pdo-sqlite=/usr --with-sqlite3=/usr --with-curl --with-libedit --with-openssl --with-zlib --with-pear --enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data --disable-cgi --with-kerberos --enable-bcmath --enable-calendar --with-imap --with-mysqli --enable-gd --with-webp --with-jpeg --with-freetype --with-imap-ssl  --enable-intl --with-iconv=/usr/local; \
make -j 4; \
find -type f -name '*.a' -delete; \
make install; \
find /usr/local/bin /usr/local/sbin -type f -perm +0111 -exec strip --strip-all '{}' + || true; \
make clean; \
cp -v php.ini-* "$PHP_INI_DIR/"; \
cd /; \
#
#install need libs
runDeps="$( \
scanelf --needed --nobanner --format '%n#p' --recursive /usr/local | tr ',' '\n' | sort -u \
| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }')"; \
apk add --no-cache $runDeps; \
#
#configuration
cd /usr/local/etc; \
cp php-fpm.conf.default php-fpm.conf; \
cp php-fpm.d/www.conf.default php-fpm.d/www.conf; \
sed -i 's!=NONE/!=!g' php-fpm.conf;\
{ \
echo '[global]'; \
echo 'error_log = /proc/self/fd/2'; \
echo 'log_limit = 8192'; \
echo '[www]'; \
echo 'access.log = /proc/self/fd/2'; \
echo 'clear_env = no'; \
echo 'catch_workers_output = yes'; \
echo 'decorate_workers_output = no'; \
} | tee php-fpm.d/docker.conf; \
\
{ \
echo '[global]'; \
echo 'daemonize = no'; \
echo '[www]'; \
echo 'listen = 9000'; \
} | tee php-fpm.d/zz-docker.conf; \
#
{ \
echo 'memory_limit=1024M'; \
echo 'upload_max_filesize=1024M'; \
echo 'post_max_size=1024M'; \
echo 'intl.default_locale = ru_RU.UTF-8'; \
} | tee /usr/local/etc/php/php.ini; \
#amqp install
pecl update-channels; \
echo "autodetect"|pecl install amqp; \
echo "zend_extension=opcache.so" > /usr/local/etc/php/conf.d/opcache.ini; \
echo "extension=amqp.so" > /usr/local/etc/php/conf.d/amqp.ini; \
#clean
rm -rf /tmp/pear ~/.pearrc /usr/src/php; \
apk del --no-network .build-deps; \
#without debug and cli
rm /usr/local/bin/phpdbg;
#rm /usr/local/bin/php;




STOPSIGNAL SIGQUIT

EXPOSE 9000
CMD ["php-fpm"]
