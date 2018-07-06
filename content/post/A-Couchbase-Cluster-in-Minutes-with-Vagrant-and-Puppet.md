+++
date = "2013-05-27"
draft = false
title = "A Couchbase Cluster in Minutes with Vagrant and Puppet"
description = "This post shows you how to easily create a multi node cluster with Couchbase installed, using Vagrant and Puppet!"
tags = ["couchbase","vagrant","puppet"]
slug = "A-Couchbase-Cluster-in-Minutes-with-Vagrant-and-Puppet"
+++

## Motivation
Since I work as part of the engineering team at Couchbase, I need to run my code against a variety of server deployments. We run a multitude of operating systems and software versions, and so do our customers. In order to fix bugs reliably and build new features, it is critical to get a cluster up and running that resembles these deployments as good as possible. I know that I can run all of these combinations on EC2, but the cost for this would be very high and most of the time its overkill.

What I need is to get such a cluster up and running in minutes and not spending too much time on configuring it. I heard about [Vagrant](http://www.vagrantup.com/) and [Puppet](https://puppetlabs.com/) in the past, but never got around to use them on my own box (though I always use [VirtualBox](https://www.virtualbox.org/) on MacOS to create virtual machines by hand).

This morning I sat down to take a closer look on how these tools can help me to get more productive - and to my huge surprise I got a 4 node Couchbase Server cluster running in less than 30 minutes (with looking up all the configuration details). Since its so easy, I want to share it with you.

## Prerequisites
Before we can provision our nodes, you need to make sure to have Vagrant and VirtualBox installed. If you use MacOS like me, just download the `.dmg` files for both and you're set. Now, create a directory somewhere to store the configuration files - I called mine 'vagrants'.

In this directory, you need to create a `Vagrantfile`. Its like the Vagrants `makefile` and it will pick it up to learn how you want to have your nodes provisioned. Note that this doesn't configure the software on top of the OS (like installing Couchbase), this is handled by puppet in a separate step. Here is the full config:

	Vagrant.configure("2") do |config|

		# Number of nodes to provision
		numNodes = 4

		# IP Address Base for private network
		ipAddrPrefix = "192.168.56.10"

		# Define Number of RAM for each node
		config.vm.provider "virtualbox" do |v|
			v.customize ["modifyvm", :id, "--memory", 1024]
		end

		# Provision the server itself with puppet
		config.vm.provision :puppet

		# Download the initial box from this url
		config.vm.box_url = "http://files.vagrantup.com/precise64.box"

		# Provision Config for each of the nodes
		1.upto(numNodes) do |num|
			nodeName = ("node" + num.to_s).to_sym
			config.vm.define nodeName do |node|
				node.vm.box = "precise64"
				node.vm.network :private_network, ip: ipAddrPrefix + num.to_s
				node.vm.provider "virtualbox" do |v|
					v.name = "Couchbase Server Node " + num.to_s
				end
			end
		end

	end

This file is just ruby code that configures Vagrant. Let's go through each directive and see what it does for us.

	# Number of nodes to provision
	numNodes = 4

	# IP Address Base for private network
	ipAddrPrefix = "192.168.56.10"

You can change these values, I just created them to fit my environment here. Depending on the amount of `numNodes` set, VMs will be created. I added a loop down below depending on this setting, so I don't have to duplicate code a lot. The ip address prefix is used to easily determine the (static) IP address for the server. The numbers will be counted upwards incrementally, so you will end up with four servers accessible through `192.168.56.101` to `192.168.56.104`.

	# Define Number of RAM for each node
	config.vm.provider "virtualbox" do |v|
		v.customize ["modifyvm", :id, "--memory", 1024]
	end

This config block is needed to increase the memory size of the VM. By default its less than that (I believe around 512MB), and I want to have 1 gig of RAM for each. Of course, feel free to tune that value or remove it completely.

	# Provision the server itself with puppet
	config.vm.provision :puppet

Because we'll be using puppet to provision the server software, we need to tell Vagrant to use it.

	# Download the initial box from this url
	config.vm.box_url = "http://files.vagrantup.com/precise64.box"

Vagrant reuses predefined images so you don't have to reinstall everything from scratch. Here we use a predefined Ubuntu 12.04 64bit box.

	# Provision Config for each of the nodes
	1.upto(numNodes) do |num|
		nodeName = ("node" + num.to_s).to_sym
		config.vm.define nodeName do |node|
			node.vm.box = "precise64"
			node.vm.network :private_network, ip: ipAddrPrefix + num.to_s
			node.vm.provider "virtualbox" do |v|
				v.name = "Couchbase Server Node " + num.to_s
			end
		end
	end

This code block configures each virtual machine. Given the number of nodes we want to create, for each of them it assigns an IP address and gives it a descriptive name inside Virtualbox. If you want to add server-dependent settings, the "node" block is the right place for it. Otherwise it will pick the cluster wide settings defined in the "config" block.

Now if we would run `vagrant up` from the command line in this directory, we'd get four Ubuntu machines setup where we could SSH into, but nothing else would be installed. In order to make them do something, we want to install Couchbase Server. Puppet is a system automation software and very good at provisioning systems. Vagrant has amazing support for it, all we need to is create a `default.pp` file inside a `manifests` directory that looks like this:

	exec { "couchbase-server-source": 
		command => "/usr/bin/wget http://packages.couchbase.com/releases/2.0.1/couchbase-server-enterprise_x86_64_2.0.1.deb",
		cwd => "/home/vagrant/",
		creates => "/home/vagrant/couchbase-server-enterprise_x86_64_2.0.1.deb",
		before => Package['couchbase-server']
	}

	exec { "install-deps":
		command => "/usr/bin/apt-get install libssl0.9.8",
		before => Package['couchbase-server']
	}

	package { "couchbase-server":
		provider => dpkg,
		ensure => installed,
		source => "/home/vagrant/couchbase-server-enterprise_x86_64_2.0.1.deb"
	}

Let's go over the internals once more.

	exec { "couchbase-server-source": 
		command => "/usr/bin/wget http://packages.couchbase.com/releases/2.0.1/couchbase-server-enterprise_x86_64_2.0.1.deb",
		cwd => "/home/vagrant/",
		creates => "/home/vagrant/couchbase-server-enterprise_x86_64_2.0.1.deb",
		before => Package['couchbase-server']
	}

In puppet, we define some tasks that we want to run. This task executes a shell command `wget` and stores the file inside the home directory of the user. We tell puppet to download the debian package of the server. Note that there is a `before` dependency to the package installation task, because we can't install it before the file wasn't downloaded.

	exec { "install-deps":
		command => "/usr/bin/apt-get install libssl0.9.8",
		before => Package['couchbase-server']
	}

We also need to install `libssl0.9.8` on the server, this is the only dependency it has. We use the command line tool `apt-get` for this.

	package { "couchbase-server":
		provider => dpkg,
		ensure => installed,
		source => "/home/vagrant/couchbase-server-enterprise_x86_64_2.0.1.deb"
	}

Finally, we can install the debian package from couchbase-server, because the file is in place and all dependencies are satisfied.

Of course, this puppet file is very simple and I'm you can do much more with it (and maybe even simplify it more) - but for my needs it is more than enough. If I want a different server version, I just need to change the puppet file and point it to the new debian package.

Now if we run `vagrant up` again, much more happens. Note that if you want to play with your puppet files, you can also use `vagrant provision` to apply the changes while the node is running.

If everything is okay, the output should look like this:

	Bringing machine 'node1' up with 'virtualbox' provider...
	Bringing machine 'node2' up with 'virtualbox' provider...
	Bringing machine 'node3' up with 'virtualbox' provider...
	Bringing machine 'node4' up with 'virtualbox' provider...
	[node1] Clearing any previously set forwarded ports...
	[node1] Creating shared folders metadata...
	[node1] Clearing any previously set network interfaces...
	[node1] Preparing network interfaces based on configuration...
	[node1] Forwarding ports...
	[node1] -- 22 => 2222 (adapter 1)
	[node1] Running any VM customizations...
	[node1] Booting VM...
	[node1] Waiting for VM to boot. This can take a few minutes.
	[node1] VM booted and ready for use!
	[node1] Configuring and enabling network interfaces...
	[node1] Mounting shared folders...
	[node1] -- /vagrant
	[node1] -- /tmp/vagrant-puppet/manifests
	[node1] Running provisioner: puppet...
	Running Puppet with default.pp...
	stdin: is not a tty
	notice: /Stage[main]//Exec[install-deps]/returns: executed successfully
	notice: Finished catalog run in 0.77 seconds
	.... more for all the other nodes.

You can then point your browser to `192.168.56.10[1-4]` and work with your Couchbase cluster. If you are done with it, you can use the `vagrant halt` command to shut it down cleanly. Very handy is also `vagrant suspend`, which will save the state of the nodes instead of shutting them down completely.

If you want to interact with one of the nodes instead of the whole cluster, you can always specify the node identifier. For example, if you want to start only the first node you can do it with the `vagrant up node1` command.

To me, this is a very fast and clean way to provision server nodes. I just need to change a few lines in a file and get a new cluster without much hassle. Even more important, I can put those config files in version control and [share them](https://github.com/daschl/vagrants) with other folks!