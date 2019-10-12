# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  config.vm.network "forwarded_port", guest: 5000, host: 5000
  config.vm.network "private_network", ip: "192.168.45.10"

  config.vm.synced_folder "./data", "/vagrant_data", type: "nfs"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
  end

  config.vm.provision "shell", inline: <<-SHELL
    add-apt-repository universe
    apt-get update
    apt-get upgrade -y
    apt-get install -y vim git tmux curl build-essential apt-transport-https
    wget -q https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
    dpkg -i packages-microsoft-prod.deb
    rm -f packages-microsoft-prod.deb
    apt-get update
    apt-get install -y dotnet-sdk-3.0 dotnet-runtime-3.0 lldb-3.9 llvm-3.9 python-lldb-3.9
  SHELL
end
