# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
  config.vm.box = "generic/ubuntu2004"
  config.vm.box_check_update = false
  config.vm.define "testing"
  config.vm.provider "virtualbox" do |vb|
     vb.gui = false
     vb.memory = "1024"
  config.vm.hostname = "testing"
  config.vm.network "public_network", ip: "192.168.0.196", bridge: 'enp0s3'
  config.vm.network "forwarded_port", guest: 3000, host: 3000, auto_correct: true
  config.vm.network "forwarded_port", guest: 9100, host: 9100, auto_correct: true
  config.vm.network "forwarded_port", guest: 9000, host: 9000, auto_correct: true
  config.vm.network "forwarded_port", guest: 9090, host: 9090, auto_correct: true
  config.ssh.forward_agent = true
  config.ssh.username = 'vagrant'
  config.ssh.password = 'vagrant'
  config.vm.provision "ansible" do |ansible|
    ansible.limit = "all"
    ansible.inventory_path = "hosts.txt"
    ansible.playbook = "ansible_vagrant_playbook.yml"
  end
  end
end
