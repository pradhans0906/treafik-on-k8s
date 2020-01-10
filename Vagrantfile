# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_NO_PARALLEL'] = 'yes'

Vagrant.configure(2) do |config|

  NodeCount = 1

  (1..NodeCount).each do |i|
    config.vm.define "ubuntuvm0#{i}" do |node|
    config.vm.network "forwarded_port", guest:80, host:8080
      node.vm.box = "ubuntu/bionic64"
      node.vm.hostname = "ubuntuvm0#{i}.example.com"
      node.vm.network "private_network", ip: "172.42.42.10#{i}"
      node.vm.provider "virtualbox" do |v|
        v.name = "ubuntuvm0#{i}"
        v.memory = 20000
        v.cpus = 8
      end
    end
  end

end
