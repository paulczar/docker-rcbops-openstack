# -*- mode: ruby -*-
# vi: set ft=ruby :

require "vagrant"

if Vagrant::VERSION < "1.2.1"
  raise "Use a newer version of Vagrant (1.2.1+)"
end

##### You Can Modify the BOX_NAME/URI to use a different OS #####

BOX_NAME = ENV['BOX_NAME'] || "precise64"
BOX_URI = ENV['BOX_URI'] || "https://opscode-vm.s3.amazonaws.com/vagrant/boxes/opscode-ubuntu-12.04.box"

# We'll mount the Chef::Config[:file_cache_path] so it persists between
# Vagrant VMs
host_cache_path = File.expand_path("../.cache", __FILE__)
guest_cache_path = "/tmp/vagrant-cache"

#### set the Env Variable to 'NO' to disable berkshelf for non-chef servers.
#### Used to allow chef client provisioner without berkshelf getting in the way.
ALLOW_BERKS = ENV['ALLOW_BERKS'] || true

Vagrant.configure("2") do |config|

  # Map .chef dir to /root/.chef to help knife etc.
  config.vm.synced_folder ".chef", "/root/.chef"
  config.vm.synced_folder ".berkshelf", "/root/.berkshelf"

  # Enable Vagrant Cachier for faster build times
  config.cache.auto_detect = true

  # Enable the berkshelf-vagrant plugin
  config.berkshelf.enabled = false
  if ALLOW_BERKS == true
    config.berkshelf.enabled = true
    # The path to the Berksfile to use with Vagrant Berkshelf
    config.berkshelf.berksfile_path = "./Berksfile-vagrant"
  end

  # Ensure Chef is installed for provisioning
  config.omnibus.chef_version = :latest



##### Docker All IN One Node ######
  config.vm.define :docker do |config|
    config.vm.hostname = "docker"
    config.vm.box = BOX_NAME
    config.vm.box_url = BOX_URI
    config.vm.network :private_network, ip: "33.33.33.50"
    config.vm.network :private_network, ip: "172.16.101.60"
    config.ssh.max_tries = 40
    config.ssh.timeout   = 120
    config.ssh.forward_agent = true

    config.vm.provision :chef_solo do |chef|
      chef.provisioning_path = guest_cache_path
      chef.json = {
        "chef-server" => {
            "version" => :latest
        },
        "languages" => {
          "ruby" => {
            "default_version" => "1.9.1"
          }
        }
      }
      chef.run_list = [
        "recipe[apt::default]",
        "recipe[ruby::default]",
        "recipe[build-essential::default]",
        "recipe[git::default]",
        "recipe[curl::default]"
      ]
    end

    config.vm.provision :shell, :inline => <<-SCRIPT
      echo lol angry red text for no reason.
      apt-get -y install libxslt-dev libxml2-dev # stupid Nokogiri!
      gem install chef-zero --no-ri --no-rdo
      /vagrant/start_chef_zero.sh
      gem install chef --no-ri --no-rdo
      gem install spiceweasel --no-ri --no-rdo
      mkdir -p /vagrant/.chef
      chown vagrant /vagrant/.chef/*
      echo "Chef server installed!!\nNow let us slurp up the cookbooks.\nPlease wait a few moments while spiceweasel gets it shit together..."
      cd /vagrant
      spiceweasel --execute /vagrant/infrastructure.yml
      cd /vagrant/nodes/; for i in $(ls *.json); do knife node from file $i; done
      ifconfig eth2 promisc
      apt-get -y links
      mkdir -p /etc/chef
      cp /vagrant/.chef/chef-validator.pem /etc/chef/validation.pem
      cp /vagrant/.chef/client.rb /etc/chef/client.rb
      chef-client
      echo "LMAO if you don't docker on openstack in vagrant"
      cd /opt
      git clone https://github.com/paulczar/openstack-docker.git
      cd openstack-docker
      sh setup_on_rcbops-openstack.sh
      echo "restart all the services for shits n giggles..."
      cd /etc/init.d/; for i in $(ls nova-*); do service $i restart; done
      service glance-registry restart
      sleep 10
      sudo nova-manage service list
      echo "##################################"
      echo "#     Openstack Installed        #"
      echo "#   visit https://33.33.33.50    #"
      echo "#   default username: admin      #"
      echo "#   default password: docker!    #"
      echo "##################################"
    SCRIPT
    config.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--cpus", 2]
      vb.customize ["modifyvm", :id, "--memory", 2048] 
      vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
    end
  end
end