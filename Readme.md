# Heroku buildpack: fakesu

Heroku buildpack that allows regular users to run commands as (fake)root, creating a chroot jail.


## Usage

    $ ls
    Gemfile      Gemfile.lock web.rb

    $ heroku create -s cedar -b http://github.com/fabiokung/heroku-buildpack-fakesu.git

    $ git push heroku master
    wait...

    $ heroku run bash

    (dyno)$ id
    uid=28171(u28171) gid=28171

    (dyno)$ fakesu

    (dyno)# id
    uid=0(root) gid=0(root)

    (dyno)# whoami
    root

    (dyno)# ls ~
    Gemfile  Gemfile.lock  bin  web.rb


### Default process type (console)

    $ heroku run console

    (dyno)# whoami
    root

### -c option for custom commands as (fake)root

    $ heroku run bash

    (dyno)$ fakesu -c id
    uid=0(root) gid=0(root)

    (dyno)$ fakesu -c touch /etc/file-created-as-root

    (dyno)$ fakesu -c ls -l /etc/file-created-as-root
    -rw------- 1 root root 0 Jan 30 09:50 /etc/file-created-as-root


## Why would I need such a crazy thing?

### Because I want to build and run my own system packages

e.g.: a different version of ruby:

    $ heroku run console

    (dyno)# cd /tmp

    (dyno)# curl -O http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.2-pXXX.tar.gz

    (dyno)# tar zxvf ruby-1.9.2-pXXX.tar.gz

    (dyno)# cd ruby-1.9.2-pXXX

    (dyno)# ./configure --prefix=/usr/local optflags="-O3" debugflags="-g3 -ggdb"
    some rubies are failing to compile (segfault) with gcc >= 4.4
    if it is your case, use CC=gcc-4.3 ./configure ...

    (dyno)# make

    (dyno)# make install


### Because I have system dependencies (e.g. libzmq-dev)

    # cat Gemfile
    source :rubygems
    gem "zmq"
    ...

    # echo "deb http://ppa.launchpad.net/chris-lea/libpgm/ubuntu lucid main" >> /etc/apt/sources.list

    # echo "deb http://ppa.launchpad.net/chris-lea/zeromq/ubuntu lucid main" >> /etc/apt/sources.list

    # apt-get update

    # apt-get install libzmq-dbg libzmq-dev libzmq1

    # bundle install

In this case, your application processes (Procfile) would need to be started inside the chroot jail, where the system dependencies were installed.

`fakesu -c` can be used for that, e.g.: `fakesu -c bundle exec thin -p $PORT ...`


### Because I want to be the (fake)root of my dyno :)


### (bonus) Because I want to install and run **services**

    $ heroku run console

    (dyno)# apt-get install nginx

    (dyno)# env | grep PORT
    PORT=59222

    (dyno)# vim /etc/nginx/sites-available/default
    edit the "listen 80" line and change it to the port given by the previous command

    (dyno)# vim /etc/init.d/nginx restart


Using another terminal, it is now possible to [create routes](https://github.com/JacobVorreuter/heroku-routing):

    $ heroku routes:create
    Creating route... done
    tcp://route.heroku.com:28392

    $ heroku routes:attach tcp://route.heroku.com:28392 run.1

    $ open http://route.heroku.com:28392

