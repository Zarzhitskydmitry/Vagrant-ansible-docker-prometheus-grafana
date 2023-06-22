# -*- mode: ruby -*-
# vi: set ft=ruby :

# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
  config.vm.box = "generic/ubuntu2004"
  config.vm.box_check_update = false
  config.vm.hostname = "testing"
  config.ssh.username="ubuntudoc"
  config.ssh.password="Ntvysqktc1723"
  config.vm.network "public_network", ip: "192.168.0.197"
  config.vm.define "testing"
  config.vm.provision "shell", inline: "cat id_rsa.pub >> .ssh/authorized_keys"
  config.vm.provider "virtualbox" do |vb|
     vb.gui = false
     vb.memory = "1024"
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible_vagrant_playbook.yml"
  end
  end
end
