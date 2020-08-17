### Docker aarch64 (arm64, armhf) LAMP-T stack (Lighttpd, Alpine, MariaDB, PHP, Tor)

Updated and ripped from:
* https://github.com/luvres/armhf
* https://github.com/foertel/rpi-alpine-tor

**USING: alpine:latest and tor-0.4.4.4-rc**

The following instructions assume that you are cross-compiling on your PC for some ARM box.


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


## Installing and running the packages

SSH into your box.

```
for name in lighttpd mariadb php7 tor; do docker load -i ${name}.bz2; done
```

Please note that the following *"docker run .."* commands are *setup commands*. They instantiate the images into what is called docker containers. If you delete a container and re-instantiate the image, all changes to the container are lost. This is why the -v parameters specify volumes outside the images for data and config files, so that even if you delete a container nothing important is lost.

The containers are linked with a network bridge. Docker creates a host file, so that you can e.g. "ping php-host" or just put "mariadb-host" into your Wordpress setup without managing IP addresses by hand.

If you want to expose the http port from your webserver (e.g. if not using Tor), simply add -p 80:80 (container:host) to the PHP container.

Please do not blindly paste but check and adapt the following sections to your needs.

```
export MYSQL_ROOT_PASS=n00b42cf113b920bfc32c8b025n00b
export UR_HOST_DIR=/storage/

# those directories from your host will be overlayed inside certain docker container directories as volumes
cd $UR_HOST_DIR
mkdir mysql httpd tor

# permissions as they are found inside the containers
chmod 770 mysql httpd 
chmod 755 tor
chown 100:101 ${UR_HOST_DIR_MYSQL}
chown 1000:1000 ${UR_HOST_DIR_HTTPD}

# only needed for hidden service
mkdir tor/hiddenservices
chown 1000:1000 tor/hiddenservices
chmod 700 tor/hiddenservices
echo -e "Log notice stdout \nSocksPort 127.0.0.1:9050 \nHiddenServiceDir /etc/tor/hiddenservices/necro69yiffparty/ \nHiddenServicePort 80 php-host:80"

# initial docker setup commands to create containers
docker run --name MariaDB -h mariadb  -v ${UR_HOST_DIR}/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASS} -d mylocalpkg/arm:mariadb &
docker run --name PHP -h php --link MariaDB:mariadb-host -v ${UR_HOST_DIR}/httpd:/var/www -d mylocalpkg/arm:php7 &
docker run --name TOR -d --link PHP:php-host -v ${UR_HOST_TOR_DIR}/tor:/etc/tor mylocalpkg/arm:tor &
```


## Configuring MariaDB

Bash into the mariadb docker image like this:

```
docker container list
docker container exec -it 3f7URxID7d15 busybox sh
```

Inside the container, go into MariaDB, change the root password and add a user and table for your web stuff:

```
mysql -u root -p

USE mysql;
UPDATE user SET password=PASSWORD('UrPasswordHere') WHERE User='root';

CREATE DATABASE necro69yiffparty;
CREATE USER necro69yiffparty@'%' IDENTIFIED BY 'UrPassword';
GRANT ALL PRIVILEGES ON necro69yiffparty.* TO necro69yiffparty@'%';
```

Now exit the container and do the following, so that you don't accidentally overwrite the root password from the bash history.

```
docker container list -a
# pick the mariadb container id
docker container stop 3f7URxID7d15
docker container rm 3f7URxID7d15
docker run --name MariaDB -h mariadb  -v ${UR_HOST_DIR}/mysql:/var/lib/mysql -d mylocalpkg/arm:mariadb 
```

To make docker containers start at boot, add this to your /etc/rc.local :

```
export UR_HOST_DIR=/storage/
docker run --name MariaDB -h mariadb  -v ${UR_HOST_DIR}/mysql:/var/lib/mysql -d mylocalpkg/arm:mariadb &
docker run --name PHP -h php --link MariaDB:mariadb-host -v ${UR_HOST_DIR}/httpd:/var/www -d mylocalpkg/arm:php7 &
docker run --name TOR -d --link PHP:php-host -v ${UR_HOST_TOR_DIR}/tor:/etc/tor mylocalpkg/arm:tor &

docker restart TOR &
docker restart MariaDB &
docker restart PHP &
```

That is it. You can now use Eschalot to generate a fancy .onion URL, like zz6a1u790iyiff69.onion. 

Don't forget you can simply use "mariadb-host" instead of some IP address from inside the httpd server.

No ports are exposed to the outside.
