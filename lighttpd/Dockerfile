FROM debian:latest

RUN apt-get update && apt-get -y install lighttpd php-cgi php-common php-iconv php-json php-gd php-curl php-xml php-pgsql php-imap php-cgi php-fpm php-soap php-xmlrpc php-posix php-gettext php-ldap php-ctype php-dom php-mysqli php-tokenizer php-mbstring php-imagick php-zip php-fileinfo php-exif \
	certbot cron \
	&& apt-get clean \
	&& rm -rf /var/lib/apt/lists/*

RUN	sed -i "s/^upload_max_filesize.*=.*/upload_max_filesize = 128M/" /etc/php/7.3/cgi/php.ini \
	&& sed -i "s/^max_execution_time.*=.*/max_execution_time = 300/" /etc/php/7.3/cgi/php.ini \
	&& sed -i "s/^post_max_size.*=.*/post_max_size = 138M/" /etc/php/7.3/cgi/php.ini \
	&& sed -i "s/^max_input_time.*=.*/max_input_time = 120/" /etc/php/7.3/cgi/php.ini \
	&& sed -i "s/^memory_limit.*=.*/memory_limit = 256M/" /etc/php/7.3/cgi/php.ini \
	&& mkdir -p /run/lighttpd/ /run/php/ /var/www/default/ \
	&& chown www-data. /run/lighttpd/ /run/php/ /var/www/default/ \
	&& ( lighttpd-enable-mod accesslog fastcgi fastcgi-php rewrite || true ) \
	&& mkdir /etc/lighttpd/certs \
	&& openssl genrsa -passout pass:client11 -des3 -out /etc/lighttpd/certs/example.com.key 2048 \ 
	&& openssl rsa -in /etc/lighttpd/certs/example.com.key -passin pass:client11 -out /etc/lighttpd/certs/example.com.key \
	&& openssl req -new -key /etc/lighttpd/certs/example.com.key -subj req -new -passin pass:client11 -out /etc/lighttpd/certs/example.com.csr -subj "/C=US/ST=City/L=Country/O=EN/OU=EN/CN=EN/emailAddress=test@example.com" \
	&& openssl x509 -req -days 10000 -in /etc/lighttpd/certs/example.com.csr -signkey /etc/lighttpd/certs/example.com.key -out /etc/lighttpd/certs/example.com.crt \
	&& cat /etc/lighttpd/certs/example.com.key /etc/lighttpd/certs/example.com.crt > /etc/lighttpd/certs/example.com.pem

ADD lighttpd.conf /etc/lighttpd/lighttpd.conf

VOLUME /var/www

EXPOSE 80

CMD php-fpm7.3 -D && ( echo "" > /etc/crontab_real && cat /etc/cron.d/* > /etc/crontab_real || true) && crontab /etc/crontab_real && cron && lighttpd -D -f /etc/lighttpd/lighttpd.conf
#ENTRYPOINT ["/usr/sbin/lighttpd", "-D", "-f", "/etc/lighttpd/lighttpd.conf"]
