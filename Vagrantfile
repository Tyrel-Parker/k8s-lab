# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Use generic Ubuntu box (works better with libvirt)
  config.vm.box = "generic/ubuntu2204"
  
  # Disable default synced folder (can cause issues)
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Control plane node
  config.vm.define "control" do |control|
    control.vm.hostname = "k8s-control"
    control.vm.network "private_network", ip: "192.168.121.10"
    
    control.vm.provider "libvirt" do |lv|
      lv.memory = "2048"
      lv.cpus = 2
      lv.storage :file, :size => '20G'
    end
  end

  # Worker nodes
  (1..3).each do |i|
    config.vm.define "worker#{i}" do |worker|
      worker.vm.hostname = "k8s-worker#{i}"
      worker.vm.network "private_network", ip: "192.168.121.#{10+i}"
      
      worker.vm.provider "libvirt" do |lv|
        lv.memory = "1024"
        lv.cpus = 1
        lv.storage :file, :size => '20G'
      end
    end
  end
end