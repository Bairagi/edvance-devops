edvance-devops
==============

### Description

  Hello, this repository contains brief overview of automating infrastructure for testing purposes. The biggest advantage is we don't have any external hardware dependency and automation to larger extent.
For now we use [Ubuntu 14.04](http://releases.ubuntu.com/14.04/) as host OS.

### Install LXC
  Install lxc and other packages to run small containers.
```bash
sudo apt-get update
sudo apt-get install lxc lxc-dev lxc-templates systemd-services uidmap python3-lxc
```

There are some configurations needed to run LXC in user-space (without root/ sudo).
You will need kernel 3.13+ to use this feature and executable permission for your home directory.
[Create lxc containers in user-space](https://www.stgraber.org/2014/01/17/lxc-1-0-unprivileged-containers/)


### Automate LXC using Ruby

  This will need ruby installed with various development libraries. For this tutorial I have used ruby 1.9.3p547

Create your first LXC container using command
```bash
lxc-create -t download -n ubuntu_1404 -- -d ubuntu -r trusty -a amd64
```

And same thing using ruby:

```ruby
require 'lxc'

ct = LXC::Container.new('tester')
ct.create('download',nil, nil, nil, %w{-r trusty -d ubuntu -a amd64})

ct.start(daemonize: true )
```
This creates new LXC container named 'tester' of Ubuntu 14.04

The basic LXC template OS may need few tools to be installed for bootstrap. Therefore we install these packages using lxc-attach using ruby.

```ruby
require 'lxc'

ct = LXC::Container.new('tester')
ct.create('download',nil, nil, nil, %w{-r trusty -d ubuntu -a amd64})

ct.start(daemonize: true )
ct.attach(wait: true) do
  LXC.run_command('apt-get update')
  LXC.run_command('apt-get install curl wget -y')
end
```

### Automate chef-zero using Ruby

  For chef we are using chef-zero to use our cookbooks, roles and environments etc.
I am using lxc bridged interface (lxcbr0) for running chef-zero server, this way all LXC containers can access uploaded chef data from [chef-zero](https://github.com/opscode/chef-zero).

```ruby
require 'chef_zero/server'
server = ChefZero::Server.new(host: '10.0.3.1', port: 4000, wait: true, log_level: 'debug')
server.start_background
```

### Automate cookbook upload using Ruby

  We are adding chef configuration parameters for chef-zero, similar to knife.rb and upload relevant data to chef-zero.

```ruby
require 'lxc'
require 'chef/knife/bootstrap'
require 'chef/knife/cookbook_upload'
require 'chef_zero/server'

Chef::Knife::CookbookUpload.load_deps
Chef::Knife::Bootstrap.load_deps

server = ChefZero::Server.new(host: '10.0.3.1', port: 4000, wait: true, log_level: 'debug')
server.start_background
puts server.running?

Chef::Config[:chef_server_url] = 'http://10.0.3.1:4000'
Chef::Config[:node_name] = 'trusty_chef'
Chef::Config[:client_key]  = '.chef/chef-zero.pem'
Chef::Config[:validation_key]  = '.chef/chef-validator.pem'

def knife_cookbook_upload(cookbook_name)
  cookbook = Chef::Knife::CookbookUpload.new
  cookbook.config[:cookbook_path] = "cookbooks"
  cookbook.name_args = cookbook_name
  cookbook.run
end

knife_cookbook_upload(['mysql-server','nginx','nagios'])
```

### Automate bootstrap of LXC containers using Ruby

  In final run we are going to create and bootstrap three containers. This describes prototype of real life scenario of our environments like web-app, DB and monitoring servers.

```ruby
require 'lxc'
require 'chef/knife/bootstrap'
require 'chef/knife/cookbook_upload'
require 'chef_zero/server'

Chef::Knife::CookbookUpload.load_deps
Chef::Knife::Bootstrap.load_deps

server = ChefZero::Server.new(host: '10.0.3.1', port: 4000, wait: true, log_level: 'debug')
server.start_background
puts server.running?

Chef::Config[:chef_server_url] = 'http://10.0.3.1:4000'
Chef::Config[:node_name] = 'trusty_chef'
Chef::Config[:client_key]  = '.chef/chef-zero.pem'
Chef::Config[:validation_key]  = '.chef/chef-validator.pem'

db = LXC::Container.new('mysql_01')
web-app = LXC::Container.new('web-app_01')
monitoring = LXC::Container.new('nagios_central')

%w{ db web-app monitoring }.each do |ct|
  ct.create('download',nil, nil, nil, %w{-r trusty -d ubuntu -a amd64})
  ct.start(daemonize: true )
  ct.attach(wait: true) do
    p `apt-get install wget curl -y`
  end
end

def knife_cookbook_upload(cookbook_name)
  cookbook = Chef::Knife::CookbookUpload.new
  cookbook.config[:cookbook_path] = "cookbooks"
  cookbook.name_args = cookbook_name
  cookbook.run
end

def knife_bootstrap(container, run_list)
  bootstrap = Chef::Knife::Bootstrap.new
  bootstrap.name_args = container
  bootstrap.config[:ssh_user] = 'ubuntu'
  bootstrap.config[:ssh_password] = 'ubuntu'
  bootstrap.config[:use_sudo] = true
  bootstrap.config[:use_sudo_password] = true
  bootstrap.config[:template_file] = false
  bootstrap.config[:run_list] = run_list
  bootstrap.config[:distro] = 'chef-full' 
  bootstrap.run
end

knife_cookbook_upload(['base','mysql-server','nginx','nagios'])
knife_bootstrap([db.ipaddress], ['base','mysql-server'])
knife_bootstrap([web-app.ipaddress], ['base','nginx'])
knife_bootstrap([monitoring.ipaddress], ['base','nagios'])
```

This will bootstrap all the three containers with specified run list.

### Feedback
It will be my pleasure to receive your feedback on above tutorial. All kinds of issues, suggestions are welcome.
For any queries email(nileshbairagi@gmail.com).

