# -*- mode: ruby -*-
# vi: set ft=ruby :

unless File.exist?(File.join(__dir__, 'secrets.rb'))
  abort "ERROR: secrets.rb not found. Copy secrets.rb.example to secrets.rb and fill in your values."
end

unless File.exist?(File.join(__dir__, 'config.rb'))
  abort "ERROR: config.rb not found. Copy config.rb.example to config.rb and fill in your values."
end

require_relative 'secrets'
require_relative 'config'

Vagrant.configure("2") do |config|
	
	config.vm.define "soc", primary: true do |soc|
		soc.vm.box = "bento/ubuntu-25.04"
		soc.vm.hostname = "soc-stack"
		soc.vm.boot_timeout = 600
		soc.vm.disk :disk, size: "100GB", primary: true
		
		# Splunk
  		soc.vm.network "forwarded_port", guest: 8000, host: SPLUNK_WEB_PORT
  		# Wazuh Dashboard
  		soc.vm.network "forwarded_port", guest: 443, host: WAZUH_DASHBOARD_PORT
  		# Wazuh API
  		soc.vm.network "forwarded_port", guest: 55000, host: WAZUH_API_PORT
  		# Wazuh Indexer (optional - for debugging)
  		soc.vm.network "forwarded_port", guest: 9200, host: WAZUH_INDEXER_PORT
		# Shuffle web UI (HTTP)
		soc.vm.network "forwarded_port", guest: 3001, host: SHUFFLE_HTTP_PORT
		# Shuffle web UI (HTTPS)
		soc.vm.network "forwarded_port", guest: 3443, host: SHUFFLE_HTTPS_PORT

		soc.vm.network "private_network", ip: SOC_IP
		soc.vm.provider "virtualbox" do |v|
  			v.memory = 16384 # Splunk 8GB + Wazuch single stack installation 8GB as per minimal requiremnts found in documentation
			v.cpus = 4
		end
		soc.vm.provision "docker" do |d|
			d.run "splunk/splunk",
			args: " -p #{SPLUNK_WEB_PORT}:8000 \
					-p #{SPLUNK_RECEIVING_PORT}:9997 \
				    -e SPLUNK_PASSWORD=#{SPLUNK_PASSWORD} \
					-e SPLUNK_START_ARGS=--accept-license \
					-e SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com"
		end
		
		soc.vm.provision "shell", inline: <<-SHELL

			# ============================================================
  			# KERNEL PARAMETERS
  			# ============================================================
			
  			# Required for Wazuh Indexer component.
			# For more information search for Docker host requirements at: https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html#single-node-stack
			echo 'vm.max_map_count=262144' > /etc/sysctl.d/99-wazuh.conf
  			sysctl -w vm.max_map_count=262144

			# Disable swap persistently - required for OpenSearch (Wazuh + Shuffle)
			# https://opensearch.org/docs/latest/install-and-configure/install-opensearch/index/
			sed -i '/swap/s/^/#/' /etc/fstab
			swapoff -a

			# ============================================================
			# DISK EXPANSION
  			# ============================================================

			# Expand partition and LVM to use full disk
  			apt-get install -y cloud-guest-utils
  			growpart /dev/sda 3
  			pvresize /dev/sda3
  			lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
  			resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
			
			# ============================================================
  			# WAZUH SINGLE NODE STACK
  			# ============================================================

			# Install Wazuh single node stack
  			# https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html#single-node-stack
  			cd /home/vagrant
  			git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.4 --depth=1
  			cd wazuh-docker/single-node

  			# Generate certificates - mandatory prerequisite
  			docker compose -f generate-indexer-certs.yml run --rm generator
			
			# Start Wazuh stack
  			docker compose up -d

			# ============================================================
  			# SPLUNK CONFIGURATION
  			# ============================================================

			# Wait for Splunk to be ready before configuring
 			echo "Waiting for Splunk to initialize..."
  			sleep 60

			# Configure Splunk receiving port and wazuh-alerts index
  			docker exec -u splunk splunk-splunk \
    		/opt/splunk/bin/splunk enable listen 9997 \
    		-auth admin:#{SPLUNK_PASSWORD}

			# Create wazuh-alerts index
			docker exec -u splunk splunk-splunk \
    		/opt/splunk/bin/splunk add index wazuh-alerts \
    		-auth admin:#{SPLUNK_PASSWORD}
			
 			# ============================================================
  			# SPLUNK UNIVERSAL FORWARDER
  			# ============================================================

			# Download and install Splunk Universal Forwarder
  			# Note: URL from https://www.splunk.com/en_us/download/universal-forwarder.html
  			wget -O /tmp/splunkforwarder.deb \
    		"https://download.splunk.com/products/universalforwarder/releases/10.2.2/linux/splunkforwarder-10.2.2-80b90d638de6-linux-amd64.deb"
  			dpkg -i /tmp/splunkforwarder.deb

			# Copy forwarder config files from shared folder
  			cp /vagrant/files/splunk-forwarder/inputs.conf \
    		/opt/splunkforwarder/etc/system/local/inputs.conf
  			cp /vagrant/files/splunk-forwarder/props.conf \
    		/opt/splunkforwarder/etc/system/local/props.conf
  			cp /vagrant/files/splunk-forwarder/outputs.conf \
    		/opt/splunkforwarder/etc/system/local/outputs.conf

			# Generate user-seed.conf replacing placeholder with actual password
			sed "s/SPLUNK_PASSWORD_PLACEHOLDER/#{SPLUNK_PASSWORD}/" \
  			/vagrant/files/splunk-forwarder/user-seed.conf.tmpl > \
  			/opt/splunkforwarder/etc/system/local/user-seed.conf

			# Restart forwarder to pick up config changes
			# Works whether it's already running or not
			/opt/splunkforwarder/bin/splunk restart \
  			--accept-license \
  			--answer-yes \
  			--no-prompt \
  			|| /opt/splunkforwarder/bin/splunk start \
  			--accept-license \
  			--answer-yes \
  			--no-prompt

			# ============================================================
  			# SHUFFLE SOAR
  			# ============================================================

  			# Install Shuffle SOAR
			# https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md
			git clone https://github.com/Shuffle/Shuffle /home/vagrant/shuffle
			cd /home/vagrant/shuffle

			# Apply lab-specific patches to official docker-compose.yml
			# Patch 1: fix port conflict with Wazuh OpenSearch on 9200
			sed -i 's/- 9200:9200/- 9201:9200/' docker-compose.yml
			# Patch 2: reduce OpenSearch heap from 3072m to 1024m for lab RAM constraints
			sed -i 's/-Xms3072m -Xmx3072m/-Xms1024m -Xmx1024m/' docker-compose.yml
			# Fix prerequisites as per official install guide
			mkdir -p shuffle-database shuffle-apps shuffle-files
			chown -R 1000:1000 shuffle-database
			swapoff -a

			# Start Shuffle
			docker compose up -d
	  		SHELL
	end   
	
	config.vm.define "win-endpoint" do |win|
		win.vm.box = "filipovo/windows-11-enter-eval-25H2"
		win.vm.hostname = "win-endpoint"
		win.vm.network "private_network", ip: WIN_ENDPOINT_IP
		win.vm.provider "virtualbox" do |v|
  			v.memory = 4196
			v.cpus = 2
		end
		
		win.vm.provision "shell", privileged: true, powershell_elevated_interactive: false, inline: <<-SHELL
    		# Download Wazuh agent MSI
    		# https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html
    		Invoke-WebRequest `
      			-Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.4-1.msi" `
      			-OutFile "$env:tmp\\wazuh-agent.msi"
    		# Install agent with manager configuration
    		Start-Process msiexec.exe -ArgumentList "/i $env:tmp\\wazuh-agent.msi /q WAZUH_MANAGER=#{SOC_IP} WAZUH_AGENT_NAME=win-endpoint" -Wait -NoNewWindow
    		# Wait for installation to complete
    		Start-Sleep -Seconds 10
    		# Start Wazuh service
    		NET START WazuhSvc
  			SHELL
	end

	config.vm.define "linux-endpoint" do |linux|
		linux.vm.box = "bento/ubuntu-25.04"
		linux.vm.hostname = "linux-endpoint"
		linux.vm.network "private_network", ip: LINUX_ENDPOINT_IP
		linux.vm.provider "virtualbox" do |v|
  			v.memory = 2048
			v.cpus = 2
		end

		linux.vm.provision "shell", inline: <<-SHELL
			# Install prerequisites
			sudo apt-get -y install gnupg apt-transport-https
			# Add Wazuh GPG key
			sudo curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
			sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && \
			sudo chmod 644 /usr/share/keyrings/wazuh.gpg
			# Add Wazuh repository
			sudo echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \ 
			sudo tee -a /etc/apt/sources.list.d/wazuh.list
			# Install Wazuh agent
			sudo apt-get update
			sudo apt-get install wazuh-agent
			# Configure manager IP directly in config file
    		# Environment variable method unreliable on Ubuntu 25.04 with v4.14.4
			sudo sed -i 's/MANAGER_IP/#{SOC_IP}/' /var/ossec/etc/ossec.conf
			# Enable and start agent
			sudo systemctl daemon-reload
			sudo systemctl enable wazuh-agent
			sudo systemctl start wazuh-agent
			# Disable repo to prevent unintended upgrades
    		# https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html
    		sudo sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list
    		sudo apt-get update

		SHELL
	
	end

	
end


