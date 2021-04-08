# -*- mode: ruby -*-
# vim: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.box_check_update = true
 #config.vbguest.auto_update = false
    
	
  config.vm.define "router01" do |router01|
    router01.vm.hostname = 'router01'
	router01.vm.network "private_network", ip: "10.10.10.1", netmask: "255.255.255.0" , adapter: 2,  virtualbox__intnet: "router01"
    router01.vm.network "private_network", ip: "172.20.0.1", netmask: "255.255.255.252" , adapter: 3,  virtualbox__intnet: "router01router02"
	router01.vm.network "private_network", ip: "192.168.100.1", netmask: "255.255.255.252" , adapter: 4,  virtualbox__intnet: "router01router03"
	router01.vm.provider :virtualbox do |v|
		v.name = "router01"
    end

	router01.vm.provision "ansible" do |ansible|
        ansible.playbook = "router01.yml"
	end
	
  end
  
  config.vm.define "router02", primary: true do |router02|
    router02.vm.hostname = 'router02'
	router02.vm.network "private_network", ip: "10.10.20.1", netmask: "255.255.255.0" , adapter: 2,  virtualbox__intnet: "router02"
    router02.vm.network "private_network", ip: "172.20.0.2", netmask: "255.255.255.252" , adapter: 3,  virtualbox__intnet: "router01router02"
	router02.vm.network "private_network", ip: "192.168.0.1", netmask: "255.255.255.252" , adapter: 4,  virtualbox__intnet: "router02router03"
	router02.vm.provider :virtualbox do |v|
		v.name = "router02"

    end

	router02.vm.provision "ansible" do |ansible|
        ansible.playbook = "router02.yml"
	end
	
  end

  config.vm.define "router03" do |router03|
    router03.vm.hostname = 'router03'
	router03.vm.network "private_network", ip: "10.10.30.1", netmask: "255.255.255.0" , adapter: 2,  virtualbox__intnet: "router03"
    router03.vm.network "private_network", ip: "192.168.100.2", netmask: "255.255.255.252" , adapter: 3,  virtualbox__intnet: "router01router03"
	router03.vm.network "private_network", ip: "192.168.0.2", netmask: "255.255.255.252" , adapter: 4,  virtualbox__intnet: "router02router03"
	router03.vm.provider :virtualbox do |v|
		v.name = "router03"
    end

	router03.vm.provision "ansible" do |ansible|
        ansible.playbook = "router03.yml"
	end
	
  end
  
end

