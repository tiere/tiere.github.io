---
layout: post
title: "Vagrant for Ruby on Rails Development on Ubuntu 13.10"
date: 2014-03-03 15:12:17 +0200
comments: true
categories: [Rails, Ruby, Vagrant]
---

## Installing

Vagrant can be installed on Ubuntu via apt-get, but the version is really old. I downloaded the .deb -file from http://www.vagrantup.com/downloads.html.

## Configuring

First create a folder for your project and go to that folder. The following command will initialize a new vagrant virtual machine and use Ubuntu Precise as the base:
    $ vagrant init precise32 http://files.vagrantup.com/precise32.box
Vagrant will also create a Vagrantfile in the working directory. Open the file for editing and the change the following line:

``` ruby
  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network :forwarded_port, guest: 80, host: 8080
```

to

``` ruby
  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  config.vm.network :forwarded_port, guest: 3000, host: 3000
```

to forward the default Rails development port to the virtual machine.
Now create two script files to the same directory with Vagrantfile. We will include all software installation in those scripts. You can name the scripts as you like, the names that I use are just examples:
    $ nano installation.sh
I have put the following commands into the first scripts that will be run as root. Feel free to customize these commands to your liking. Node.js is included because its needed for some javascript operations:

``` bash
#!/usr/bin/env bash
echo -e "deb mirror://mirrors.ubuntu.com/mirrors.txt precise main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-updates main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-backports main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-security main restricted universe multiverse\n$(cat /etc/apt/sources.list)" > /etc/sources.list
apt-get update
apt-get install -y git build-essential nodejs libsqlite3-dev
```

After this create another script file to be run as normal user. The script will first install rbenv for easy switching between different Ruby versions. Then it will install a version of Ruby, in my case 2.1.0. It will then set that version of Ruby to be the global default. Finally it will install Rails and Bundler:
    $ nano installation_user.sh

``` bash
#!/usr/bin/env bash
git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
source ~/.bash_profile
rbenv install -v 2.1.0 # replace with the ruby version that you want to use
rbenv global 2.1.0 # replace with the ruby version that you want to use
gem install rails
gem install bundler
```

After this add the following lines to Vagrantfile to include your new shell scripts:

``` ruby
  config.vm.provision :shell, path: "installation.sh"
  config.vm.provision :shell, path: "installation_user.sh", privileged: false
```

After this the configuration for your Ruby on Rails development environment should be configured and you can start the Vagrant box with the following command:
    $ vagrant up
The startup will take some time the first time because it will install the whole system like you previously configured. After the process has finished you can ssh to your new virtual machine with the command:
    $ vagrant ssh
The directory where you most likely will want to initialize your Rails project(s) inside the virtual machine is:
    /vagrant/
Since it will be shared between your guest and host machine. You should now be set up to start developing your application.