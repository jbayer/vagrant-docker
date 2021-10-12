Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-20.04"
  config.vm.provision "docker"
  config.vm.network "forwarded_port", guest: 2375, host: 2375   # docker
  config.vm.synced_folder ".", "/vagrant"
end


