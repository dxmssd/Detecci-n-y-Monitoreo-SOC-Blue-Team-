Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  # Máquina 1: Wazuh Server
  config.vm.define "wazuh-server" do |server|
    server.vm.hostname = "wazuh-server"
    # Acceso al dashboard desde el host (Fedora)
    server.vm.network "forwarded_port", guest: 443, host: 8443, id: "wazuh_dashboard"
    
    server.vm.provider "libvirt" do |lv|
      lv.memory = 6144
      lv.cpus = 2
    end
  end

  # Máquina 2: Agente Ubuntu (Víctima / Cliente)
  config.vm.define "wazuh-agent-01" do |agent|
    agent.vm.hostname = "wazuh-agent-01"
    
    agent.vm.provider "libvirt" do |lv|
      lv.memory = 2048
      lv.cpus = 1
    end
  end
end