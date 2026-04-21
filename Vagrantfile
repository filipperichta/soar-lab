# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
	
	config.vm.define "splunk", primary: true do |splunk|
		splunk.vm.box = "bento/ubuntu-25.04"
		splunk.vm.network "private_network", ip: "192.168.33.10"

		splunk.vm.provider "virtualbox" do |vb|
  			vb.memory = "8192"
		end

		splunk.vm.provision "docker" do |d|
			d.run "splunk/splunk"
		end
 	 	# splunk.vm.provision "shell", inline: <<-SHELL
 	 	#   apt-get update
	  	#   apt-get install -y apache2
	  	# SHELL
	end   
	

	config.vm.define "wazuch" do |wazuch|
		wazuch.vm.box = "bento/ubuntu-25.04"
		wazuch.vm.network "private_network", ip: "192.168.33.11"

		wazuch.vm.provider "virtualbox" do |vb|
  			vb.memory = "8192"
		end

		#wazuch.vm.provision "docker" do |d|
		#	d.run "wazuch/wazuch"
		#end
 	 	# wazuch.vm.provision "shell", inline: <<-SHELL
 	 	#   apt-get update
	  	#   apt-get install -y apache2
	  	# SHELL
	end   


config.vm.define "dockerreg" do |dockerreg|
		dockerreg.vm.box = "bento/ubuntu-25.04"
		dockerreg.vm.network "private_network", ip: "192.168.33.12"

		dockerreg.vm.provider "virtualbox" do |vb|
  			vb.memory = "4196"
		end

		dockerreg.vm.provision "docker" do |d|
			d.run "registry"
		end
 	 	# docker.vm.provision "shell", inline: <<-SHELL
 	 	#   apt-get update
	  	#   apt-get install -y apache2
	  	# SHELL
	end   



	
end
