# docker-workshop

## Docker 简介

## Docker子命令分类


## Set up docker environment

### Install Vagrant

```
$ brew cask install virtualbox
$ brew cask install vagrant
$ brew cask install vagrant-manager

```

Add the Vagrant box you want to use.

```
$ vagrant box add ubuntu/trusty64
```

Now create a test directory and cd into the test directory. Then we'll initialize the vagrant machine.

```
$ vagrant init ubuntu/trusty64
```

Now lets start the machine using the following command.

```
$ vagrant up
```

You can ssh into the machine now.

```
$ vagrant ssh
```

Halt the vagrant machine now.

```
$ vagrant halt
```

[Learn More >>](http://sourabhbajaj.com/mac-setup/Vagrant/README.html)

### Install Docker

#### Uninstall old versions

```
$ sudo apt-get remove docker docker-engine docker.io
```

#### Supported storage drivers

```
$ sudo apt-get update

$ sudo apt-get install \
    linux-image-extra-$(uname -r) \
    linux-image-extra-virtual
```

#### Install Docker CE

##### Install using the repository

```
$ sudo apt-get update

# Install packages to allow apt to use a repository over HTTPS:
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

# Add Docker’s official GPG key:
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Verify that you now have the key with the fingerprint
$ sudo apt-key fingerprint 0EBFCD88

# Use the following command to set up the stable repository
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

##### INSTALL DOCKER CE

```
$ sudo apt-get update

# Install the latest version of Docker CE:
$ sudo apt-get install docker-ce

# Verify that Docker CE is installed correctly by running the hello-world image.
$ sudo docker run hello-world
```

[Learn More >>](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1)

##### Mapping host port to vm port

```
$ vim Vagrantfile

config.vm.network "forwarded_port", guest: 6301, host: 6301
```

#### Pull docker images

Those images will be used in our workshop, because pull images will cost some time, so please pull those images before we start the workshop.

```
$ sudo docker pull ubuntu
$ sudo docker pull django
$ sudo docker pull haproxy
$ sudo docker pull redis

$ sudo docker images
```

#### 启动应用容器栈

```
$ mkdir workspace && cd workspace
$ mkdir redis-master
$ mkdir redis-slave
$ mkdir app
$ mkdir haproxy
```

```
$ sudo docker run -it \
                  --name redis-master \
                  -v ~/workspace/redis-master:/redis-config \
                  redis /bin/bash

$ sudo docker run -it \
                  --name redis-slave1 \
                  --link redis-master:master \
                  -v ~/workspace/redis-slave:/redis-config \
                  redis /bin/bash

$ sudo docker run -it \
                  --name redis-slave2 \
                  --link redis-master:master \
                  -v ~/workspace/redis-slave:/redis-config \
                  redis /bin/bash

$ sudo docker run -it \
                  --name APP1 \
                  --link redis-master:db \
                  -v ~/workspace/app:/usr/src/app \
                  django /bin/bash

$ sudo docker run -it \
                  --name APP2 \
                  --link redis-master:db \
                  -v ~/workspace/app:/usr/src/app \
                  django /bin/bash

$ sudo docker run -it \
                  --name HAProxy \
                  --link APP1:APP1 \
                  --link APP2:APP2 \
                  -p 6301:6301 \
                  -v ~/workspace/haproxy:/tmp \
                  haproxy /bin/bash
```



##### Redis Master容器节点配置

Downlaod redis.conf template, then copy to the volume dir：

```
# In vagrant console
$ wget http://download.redis.io/redis-stable/redis.conf
$ cp redis.conf redis-master
$ cp redis.conf redis-slave
```

Modify the config of redis master

```
# In vagrant console
$ vim redis-master/redis.conf

bind 0.0.0.0
daemonize yes
pidfile /var/run/redis.pid
```

Start redis server:

```
# In redis-master container console
$ cd /usr/local/bin
$ cp /redis-config/redis.conf .
$ redis-server redis.conf
```

##### Redis Slave容器节点配置

Modify the config of redis slave

```
# In vagrant console
$ vim redis-slave/redis.conf

bind 0.0.0.0
daemonize yes
pidfile /var/run/redis.pid

# saveof <masterip> <masterport>
slaveof master 6379
```

Start redis server:

```
# In redis-slave container console
$ cd /usr/local/bin
$ cp /redis-config/redis.conf .
$ redis-server redis.conf
```

The same way to finish the config of another rediss slave.

##### Redis 容器节点测试

First, In redis master container, start redis cli:

```
$ redis-cli

127.0.0.1:6379> set master hello-world
OK
```

Then, check in 2 redis slaves contaner:

```
$ redis-cli

127.0.0.1:6379> get master
"hello-world"
```

##### APP容器节点(Django)配置

Install python redis package

```
$ pip install redis
```

```
$ cd /usr/src/app/
$ make dockerweb && cd dockerweb
$ django-admin.py startproject redisweb
$ cd redisweb
$ ls
manage.py  redisweb
$ python manage.py startapp helloworld
$ ls
helloworld  manage.py  redisweb
```

After create APP in container, switch to the volume dir `~/workspace/app` of the host machine, then edit the configuration of APP:

```
$ cd ~/workspace/app
$ ls
dockerweb
$ cd dockerweb/redisweb/helloworld
$ ls
__init__.py  __pycache__  admin.py  apps.py  migrations  models.py  tests.py  views.py
```

Edit views.py

```
from django.shortcuts import render
from django.http import HttpResponse

import redis

def hello(request):
    str = redis.__file__
    str = str + "<br>"
    r = redis.Redis(host='db', port=6379, db=0)
    info = r.info()
    str = str + ("Set Hi <br>")
    r.set('Hi', 'HelloWorld-APP')
    str = str + ('Get Hi: %s <br>' % r.get('Hi'))
    str = str + ('Redis Info: <br>')
    str = str + ('Key: Info Value')

    for key in info:
        str = str + ("%s: %s <br>" % (key, info[key]))
    return HttpResponse(str)

```

Then, edit the configuration file `setting.py` of redisweb.

```
$ cd ../redisweb/
$ ls
__init__.py  __pycache__  settings.py  urls.py  wsgi.py
```

Add `helloworld` into `INSTALLED_APPS` and Add `*` into `ALLOWED_HOSTS`

```
$ vim settings.py

# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'helloworld',
]

ALLOWED_HOSTS = [
 '*',
]
```

Then, edit urls.py

```
$ vim urls.py

from django.conf.urls import url
from django.contrib import admin
from helloworld.views import hello

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^helloworld$', hello),
]
```

Then, go back to container:

```
python manage.py migrate
```

Then, Start server:

```
python manage.py runserver 0.0.0.0:8001
```

The configuration of another APP `APP2` is same.

##### HAProxy容器节点配置

```
$ cd ~/workspace/haproxy
$ vim haproxy.cfg

global
    log         127.0.0.1 local2
    chroot      /usr/local/sbin
    pidfile     /usr/local/sbin/haproxy.pid
    maxconn     4000
    nbproc 4
    daemon

defaults
    mode        http
    timeout connect 10000 # default 10 second time out if a backend is not found
    timeout client 300000
    timeout server 300000

frontend http-in
   bind 0.0.0.0:6301
   default_backend servers

backend servers
   bind-process 2
   server APP1 APP1:8001 check inter 2000 rise 2 fall 5
   server APP2 APP2:8001 check inter 2000 rise 2 fall 5
```

Start Haproxy in container

```
$ cd /usr/local/sbin/
$ haproxy -c -f /tmp/haproxy.cfg
$ haproxy -f /tmp/haproxy.cfg
```

If you want stop haproxy:

```
$ apt-get install psmisc
$ killall haproxy
```