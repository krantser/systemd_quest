# -*- mode: ruby -*-
# vim: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "systemdquest", primary: true do |systemdquest|
    systemdquest.vm.box = "centos/7"
    systemdquest.vm.box_version = "2004.01"
    systemdquest.vm.host_name = "systemdquest-machine"
    systemdquest.vm.provider "virtualbox" do |v|
        v.memory = "512"
        v.cpus = "2"
    end
    systemdquest.vm.network "public_network", bridge: "wlp2s0", ip: "192.168.1.102"
    systemdquest.vm.provision "shell", inline: <<-SHELL
        sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/g' /etc/ssh/sshd_config
        sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
        systemctl restart sshd
    SHELL
  end
end

