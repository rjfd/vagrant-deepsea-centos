# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

raise Vagrant::Errors::VagrantError.new,
  "'settings.yml' file not found.\n\n" \
  "Please define your 'settings.yml'. " \
  "See 'settings.sample.yml' for an example." unless File.exist?("settings.yml")

settings = YAML.load_file('settings.yml')

num_volumes = settings.has_key?('vm_num_volumes') ?
              settings['vm_num_volumes'] : 2

volume_size = settings.has_key?('vm_volume_size') ?
              settings['vm_volume_size'] : '8G'

num_nodes = settings.has_key?('num_nodes') ?
            settings['num_nodes'] : 3

deploy_ceph = settings.has_key?('deploy_ceph') ? settings['deploy_ceph'] : false

raise Vagrant::Errors::VagrantError.new,
  "Invalid 'num_nodes' (#{num_nodes}).\n\n" \
  "Supported values: 2 or 3" unless ([2,3]).include?(num_nodes)

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false

  config.vm.box = "centos/7"

  config.vm.provider "libvirt" do |lv|
    if settings.has_key?('libvirt_host') then
      lv.host = settings['libvirt_host']
    end
    if settings.has_key?('libvirt_user') then
      lv.username = settings['libvirt_user']
    end
    if settings.has_key?('libvirt_use_ssl') then
      lv.connect_via_ssh = true
    end

    lv.memory = settings.has_key?('vm_memory') ? settings['vm_memory'] : 4096
    lv.cpus = settings.has_key?('vm_cpus') ? settings['vm_cpus'] : 2
    if settings.has_key?('vm_storage_pool') then
      lv.storage_pool_name = settings['vm_storage_pool']
    end
  end
  config.vm.provider :virtualbox do |vb|
    vb.memory = settings.has_key?('vm_memory') ? settings['vm_memory'] : 4096
    vb.cpus = settings.has_key?('vm_cpus') ? settings['vm_cpus'] : 2
  end

  config.vm.define :node1 do |node|
    node.vm.hostname = "node1.oa.local"
    node.vm.network :private_network, ip: "192.168.100.201"
    node.vm.network :private_network, ip: "192.168.170.201"

    node.vm.provision "file", source: "keys/id_rsa",
                              destination:".ssh/id_rsa"
    node.vm.provision "file", source: "keys/id_rsa.pub",
                              destination:".ssh/id_rsa.pub"

    node.vm.synced_folder ".", "/vagrant", disabled: true

    node.vm.provider "libvirt" do |lv|
      (1..num_volumes).each do |d|
        lv.storage :file, size: volume_size, type: 'qcow2'
      end
    end
    node.vm.provider :virtualbox do |vb|
      for i in 1..num_volumes do
        file_to_disk = "./disks/#{node.vm.hostname}-disk#{i}.vmdk"
        unless File.exist?(file_to_disk)
          vb.customize ['createmedium', 'disk', '--filename', file_to_disk,
            '--size', volume_size]
          vb.customize ['storageattach', :id,
            '--storagectl', 'SATA Controller',
            '--port', i, '--device', 0,
            '--type', 'hdd', '--medium', file_to_disk]
        end
      end
    end

    node.vm.provision "shell", inline: <<-SHELL
      echo "192.168.100.200 salt salt.oa.local" >> /etc/hosts
      echo "192.168.100.201 node1 node1.oa.local" >> /etc/hosts
      echo "192.168.100.202 node2 node2.oa.local" >> /etc/hosts
      echo "192.168.100.203 node3 node3.oa.local" >> /etc/hosts
      cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
      mkdir /root/.ssh
      chmod 600 /home/vagrant/.ssh/id_rsa
      cp /home/vagrant/.ssh/id_rsa* /root/.ssh/
      cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
      hostname node1

      ssh-keyscan -H salt >> ~/.ssh/known_hosts
      ssh-keyscan -H node1 >> ~/.ssh/known_hosts
      ssh-keyscan -H node3 >> ~/.ssh/known_hosts

      yum install -y centos-release-ceph-luminous

      yum install -y https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
      yum install -y salt-minion
      systemctl enable salt-minion
      systemctl start salt-minion

      touch /tmp/ready
    SHELL
  end

  if num_nodes > 1 then
    config.vm.define :node2 do |node|
        node.vm.hostname = "node2.oa.local"
        node.vm.network :private_network, ip: "192.168.100.202"
        node.vm.network :private_network, ip: "192.168.170.202"

        node.vm.provision "file", source: "keys/id_rsa",
                                destination:".ssh/id_rsa"
        node.vm.provision "file", source: "keys/id_rsa.pub",
                                destination:".ssh/id_rsa.pub"

        node.vm.synced_folder ".", "/vagrant", disabled: true

        node.vm.provider "libvirt" do |lv|
          (1..num_volumes).each do |d|
            lv.storage :file, size: volume_size, type: 'qcow2'
          end
        end
        node.vm.provider :virtualbox do |vb|
          for i in 1..num_volumes do
            file_to_disk = "./disks/#{node.vm.hostname}-disk#{i}.vmdk"
            unless File.exist?(file_to_disk)
              vb.customize ['createmedium', 'disk', '--filename', file_to_disk,
                '--size', volume_size]
              vb.customize ['storageattach', :id,
                '--storagectl', 'SATA Controller',
                '--port', i, '--device', 0,
                '--type', 'hdd', '--medium', file_to_disk]
            end
          end
        end

      node.vm.provision "shell", inline: <<-SHELL
        echo "192.168.100.200 salt salt.oa.local" >> /etc/hosts
        echo "192.168.100.201 node1 node1.oa.local" >> /etc/hosts
        echo "192.168.100.202 node2 node2.oa.local" >> /etc/hosts
        echo "192.168.100.203 node3 node3.oa.local" >> /etc/hosts
        cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
        mkdir /root/.ssh
        chmod 600 /home/vagrant/.ssh/id_rsa
        cp /home/vagrant/.ssh/id_rsa* /root/.ssh/
        cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
        hostname node2

        ssh-keyscan -H salt >> ~/.ssh/known_hosts
        ssh-keyscan -H node1 >> ~/.ssh/known_hosts
        ssh-keyscan -H node3 >> ~/.ssh/known_hosts

        yum install -y centos-release-ceph-luminous

        yum install -y https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
        yum install -y salt-minion
        systemctl enable salt-minion
        systemctl start salt-minion

        touch /tmp/ready
        SHELL
    end
  end

  if num_nodes > 2 then
    config.vm.define :node3 do |node|
        node.vm.hostname = "node3.oa.local"
        node.vm.network :private_network, ip: "192.168.100.203"
        node.vm.network :private_network, ip: "192.168.170.203"

        node.vm.provision "file", source: "keys/id_rsa",
                                destination:".ssh/id_rsa"
        node.vm.provision "file", source: "keys/id_rsa.pub",
                                destination:".ssh/id_rsa.pub"

        node.vm.synced_folder ".", "/vagrant", disabled: true

        node.vm.provider "libvirt" do |lv|
          (1..num_volumes).each do |d|
            lv.storage :file, size: volume_size, type: 'qcow2'
          end
        end
        node.vm.provider :virtualbox do |vb|
          for i in 1..num_volumes do
            file_to_disk = "./disks/#{node.vm.hostname}-disk#{i}.vmdk"
            unless File.exist?(file_to_disk)
            vb.customize ['createmedium', 'disk', '--filename', file_to_disk,
                '--size', volume_size]
            vb.customize ['storageattach', :id,
                '--storagectl', 'SATA Controller',
                '--port', i, '--device', 0,
                '--type', 'hdd', '--medium', file_to_disk]
            end
          end
        end

        node.vm.provision "shell", inline: <<-SHELL
        echo "192.168.100.200 salt salt.oa.local" >> /etc/hosts
        echo "192.168.100.201 node1 node1.oa.local" >> /etc/hosts
        echo "192.168.100.202 node2 node2.oa.local" >> /etc/hosts
        echo "192.168.100.203 node3 node3.oa.local" >> /etc/hosts
        cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
        mkdir /root/.ssh
        chmod 600 /home/vagrant/.ssh/id_rsa
        cp /home/vagrant/.ssh/id_rsa* /root/.ssh/
        cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys

        ssh-keyscan -H salt >> ~/.ssh/known_hosts
        ssh-keyscan -H node1 >> ~/.ssh/known_hosts
        ssh-keyscan -H node2 >> ~/.ssh/known_hosts

        yum install -y centos-release-ceph-luminous

        yum install -y https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
        yum install -y salt-minion
        systemctl enable salt-minion
        systemctl start salt-minion

        touch /tmp/ready
        SHELL
    end
  end

  config.vm.define :salt do |salt|
    salt.vm.hostname = "salt.oa.local"
    salt.vm.network :private_network, ip: "192.168.100.200"

    salt.vm.provision "file", source: "keys/id_rsa",
                              destination:".ssh/id_rsa"
    salt.vm.provision "file", source: "keys/id_rsa.pub",
                              destination:".ssh/id_rsa.pub"

    salt.vm.synced_folder ".", "/vagrant", disabled: true

    roles = {
        "mon" => "node*",
        "igw" => ("node[12]*" if num_nodes > 1) || "node*",
        "rgw" => ("node[13]*" if num_nodes > 2) || "node*",
        "mds" => ("node[23]*" if num_nodes > 2) || "node*",
        "ganesha" => ("node[23]*" if num_nodes > 2) || "node*",
        "mgr" => ("node[12]*" if num_nodes > 1) || "node*"
    }

    salt.vm.provision "shell", inline: <<-SHELL
      echo "192.168.100.200 salt salt.oa.local" >> /etc/hosts
      echo "192.168.100.201 node1 node1.oa.local" >> /etc/hosts
      echo "192.168.100.202 node2 node2.oa.local" >> /etc/hosts
      echo "192.168.100.203 node3 node3.oa.local" >> /etc/hosts
      cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
      mkdir /root/.ssh
      chmod 600 /home/vagrant/.ssh/id_rsa
      cp /home/vagrant/.ssh/id_rsa* /root/.ssh/
      cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
      hostname salt

      yum install -y centos-release-ceph-luminous
      yum install -y https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
      yum install -y salt-master salt-minion ntp
      yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/p/python-click-6.3-1.el7.noarch.rpm
      yum install -y yum-plugin-copr
      yum copr enable rjdias/home -y

      systemctl enable ntpd
      systemctl start ntpd

      sed -i 's/#worker_threads: 5/worker_threads: 10/g' /etc/salt/master

      systemctl enable salt-master
      systemctl start salt-master
      sleep 5
      systemctl enable salt-minion
      systemctl start salt-minion

      while : ; do
        PROVISIONED_NODES=`ls -l /tmp/ready-* 2>/dev/null | wc -l`
        echo "waiting for nodes (${PROVISIONED_NODES}/#{num_nodes})";
        [[ "${PROVISIONED_NODES}" != "#{num_nodes}" ]] || break
        sleep 5;
        scp -o StrictHostKeyChecking=no node1:/tmp/ready /tmp/ready-node1 2>/dev/null;
        scp -o StrictHostKeyChecking=no node2:/tmp/ready /tmp/ready-node2 2>/dev/null;
        scp -o StrictHostKeyChecking=no node3:/tmp/ready /tmp/ready-node3 2>/dev/null;
      done
      sleep 5

      salt-key -Ay

      yum install -y deepsea

      cat > /tmp/policy.cfg <<EOF
# Cluster assignment
cluster-ceph/cluster/*.sls
# Hardware Profile
profile-default/cluster/*.sls
profile-default/stack/default/ceph/minions/*yml
# Common configuration
config/stack/default/global.yml
config/stack/default/ceph/cluster.yml
# Role assignment
role-master/cluster/salt*.sls
role-admin/cluster/salt*.sls
role-mon/cluster/#{roles['mon']}.sls
#role-igw/cluster/#{roles['igw']}.sls
role-rgw/cluster/#{roles['rgw']}.sls
role-mds/cluster/#{roles['mds']}.sls
role-mon/stack/default/ceph/minions/#{roles['mon']}.yml
role-ganesha/cluster/#{roles['ganesha']}.sls
role-mgr/cluster/#{roles['mgr']}.sls
EOF

      echo "deepsea_minions: '*'" > /srv/pillar/ceph/deepsea_minions.sls
    SHELL

    if deploy_ceph then
      salt.vm.provision "shell", inline: <<-SHELL
        sleep 2
        echo "[DeepSea] Stage 0 - prep"
        deepsea stage run ceph.stage.prep --simple-output

        sleep 2
        echo "[DeepSea] Stage 1 - discovery"
        deepsea stage run ceph.stage.discovery --simple-output

        cp /tmp/policy.cfg /srv/pillar/ceph/proposals

        sleep 2
        echo "[DeepSea] Stage 2 - configure"
        deepsea stage run ceph.stage.configure --simple-output

        sleep 2
        echo "[DeepSea] Stage 3 - deploy"
        DEV_ENV='true' deepsea stage run ceph.stage.deploy --simple-output

        sleep 2
        echo "[DeepSea] Stage 4 - services"
        DEV_ENV='true' deepsea stage run ceph.stage.services --simple-output
      SHELL
    end
  end
end
