FROM debian:latest

RUN apt-get update && apt-get -y install lighttpd php-cgi php-common php-iconv php-json php-gd php-curl php-xml php-pgsql php-imap php-cgi php-fpm php-soap php-xmlrpc php-posix php-gettext php-ldap php-ctype php-dom php-mysqli php-tokenizer php-mbstring php-imagick php-zip php-fileinfo php-exif \
	&& apt-get clean \
	&& rm -rf /var/lib/apt/lists/* 

RUN	sed -i "s/^upload_max_filesize.*=.*/upload_max_filesize = 128M/" /etc/php/7.3/cgi/php.ini \
	&& sed -i "s/^max_execution_time.*=.*/max_execution_time = 300/" /etc/php/7.3/cgi/php.ini \
	&& sed -i "s/^post_max_size.*=.*/post_max_size = 138M/" /etc/php/7.3/cgi/php.ini \
	&& sed -i "s/^max_input_time.*=.*/max_input_time = 120/" /etc/php/7.3/cgi/php.ini \
	&& sed -i "s/^memory_limit.*=.*/memory_limit = 256M/" /etc/php/7.3/cgi/php.ini \
	&& mkdir -p /run/lighttpd/ /run/php/ /var/www/default/ \
	&& chown www-data. /run/lighttpd/ /run/php/ /var/www/default/ \
	&& ( lighttpd-enable-mod accesslog fastcgi fastcgi-php rewrite || true )

ADD lighttpd.conf /etc/lighttpd/lighttpd.conf

VOLUME /var/www

EXPOSE 80

CMD php-fpm7.3 -D && lighttpd -D -f /etc/lighttpd/lighttpd.conf
#ENTRYPOINT ["/usr/sbin/lighttpd", "-D", "-f", "/etc/lighttpd/lighttpd.conf"]
