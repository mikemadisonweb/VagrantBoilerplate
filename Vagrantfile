require 'yaml'
require 'fileutils'

config = {
  local: './vagrant/vagrant-local.yml',
  main: './vagrant/vagrant-main.yml'
}

# Read and write configuration from file
FileUtils.cp config[:main], config[:local] unless File.exist?(config[:local])
options = YAML.load_file config[:local]

Vagrant.require_version ">= 1.5"

# Patch for Windows
# source: https://stackoverflow.com/questions/2108727/which-in-ruby-checking-if-program-exists-in-path-from-ruby
def which(cmd)
    exts = ENV['PATHEXT'] ? ENV['PATHEXT'].split(';') : ['']
    ENV['PATH'].split(File::PATH_SEPARATOR).each do |path|
        exts.each { |ext|
            exe = File.join(path, "#{cmd}#{ext}")
            return exe if File.executable? exe
        }
    end
    return nil
end

Vagrant.configure(2) do |config|
    config.vm.provider :virtualbox do |v|
        host = RbConfig::CONFIG['host_os']
        # VM will get 1/4 of RAM
        if host =~ /darwin/
            cpus = `sysctl -n hw.ncpu`.to_i
            mem = `sysctl -n hw.memsize`.to_i / 1024 / 1024 / 4
        elsif host =~ /linux/
            cpus = `nproc`.to_i
            mem = `grep 'MemTotal' /proc/meminfo | sed -e 's/MemTotal://' -e 's/ kB//'`.to_i / 1024 / 4
        else # Windows
            cpus = 2
            mem = 1024
        end

        v.name = options['machine_name']
        v.customize [
            "modifyvm", :id,
            "--name", options['machine_name'],
            "--memory", mem,
            "--natdnshostresolver1", "on",
            "--cpus", cpus,
        ]
    end

    config.vm.box = options['os']
    config.ssh.username = options['username']
    config.ssh.forward_agent = true
    config.vm.network "private_network", ip: options['ip']
    # rsync version >=3.1 on both host and VM is required for instant synchronization
    if Vagrant.has_plugin?("vagrant-instant-rsync-auto")
        config.vm.synced_folder "./", "/vagrant",
            type: "rsync",
            rsync__exclude: [
                ".git",
                ".idea",
                "hooks",
                "ansible",
                "Vagrantfile",
                "dumps",
                "backend/runtime",
                "frontend/runtime",
                "console/runtime",
                "backend/web/assets",
                "frontend/web/assets"
                ],
            rsync__args: [
                "--verbose",
                "--rsync-path='sudo rsync'",
                "--archive",
                "--delete",
                "-z"
            ]
    end

    # Automatically add record to /etc/hosts
    config.vm.provision :hostmanager
    config.hostmanager.enabled            = true
    config.hostmanager.manage_host        = true
    config.hostmanager.ignore_private_ip  = false
    config.hostmanager.include_offline    = true
    config.hostmanager.aliases            = options['domains']

    # If we can't install Ansible on host(e.g. Windows) install it on VM
    if which('ansible-playbook')
        config.vm.provision :ansible do |ansible|
            ansible.limit = "local" #limit vagrant to local group
            ansible.inventory_path = "ansible/hosts"
            ansible.playbook = options['provision_path']
            # ansible.verbose = "vvvv"
        end
    else
        config.vm.provision :shell, path: "ansible/windows.sh", args: [options['username']]
    end

    if Vagrant.has_plugin?("vagrant-triggers")
        config.trigger.after [:up, :reload] do
            run_remote "runuser -l #{options['username']} -c 'pm2 kill -s'"
            run_remote "runuser -l #{options['username']} -c 'pm2 start --name=websockets /vagrant/nodejs/socketio.js'"
            run_remote "runuser -l #{options['username']} -c 'pm2 start --interpreter=php --name=consumer /vagrant/yii -- rabbitmq-consumer/multiple search -m=100'"
            run_remote "runuser -l #{options['username']} -c 'pm2 start --interpreter=php --name=material-import /vagrant/yii -- material-import/update -i=10'"
            run "vagrant hostmanager"
            run "vagrant instant-rsync-auto"
        end
    end
end
