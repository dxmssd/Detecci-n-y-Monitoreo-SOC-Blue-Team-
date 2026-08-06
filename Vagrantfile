Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.hostname = "wazuh-server"

  # Acceso al dashboard desde el host (Fedora)
  config.vm.network "forwarded_port", guest: 443, host: 8443, id: "wazuh-dashboard"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "6144"
    vb.cpus = 2
    vb.name = "wazuh-server"
  end
end
