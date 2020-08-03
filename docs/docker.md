# Docker tips

[How To Remove Docker Containers, Images, Volumes, and Networks
](https://linuxize.com/post/how-to-remove-docker-images-containers-volumes-and-networks/)

#### Remove all stopped containers

Before performing the removal command, you can get a list of all non-running (stopped) containers that will be removed using the following command:

    $ docker container ls -a --filter status=exited --filter status=created 
 
To remove all stopped containers use the docker container prune command:

    $ docker container prune
    
You’ll be prompted to continue, use the -f or --force flag to bypass the prompt.

#### Stop all containers

    $ docker container stop $(docker container ls -aq)
    
#### Remove dangling images

A dangling image is an image that is not tagged and is not used by any container. To remove dangling images type:

    $ docker image prune

## Debug Docker images

### Start/run with a different entry point

Start a stopped Docker container with a different command

    $ docker run -ti --entrypoint=sh telopromo:v0.1

## Docker machine

### Install on macOS 
[ref](https://docs.docker.com/machine/install-machine/)

    base=https://github.com/docker/machine/releases/download/v0.16.0 &&
    curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/usr/local/bin/docker-machine &&
    chmod +x /usr/local/bin/docker-machine
    
Test

    $ docker-machine version
    docker-machine version 0.16.0, build 702c267f

It's good idea to also install `docker-machine` completion scripts

    base=https://raw.githubusercontent.com/docker/machine/v0.16.0
    for i in docker-machine-prompt.bash docker-machine-wrapper.bash docker-machine.bash
    do
       sudo wget "$base/contrib/completion/bash/${i}" -P /etc/bash_completion.d
    done

## Docker Engine

[ref](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

If you want to create your own Docker server

    # apt-get remove docker docker-engine docker.io containerd runc
    # apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
    # curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    # apt-key fingerprint 0EBFCD88
    # add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    # apt-get update
    # apt-get install docker-ce docker-ce-cli containerd.io
    # apt-cache madison docker-ce
    # docker run hello-world
    # groupadd docker
    # usermod -aG docker pablito
    # su - pablito
    # systemctl enable docker
    # chkconfig docker on
    
### DNS

[ref](https://development.robinwinslow.uk/2016/06/23/fix-docker-networking-dns/)


You need to change the DNS settings of the Docker daemon. You can set the default options for the docker daemon by creating a daemon configuration file at `/etc/docker/daemon.json`.

You should create this file with the following contents to set two DNS, firstly your network’s DNS server, and secondly the Google DNS server to fall back to in case that server isn’t available:

/etc/docker/daemon.json:

    {
        "dns": ["10.0.0.2", "8.8.8.8"]
    }


### Deployment

on chofero user env

-- local machine

    $ docker save -o image.zip chofero-docker
    $ scp image.zip chofero@beta.stupidfriendly.com:~/incoming

-- on server

    $ docker load -i ~/incoming/image.zip

Another option to deployment is to setup `docker host` 

* [ref](https://www.digitalocean.com/community/tutorials/how-to-use-a-remote-docker-server-to-speed-up-your-workflow)
* [How To Set Up a Private Docker Registry on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04)

```
$ export DOCKER_HOST=ssh://chofero@beta.stupidfriendly.com
$ docker build --rm -f Dockerfile -t chofero:latest .
```

And with [watchtower](https://hub.docker.com/r/v2tec/watchtower/) the container will update automatically with the new image.


#### Setting the network

Create a docker network, Every container on that network will be able to communicate with each other using the container name as hostname.

    $ docker network create -d bridge chofero-net

#### Database

1. Create a data directory on the host system, e.g. `/home/chofero/data`
2. Start your mysql container like this (mysql v5.7): 


    $ docker run --name chofero-mysql -v /home/chofero/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d --network chofero-net --restart unless-stopped mysql:5.7
    
Run commands inside container

    $ docker exec -it chofero-mysql bash
    
Create database

    mysql> CREATE DATABASE IF NOT EXISTS `chofero` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

Create local user

    mysql> CREATE USER 'chofero'@'localhost' IDENTIFIED WITH mysql_native_password AS '***';
    mysql> GRANT USAGE ON *.* TO 'chofero'@'localhost';
    mysql> GRANT ALL PRIVILEGES ON `chofero`.* TO 'chofero'@'localhost';
    mysql> GRANT ALL PRIVILEGES ON `chofero\_%`.* TO 'chofero'@'localhost';

Create remote user, try to limit access only to your app host

    mysql> CREATE USER 'chofero'@'%' IDENTIFIED WITH mysql_native_password AS '***';
    mysql> GRANT USAGE ON *.* TO 'chofero'@'%';
    mysql> GRANT ALL PRIVILEGES ON `chofero`.* TO 'chofero'@'%';
    mysql> GRANT ALL PRIVILEGES ON `chofero\_%`.* TO 'chofero'@'%';

##### Importing data

    $ docker exec -i chofero-mysql mysql -uroot -psecret chofero < chofero.sql

or

    $ docker exec -i chofero-mysql mysql -uroot -p"$CHOFERO_PASS" chofero < chofero.sql

#### SSL on nginx

ref: https://medium.com/faun/setting-up-ssl-certificates-for-nginx-in-docker-environ-e7eec5ebb418

Copy private key and crt to `/etc/ssl/chofero`

#### certbot with nginx

certbot is a tool for handlig free SSL certificates. ref: https://absolutecommerce.co.uk/blog/auto-renew-letsencrypt-nginx-certbot

    # apk add certbot certbot-nginx
    
Run certbox for nginx configuration

    # certbot --nginx
    
Configuration generated by certbot `/etc/nginx/conf.d/nginx.conf`

```
server {
    listen 443 ssl;
    server_name stage.chofero.com;
    ssl_certificate /etc/letsencrypt/live/stage.chofero.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/stage.chofero.com/privkey.pem; # managed by Certbot

    location / {
        include uwsgi_params;
        uwsgi_param SCRIPT_NAME '';
        uwsgi_pass localhost:9000;
        # uwsgi_pass 172.17.0.2:9000;
    }

}
server {
    if ($host = stage.chofero.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    server_name stage.chofero.com;
    return 404; # managed by Certbot


}
```
    
    

#### Run the application

With new image, this will create a new container

    $ docker run -d -p 80:80 -p 443:443 --name chofero-run -v /etc/ssl/chofero:/etc/ssl/chofero --network chofero-net --restart unless-stopped chofero
    
Stop and Delete previous container

    $ docker stop chofero-run
    $ docker rm chofero-run 
        
With existing container

    $ docker container start chofero-run
    
View logs

    $ docker logs chofero-run

#### PHPMyAdmin

    $ docker pull phpmyadmin/phpmyadmin:latest
    
    $ docker run --name chofero-phpmyadmin -e PMA_HOST=chofero-mysql -d --network chofero-net --restart unless-stopped -p 8081:80 phpmyadmin/phpmyadmin
    
#### Watchtower

    $ docker run -d --name watchtower -v /var/run/docker.sock:/var/run/docker.sock --restart unless-stopped --no-pull containrrr/watchtower 
    
