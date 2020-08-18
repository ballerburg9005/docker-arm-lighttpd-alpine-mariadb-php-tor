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

RUN sed -i "/.*skip-networking.*/d" /etc/my.cnf
RUN sed -i "/.*skip-networking.*/d" /etc/my.cnf.d/mariadb-server.cnf
RUN sed -i "/.*bind-address.*/d" /etc/my.cnf.d/mariadb-server.cnf
RUN sed -i "s/^\[mysqld\].*$/[mysqld]\nbind-address=0.0.0.0/g" /etc/my.cnf

EXPOSE 3306

USER mysql
CMD ["mysqld"]