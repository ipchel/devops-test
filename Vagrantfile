Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "ubuntu2004.localdomain"
  
  # Проброс портов
  config.vm.network "forwarded_port", guest: 9090, host: 9090
  config.vm.network "forwarded_port", guest: 3000, host: 3000
  
  # Shared folder
  config.vm.synced_folder ".", "/vagrant", disabled: false
  
  # Provisioning
  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.become = true
  end
end
