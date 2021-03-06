# -*- mode: ruby -*-
# vi: set ft=ruby :

env = ENV.has_key?('VM_ENV') ? ENV['VM_ENV'] : "allinone_prod"

require 'yaml'
configuration = YAML.load_file(File.dirname(__FILE__) + "/environments/#{env}.yml")
servers = configuration['servers']
ansible_config = configuration['ansible_config'] || false
libvirt_config = configuration['libvirt_config'] || false

# set minimal Vagrant version
Vagrant.require_version ">= 1.8.0"

Vagrant.configure("2") do |config|
  config.ssh.forward_agent = true
  # see https://twitter.com/mitchellh/status/525704126647128064
  config.ssh.insert_key = false

  # Plugins section
  if Vagrant.has_plugin?("vagrant-cachier")
    if ["demo"].include? env
       config.cache.disable!
    end
    config.cache.scope = :machine
    config.cache.auto_detect = false
    config.cache.enable :apt
  end

  # Disable VBGuest installation in demo env, as it's VirtualBox specific and
  # the demo env is used to generate cross hypervisor VMDK files.
  if Vagrant.has_plugin?("vagrant-vbguest")
    if ["demo"].include? env
      config.vbguest.no_install = true
    end
  end

  # Multi machine environment
  servers.each do |server|
    config.vm.define server['name'] do |machine|
      machine.vm.hostname = server['hostname'] || nil
      machine.vm.box = server['box']
      machine.vm.box_version = server['box_version'] || ">= 0"
      machine.vm.synced_folder '.', '/vagrant', disabled: true

      if server['ip']
        print server['ip'], "\n"
        machine.vm.network "private_network", ip: server['ip']
      end

      if server.has_key?('shares')
        server['shares'].each do |share|
          if server['windows'] || false
            machine.vm.synced_folder share["share_from"], share["share_to"]
          else
            machine.vm.synced_folder share["share_from"], share["share_to"], type: "rsync", rsync__chown: false, rsync__exclude: share["share_exclude"]
          end
        end
      end

      if server['windows'] || false
        machine.vm.communicator = :winrm
        machine.vm.guest = :windows
        machine.ssh.port = 5985
        machine.vm.network :forwarded_port, guest: 3389, host: 3389, id: "rdp", auto_correct:true 
        machine.vm.network :forwarded_port, guest: 5985, host: 5985, id: "winrm", auto_correct:true
      end

      # Providers specific section
      machine.vm.provider "virtualbox" do |v|
        v.gui = false
        v.cpus = server['cpus'] || 1
        v.memory = server['memory'] || 1024
        v.customize ["modifyvm", :id, "--cpuexecutioncap", server['cpuexecutioncap'] || 50]
      end

      if libvirt_config
        machine.vm.provider "libvirt" do |libvirt|
          libvirt.driver = libvirt_config['driver']
          libvirt.connect_via_ssh = libvirt_config['connect_via_ssh'] || false
          if libvirt.connect_via_ssh
            libvirt.username = libvirt_config['username']
            libvirt.host = libvirt_config['host']
            libvirt.id_ssh_key_file = libvirt_config['id_ssh_key_file'] || "id_rsa"
          end
          libvirt.storage_pool_name = libvirt_config['storage_pool_name'] || "default"
          # Equivalent to
          #libvirt.uri = "qemu+ssh://<username>@<host>/system"

          libvirt.machine_arch = server['machine_arch'] || "x86_64"
          libvirt.memory =  server['memory'] || 1024
          libvirt.cpus =  server['cpus'] || 1
          libvirt.volume_cache = "none"
          libvirt.autostart = true

          if server.has_key?('extra_storage')
            libvirt.storage :file, :size => server['extra_storage']
          end
        end
      end
    end
  end

  if ansible_config
    config.vm.provision :ansible do |ansible|
      ansible.playbook = 'playbooks/playbook.yml'
      ansible.extra_vars = ansible_config['extra_vars']
      ansible.groups = ansible_config['groups']
      ansible.force_remote_user = false

      # NOTE: ansible.limit = 'all' is incompatible when provisioning windows
      #ansible.limit = 'all'
      #ansible.tags = ['']
      #ansible.skip_tags = ['']
      #ansible.verbose = 'vvvv'
      #ansible.raw_arguments = ['--check','--diff']
    end
  end
end
