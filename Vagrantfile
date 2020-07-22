# -*- mode: ruby -*-
# vim: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "systemdquest", primary: true do |systemdquest|
    systemdquest.vm.box = "centos/7"
    systemdquest.vm.box_version = "2004.01"
    systemdquest.vm.provider "virtualbox" do |v|
        v.gui = true
    end
    systemdquest.vm.host_name = "systemdquest-machine"
    systemdquest.vm.provider "virtualbox" do |v|
        v.memory = "512"
        v.cpus = "2"
    end
    systemdquest.vm.network "private_network", ip: "192.168.1.32"
    systemdquest.vm.provision "shell", inline: <<-SHELL
    SHELL
  end
end

