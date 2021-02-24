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
  
def read_env key, default_value = nil, false_value = false
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

required_plugins = []
required_plugins << 'vagrant-vbguest'
required_plugins << 'vagrant-scp' if read_bool_env 'SCP', true

plugins_to_install = required_plugins.select { |plugin| not Vagrant.has_plugin? plugin }
if not plugins_to_install.empty?
  puts "Installing plugins: #{plugins_to_install.join(' ')}"
  if system "vagrant plugin install #{plugins_to_install.join(' ')}"
    exec "vagrant #{ARGV.join(' ')}"
  else
    abort "Installation of one or more plugins has failed. Aborting."
  end
end

memory = read_env 'MEM', '2048'
master_memory = read_env 'MASTER_MEM', '2048'
cpus = read_env 'CPU', '1'
master_cpus = read_env 'MASTER_CPU', ([cpus.to_i, 2].max).to_s # 2 CPU min for master
nodes = (read_env 'NODES', 3).to_i
raise "There should be at least one node and at most 255 while prescribed #{nodes} ; you can set up node number like this: NODES=2 vagrant up" unless nodes.is_a? Integer and nodes >= 1 and nodes <= 255

docker_version = read_env 'DOCKER_VERSION', 'latest'
docker_repo_fingerprint = read_env 'DOCKER_APT_FINGERPRINT', '0EBFCD88'
containerd_version = read_env 'CONTAINERD_VERSION', 'latest'
compose_version = read_env 'COMPOSE_VERSION', 'latest'
raise "Docker Compose requires Docker to be installed first" unless docker_version

etcd_version = read_env 'ETCD_VERSION', 'latest'
raise "ETCD requires Docker to be installed first" unless docker_version
etcd_size = read_env 'ETCD_SIZE', 3
if etcd_size && etcd_version
  etcd_size = etcd_size.to_i
  raise "Not enough servers configured: stated #{nodes} nodes while requested #{etcd_size} etcd nodes" if etcd_size > nodes
else
  etcd_size = 0
end

box = read_env 'BOX', 'bento/debian-10' # must be debian-based
box_url = read_env 'BOX_URL', false # e.g. https://svn.ensisa.uha.fr/vagrant/k8s.json
# Box-dependent
vagrant_user = read_env 'VAGRANT_GUEST_USER', 'vagrant'
vagrant_group = read_env 'VAGRANT_GUEST_GROUP', 'vagrant'
vagrant_home = read_env 'VAGRANT_GUEST_HOME', '/home/vagrant'

host_itf = read_env 'ITF', false

leader_ip = (read_env 'MASTER_IP', "192.168.10.100").split('.').map {|nbr| nbr.to_i} # private ip ; public ip is to be set up with DHCP
hostname_prefix = read_env 'PREFIX', 'node'

upgrade = read_bool_env 'UPGRADE', false
guest_additions = read_bool_env 'GUEST_ADDITIONS', false

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
exposed_ip_script = "ip -4 addr list #{default_itf} |  grep -v secondary | grep inet | sed 's/.*inet\\s*\\([0-9.]*\\).*/\\1/'"
internal_ip_script = "ip -4 addr list #{internal_itf} |  grep -v secondary | grep inet | sed 's/.*inet\\s*\\([0-9.]*\\).*/\\1/'"

definitions = (1..nodes).map do |node_number|
    hostname = "%s%02d" % [hostname_prefix, node_number]
    ip = leader_ip.dup
    ip[-1] += node_number-1
    ip_str = ip.join('.')
    raise "Not enough addresses available for all nodes, e.g. can't assign IP #{ip_str} to #{hostname} ; lower NODES number or give another MASTER_IP" if ip[-1] > 255
    {:hostname => hostname, :ip => ip_str}
end
etcd_cluster = definitions[0, etcd_size].map { |definition| "#{definition[:hostname]}=http://#{definition[:ip]}:2380"}.join ',' if etcd_size > 0

if public
    require 'socket'
    vagrant_host = Socket.gethostname || Socket.ip_address_list.find { |ai| ai.ipv4? && !ai.ipv4_loopback? }.ip_address
    puts "this host is #{vagrant_host}"
    require 'digest/md5' # used later for machine id generation so that dhcp returns the same IP
end

pub_key = read_env 'PUBLIC_ROOT_KEY', 'AAAAB3NzaC1yc2EAAAADAQABAAABAQDFCEEemETfqtunwT8G2A+aaqJlXME99G0LtSk2Nd7ER1uPt54lY6uxCs+5lz6c6WXS58XPHNOOfz8F9iUgyJqOM97Dj9HOaSAdmE+xvOHa5lf8fUpeb3GhRNvp8vnwQDfKG3wdrMLUlZjqMbJnH63C/H5nwQ4LybbfLc9XtL8D7PQEGW5SbUaEmULNO46JydEUWgtGodjc6UHs0YVor8e89Up5uy5a0MGIQeB2B6y6rkVc2+aNwUka8bY3O9HuLlJmB+iYKu9IP/pVwy3Y733FRyB7XJJL4T1jsMZfjbQoyPoEGVU5EC8j8dUy+XkUfCe5dWY1wdNDG9oBbwWz1+B5'
priv_key = read_env 'PRIVATE_ROOT_KEY', '-----BEGIN OPENSSH PRIVATE KEY-----
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
    begin config_all.vm.box_url = box_url if box_url rescue nil end

    config_all.vm.synced_folder ".", "/vagrant", disabled: true
    config_all.vm.synced_folder ".", "/home/vagrant/sync", disabled: true

    root_hostname = definitions[0][:hostname]
    root_ip = definitions[0][:ip]

    config_all.vm.provider :virtualbox do |vb|
        #config_all.timezone.value = :host
        vb.check_guest_additions = guest_additions
        vb.functional_vboxsf     = false
        if Vagrant.has_plugin?("vagrant-vbguest") then
            config_all.vbguest.auto_update = upgrade
        end
    end

    # Generic
    config_all.vm.provision "Aliases", :type => "shell", :name => "Setting up aliases", :inline => "
        grep -q 'alias ll=' /etc/bash.bashrc || echo 'alias ll=\"ls -alh\"' >> /etc/bash.bashrc
    "
    config_all.vm.provision "Upgrade", :type => "shell", :name => "Upgrading system", :inline => "
        export APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1
        export DEBIAN_FRONTEND=noninteractive
        apt-get update
        apt-get dist-upgrade --yes
        apt-get -y autoremove
        apt-get -y autoclean
    " if upgrade

    # Referencing all IPs in /etc/hosts
    config_all.vm.provision "Network", :type => "shell", :name => "Configuring network", :inline => "
        echo 'nameserver 8.8.8.8 8.8.4.4' > /etc/resolv.conf
        sed -i 's/^DNS=.*/DNS=8.8.8.8 8.8.4.4/' /etc/systemd/resolved.conf
        sed -i '/^127\\.\\0\\.1\\.1/d' /etc/hosts
    "
    definitions.each do |node|
        config_all.vm.provision "#{node[:hostname]}Access", :type => "shell", :name  => "Referencing #{node[:hostname]}", :inline => "grep -q " + node[:hostname] + " /etc/hosts || echo \"" + node[:ip] + " " + node[:hostname] + "\" >> /etc/hosts"
    end

    # Auto SSH
    config_all.vm.provision "SSHRootAuthorizationFile", :type => "shell", :name => 'auto ssh', :inline => "mkdir -m 0700 -p /root/.ssh; touch /root/.ssh/authorized_keys; chmod 600 /root/.ssh/authorized_keys"
    (1..nodes).each do |node_number|
        node_name = definitions[node_number-1][:hostname]
        config_all.vm.provision "SSHRootAuthorizationFrom#{node_name}", :type => "shell", :name => "auto ssh from #{node_name}", :inline => "
            grep -q 'root@#{node_name}' /root/.ssh/authorized_keys || echo 'ssh-rsa #{pub_key} root@#{node_name}' >> /root/.ssh/authorized_keys
        "
    end

    # Docker Installation
    if docker_version
        config_all.vm.provision "DockerInstall", :type => "shell", :name => 'Installing Docker', :inline => "
            if ! which docker >/dev/null; then
                export APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1
                export DEBIAN_FRONTEND=noninteractive
                apt-get update
                apt-get install --yes apt-transport-https ca-certificates curl gnupg2 software-properties-common
                DIST=$(lsb_release -i -s  | tr '[:upper:]' '[:lower:]')
                curl -fsSL https://download.docker.com/linux/$DIST/gpg | apt-key add -
                apt-key fingerprint #{docker_repo_fingerprint}
                add-apt-repository \"deb [arch=amd64] https://download.docker.com/linux/$DIST $(lsb_release -cs) stable\"
                apt-get update
                if [ \"#{docker_version}\" == \"latest\" ]; then
                    DOCKER_VERSION='*';
                else
                    DOCKER_VERSION=$(apt-cache madison docker-ce | grep '#{docker_version}' | head -1 | awk '{print $3}')
                fi
                if [ \"#{containerd_version}\" == \"latest\" ]; then
                    CONTAINERD_VERSION='*'
                else
                    CONTAINERD_VERSION=$(apt-cache madison containerd.io | grep '#{containerd_version}' | head -1 | awk '{print $3}')
                fi
                echo \" == Installing Docker $DOCKER_VERSION and containerd $CONTAINERD_VERSION ==\"
                apt-get install --yes docker-ce=$DOCKER_VERSION docker-ce-cli=$DOCKER_VERSION containerd.io=$CONTAINERD_VERSION
                # apt-mark hold docker-ce docker-ce-cli containerd.io
            fi
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

    (1..nodes).each do |node_number|

        definition = definitions[node_number-1]
        hostname = definition[:hostname]
        ip = definition[:ip]
        master = node_number == 1

        config_all.vm.define hostname, primary: node_number == 1 do |config|
            config.vm.hostname = hostname
            config.vm.provider :virtualbox do |vb, override|
                vb.memory = if master then master_memory else memory end
                vb.cpus = if master then master_cpus else cpus end
                vb.customize [
                    'modifyvm', :id,
                    '--name', hostname,
                    '--cpuexecutioncap', '100',
                    '--paravirtprovider', 'kvm',
                    '--natdnshostresolver1', 'on',
                    '--natdnsproxy1', 'on',
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

            if compose_version && master
                config_all.vm.provision "DockerComposeInstall", :type => "shell", :name => 'Installing Docker Compose', :inline => "
                    if [ ! -x /usr/local/bin/docker-compose ]; then
                        if [ \"#{compose_version}\" == \"latest\" ]; then
                            COMPOSE_VERSION=$(curl -Isw \"%{redirect_url}\" --HEAD -o /dev/null \"https://github.com/docker/compose/releases/latest\")
                            COMPOSE_VERSION=${COMPOSE_VERSION##*/}
                        else
                            COMPOSE_VERSION=#{compose_version}
                        fi
                        echo \" == Installing Docker Compose ${COMPOSE_VERSION} == \"
                        curl -s -L \"https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)\" -o /usr/local/bin/docker-compose
                        chmod +x /usr/local/bin/docker-compose
                        curl -s -L https://raw.githubusercontent.com/docker/compose/${COMPOSE_VERSION}/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose
                    fi
                "
            end # Compose

            if etcd_version
                config_all.vm.provision "EtcdInstall", :type => "shell", :name => 'Installing etcd', :inline => "
                    if [ $(docker ps -qf 'name=etcd' | wc -l) == 0 ]; then
                        DATA=/var/lib/etcd/data
                        mkdir -p ${DATA}
                        ETCD_VERSION=#{etcd_version}
                        if [[ \"${ETCD_VERSION}\" != \"latest\" ]] && [[ \"${ETCD_VERSION}\" != v* ]]; then
                            ETCD_VERSION=\"v${ETCD_VERSION}\"
                        fi
                        IMG=\"quay.io/coreos/etcd:${ETCD_VERSION}\"
                        CLIENT_IP=$(#{exposed_ip_script})
                        PEER_IP=$(#{internal_ip_script})
                        echo \" == Running ETCD on ${CLIENT_IP} (internal comm on ${PEER_IP}) == \"
                        docker pull -q ${IMG}
                        docker run -d \
                            -p 2379:2379 \
                            -p ${PEER_IP}:2380:2380 \
                            --restart always \
                            --mount type=bind,source=${DATA},destination=/etcd-data \
                            --name etcd \
                            ${IMG} \
                            /usr/local/bin/etcd \
                            --name $(hostname) \
                            --data-dir /etcd-data \
                            --listen-client-urls http://0.0.0.0:2379 \
                            --advertise-client-urls http://${CLIENT_IP}:2379 \
                            --listen-peer-urls http://0.0.0.0:2380 \
                            --initial-advertise-peer-urls http://${PEER_IP}:2380 \
                            --initial-cluster-state new \
                            --initial-cluster #{etcd_cluster}
                            
                    fi
                "

                config_all.vm.provision "EtcdctlInstall", :type => "shell", :name => 'Installing etcdctl', :inline => "
                    if ! which etcdctl >/dev/null; then
                        docker cp etcd:$(docker exec etcd which etcdctl) /usr/local/bin
                    fi
                "
            end # etcd

        end # Config node
    end # node
end # all
