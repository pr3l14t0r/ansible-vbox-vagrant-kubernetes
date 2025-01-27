IMAGE_NAME = "generic/ubuntu2004"
IMAGE_VERSION = "3.4.0"
K8S_NAME = "kubernetes-forensics"

# Stats for master node(s)
MASTERS_NUM = 1
MASTERS_CPU = 2 
MASTERS_MEM = 4096

# stats for worker node(s)
NODES_NUM = 1
NODES_CPU = 2
NODES_MEM = 4096

IP_BASE = "192.168.56."

VAGRANT_DISABLE_VBOXSYMLINKCREATE=1

# Windows WSL users should use an own ssh private key and inject its corresponding public-key into the VM.
# uncomment the next line, providing a path to your ssh private key within the WSL2 distro
SECURE_SSH_PRIVATE_KEY = "~/.ssh/k8sDeploy"

Vagrant.configure("2") do |config|

    # specifiy same image and version for each machine 
    config.vm.box = IMAGE_NAME
    config.vm.box_version = IMAGE_VERSION
    #config.disksize.size = '60GB'
    config.ssh.insert_key = false

    # Insert own ssh key to the machines. 
    # Taken from: https://stackoverflow.com/questions/30075461/how-do-i-add-my-own-public-key-to-vagrant-vm
    # uncomment the next block
    vagrant_home_path = ENV["VAGRANT_HOME"] ||= "~/.vagrant.d"
    config.ssh.private_key_path = ["#{vagrant_home_path}/insecure_private_key", "#{SECURE_SSH_PRIVATE_KEY}"]

    # inject the ssh key file to the VMs
    config.vm.provision "file", source: "#{SECURE_SSH_PRIVATE_KEY}.pub", destination: "/home/vagrant/.ssh/SecureKey.pub"
    config.vm.provision :shell, privileged: false do |shell_action|
       shell_action.inline = <<-SHELL
           cat /home/vagrant/.ssh/SecureKey.pub >> /home/vagrant/.ssh/authorized_keys
       SHELL
     end

    # update the machines and disable updater afterwards (thanks @ bento!)
    config.vm.provision "shell", path: "disableUpdates.sh"

    # Provisionign of the master node(s)
    (1..MASTERS_NUM).each do |i|      
        config.vm.define "k8s-m-#{i}" do |master|
            master.vm.network "private_network", ip: "#{IP_BASE}#{i + 10}"
            master.vm.hostname = "k8s-m-#{i}"
            master.vm.provider "virtualbox" do |v|
                v.name = "kubernetes-master-#{i}"
                v.memory = MASTERS_MEM
                v.cpus = MASTERS_CPU
            end            
            master.vm.provision "ansible" do |ansible|
                ansible.playbook = "roles/k8s.yml"
                #Redefine defaults
                ansible.extra_vars = {
                    k8s_cluster_name:       K8S_NAME,                    
                    k8s_master_admin_user:  "vagrant",
                    k8s_master_admin_group: "vagrant",
                    k8s_master_apiserver_advertise_address: "#{IP_BASE}#{i + 10}",
                    k8s_master_node_name: "k8s-m-#{i}",
                    k8s_node_public_ip: "#{IP_BASE}#{i + 10}"
                }                
            end
        end
    end

    # Provisioning of the worker node(s)
    (1..NODES_NUM).each do |j|
        config.vm.define "k8s-n-#{j}" do |node|
            node.vm.network "private_network", ip: "#{IP_BASE}#{j + 10 + MASTERS_NUM}"
            node.vm.hostname = "k8s-n-#{j}"
            node.vm.provider "virtualbox" do |v|
                v.name = "kubernetes-worker-#{j}"
                v.memory = NODES_MEM
                v.cpus = NODES_CPU
                #v.customize ["modifyvm", :id, "--cpuexecutioncap", "20"]
            end             
            node.vm.provision "ansible" do |ansible|
                ansible.playbook = "roles/k8s.yml"                   
                #Redefine defaults
                ansible.extra_vars = {
                    k8s_cluster_name:     K8S_NAME,
                    k8s_node_admin_user:  "vagrant",
                    k8s_node_admin_group: "vagrant",
                    k8s_node_public_ip: "#{IP_BASE}#{j + 10 + MASTERS_NUM}"
                }
            end
        end
    end
end
