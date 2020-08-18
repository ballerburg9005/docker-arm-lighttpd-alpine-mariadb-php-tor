### Docker aarch64 (arm64, armhf) LAMP-T stack (Lighttpd, Alpine, MariaDB, PHP, Tor)

Check out https://github.com/luvres/armhf for more stuff. Its dated though, but can be easily adopted into this project folder.

**USING: Alpine 3.12 with PHP 7.3**

If you want to use newer versions, download a newer Alpine miniroot and simply change the Dockerfile in ./alpine/ appropriately. Drop a PR for that if you want.

The following instructions assume that you are cross-compiling on your PC for some ARM box. I am using a repurposed Android TV Box running LibreELEC, but it should also work without issues on Rasperry Pi, Odroid, Wetek, S805/S812/S905 devices, etc.

## Docker setup
* Use root account for all commands
* Install the following packages in your distribution: docker, qemu-user-static


## Enable buildx for docker
```
 if [[  -a ~/.docker/config.json ]]; then echo "\n\nplease add it by hand\!"; else  mkdir ~/.docker/ >& /dev/null; echo '{"experimental": "enabled"}' > ~/.docker/config.json; fi
```


## Sanity check on docker builder

Check for builders and delete them.
```
docker buildx ls
docker buildx rm default
docker buildx rm somebuilder

# run this several times if the next step doesn't work out.
systemctl restart docker
docker run --rm --privileged multiarch/qemu-user-static:register --reset
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

Now it should look like this:
```
NAME/NODE DRIVER/ENDPOINT STATUS  PLATFORMS
default * docker                  
  default default         running linux/amd64, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
```

Make sure it says "default * docker", that means that you are using the "docker" driver and not Buildkit, so that local packages can be accessed by other local packages.
Make sure it says linux/arm64 obviously.


## Building the packages
``` 
docker buildx build --platform linux/arm64 -t mylocalpkg/alpine ./alpine
docker buildx build --platform linux/arm64 -t mylocalpkg/arm:lighttpd ./lighttpd
docker buildx build --platform linux/arm64 -t mylocalpkg/arm:mariadb ./mariadb
docker buildx build --platform linux/arm64 -t mylocalpkg/arm:php7 ./php7
docker buildx build --platform linux/arm64 -t mylocalpkg/arm:tor ./tor
```


## Saving and deploying the packages
 
```
for name in lighttpd mariadb php7 tor; do docker save mylocalpkg/arm:$name | bzip2 > $name.bz2; done
scp lighttpd.bz2 mariadb.bz2 php7.bz2 tor.bz2 root@192.168.0.10:/home/mydestination/
```


## Setting up your host system

SSH into your ARM box.

```
for name in lighttpd mariadb php7 tor; do docker load -i ${name}.bz2; done
```

Please do not blindly paste but check and adapt the following sections to your needs.

```
export UR_HOST_DIR=/storage/

# those directories from your host will be overlayed inside certain docker container directories as volumes
cd $UR_HOST_DIR
mkdir mysql httpd tor

# permissions as they are found inside the containers
chmod 750 mysql httpd 
chmod 755 tor
chown 100:101 ${UR_HOST_DIR_MYSQL}
chown 1000:1000 ${UR_HOST_DIR_HTTPD}

# for Tor 
mkdir tor/hiddenservices
chown 100:0 tor/hiddenservices
chmod 700 tor/hiddenservices
echo -e "Log notice stdout \nSocksPort 127.0.0.1:9050 \nHiddenServiceDir /etc/tor/hiddenservices/necro69yiffparty/ \nHiddenServicePort 80 php-host:80"
```

## Configuring MariaDB

This command bypasses the default startup and drops you into a shell inside the mariadb container.

```
export UR_HOST_DIR=/storage/

docker run -ti --entrypoint=sh --user 0 --name MariaDB -h mariadb  -v ${UR_HOST_DIR}/mysql:/var/lib/mysql mylocalpkg/arm:mariadb
```

Inside, set up the database with your new root password.

```
mariadb-install-db --user=mysql --datadir=/var/lib/mysql
su-exec mysql mysqld&


mysql

SET @@SESSION.SQL_LOG_BIN=0;
CREATE USER 'root'@'%' IDENTIFIED BY 'UrRootPassword' ;
GRANT ALL ON *.* TO 'root'@'%' WITH GRANT OPTION ;
DROP DATABASE IF EXISTS test ;

/* at this chance, you can also add a database/user for your webstuff */
CREATE DATABASE necro69yiffparty;
CREATE USER necro69yiffparty@'%' IDENTIFIED BY 'UrOtherPassword';
GRANT ALL PRIVILEGES ON necro69yiffparty.* TO necro69yiffparty@'%';

FLUSH PRIVILEGES ;
```

Exit the container and remove it. 

```
docker container stop MariaDB
docker container rm MariaDB
```

## Setting up docker containers

Please note that the following *"docker run .."* commands are *setup commands*. They instantiate the images into what is called docker containers. If you delete a container and re-instantiate the image, all changes to the container are lost. This is why the -v parameters specify volumes outside the images for data and config files, so that even if you delete a container nothing important is lost.

The containers are linked with a network bridge. Docker creates a host file, so that you can e.g. "ping php-host" or just put "mariadb-host" into your Wordpress setup without managing IP addresses by hand.

If you want to expose the http port from your webserver (e.g. if not using Tor), simply add -p 80:80 (container:host) to the PHP container.

```
export UR_HOST_DIR=/storage/

# initial docker setup commands to create containers
docker run --name MariaDB -h mariadb  -v ${UR_HOST_DIR}/mysql:/var/lib/mysql -d mylocalpkg/arm:mariadb &
docker run --name PHP -h php --link MariaDB:mariadb-host -v ${UR_HOST_DIR}/httpd:/var/www -d mylocalpkg/arm:php7 &
docker run --name TOR -d --link PHP:php-host -v ${UR_HOST_TOR_DIR}/tor:/etc/tor mylocalpkg/arm:tor &
```

To make docker containers start at boot, add this to your /etc/rc.local :

```
docker restart TOR &
docker restart MariaDB &
docker restart PHP &
```

FYI, you can sh into any running container like this:

```
docker container exec -it MariaDB busybox sh
```

You should now find an auto-generated "hostname" and "private_key" file in ${UR_HOST_TOR_DIR}/hiddenservices/necro69yiffparty/ . You can replace them with fancy ones generated by Eschalot (like zz6a1u790iyiff69.onion ).

Don't forget you can simply use "mariadb-host" instead of some IP address from inside the httpd server.

No ports are exposed to the outside by default. You have to use the .onion URL to access your site.
