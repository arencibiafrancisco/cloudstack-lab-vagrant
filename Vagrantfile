# Vagrantfile profesional para entorno CloudStack + KVM con Ansible sin bonds, usando una interfaz detectada dinámicamente

VAGRANTFILE_API_VERSION = "2"
iface = IO.popen("ip -o link show | awk -F': ' '{print $2}' | grep -Ev 'lo|eth0|virbr0|docker|br-|veth' | head -n 1").read.strip

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/jammy64"
  config.ssh.insert_key = true

  # Variables comunes
  common_provisioning = <<-SCRIPT
    sudo touch /etc/cloud/cloud-init.disabled
    sudo apt-get update && sudo apt-get install -y software-properties-common git curl jq
    sudo usermod -aG sudo vagrant
    echo "vagrant ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/vagrant

    mkdir -p /home/vagrant/.ssh
    cp /vagrant/id_rsa.pub /home/vagrant/.ssh/authorized_keys
    chown -R vagrant:vagrant /home/vagrant/.ssh
    chmod 700 /home/vagrant/.ssh
    chmod 600 /home/vagrant/.ssh/authorized_keys

    # Habilitar root con clave pública SSH
    mkdir -p /root/.ssh
    cp /vagrant/id_rsa.pub /root/.ssh/authorized_keys
    chmod 700 /root/.ssh
    chmod 600 /root/.ssh/authorized_keys
  SCRIPT

  # VM 1: CloudStack Management + NFS
  config.vm.define "cloudstack-mgmt" do |mgmt|
    mgmt.vm.hostname = "cloudstack-mgmt"
    mgmt.vm.network "private_network", ip: "192.168.56.10"
    mgmt.vm.network "forwarded_port", guest: 22, host: 2223, id: "ssh"
    mgmt.vm.provider "virtualbox" do |vb|
      vb.name = "cloudstack-mgmt"
      vb.memory = 4096
      vb.cpus = 2
    end
    mgmt.vm.provision "file", source: File.expand_path("~/.ssh/id_rsa.pub"), destination: "/vagrant/id_rsa.pub"
    mgmt.vm.provision "shell", inline: common_provisioning
    mgmt.vm.provision "shell", inline: <<-SHELL
      sudo apt-add-repository --yes --update ppa:ansible/ansible
      sudo apt-get install -y ansible

      git clone https://github.com/arencibiafrancisco/cloudstack-installer.git || true
      cd cloudstack-installer

      echo '[acs-manager]' > hosts
      echo '127.0.0.1 ansible_connection=local' >> hosts

      sudo ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook deploy-cloudstack.yml -i hosts \
        -e "nodetype=master mysql_root_password=Cl0ud5tack-MySQL \
        mysql_cloud_password=Cl0ud5tack-Cl0ud cloudstack_release=4.19 \
        cloudstack_systemvmtemplate=4.19.1 install_local_db=true"
    SHELL
    mgmt.vm.provision "shell", inline: "echo '[INFO] CloudStack Manager IP: 192.168.56.10'"
  end

  # VM 2: KVM Host
  config.vm.define "kvm-node" do |kvm|
    kvm.vm.hostname = "kvm-node"
    kvm.vm.network "private_network", type: "static", ip: "192.168.56.20"
    kvm.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh"
    kvm.vm.provider "virtualbox" do |vb|
      vb.name = "kvm-node"
      vb.memory = 4096
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
      vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    end

    kvm.vm.provision "file", source: File.expand_path("~/.ssh/id_rsa.pub"), destination: "/vagrant/id_rsa.pub"
    kvm.vm.provision "shell", inline: common_provisioning
    kvm.vm.provision "shell", inline: <<-SHELL
      sudo apt-add-repository --yes --update ppa:ansible/ansible
      sudo apt-get install -y ansible

      git clone https://github.com/arencibiafrancisco/kvm.git || true
      cd kvm

      echo '[hypervisor]' > hosts
      echo '127.0.0.1 ansible_connection=local' >> hosts

      ansible-playbook kvm-playbook.yml -i hosts -u root --tags "kvm,cockpit"

      cat <<EOF | sudo tee /etc/netplan/01-cloudbr0.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      dhcp4: no
  bridges:
    cloudbr0:
      interfaces: [enp0s8]
      addresses: [192.168.56.20/24]
      routes:
        - to: default
          via: 192.168.56.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      parameters:
        stp: false
        forward-delay: 0
EOF

      sudo netplan generate
      sudo netplan apply

      # Verificar conectividad
      sleep 5
      ping -c 3 192.168.56.1 || echo "⚠️ Verificando conectividad con el gateway"
      ping -c 3 8.8.8.8 || echo "⚠️ Verificando conectividad a Internet"
    SHELL
    kvm.vm.provision "shell", inline: "echo '[INFO] KVM Node IP: 192.168.56.20'"
  end
end
