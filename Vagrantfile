servers = [
    { :hostname => 'web', :ips => '192.168.31.200', :ram => 2048, :cpuz => 2 },
]

Vagrant.configure(2) do |config|
    servers.each do |machine|
        config.vm.define machine[:hostname] do |node|
            node.vm.box = 'centos/7'
            node.vm.box_version = "2004.01"
            node.vm.hostname = machine[:hostname]
            node.vm.network :public_network, ip: machine[:ips], bridge: "wlp58s0"
            node.vm.provider "virtualbox" do |vb|
                vb.memory = machine[:ram]
                vb.cpus = machine[:cpuz]
            end
            node.vm.provision "shell", inline: <<-SHELL
                mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
                sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
                systemctl restart sshd
            SHELL
            node.vm.provision :ansible do |ansible|
                ansible.playbook = "playbooks/playbook-#{node.vm.hostname}.yml"
            end
        end
        
    end
end
