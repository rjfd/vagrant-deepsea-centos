# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

raise Vagrant::Errors::VagrantError.new,
  "'settings.yml' file not found.\n\n" \
  "Please define your 'settings.yml'. " \
  "See 'settings.sample.yml' for an example." unless File.exist?("settings.yml")

settings = YAML.load_file('settings.yml')

openattic_repo = settings.has_key?('openattic_repo') ?
                 settings['openattic_repo'] : "~/openattic"

deepsea_repo = settings.has_key?('deepsea_repo') ?
               settings['deepsea_repo'] : "~/DeepSea"

openattic_docker_repo = settings.has_key?('openattic_docker_repo') ?
                        settings['openattic_docker_repo'] :
                        'https://github.com/openattic/openattic-docker.git'

openattic_docker_branch = settings.has_key?('openattic_docker_branch') ?
                          settings['openattic_docker_branch'] : 'master'

num_volumes = settings.has_key?('vm_num_volumes') ?
              settings['vm_num_volumes'] : 2

volume_size = settings.has_key?('vm_volume_size') ?
              settings['vm_volume_size'] : '8G'

nfs_auto_export = settings.has_key?('nfs_auto_export') ?
                  settings['nfs_auto_export'] : true

build_openattic_docker_image = settings.has_key?('build_openattic_docker_image') ?
                               settings['build_openattic_docker_image'] : false

create_openattic_node = settings.has_key?('create_openattic_node') ?
                        settings['create_openattic_node'] : false

num_nodes = settings.has_key?('num_nodes') ?
            settings['num_nodes'] : 3

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
  
  if create_openattic_node then
    config.vm.define :openattic do |node|
      node.vm.hostname = "openattic.oa.local"
      node.vm.network :private_network, ip: "192.168.100.204"

      node.vm.provision "file", source: "keys/id_rsa",
                                destination:".ssh/id_rsa"
      node.vm.provision "file", source: "keys/id_rsa.pub",
                                destination:".ssh/id_rsa.pub"

      node.vm.synced_folder ".", "/vagrant", disabled: true


      node.vm.provision "shell", inline: <<-SHELL
        echo "192.168.100.204 openattic openattic.oa.local" >> /etc/hosts
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

        while : ; do
          PROVISIONED_NODES=`ls -l /tmp/ready-salt 2>/dev/null | wc -l`
          echo "waiting for salt (${PROVISIONED_NODES}/1)";
          [[ "${PROVISIONED_NODES}" != "1" ]] || break
          sleep 10;
          scp -o StrictHostKeyChecking=no salt:/tmp/ready /tmp/ready-salt 2>/dev/null;
        done

        sleep 5

        scp -o StrictHostKeyChecking=no -r salt:/etc/ceph /etc
        chmod -R 664 /etc/ceph
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

    salt.vm.provision "file", source: "bin",
                              destination:"."

    salt.vm.synced_folder openattic_repo, '/home/vagrant/openattic', type: 'nfs',
                            :nfs_export => nfs_auto_export,
                            :mount_options => ['nolock,vers=3,udp,noatime,actimeo=1'],
                            :linux__nfs_options => ['rw','no_subtree_check','all_squash','insecure']

    salt.vm.synced_folder deepsea_repo, '/home/vagrant/DeepSea', type: 'nfs',
                            :nfs_export => nfs_auto_export,
                            :mount_options => ['nolock,vers=3,udp,noatime,actimeo=1'],
                            :linux__nfs_options => ['rw','no_subtree_check','all_squash','insecure']

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

      chmod 755 -R bin/

      yum install -y centos-release-ceph-luminous

      yum install -y https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
      yum install -y salt-master salt-minion

      yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/p/python-click-6.3-1.el7.noarch.rpm

      # systemctl enable docker
      # systemctl restart docker

      yum install -y ntp
      systemctl enable ntpd
      systemctl start ntpd

      systemctl enable salt-master
      systemctl start salt-master
      sleep 5
      systemctl enable salt-minion
      systemctl start salt-minion

      # git clone #{openattic_docker_repo} openattic-docker
      # cd openattic-docker
      # git checkout #{openattic_docker_branch}
      # cd openattic-dev/opensuse_leap_42.3
      # [[ "#{build_openattic_docker_image}" == "true" ]] && docker build -t openattic-dev .

      # SuSEfirewall2 off

      while : ; do
        PROVISIONED_NODES=`ls -l /tmp/ready-* 2>/dev/null | wc -l`
        echo "waiting for nodes (${PROVISIONED_NODES}/#{num_nodes})";
        [[ "${PROVISIONED_NODES}" != "#{num_nodes}" ]] || break
        sleep 10;
        scp -o StrictHostKeyChecking=no node1:/tmp/ready /tmp/ready-node1 2>/dev/null;
        scp -o StrictHostKeyChecking=no node2:/tmp/ready /tmp/ready-node2 2>/dev/null;
        scp -o StrictHostKeyChecking=no node3:/tmp/ready /tmp/ready-node3 2>/dev/null;
      done

      sleep 5

      salt-key -Ay

      cd /home/vagrant
      sleep 5
      umount DeepSea
      mount.nfs 192.168.1.102:/home/rdias/Work/DeepSea DeepSea
      sleep 5
      cd DeepSea
      if [[ -e Makefile ]]; then
         make install

         sed -i 's/#worker_threads: 5/worker_threads: 10/g' /etc/salt/master

         systemctl restart salt-master
         systemctl restart salt-minion
         sleep 5

         sleep 2
         echo "[DeepSea] Stage 0 - prep"
         deepsea stage run ceph.stage.prep --simple-output

         sleep 2
         echo "[DeepSea] Stage 1 - discovery"
         deepsea stage run ceph.stage.discovery --simple-output
         sleep 2

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
         cp /tmp/policy.cfg /srv/pillar/ceph/proposals

         echo "[DeepSea] Stage 2 - configure"
         deepsea stage run ceph.stage.configure --simple-output

         sleep 5
         echo "[DeepSea] Stage 3 - deploy"
         DEV_ENV='true' deepsea stage run ceph.stage.deploy --simple-output

         sleep 5
         echo "[DeepSea] Stage 4 - services"
         DEV_ENV='true' deepsea stage run ceph.stage.4 --simple-output

         chmod 644 /etc/ceph/ceph.client.admin.keyring

         touch /tmp/ready
      fi
    SHELL
  end
end
