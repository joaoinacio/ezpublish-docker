# eZ Publish5 in Docker

eZ Publish Inception, installed in docker containers, inside a virtual machine (configurable in VagrantFile).
Project is work in progress!

## Project goal

Cover the following use cases (with time), including (in prioritized order):

1.  Single server images, for use by:
 1.  Sprint / Release Demo (PS, Sales, PM)
 2.  Testing (QA, Support, Dev)
 3.  Production (internal Ops, but potentially also partners)
2.  Cluster support (Memcached, Varnish4, DFS)
3.  Add additional containers for the different platforms we support to be able to automate Behat (BDD) testing across
    a wide range of platforms and combinations.
4. (outside scope of this Repository) Investigate use of Meos++ for use with massive clustering


### Spec

#### "Host" (VM Guest when using Vagrant)

Host machine is running CoreOS allowing us to not have to maintain the host (autoupdate), have a light host OS,
and allowing us to take advantage of its clustering capabilities in the future.

eZ Publish is placed in .vagrant/ezpublish as mounted by Vagrant on boot, this is used as volume for all
containers that need access to it.

    Port (Listen): 80
    Software: CoreOs

#### Configuration

In the beginning VagrantFile will be used, and config is up to user to adjust eZ Publish install and VagrantFile.

However eventually the following needs to be configured in a "Docker way"(TM):
- Container linking (typically fig or Meos for cluster setup or manually using Weave)
- Configuring of eZ Publish (typically using Ansible)
    - database (example: mysql/postgres/oracle/..)
    - cache (example: memcached/file/redis/..)
    - http cache (example: varnish4/nginx/file/..)
    - search (example: solr/es/db/..)
    - io (example: file/nfs/s3/..)


#### Docker Images

As much as possible we would like to reuse existing images out there by contributing features to them.
A lot of existing images exists for ubuntu and debian.

##### Images varying by Base

Base images are provided by Docker, and the once we would like to use are currently (prioritized):
- `centos:centos7`
- `ubuntu:trusty`
- `debian:wheezy` (for later, might make sense ot wait for `debian:jessie`)
- `opensuse:latest` (for later, if/when we re add suse to supported platforms)

For test coverage of our different supported platforms we would thus like to eventually have the following images
either from docker hub or home grown implemented using all the listed base images above:

###### DB

    Port (Listen): <depends on server>
    CMD: start db service
    Software: mysql/mariadb, postgres, ..
    On run:
        Should 1. create database and give rights to a user, 2. accept existing database dump


###### HTTPD

    Port (Listen): 80
    CMD: start httpd service
    Software: Nginx, Apache
    Volume: .vagrant/ezpublish mounted to for instance /var/www
    Configuration:
        FastCGI on port 9000
        VirtualHost config with <volume-mount-dir>/web as configured as web root (at docker run before starting server?)


###### PHP

Image to use both as php-fpm (fast cgi) container, as cronjob container running php scripts and can be used
for starting own instance on demand for running own php commands against the eZ install.

    Port (Listen): 9000
    Software: php-fpm, php-cli
    Volume: .vagrant/ezpublish mounted to for instance /var/www
    Configuration:
        Configure php-fpm to execute on <volume-mount-dir>/web.

Should probably be a dev version extending this adding all kind of debug things like xdebug and modifying settings.


##### Other images

List of images that does not need to be strictly tested on the misc distros.
TBD but if we continue with a Reference platform these should use CentOS, if not we are free to choose.

###### Memcached
###### Varnish v4
###### Solr
###### Elastic Search
###### NFS


### Docker images

These should be refactored in the future to be something like the following list of containers:
- http server with fastcgi
- php fpm for serving php
- crontab container with php cli
- Database: Mysql|Postgres|..
- Clustering:
 - Cache: Memcached|Redis|..
 - HTTP cache: Varnish|Nginx|..
 - FS: NFS|GridFS|..  (services like S3 might not need a container, but could have a container acting as proxy)

**Note** We should take advantage of official images from Docker as much as possible, however they now use Debian
as it takes less space then Ubuntu, and for lowest space use (and memory?) base images should be the same across
all our official images.

And development mode or not should probably rather be a global parameter then special images.

However right now following images exists:

## Installation

NB: This section reflects current status with images not reflecting spec above!

- Ensure you have the following tools installed on our computer:
 - Vagrant 1.6+ (http://vagrantup.com)
 - VirtualBox 4.3.12+ (http://www.virtualbox.org)
- Put your eZ Publish directly inside "ezpublish/" directory, or symlink it there (overwriting the folder)
- TODO: It is currently not supported to provide database dump, only clean install is currently supported!
- Copy files/vagrant.yml-EXAMPLE to files/vagrant.yml. Then adjust settings in .yml file as needed
- Run `vagrant up`

When this is done you should be able to browse to eZ Publish setup wizard by going to http://localhost/:8080

If you later want to do changes to your docker/vagrant files, you need to stop and remove the corresponding container ```docker stop [containerid]```, remove the image ```docker rmi [imageid]``` and then run ```vagrant provision``` instead of ```vagrant up```

#### SSH

##### VM

To enter virtual machine:
- ```vagrant ssh```

From there you can check running containers:
- ```docker ps```

And inspect the eZ Publish folder which was rsynced into the vm and is used as volume for eZ Publish container:
- ```ls -al /vagrant/ezpublish/```


##### Running php-cli and mysql commands

To run php/mysql commands you'll need to start a new container which contains php-cli:
- ```vagrant ssh```
- ```docker run --rm -i -t --link db-1:db --dns 8.8.8.8 --dns 8.8.4.4 -v '/vagrant/ezpublish/:/var/www:rw' ezsystems/php-cli /bin/bash```

From there you can run symfony commands like normal:
- ```php ezpublish/console ezpublish:legacy:assets_install --symlink --relative --env dev``

You can also access mysql from this container as it has the mysql client installed:
- ```mysql -uadmin --password=[mysqlpasswd] --protocol=tcp --host=db```

Mysql password is defined in files/vagrant.yml

( For other environment variables see ```env```, basically these typically comes from parent images and links )

To get out, type ```exit``` two times ;)


##### The containers

Once you have the system up, doing a ```docker ps -a``` will reveal that the following containers:
 - web-nginx
 - php-fpm
 - db-1
 - db-vol

The db-vol container is stopped ( meaning no processes are running in it ). This is correct. This container is only a data volume container for the mysql raw db files.
The content of this data volume container is mapped to /vagrant/volumes/mysql/ on VM. 
If you want to reset the mysql databases, you'll need to stop and remove the db-1 container, remove all files in volumes/mysql and make sure that is synced to VM ( /vagrant/volumes/mysql ), then recreate db-1 container


##### Running vagrant from windows

It is possible to run this from Windows. However, it is not possible to use synced_folder with type virtualbox ( seems to be some incompatibility between virtualbox and docker's provision plugin in Vagrant or between CoreOS and Vagrant ? )
So, you need to use rsync on Windows too. In order to do this, you need rsync and ssh.
Easiest way to accomplish this is to install MinGW ( minimalist GNU for Windows ), http://sourceforge.net/projects/mingw/files/MSYS/Extension/rsync/rsync-3.0.8-1/
Download wingw-get-setup.exe and install openssh and rsync. You should then add "C:\MinGW\msys\1.0\bin" to your path and you should be all set to run "vagrant up"