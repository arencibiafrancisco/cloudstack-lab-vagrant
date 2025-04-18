# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/jammy64"
  config.ssh.insert_key = false
  config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", "~/.ssh/id_rsa"]
  config.ssh.forward_agent = true

  # Configuración común para todas las VMs
  config.vm.provision "shell", inline: <<-SHELL
    # Configuración básica de SSH
    mkdir -p /home/vagrant/.ssh
    chmod 700 /home/vagrant/.ssh
    echo "vagrant ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/vagrant
    sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
    sudo sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
    sudo systemctl restart ssh

        # Establecer contraseñas
    echo "root:password" | sudo chpasswd
    echo "vagrant:vagrant" | sudo chpasswd

    # Deshabilitar cloud-init
    sudo touch /etc/cloud/cloud-init.disabled
  SHELL

  # VM 1: CloudStack Management
  config.vm.define "cloudstack-mgmt" do |mgmt|
    mgmt.vm.hostname = "cloudstack-mgmt"
    mgmt.vm.network "private_network", ip: "192.168.56.10", netmask: "24"
    mgmt.vm.network "forwarded_port", guest: 22, host: 2223, id: "ssh"
    mgmt.vm.network "forwarded_port", guest: 8080, host: 8080, host_ip: "127.0.0.1"

    mgmt.vm.provider "virtualbox" do |vb|
      vb.name = "cloudstack-mgmt"
      vb.memory = 4096
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--nic1", "nat"]
      vb.customize ["modifyvm", :id, "--nic2", "hostonly"]
      vb.customize ["modifyvm", :id, "--hostonlyadapter2", "vboxnet0"]
    end

    mgmt.vm.provision "shell", inline: <<-SHELL
      # Eliminar configuraciones de netplan existentes
      sudo rm -f /etc/netplan/*

      # Configuración de red
      cat <<EOF | sudo tee /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses: [192.168.56.10/24]
EOF
      sudo netplan apply

      # Instalar dependencias
      sudo apt-get update && sudo apt-get install -y software-properties-common
      sudo add-apt-repository -y ppa:ansible/ansible
      sudo apt-get update && sudo apt-get install -y ansible git python3-pip

      # Clonar repositorio con roles de Ansible
      git clone https://github.com/arencibiafrancisco/cloudstack-installer.git || true
      cd cloudstack-installer

      # Crear inventario de Ansible
      cat <<EOF > hosts
[acs-manager]
127.0.0.1 ansible_connection=local

EOF

      # Ejecutar playbook de CloudStack
      sudo ansible-playbook deploy-cloudstack.yml -i hosts \
        -e "nodetype=master mysql_root_password=Cl0ud5tack-MySQL \
        mysql_cloud_password=Cl0ud5tack-Cl0ud cloudstack_release=4.19 \
        cloudstack_systemvmtemplate=4.19.1 install_local_db=true"
    SHELL
  end

  # VM 2: KVM Host
  config.vm.define "kvm-node" do |kvm|
    kvm.vm.hostname = "kvm-node"
    kvm.vm.network "private_network", ip: "192.168.56.20", netmask: "24"
    kvm.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh"
    
    kvm.vm.provider "virtualbox" do |vb|
      vb.name = "kvm-node"
      vb.memory = 4096
      vb.cpus = 2
      vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
      vb.customize ["modifyvm", :id, "--nic1", "nat"]
      vb.customize ["modifyvm", :id, "--nic2", "hostonly"]
      vb.customize ["modifyvm", :id, "--hostonlyadapter2", "vboxnet0"]
      vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
    end

    kvm.vm.provision "shell", inline: <<-SHELL
      # Eliminar configuraciones de netplan existentes
      sudo rm -f /etc/netplan/*

      # Configuración de red y bridge cloudbr0
      cat <<EOF | sudo tee /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses: [192.168.56.20/24]
  bridges:
    cloudbr0:
      interfaces: [enp0s8]
      addresses: [192.168.56.20/24]
      parameters:
        stp: false
        forward-delay: 0
EOF
      sudo netplan apply

      # Instalar dependencias
      sudo apt-get update && sudo apt-get install -y software-properties-common
      sudo add-apt-repository -y ppa:ansible/ansible
      sudo apt-get update && sudo apt-get install -y ansible git python3-pip qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils

      # Clonar repositorio con roles de KVM
      git clone https://github.com/arencibiafrancisco/kvm.git || true
      cd kvm

      # Crear inventario de Ansible
      cat <<EOF > hosts
[hypervisors]
127.0.0.1 ansible_connection=local
EOF

      # Ejecutar playbook de KVM
      sudo ansible-playbook kvm-playbook.yml -i hosts -u root --tags "kvm,cockpit"

      # Configurar CloudStack Agent
      sudo apt-get install -y cloudstack-agent
      cat <<EOF | sudo tee /etc/cloudstack/agent/agent.properties
host=192.168.56.10
port=8250
zone=default
pod=pod1
cluster=cluster1
guid=KVM-$(hostname | md5sum | cut -c1-8)
EOF
      sudo systemctl enable cloudstack-agent
      sudo systemctl restart cloudstack-agent
    SHELL
  end

  # Configuración post-instalación
  config.vm.provision "shell", inline: <<-SHELL
    echo "-----------------------------------------------------------"
    echo "          Entorno CloudStack + KVM configurado"
    echo "-----------------------------------------------------------"
    echo " CloudStack Management: http://192.168.56.10:8080/client"
    echo " Usuario: admin"
    echo " Password: password"
    echo ""
    echo " KVM Host: 192.168.56.20"
    echo " Bridge cloudbr0 configurado en KVM host"
    echo " Acceso SSH: vagrant ssh kvm-node"
    echo "-----------------------------------------------------------"
  SHELL
end
