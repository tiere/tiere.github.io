---
layout: post
title: "Ruby on Rails Server With nginx, Phusion Passenger and PostgreSQL"
date: 2014-03-07 16:36:02 +0200
comments: true
categories: [Ruby, Rails, nginx, Passenger, PostgreSQL, server]
---

## Introduction
Instructions for setting up Ubuntu 12.04.4 LTS to host a Ruby on Rails Application with nginx as the web server, Phusion Passenger as the application server and PostgreSQL as the database. This has been tested on a clean installation of Ubuntu 12.04.4, but should work on existing installation and different versions with little changes as well. This is one of my first blog posts so take that into account when giving feedback.

## Installation
Write the following inside a script file and run it (Or run the commands one by one). It will add the Phusion Passenger repositories to apt sources, install it, install rbenv, ruby and bundler, stuff required to compile Ruby, PostgreSQL and nginx. Feel free to customize the script to your needs. If you prefer RVM to rbenv just change the corresponding lines etc.:

``` bash
#!/usr/bin/env bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 561F9B9CAC40B2F7
echo "deb https://oss-binaries.phusionpassenger.com/apt/passenger precise main" | sudo tee -a /etc/apt/sources.list.d/passenger.list
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates
sudo apt-get install -y git build-essential nodejs postgresql postgresql-server-dev-9.1 libssl-dev nginx-extras passenger libcurl4-openssl-dev
git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
source ~/.bash_profile
rbenv install -v 2.1.1 # replace with the ruby version that you want to use
rbenv global 2.1.1 # replace with the ruby version that you want to use
gem install -V bundler
```

## Configuration

Change the following lines in /etc/nginx/nginx.conf:

```
    # Phusion Passenger config
    ##
    # Uncomment it if you installed passenger or passenger-enterprise
    ##

    # passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
    # passenger_ruby /usr/bin/ruby;

    ##
    # Virtual Host Configs
    ##
```



to:

```
    # Phusion Passenger config
    ##
    # Uncomment it if you installed passenger or passenger-enterprise
    ##

    passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
    passenger_ruby /<path/to/where/you/installed/ruby>/bin/ruby;

    #Add the following line if you don't want passenger to display friendly debug information when your application fails to start:
    passenger_friendly_error_pages off;

    ##
    # Virtual Host Configs
    ##
```
(Default ruby installation directory is /home/\<username\>/.rbenv/versions/<ruby-version>)

Generate a ssh keypair and add your public key to where you deploy from (If you need to and use git with ssh). Otherwise just do whatever you need to get your application to the server. Guide to ssh key generation can be found at <a href="https://help.github.com/articles/generating-ssh-keys" title="https://help.github.com/articles/generating-ssh-keys" target="_blank">https://help.github.com/articles/generating-ssh-keys</a>

Clone your repo to a directory of your choosing (If you use git):

```
    $ git clone <your-git-repo-address>
```

Add an user to postgresql with the name and password you specified in the production database config in your database.yml:

```
    $ sudo -u postgres psql
```

```
    create user <username>  with password '<password>';
```

```
    alter role <username> createdb;
```

To exit psql prompt, type:

```
    \q
```

Change the local authentication line in /etc/postgresql/9.1/main/pg_hba.conf:

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
```



to:

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
```

to allow password authentication to postgresql


And then restart postgresql:
```
    $ sudo service postgresql restart
```

source ~/.bash_profile to get access to rbenv and new commands of the gems that were installed:
```
    $ source ~/.bash_profile
```
Go to your rails application root and run bundle install:
```
    $ bundle install --without development test
```
Make sure your secret token is specified in your application's config/application.yml if you're using the <a href="https://github.com/laserlemon/figaro" title="https://github.com/laserlemon/figaro" target="_blank">figaro</a> gem. If not, you can generate a new one with the following command in your rails apps's root:
```
    $ rake secret
```
Then just add that to \<your/rails/application/root\>/config/application.yml:

```
SECRET_TOKEN: <generated-token-here>
```

If you are using ssl, now is the time to import your certificates or generate them. The following configurations assume they are stored in /etc/nginx/ssl/

Create a file to /etc/nginx/sites-enabled/ with the name of the name of your application and add the following contents Replace leading whitespaces with tabs if nginx complains about them:

```
server {
        listen 80;
        server_name <your-domain-here>;
        root <your/rails/application/root>/public;
        passenger_enabled on;
}

# Comment this out if you are using SSL
# server {
        # listen 443 ssl;
        # server_name <your-domain-here>;
        # passenger_enabled on;
        # root <your/rails/application/root>/public;
        # ssl_certificate /etc/nginx/ssl/server.crt; # Replace with the path to your certificate
        # ssl_certificate_key /etc/nginx/ssl/server.key; # Replace with the path to your key
# }
```



Then modify /etc/nginx/nginx.conf to only include that site configuration:

```
        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/<the-server-conf-file-you-created>;
}
```
Go to your rails application root and run the following commands:
```
    $ RAILS_ENV=production rake db:setup assets:precompile
```
Then restart nginx:
```
    $ sudo service nginx restart
```
Your application should now be served properly by nginx on port 80 :) If you run into problems, check the logs at /var/log/nginx to see whether they help identify the problem.