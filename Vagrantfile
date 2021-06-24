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

memory = read_env 'MEM', '1024'
master_memory = read_env 'MASTER_MEM', '1024'
cpus = read_env 'CPU', '1'
master_cpus = read_env 'MASTER_CPU', 1
nodes = (read_env 'NODES', 3).to_i
raise "There should be at least one node and at most 255 while prescribed #{nodes} ; you can set up node number like this: NODES=2 vagrant up" unless nodes.is_a? Integer and nodes >= 1 and nodes <= 255
leader_ip = (read_env 'MASTER_IP', "192.168.99.99").split('.').map {|nbr| nbr.to_i} # private ip
hostname_prefix = read_env 'PREFIX', 'node'

docker_version = read_env 'DOCKER_VERSION', 'latest'
docker_repo_fingerprint = read_env 'DOCKER_APT_FINGERPRINT', '0EBFCD88'
containerd_version = read_env 'CONTAINERD_VERSION', 'latest'
compose_version = read_env 'COMPOSE_VERSION', 'latest'
raise "Docker Compose requires Docker to be installed first" unless docker_version

etcd_version = read_env 'ETCD_VERSION', 'latest'
raise "ETCD requires Docker to be installed first" unless docker_version
etcd_size = read_env 'ETCD_SIZE', 3
if etcd_size && etcd_size > 0 && etcd_version
  raise "Not enough servers configured: stated #{nodes} nodes while requested #{etcd_size} etcd nodes" if etcd_size > nodes
else
    etcd_version = false
    etcd_size = 0
end
etcdctl_api = read_env 'ETCDCTL_API', 3
if read_bool_env 'ETCD_PORT'
    etcd_port = read_env 'ETCD_PORT', 0
    etcd_port = 2379 unless etcd_port.is_a? Integer
else
    etcd_port = 0
end

swarm = read_bool_env 'SWARM' # swarm mode is disabled by default ; use SWARM=on for setting up (only at node creation of leader)
raise "You shouldn't disable etcd when swarm mode is enabled" if swarm && ! etcd_version
swarm_managers = (ENV['SWARM_MANAGERS'] || "#{hostname_prefix}01").split(',').map { |node| node.strip }.select { |node| node and node != ''}

box = read_env 'BOX', 'bento/debian-10' # must be debian-based
box_url = read_env 'BOX_URL', false # e.g. https://svn.ensisa.uha.fr/vagrant/k8s.json
# Box-dependent
vagrant_user = read_env 'VAGRANT_GUEST_USER', 'vagrant'
vagrant_group = read_env 'VAGRANT_GUEST_GROUP', 'vagrant'
vagrant_home = read_env 'VAGRANT_GUEST_HOME', '/home/vagrant'

host_itf = read_env 'ITF', false

upgrade = read_bool_env 'UPGRADE', false
guest_additions = read_bool_env 'GUEST_ADDITIONS', false

default_itf = read_env 'DEFAULT_ITF', 'eth1'  # depends on chosen box
exposed_ip_script = "ip -4 addr list #{default_itf} |  grep -v secondary | grep inet | sed 's/.*inet\\s*\\([0-9.]*\\).*/\\1/'"

definitions = (1..nodes).map do |node_number|
    hostname = "%s%02d" % [hostname_prefix, node_number]
    ip = leader_ip.dup
    ip[-1] += node_number-1
    ip_str = ip.join('.')
    raise "Not enough addresses available for all nodes, e.g. can't assign IP #{ip_str} to #{hostname} ; lower NODES number or give another MASTER_IP" if ip[-1] > 255
    {:hostname => hostname, :ip => ip_str}
end
etcd_cluster = definitions[0, etcd_size].map { |definition| "#{definition[:hostname]}=http://#{definition[:ip]}:2380"}.join ',' if etcd_size > 0

Vagrant.configure("2") do |config_all|
    # forward ssh agent to easily ssh into the different machines
    #config_all.ssh.forward_agent = true
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
        apt-get update -qqq
        apt-get dist-upgrade --yes -qqq
        apt-get -y -qqq autoremove
        apt-get -y -qqq autoclean
    " if upgrade

    definitions.each do |node|
        config_all.vm.provision "#{node[:hostname]}Access", :type => "shell", :name  => "Referencing #{node[:hostname]}", :inline => "grep -q " + node[:hostname] + " /etc/hosts || echo \"" + node[:ip] + " " + node[:hostname] + "\" >> /etc/hosts"
    end

    # Docker Installation
    if docker_version
        config_all.vm.provision "DockerInstall", :type => "shell", :name => 'Installing Docker', :inline => "
            if ! which docker >/dev/null; then
                export APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1
                export DEBIAN_FRONTEND=noninteractive
                apt-get update -qqq
                apt-get install --yes -qqq apt-transport-https ca-certificates curl gnupg2 software-properties-common
                DIST=$(lsb_release -i -s  | tr '[:upper:]' '[:lower:]')
                curl -fsSL https://download.docker.com/linux/$DIST/gpg | apt-key add -
                apt-key fingerprint #{docker_repo_fingerprint}
                add-apt-repository \"deb [arch=amd64] https://download.docker.com/linux/$DIST $(lsb_release -cs) stable\"
                apt-get update -qqq
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
                apt-get install --yes -qqq docker-ce=$DOCKER_VERSION docker-ce-cli=$DOCKER_VERSION containerd.io=$CONTAINERD_VERSION
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

            config.vm.network :private_network, ip: ip

            if compose_version && master
                config.vm.provision "DockerComposeInstall", :type => "shell", :name => 'Installing Docker Compose', :inline => "
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

            if etcd_version && node_number <= etcd_size
                config.vm.provision "EtcdInstall", :type => "shell", :name => 'Installing etcd', :inline => "
                    if [ $(docker ps -qf 'name=etcd' | wc -l) == 0 ]; then
                        DATA=/var/lib/etcd/data
                        mkdir -p ${DATA}
                        ETCD_VERSION=#{etcd_version}
                        if [[ \"${ETCD_VERSION}\" != \"latest\" ]] && [[ \"${ETCD_VERSION}\" != v* ]]; then
                            ETCD_VERSION=\"v${ETCD_VERSION}\"
                        fi
                        IMG=\"quay.io/coreos/etcd:${ETCD_VERSION}\"
                        CLIENT_IP=#{ip}
                        PEER_IP=#{ip}
                        echo \" == Running ETCD on ${CLIENT_IP} == \"
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

                config.vm.provision "EtcdctlInstall", :type => "shell", :name => 'Installing etcdctl', :inline => "
                    if ! which etcdctl >/dev/null; then
                        docker cp etcd:$(docker exec etcd which etcdctl) /usr/local/bin
                    fi
                    grep -q 'ETCDCTL_API=' #{vagrant_home}/.bashrc || echo \"ETCDCTL_API=#{etcdctl_api}\" >> #{vagrant_home}/.bashrc
                "

                if master && etcd_port > 0
                    config.vm.network "forwarded_port", guest: 2379, host: etcd_port
                end # etcd master
            end # etcd

            if swarm
          
                role = if node_number == 1 or swarm_managers.include? hostname then 'manager' else 'worker' end
                config.vm.provision "EtcdctlInstall", :type => "shell", run: "always", :name => "Checking Docker SWARM", :inline => <<-EOF
\# Checking whether a swarm already exists
export ETCDCTL_API=2
echo "Waiting for etcd to be availabble"
until etcdctl cluster-health ; do
	sleep 1
done
etcdctl ls | grep vagrant-swarm >/dev/null || etcdctl mkdir vagrant-swarm
MANAGERS=$(etcdctl ls /vagrant-swarm | grep '_address$' | sed 's;^/vagrant-swarm/swarm_\\(.*\\)_address;\\1;')
if [ -z "$MANAGERS" ]; then
    \# first node to appear: creating swarm (if not already)
    echo " == Initializing Docker SWARM ==";
    docker swarm init --advertise-addr #{ip} --listen-addr #{ip}
elif [ "0" != $(docker node list 1>/dev/null 2>&1;echo $?) ]; then
    echo \"joining swarm as #{role}\"
    \# iterating over manager nodes to join
    for MANAGER in $MANAGERS; do
        docker swarm join --token $(etcdctl get "/vagrant-swarm/swarm_${MANAGER}_token_#{role}") $(etcdctl get "/vagrant-swarm/swarm_${MANAGER}_address") && break
    done
fi
if [ "#{role}" == "manager" ]; then
    \# Storing join token in etcd
    etcdctl get /vagrant-swarm/swarm_#{hostname}_address 1>/dev/null 2>&1 || etcdctl set /vagrant-swarm/swarm_#{hostname}_address $(docker swarm join-token worker | grep 'docker swarm join' | grep -o '[0-9]*\\.[0-9]*\\.[0-9]*\\.[0-9]*:[0-9]*$')
    etcdctl get /vagrant-swarm/swarm_#{hostname}_token_worker 1>/dev/null 2>&1 || etcdctl set /vagrant-swarm/swarm_#{hostname}_token_worker $(docker swarm join-token -q worker)
    etcdctl get /vagrant-swarm/swarm_#{hostname}_token_manager 1>/dev/null 2>&1 || etcdctl set /vagrant-swarm/swarm_#{hostname}_token_manager $(docker swarm join-token -q manager)
fi
      EOF
         
                # TODO: docker service create     --name portainer     --publish 9000:9000     --constraint 'node.role == manager'     --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock     portainer/portainer     -H unix:///var/run/docker.sock
             
            end # swarm

        end # Config node
    end # node
end # all
