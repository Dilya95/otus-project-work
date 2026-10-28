VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "bento/ubuntu-24.04"

  config.ssh.insert_key = false

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
  end

  servers = {
    "frontend" => {
      management_ip: "192.168.56.10",
      app_ip: "172.16.20.10",
      memory: 1024,
      cpus: 1
    },

    "backend01" => {
      management_ip: "192.168.56.21",
      app_ip: "172.16.20.21",
      db_ip: "172.16.30.21",
      memory: 1536,
      cpus: 1
    },

    "backend02" => {
      management_ip: "192.168.56.22",
      app_ip: "172.16.20.22",
      db_ip: "172.16.30.22",
      memory: 1536,
      cpus: 1
    },

    "db01" => {
      management_ip: "192.168.56.31",
      db_ip: "172.16.30.31",
      memory: 1536,
      cpus: 1
    },

    "db02" => {
      management_ip: "192.168.56.32",
      db_ip: "172.16.30.32",
      memory: 1536,
      cpus: 1
    },

    "monitoring" => {
      management_ip: "192.168.56.40",
      memory: 3072,
      cpus: 2
    }
  }

  servers.each do |name, options|
    config.vm.define name do |node|
      node.vm.hostname = name

      node.vm.network "private_network",
        ip: options[:management_ip],
        netmask: "255.255.255.0"

      if options[:app_ip]
        node.vm.network "private_network",
          ip: options[:app_ip],
          netmask: "255.255.255.0"
      end

      if options[:db_ip]
        node.vm.network "private_network",
          ip: options[:db_ip],
          netmask: "255.255.255.0"
      end

      node.vm.provider "virtualbox" do |vb|
        vb.name = "otus-#{name}"
        vb.memory = options[:memory]
        vb.cpus = options[:cpus]
      end
    end
  end
end
