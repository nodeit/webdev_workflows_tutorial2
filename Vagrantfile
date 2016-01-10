# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu/trusty64"
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.synced_folder "myapp", "/var/www/myapp", type: "rsync"

  # Ansible Docs:
  # Playbooks: http://docs.ansible.com/ansible/playbooks.html
  # Best Practices: http://docs.ansible.com/ansible/playbooks_best_practices.html
  config.vm.provision "ansible" do |ansible|
    
    # ansible.verbose = "v"
    ansible.sudo = true
    ansible.playbook = "provisioning/playbook.yml"

    ansible.extra_vars = {
      nginx_hostname: "myapp",
      web_project_name: "myapp",
      node: {
        port: 8080,
        version: "4.2.4"
      }
    }

  end

end
