### Docker Lighttpd+PHP, MariaDB, Tor containers made with ARM boxes in mind (aarch64, arm64, armhf)

**USING: Debian:latest with PHP 7 & Alpine 3.12 for the rest**

**Time required: 20 minutes**

**Fiddle factor: negligible**

You will end up with a working .onion address, where you can host a blog, forum and similar.

You can edit the Dockerfiles for a newer version of Alpine. 3.12 currently is the newest, but there is also alpine:edge which has PHP 7.4. Alpine:latest however is very old and might not even have PHP7.

You can also build this for different architectures like linux/amd64.

The following instructions assume that you are cross-compiling on your Linux PC for some ARM box. I am using a repurposed Android TV Box running LibreELEC, but it should also work without issues on Rasperry Pi, Odroid, Wetek, S805/S812/S905 devices, etc. Maybe you need to use armv7 for older boxes.

Please note that I have not included any of the typical zero-configuration docker setup scripts on purpose. They break easily with future versions, and checking them for loopholes and errors is very tedious to the user. Quite frankly, autosetup scripts are a totally fragile and untransparent mess. That is why most of the other docker images I found have security holes and mostly still use obsolete versions like PHP5 with hardly any new updated versions available.

Note: I changed the PHP container to Debian, because Alpine implements some weirdo version of glob() that doesn't really work but is required by some PHP applications. I should probably switch MariaDB to Debian as well, but I was too lazy for that.

## Build setup on your PC
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
docker buildx build --platform linux/arm64 -t mylocalpkg/arm:tor ./tor
```


## Saving and deploying the packages
 
```
for name in lighttpd mariadb tor; do docker save mylocalpkg/arm:$name | bzip2 > $name.bz2; done
scp lighttpd.bz2 mariadb.bz2 tor.bz2 root@192.168.0.10:/home/mydestination/
```


## Setting up your host system

SSH into your ARM box.

```
for name in lighttpd mariadb tor; do docker load -i ${name}.bz2; done
```

Please adapt the exported variables appropriately and don't switch ssh sessions to not lose them.

```
export UR_HOST_DIR=/storage/
export UR_HIDDEN_SERVICE=necro69yiffparty

# those directories on your host will be overlayed inside corresponding docker container directories as volumes
cd $UR_HOST_DIR
mkdir mysql httpd tor

# permissions as they are found inside the containers
chmod 750 mysql httpd 
chmod 755 tor
chown 100:101 mysql
chown 33:33 httpd

# for Tor 
mkdir tor/hiddenservices
chown 100:0 tor/hiddenservices
chmod 700 tor/hiddenservices
echo -e "Log notice stdout \nSocksPort 127.0.0.1:9050 \nHiddenServiceDir /etc/tor/hiddenservices/${UR_HIDDEN_SERVICE}/ \nHiddenServicePort 80 php-host:80"
```

## Configuring MariaDB on host system

This command bypasses the default startup and drops you into a shell inside the mariadb container.

```
docker run -ti --entrypoint=sh --user 0 --name MariaDB -h mariadb  -v ${UR_HOST_DIR}/mysql:/var/lib/mysql mylocalpkg/arm:mariadb
```

Inside, set up the database with your new root password.

```
mariadb-install-db --user=mysql --datadir=/var/lib/mysql
mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql mysql
su-exec mysql mysqld&


mysql

SET @@SESSION.SQL_LOG_BIN=0;
CREATE USER 'root'@'%' IDENTIFIED BY 'UrRootPassword' ;
GRANT ALL ON *.* TO 'root'@'%' WITH GRANT OPTION ;
DROP DATABASE IF EXISTS test ;

/* at this chance, you can also add a database/user for your webstuff */
CREATE DATABASE necro69yiffparty;
CREATE USER 'necro69yiffparty'@'%' IDENTIFIED BY 'UrOtherPassword';
GRANT ALL PRIVILEGES ON necro69yiffparty.* TO 'necro69yiffparty'@'%';

FLUSH PRIVILEGES ;
```

Exit the container and remove it. 

```
docker container stop MariaDB
docker container rm MariaDB
```


## Setting up docker containers on host system

Please note that the following *"docker run .."* commands are *setup commands*. They instantiate the images into what is called docker containers. If you delete a container and re-instantiate the image, all changes to the container are lost. This is why the -v parameters specify volumes outside the container for data and config files, so that even if you delete a container nothing important is lost.

Docker will keep multiple containers and multiple images loaded, when it is probably not desirable to do so at all. You should therefore always check with "docker [image|container] ls -a" and remove with "docker [image|container] rm [id]" anything old before loading something new.

The containers are linked with a network bridge. Docker creates a host file with the linked containers, so that you can e.g. put "mariadb-host" into your Wordpress setup, instead of specifying some IP address.

If you want to expose the http port from your webserver (e.g. if not using Tor), simply add -p 80:80 (host:container) to the PHP container.


```
export UR_HOST_DIR=/storage/

# initial docker setup commands to create containers
docker run --name MariaDB -h mariadb  -v ${UR_HOST_DIR}/mysql:/var/lib/mysql -d mylocalpkg/arm:mariadb &
docker run --name PHP -h php --link MariaDB:mariadb-host -v ${UR_HOST_DIR}/httpd:/var/www -d mylocalpkg/arm:lighttpd &
docker run --name TOR -d --link PHP:php-host -v ${UR_HOST_TOR_DIR}/tor:/etc/tor mylocalpkg/arm:tor &
```

To make docker containers start at boot, add this to your /etc/rc.local :

```
docker restart TOR &
docker restart MariaDB &
docker restart PHP &
```


## Addendum

Don't forget that you can simply use "mariadb-host" instead of some IP address from inside the httpd server.

You can also view logs:

```
docker container logs PHP
```

Or sh into a running container:

```
docker container exec --user 0 -it MariaDB sh
```

If you want a fancy URL like zz6a1u790iyiff69.onion , you can use [Eschalot](<https://github.com/ReclaimYourPrivacy/eschalot>) .

Simply replace the contents of the auto-generated "hostname" and "private_key" files in ${UR_HOST_DIR}/tor/hiddenservices/${UR_HIDDEN_SERVICE}/ .


```
eschalot -t4 -v -c -r "yiff69$"
```

No ports are exposed to the outside by default. You have to use the .onion URL to access your site.
