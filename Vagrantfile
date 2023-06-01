# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
    :prometheus => {
        :box_name => "almalinux/9",
        :cpus => 1,
        :memory => 512,
        :ip_addr => '192.168.50.10',
        #:script => 'vm_prepare.sh',
    },
}

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false
  MACHINES.each do |boxname, boxconfig|
      config.vm.define boxname do |box|
          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s
          box.vm.network "private_network", ip: boxconfig[:ip_addr]
          box.vm.provider :virtualbox do |vb|
            vb.name = boxname.to_s
            vb.memory = boxconfig[:memory]
            vb.cpus = boxconfig[:cpus]
          end
          #box.vm.provision "shell", path: boxconfig[:script]
      end
  end
end
