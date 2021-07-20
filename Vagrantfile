# -*- mode: ruby -*-

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.ssh.forward_agent = true
  config.vm.synced_folder "#{ENV['HOME']}/Desktop/vbshare", "/vbshare"

  config.vm.define 'rails-env', primary: true do |ubuntu|
    ubuntu.vm.hostname = 'rails-env'
    ubuntu.vm.network :private_network, ip: "192.168.200.2"
    ubuntu.vm.network :forwarded_port, host: 3000, guest: 3000
    ubuntu.vm.provider :virtualbox do |vb|
      vb.gui = false
      vb.memory = 4096
      vb.cpus = 2
    end
    ubuntu.vm.provision :ansible do |ansible|
      ansible.limit = "all"
      ansible.compatibility_mode = "2.0"
      ansible.inventory_path = "hosts.vagrant"
      ansible.playbook = "playbook_vagrant.yml"
    end
  end
end
