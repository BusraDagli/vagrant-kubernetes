# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.6.0"

def read_bool_env key, default_value = false
    key = key.to_s
    if ENV.include?(key)
      return ! (['no', 'off', 'false', '0']).include?(ENV[key].strip.downcase)
    else
      return default_value
    end
end
  
def read_env key, default_value = nil, false_value = false, true_value = true
    key = key.to_s
    if ENV.include?(key)
        val = ENV[key].strip
        if  (['no', 'off', 'false', '0']).include? val
            return false_value
        else
            return val
        end
    else
        return default_value
    end
end

memory = read_env 'MEM', '4096'
cpus = read_env 'CPU', '1'
master_cpus = read_env 'MASTER_CPU', ([cpus.to_i, 2].max).to_s # 2 CPU min for master
master_runs_pods = read_bool_env 'MASTER_RUNS_PODS', true

docker_version = read_env 'DOCKER_VERSION', '5:19.03.5~3-0' # check https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.17.md and apt-cache madison docker-ce
docker_repo_fingerprint = read_env 'DOCKER_APT_FINGERPRINT', '0EBFCD88'

k8s_version = read_env 'K8S_VERSION', '1.17.0-00'

gluster_version = read_env 'GLUSTER_VERSION', '7'

host_itf = read_env 'ITF', false

leader_ip = (read_env 'MASTER_IP', "192.168.2.100").split('.').map {|nbr| nbr.to_i} # private ip ; public ip is to be set up with DHCP
hostname_prefix = read_env 'PREFIX', 'k8s'

nodes = (read_env 'NODES', 3).to_i
raise "There should be at least one node and at most 255 while prescribed #{nodes} ; you can set up node number like this: NODES=2 vagrant up" unless nodes.is_a? Integer and nodes >= 1 and nodes <= 255

box = read_env 'BOX', 'bento/debian-10' # must be debian-based
# Box-dependent
vagrant_user = read_env 'VAGRANT_USER', 'vagrant'
vagrant_group = read_env 'VAGRANT_GROUP', 'vagrant'
vagrant_home = read_env 'VAGRANT_HOME', '/home/vagrant'

guest_additions = read_bool_env 'GUEST_ADDITIONS'

public = read_bool_env 'PUBLIC', false
private = read_bool_env 'PRIVATE', true

public_itf = 'eth1' # depends on chosen box and order of interface declaration
private_itf = if public then 'eth2' else 'eth1' end # depends on chosen box
default_itf = read_env 'DEFAULT_PUBLIC_ITF', if public then public_itf else private_itf end # default gateway
internal_itf = case ENV['INTERNAL_ITF']
    when 'public'
        raise 'Cannot use public interface in case it is disabled ; state PUBLIC=yes' unless public
        public_itf
    when 'private'
        raise 'Cannot use private interface in case it is disabled ; state PRIVATE=yes' unless private
        private_itf
    when String
        ENV['ETCD_ITF'].strip
    else
        if public then public_itf else private_itf end
end # interface used for internal node communication (i.e. should it be public or private ?)
host_ip_script = "ip -4 addr list #{internal_itf} |  grep -v secondary | grep inet | sed 's/.*inet\\s*\\([0-9.]*\\).*/\\1/'"

definitions = (1..nodes).map do |node_number|
    hostname = "%s%02d" % [hostname_prefix, node_number]
    ip = leader_ip.dup
    ip[-1] += node_number-1
    ip_str = ip.join('.')
    raise "Not enough addresses available for all nodes, e.g. can't assign IP #{ip_str} to #{hostname} ; lower NODES number or give another MASTER_IP" if ip[-1] > 255
    {:hostname => hostname, :ip => ip_str}
end

if public
    require 'socket'
    vagrant_host = Socket.gethostname || Socket.ip_address_list.find { |ai| ai.ipv4? && !ai.ipv4_loopback? }.ip_address
    puts "this host is #{vagrant_host}"
    require 'digest/md5' # used later for machine id generation so that dhcp returns the same IP
end

pub_key = 'AAAAB3NzaC1yc2EAAAADAQABAAABAQDFCEEemETfqtunwT8G2A+aaqJlXME99G0LtSk2Nd7ER1uPt54lY6uxCs+5lz6c6WXS58XPHNOOfz8F9iUgyJqOM97Dj9HOaSAdmE+xvOHa5lf8fUpeb3GhRNvp8vnwQDfKG3wdrMLUlZjqMbJnH63C/H5nwQ4LybbfLc9XtL8D7PQEGW5SbUaEmULNO46JydEUWgtGodjc6UHs0YVor8e89Up5uy5a0MGIQeB2B6y6rkVc2+aNwUka8bY3O9HuLlJmB+iYKu9IP/pVwy3Y733FRyB7XJJL4T1jsMZfjbQoyPoEGVU5EC8j8dUy+XkUfCe5dWY1wdNDG9oBbwWz1+B5'
priv_key = '-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAxQhBHphE36rbp8E/BtgPmmqiZVzBPfRtC7UpNjXexEdbj7eeJWOr
sQrPuZc+nOll0ufFzxzTjn8/BfYlIMiajjPew4/RzmkgHZhPsbzh2uZX/H1KXm9xoUTb6f
L58EA3yht8HazC1JWY6jGyZx+twvx+Z8EOC8m23y3PV7S/A+z0BBluUm1GhJlCzTuOicnR
FFoLRqHY3OlB7NGFaK/HvPVKebsuWtDBiEHgdgesuq5FXNvmjcFJGvG2NzvR7i5SZgfomC
rvSD/6VcMt2O99xUcge1ySS+E9Y7DGX420KMj6BBlVORAvI/HVMvl5FHwnuXVmNcHTQxva
AW8Fs9fgeQAAA8h6qyIHeqsiBwAAAAdzc2gtcnNhAAABAQDFCEEemETfqtunwT8G2A+aaq
JlXME99G0LtSk2Nd7ER1uPt54lY6uxCs+5lz6c6WXS58XPHNOOfz8F9iUgyJqOM97Dj9HO
aSAdmE+xvOHa5lf8fUpeb3GhRNvp8vnwQDfKG3wdrMLUlZjqMbJnH63C/H5nwQ4LybbfLc
9XtL8D7PQEGW5SbUaEmULNO46JydEUWgtGodjc6UHs0YVor8e89Up5uy5a0MGIQeB2B6y6
rkVc2+aNwUka8bY3O9HuLlJmB+iYKu9IP/pVwy3Y733FRyB7XJJL4T1jsMZfjbQoyPoEGV
U5EC8j8dUy+XkUfCe5dWY1wdNDG9oBbwWz1+B5AAAAAwEAAQAAAQEAiWQHHJFrPVgD0Qdk
rp4My01eLjYuncgKHebWdPG9g7qKcz3DrijBOTPjw3NeesYZdaaefZyJPM0oIj0QiLq5Yz
1yMYXg9ADEHz7tG3AtQZnrcqnfKNinMKA2hP0kIc512J2vv3WPafNi7LN4xoYFgXjVn/2z
kK64sQldkrf7lnzzFq7/3/hxiCKhkYd3MO3n213WmvCXnv/fogliIQUMRUov5A4Ib+VgMt
livMFssyktZUK0p8Lnq4MT8G7vsPGfOC4KNVORyvhQWL3AavdrxXjm6Ss4ycudv08ZrFuw
wvMmlZ79MpANcuc8zdZJM0qoES9PKHx8bh0EN1HitbMr3QAAAIB+NbJ8UKeFXiCnJ+IE3y
uhCteLo8jWwThQDBHoueP7cNVIsNT2c1sryKwRS576hUy0vmoNeAUhePFiFZ1hKIJjbqtz
wuZYb7TG0W2ohgxurTW7OEShhOsv17Y6APYd5G2fNQ15CX8D/Ij8QcPqtrxwUOJYaeQgod
/2+2QG5ynjDgAAAIEA9Hd5M8sxmeGkLUJmQ64Xnk4f+ClzR7JWRSjFO6YPj4c5ZpBlBxvj
mBMt/CGlMV4+29F/rWmRW0SgHNHQUUSfcqQ2tUFXMnKo5YO0vDIXV6ZSYOq7P8VFGY7qiu
WeA3gQboC1afHP2UE+JWVA/lrQK9FRYA1mVU6dH6a75OTTRN8AAACBAM5T5S3P6mZKZA2l
/DocqoOGF7w/Op6ARNOU0vl0hJhY7B8TvM97TfB6u3lpVyQhixxFChPrfj2mfFsT/HeX9I
wQ9BHtc5YfU7ePa+1XuXDfd1wDgF3lxETMcIpjKDODS7hRfFD0b/q3Hv9zWzaug4C70+pU
JMSNVvJ7sbXxrW2nAAAADnZhZ3JhbnRAazhzLTAxAQIDBA==
-----END OPENSSH PRIVATE KEY-----'

Vagrant.configure("2") do |config_all|
    # always use Vagrants insecure key
    config_all.ssh.insert_key = false
    # forward ssh agent to easily ssh into the different machines
    config_all.ssh.forward_agent = true
    config_all.vm.box = box

    config_all.vm.synced_folder ".", "/vagrant", disabled: true
    config_all.vm.synced_folder ".", "/home/vagrant/sync", disabled: true

    root_hostname = definitions[0][:hostname]
    root_ip = definitions[0][:ip]

    unless guest_additions
        config_all.vm.provider :virtualbox do |vb|
            vb.check_guest_additions = false
            vb.functional_vboxsf     = false
            if Vagrant.has_plugin?("vagrant-vbguest") then
                config_all.vbguest.auto_update = false
            end
        end
    end

    # Generic
    config_all.vm.provision :shell, :name => "Setting up aliases", :inline => "
        grep -q 'alias ll=' || echo 'alias ll=\"ls -alh\"' >> /etc/bash.bashrc
    "

    # Referencing all IPs in /etc/hosts
    definitions.each do |node|
        config_all.vm.provision :shell, :name  => "referencing #{node[:hostname]}", :run => "always", :inline => "grep -q " + node[:hostname] + " /etc/hosts || echo \"" + node[:ip] + " " + node[:hostname] + "\" >> /etc/hosts"
    end

    # Auto SSH
    config_all.vm.provision "shell", name: 'auto ssh', inline: "mkdir -m 0700 -p /root/.ssh; touch /root/.ssh/authorized_keys; chmod 600 /root/.ssh/authorized_keys"
    (1..nodes).each do |node_number|
        node_name = definitions[node_number-1][:hostname]
        config_all.vm.provision "shell", name: "auto ssh from #{node_name}", inline: "grep -q 'root@#{node_name}' /root/.ssh/authorized_keys || echo 'ssh-rsa #{pub_key} root@#{node_name}' >> /root/.ssh/authorized_keys"
    end

    # Docker Installation
    if docker_version
        config_all.vm.provision "DockerInstall", type: "shell", name: 'Installing Docker', inline: "
            export APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1
            export DEBIAN_FRONTEND=noninteractive
            apt-get update
            apt-get install --yes apt-transport-https ca-certificates curl gnupg2 software-properties-common
            curl -fsSL https://download.docker.com/linux/$(lsb_release -i -s  | tr '[:upper:]' '[:lower:]')/gpg | apt-key add -
            apt-key fingerprint #{docker_repo_fingerprint}
            add-apt-repository \"deb [arch=amd64] https://download.docker.com/linux/$(lsb_release -i -s  | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable\"
            apt-get update
            apt-get install --yes docker-ce=#{docker_version}~$(lsb_release -i -s  | tr '[:upper:]' '[:lower:]')-$(lsb_release -cs) docker-ce-cli=#{docker_version}~$(lsb_release -i -s  | tr '[:upper:]' '[:lower:]')-$(lsb_release -cs) containerd.io
            apt-mark hold docker-ce docker-ce-cli containerd.io
            if [ ! -f /etc/docker/daemon.json ]; then
                cat > /etc/docker/daemon.json <<EOF
{
    \"exec-opts\": [\"native.cgroupdriver=systemd\"],
    \"log-driver\": \"json-file\",
    \"log-opts\": {
        \"max-size\": \"100m\"
    },
    \"storage-driver\": \"overlay2\"
}
EOF
                mkdir -p /etc/systemd/system/docker.service.d
                systemctl daemon-reload
                systemctl restart docker
                echo \"Docker daemon restarted\"
            fi
            usermod -aG docker vagrant
            "
    end

    # Kubernetes installation
    if k8s_version
        raise "Cannot install Kubernetes without Docker" unless docker_version
        config_all.vm.provision "K8SInstall", type: "shell", name: 'Installing Kubernetes', inline: "
                export APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1
                export DEBIAN_FRONTEND=noninteractive
                swapoff -a;sed -i '/swap/d' /etc/fstab
                update-alternatives --set iptables /usr/sbin/iptables-legacy
                update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
                update-alternatives --set arptables /usr/sbin/arptables-legacy
                update-alternatives --set ebtables /usr/sbin/ebtables-legacy
                curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
                echo 'deb https://apt.kubernetes.io/ kubernetes-xenial main' >/etc/apt/sources.list.d/kubernetes.list
                apt-get update
                apt-get install --yes kubelet=#{k8s_version} kubeadm=#{k8s_version} kubectl=#{k8s_version}
                apt-mark hold kubelet kubeadm kubectl
                [ -f /etc/bash_completion.d/kubectl ] || kubectl completion bash >/etc/bash_completion.d/kubectl
            "
    end

    # Gluster installation
    if gluster_version
        config_all.vm.provision "GlusterInstall", type: "shell", name: 'Installing GlusterFS', inline: "
            export APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1
            export DEBIAN_FRONTEND=noninteractive
            curl -s https://download.gluster.org/pub/gluster/glusterfs/#{gluster_version}/rsa.pub | apt-key add -
            DEBID=$(grep 'VERSION_ID=' /etc/os-release | cut -d '=' -f 2 | tr -d '\"')
            DEBVER=$(grep 'VERSION=' /etc/os-release | grep -Eo '[a-z]+')
            DEBARCH=$(dpkg --print-architecture)
            echo \"deb https://download.gluster.org/pub/gluster/glusterfs/#{gluster_version}/LATEST/Debian/${DEBID}/${DEBARCH}/apt ${DEBVER} main\" > /etc/apt/sources.list.d/gluster.list
            apt-get update
            apt-get install --yes glusterfs-server
            systemctl start glusterd
            systemctl enable glusterd
        "
    end
        
    (1..nodes).each do |node_number|
        definition = definitions[node_number-1]
        hostname = definition[:hostname]
        ip = definition[:ip]
        master = node_number == 1

        config_all.vm.define hostname, primary: node_number == 1 do |config|
            config.vm.hostname = hostname
            config.vm.provider :virtualbox do |vb, override|
                vb.memory = memory
                vb.cpus = if master then master_cpus else cpus end
                vb.customize [
                  'modifyvm', :id,
                  '--name', hostname,
                  '--cpuexecutioncap', '100',
                  '--paravirtprovider', 'kvm',
                ]
            end

            if public
                options = {}
                options[:use_dhcp_assigned_default_route] = true
                options[:bridge] = host_itf if host_itf
                options[:auto_config] = false
                config.vm.network "public_network", **options
                
                machine_id = (Digest::MD5.hexdigest "#{hostname} on #{vagrant_host}").upcase
                machine_id[2] = (machine_id[2].to_i(16) & 0xFE).to_s(16).upcase # generated MAC must not be multicast
                machine_mac = "#{machine_id[1, 2]}:#{machine_id[3, 2]}:#{machine_id[5, 2]}:#{machine_id[7, 2]}:#{machine_id[9, 2]}:#{machine_id[11, 2]}"
                
                config.vm.provider :virtualbox do |vb, override|
                    vb.customize [
                        'modifyvm', :id,
                        '--macaddress2', "#{machine_mac.delete ':'}",
                    ]
                end
            end # public itf

            config.vm.network :private_network, ip: ip

            # Same SSH keys for everyone
            config.vm.provision "shell", name: 'keys', inline: "mkdir -m 0700 -p /root/.ssh; echo '#{priv_key}' > /root/.ssh/id_rsa; echo 'ssh-rsa #{pub_key} root@#{hostname}' > /root/.ssh/id_rsa.pub ; chmod 600 /root/.ssh/id_rsa"

            if k8s_version
                if master
                    # Initializing K8s
                    config.vm.provision "K8SInit", type: "shell", name: 'Initializing the Kubernetes cluster', inline: "
                        if [ ! -f /etc/kubernetes/admin.conf ]; then echo 'Initializing Kubernetes' ; kubeadm init --apiserver-advertise-address=$(#{host_ip_script}) --pod-network-cidr=10.244.0.0/16 | tee /root/k8sjoin.txt; fi
                        if [ ! -d $HOME/.kube ]; then mkdir -p $HOME/.kube ; cp -f -i /etc/kubernetes/admin.conf $HOME/.kube/config ; fi
                        if [ ! -d #{vagrant_home}/.kube ]; then mkdir -p #{vagrant_home}/.kube ; cp -f -i /etc/kubernetes/admin.conf #{vagrant_home}/.kube/config ; chown #{vagrant_user}:#{vagrant_group} #{vagrant_home}/.kube/config ; fi
                        kubectl get pods --namespace kube-system | grep -q flannel || kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
                        "
                else
                    # Joining K8s
                    config.vm.provision "K8SJoin", type: "shell", name: 'Joining the Kubernetes cluster', inline: "
                        [ -d ~/.kube ] || scp -o StrictHostKeyChecking=no -r #{root_hostname}:~/.kube .
                        [ -f /etc/kubernetes/kubelet.conf ] || kubeadm join --discovery-file .kube/config
                    "
                end
            end

            if gluster_version
                unless master
                    # Joining Gluster
                    config.vm.provision "GlusterJoin", type: "shell", name: 'Joining the Gluster cluster', inline: "
                        ssh -o StrictHostKeyChecking=no #{root_hostname} 'gluster pool list' | grep -q #{hostname} || ssh #{root_hostname} 'gluster peer probe #{hostname}'
                    "
                end
            end
            
        end # node cfg
    end # node
end # config
