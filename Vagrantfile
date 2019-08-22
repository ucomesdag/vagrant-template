# -*- mode: ruby -*-
# vi: set ft=ruby :

# Temporary fix for: https://github.com/projectatomic/adb-vagrant-registration/issues/126
# Registration is skiped on rhel7.5 because subscription_manager_registered cap no longer works
if Vagrant.has_plugin?('vagrant-registration')
  module SubscriptionManagerMonkeyPatches
    def self.subscription_manager_registered?(machine)
      true if machine.communicate.sudo("/usr/sbin/subscription-manager list --consumed --pool-only | grep -E '^[a-f0-9]{32}$'")
    rescue
      false
    end
  end
  VagrantPlugins::Registration::Plugin.guest_capability 'redhat', 'subscription_manager_registered?' do
    SubscriptionManagerMonkeyPatches
  end
end
# End fix

# Install plugins
need_restart = false
required_plugins = %w(vagrant-registration vagrant-hostmanager vagrant-protect vagrant-cachier vagrant-vbguest vagrant-disksize) # nugrant vagrant-bindfs vagrant-proxyconf
required_plugins.each do |plugin|
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
    need_restart = true
  end
end
exec "vagrant #{ARGV.join' '}" if need_restart

# Imports :
# Require YAML module
require 'yaml'

# variables :
VAGRANT_DIR ||= File.expand_path(File.dirname(__FILE__))
VAGRANTFILE_API_VERSION ||= "2"
Vagrant.require_version ">=1.8.4"

# Read YAML file with box details
servers = YAML.load_file('servers.yml')

# Create boxes
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # vagrant-cachier plugin
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
  servers.each do |servers|

    if servers["username"]
      config.ssh.username = servers["username"]
    end

    if servers["password"]
      config.ssh.password = servers["password"]
      config.ssh.insert_key = false
    end

    config.vm.define servers["name"] do |srv|

      # vagrant box
      if servers["box_url"] != nil
        srv.vm.box_url = servers["box_url"]
      else
        if servers["box_version"] and
            srv.vm.box_version = servers["box_version"].to_s
            srv.vm.box_check_update = false
          end
      end
      srv.vm.box = servers["box"]

      # ip
      if servers["ip"]
        srv.vm.network "private_network", ip: servers["ip"]
      end

      # ssh
      if servers["ssh_host_port"]
        # https://realguess.net/2015/10/06/overriding-the-default-forwarded-ssh-port-in-vagrant/
        srv.vm.network :forwarded_port, guest: 22, host: servers['ssh_host_port'], id: "ssh"
      end

      # rdp
      if servers["rdp_host_port"]
        srv.vm.network :forwarded_port, guest: 3389, host: servers['rdp_host_port'], id: "rdp"
      end

      # winrm
      if servers["winrm_host_port"]
        srv.vm.network :forwarded_port, guest: 5985, host: servers['winrm_host_port'], id: "winrm", auto_correct: true
      end

      # hostname
      if servers["hostname"] != nil
        srv.vm.hostname = servers["hostname"]
      end

      # vagrant-protect plugin
      if Vagrant.has_plugin?("vagrant-protect")
        srv.protect.enabled = servers["protected"]
      end

      # vagrant-registration plugin
      if Vagrant.has_plugin?('vagrant-registration')
        if ENV['RHSM_USERNAME'] && ENV['RHSM_PASSWORD']
          now = Time.new
          if servers["hostname"] != nil
            registration_name = servers["hostname"]
          else
            registration_name = servers["name"].gsub(/\s+/, '_').gsub!(/[^0-9A-Za-z\.-_]/, '')
          end
          srv.registration.name = registration_name + "-" + now.strftime("%Y%m%d%H%M")
          srv.registration.username = ENV['RHSM_USERNAME']
          srv.registration.password = ENV['RHSM_PASSWORD']
        end
      end

      # vagrant-hostmanager plugin
      if Vagrant.has_plugin?("vagrant-hostmanager")
        srv.hostmanager.enabled = true
        srv.hostmanager.manage_host = true
        srv.hostmanager.manage_guest = true
        srv.hostmanager.ignore_private_ip = false
        srv.hostmanager.include_offline = true
        if servers["hostname"] != nil
          hostname = servers["hostname"].gsub(/^(\w+)\..*$/, '\1')
          srv.hostmanager.aliases = %w("#{hostname}.localdomain" "#{hostname}")
        end
      end

      # vagrant-vbguest plugin
      if Vagrant.has_plugin?("vagrant-vbguest")
        if servers["guest_additions"] == nil || !servers["guest_additions"]
          srv.vbguest.auto_update = false
        end
      end

      # sync folder
      if File.exist?(".vagrant/machines/#{servers["name"]}/virtualbox/action_provision")
        srv.vm.synced_folder ".", "/vagrant"
      end

      # configure provider
      if servers["provider"] != nil
        case servers["provider"]
        when "virtualbox"
          srv.vm.provider :virtualbox do |vb|
            vb.name = "#{servers["name"]}"
            if servers["cpu"] != nil
              vb.cpus = servers["cpu"]
            end
            if servers["ram"] != nil
              vb.memory = servers["ram"]
            end
            if servers["disk"] != nil && Vagrant.has_plugin?("vagrant-disksize")
              srv.disksize.size = servers["disk"]
            end
            vb.gui = servers["gui"]
            vb.customize ["setextradata", "global", "GUI/SuppressMessages", "all"]
          end
        else
          puts "Supported provider ATM is Virtualbox ... exiting"
        end
      end

      # ansible provisioning
      if servers["ansible_provision"] != nil && servers["ansible_provision"]
        srv.vm.provision :ansible do |ansible|
          ansible.limit = servers[:name]
          # Use playbook name from servers.yml
          if servers["playbook"] == nil
             ansible.playbook = "./playbooks/default.yml"
          else
             ansible.playbook = "#{servers["playbook"]}"
          end
          # Pass a custom inventory file
          if servers["inventory"] != nil
             ansible.inventory_path = "#{servers["inventory"]}"
          end

          ansible.become = "true"
          ansible.become_user = "root"
          ansible.compatibility_mode = "2.0"
          ansible.host_key_checking = "false"

          # Run in verbose mode
          if servers["ansible_verbose"] != nil && servers["ansible_verbose"]
            if servers["ansible_verbose"].is_a? Numeric
              if servers["ansible_verbose"] > 0
                ansible.verbose = "-"
                for i in 1..servers["ansible_verbose"]
                  ansible.verbose << "v" 
                end
              end
            else
               ansible.verbose = "-v"
            end
          end

          # Set the password for the vault
          if servers["ask_vault_pass"]
            ansible.ask_vault_pass = true
          elsif servers["vault_password_file"] != nil
            ansible.vault_password_file = "#{servers["vault_password_file"]}"
          end

          # Set roles path
          if servers["ansible_roles_path"] == nil 
            ansible.galaxy_roles_path = "#{servers["ansible_roles_path"]}"
          end
          
          # requirements file
          if servers["ansible_requirements"] != nil
             ansible.galaxy_role_file  = "#{servers["ansible_requirements"]}"
          end

          # extra_vars
          if servers["ansible_extra_vars"] != nil
            ansible.extra_vars = servers["ansible_extra_vars"]
          end

        end
      end
    end
  end
end
