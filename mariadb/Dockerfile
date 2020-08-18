FROM mylocalpkg/alpine

RUN apk add --no-cache \
    bash \
    pwgen \
    su-exec \
    tzdata \
    mysql \
    mysql-client \
    && mkdir /run/mysqld \
    && chmod 777 /run/mysqld

EXPOSE 3306

USER mysql
CMD ["mysqld"]

