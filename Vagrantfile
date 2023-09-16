# -*- mode: ruby -*-
# vi: set ft=ruby :
# version 0.2

# Enable experimental features
ENV["VAGRANT_EXPERIMENTAL"] = "disks"

# Install plugins
# https://github.com/hashicorp/vagrant/wiki/Available-Vagrant-Plugins
need_restart = false
required_plugins = %w(vagrant-registration vagrant-hostmanager vagrant-protect vagrant-cachier vagrant-scp)
required_plugins.each do |plugin|
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
    need_restart = true
  end
end
exec "vagrant #{ARGV.join' '}" if need_restart

# Imports
# Require YAML module
require 'yaml'

# Variables
VAGRANT_DIR ||= File.expand_path(File.dirname(__FILE__))
VAGRANTFILE_API_VERSION ||= "2"
Vagrant.require_version ">=2.2.10"

# Read YAML file with box details
servers = YAML.load_file(VAGRANT_DIR + '/servers.yml')

# Create boxes
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # Configure vagrant-cachier plugin
  if Vagrant.has_plugin?("vagrant-cachier")
    # https://github.com/fgrehm/vagrant-cachier
    host = RbConfig::CONFIG['host_os']
    if host == /linux/ || host == /darwin/
      config.cache.scope = :machine
        config.cache.synced_folder_opts = {
          type: :nfs, mount_options: ['rw', 'vers=3', 'tcp', 'nolock']
        }
    end
  end

  # Iterate through entries in YAML file
  servers.each do |server|

    if server["username"]
      config.ssh.username = server["username"]
    end

    if server["password"]
      config.ssh.password = server["password"]
      config.ssh.insert_key = false
    end

    config.vm.define server["name"] do |srv|

      # Vagrant box
      if server["box_url"] != nil
        srv.vm.box_url = server["box_url"]
      else
        if server["box_version"] and
            srv.vm.box_version = server["box_version"].to_s
            srv.vm.box_check_update = false
          end
      end
      srv.vm.box = server["box"]

      # Set hostname
      if server["hostname"] != nil
        srv.vm.hostname = server["hostname"]
      end

      # Set ip
      if server["ip"]
        srv.vm.network "private_network", ip: server["ip"]
        # Fix for vmware_fusion: Networks with custom subnet/mask values are not supported on this platform
        # srv.vm.network :private_network
      end

      # Configure ssh
      if server["ssh_host_port"] != nil
        # https://realguess.net/2015/10/06/overriding-the-default-forwarded-ssh-port-in-vagrant/
        srv.vm.network :forwarded_port, guest: 22, host: server['ssh_host_port'], id: "ssh"
      end

      # Configure rdp
      if server["rdp_host_port"] != nil
        srv.vm.network :forwarded_port, guest: 3389, host: server['rdp_host_port'], id: "rdp"
      end

      # Configure winrm
      if server["winrm_host_port"] != nil
        srv.vm.network :forwarded_port, guest: 5985, host: server['winrm_host_port'], id: "winrm", auto_correct: true
      end

      # Configure Disks
      if server["disk"] != nil
        server["disk"].each do |disk|
          srv.vm.disk :disk, size: disk["size"], name: disk["name"]
        end
      end

      # Configure vagrant-protect plugin
      if Vagrant.has_plugin?("vagrant-protect")
        srv.protect.enabled = server["protected"]
      end

      # Configure vagrant-registration plugin
      if Vagrant.has_plugin?('vagrant-registration')
        if ENV['RHSM_USERNAME'] && ENV['RHSM_PASSWORD']
          now = Time.new
          if server["hostname"] != nil
            registration_name = server["hostname"]
          else
            registration_name = server["name"].gsub(/\s+/, '_').gsub!(/[^0-9A-Za-z\.-_]/, '')
          end
          srv.registration.name = registration_name + "-" + now.strftime("%Y%m%d%H%M")
          srv.registration.username = ENV['RHSM_USERNAME']
          srv.registration.password = ENV['RHSM_PASSWORD']
        end
      end

      # Configure vagrant-hostmanager plugin
      if Vagrant.has_plugin?("vagrant-hostmanager")
        srv.hostmanager.enabled = true
        srv.hostmanager.manage_host = true
        srv.hostmanager.manage_guest = true
        srv.hostmanager.ignore_private_ip = false
        srv.hostmanager.include_offline = true
        if server["hostname"] != nil
          hostname = server["hostname"].gsub(/^(\w+)\..*$/, '\1')
          srv.hostmanager.aliases = %w("#{hostname}.localdomain" "#{hostname}")
        end
      end

      # Sync folder
      if File.exist?(".vagrant/machines/#{server["name"]}/virtualbox/action_provision")
        srv.vm.synced_folder ".", "/vagrant"
      end

      # Providers
      srv.vm.provider :virtualbox do |vb|
        vb.name = "#{server["name"]}"
        if server["cpu"] != nil
          vb.cpus = server["cpu"]
        end
        if server["ram"] != nil
          vb.memory = server["ram"]
        end
        if server["cap"] != nil
          vb.customize ["modifyvm", :id, "--cpuexecutioncap", "#{server["cap"]}"]
        end
        vb.gui = server["gui"]
        vb.customize ["setextradata", "global", "GUI/SuppressMessages", "all"]
        # vagrant-vbguest plugin
        if server["guest_tools"] == nil || !server["guest_tools"]
          if Vagrant.has_plugin?("vagrant-vbguest")
            srv.vbguest.auto_update = false
          end
        else
          unless Vagrant.has_plugin?("vagrant-vbguest")
            system "vagrant plugin install vagrant-vbguest"
          end
        end
      end

      srv.vm.provider :parallels do |prl|
        prl.name = "#{server["name"]}"
        if server["cpu"] != nil
          prl.cpus = server["cpu"]
        end
        if server["ram"] != nil
          prl.memory = server["ram"]
        end
        if server["disk"] != nil
          server["disk"].each do |disk|
            # Size in megabytes
            size = disk["size"]
            if size.include? "MB"
              size = size[0..-2].to_i
            elsif size.include? "GB"
              size = size[0..-2].to_i * 1024
            end
            prl.customize ["set", :id, "--device-add", "hdd", "--size", size, "--iface", "sata"]
          end
        end
        if server["cap"] != nil
          (server["cap"] < 40) ? quota = "low" : (server["cap"] < 80) ? quota = "medium" : quota = "unlimited"
          prl.customize ["set", :id, "--resource-quota", "#{quota}"]
        end
        if server["guest_tools"] == nil || !server["guest_tools"]
          prl.update_guest_tools = false
        end
      end

      srv.vm.provider :vmware_desktop do |v|
        v.vmx["displayName"] = "#{server["name"]}"
        if server["cpu"] != nil
          v.vmx["numvcpu"] = server["cpu"]
        end
        if server["ram"] != nil
          v.vmx["memsize"] = server["ram"]
        end
        if server["disk"] != nil
          server["disk"].each do |disk|
            # Not implemented in vmware
          end
        end
        if server["cap"] != nil
          # Not implemented in vmware
          # v.vmx[""] = server["cap"]
        end
        v.gui = server["gui"]
        if server["guest_tools"] == nil || !server["guest_tools"]
          # Not implemented in vmware
          # v.vmx[""] = false
        end
      end

      srv.vm.provider :hyperv do |h|
        # Hyper-V provider not supported ATM... exiting
      end

      srv.vm.provider :docker do |d|
        # Docker provider not supported ATM... exiting
      end

      # Add your public key from ~/.ssh/id_rsa.pub
      config.vm.provision "shell" do |s|
        ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
        s.inline = <<-SHELL
          mkdir -p /root/.ssh/
          echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
          echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
        SHELL
      end

      # Ansible provisioning
      if server["ansible_provision"] != nil && server["ansible_provision"]
        srv.vm.provision :ansible do |ansible|
          ansible.limit = server[:name]
          # Use playbook name from server.yml
          if server["playbook"] == nil
             ansible.playbook = "./playbooks/default.yml"
          else
             ansible.playbook = "#{server["playbook"]}"
          end
          # Pass a custom inventory file
          if server["inventory"] != nil
             ansible.inventory_path = "#{server["inventory"]}"
          end

          ansible.become = "true"
          ansible.become_user = "root"
          ansible.compatibility_mode = "2.0"
          ansible.host_key_checking = "false"

          # Run Ansible in verbose mode
          if server["ansible_verbose"] != nil && server["ansible_verbose"]
            if server["ansible_verbose"].is_a? Numeric
              if server["ansible_verbose"] > 0
                ansible.verbose = "-"
                for i in 1..server["ansible_verbose"]
                  ansible.verbose << "v"
                end
              end
            else
               ansible.verbose = "-v"
            end
          end

          # Set the password for the vault
          if server["ask_vault_pass"]
            ansible.ask_vault_pass = true
          elsif server["vault_password_file"] != nil
            ansible.vault_password_file = "#{server["vault_password_file"]}"
          end

          # Set Ansible roles path
          if server["ansible_roles_path"] == nil
            ansible.galaxy_roles_path = "#{server["ansible_roles_path"]}"
          end

          # Set Ansible requirements file
          if server["ansible_requirements"] != nil
             ansible.galaxy_role_file  = "#{server["ansible_requirements"]}"
          end

          # Ansible extra_vars
          if server["ansible_extra_vars"] != nil
            ansible.extra_vars = server["ansible_extra_vars"]
          end

        end
      end
    end
  end
end
