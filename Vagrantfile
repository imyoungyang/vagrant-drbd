# -*- mode: ruby -*-
# vi: set ft=ruby :
# http://qiita.com/Yuki-Inamoto/items/ce65468abba10bce58f0
# https://www.theurbanpenguin.com/create-3-node-drbd-9-cluster-using-drbd-manage/
# http://www.thegeekstuff.com/2008/11/3-steps-to-perform-ssh-login-without-password-using-ssh-keygen-ssh-copy-id/

servers=[
  {
    :hostname => "drbd0",
    :ip => "192.168.100.60",
    :disk => "./drbd0.vdi"
  },{
   :hostname => "drbd1",
   :ip => "192.168.100.61",
   :disk => "./drbd1.vdi"
 },{
   :hostname => "drbd2",
   :ip => "192.168.100.62",
   :disk => "./drbd2.vdi"
 }
]

$script = <<SCRIPT
  sudo su root
  yum check-update
  yum makecache fast

  export LANGUAGE="en_US.UTF-8"
  export LC_ALL="en_US.UTF-8"
  echo "nameserver 8.8.8.8" >> /etc/resolv.conf 

  ## add user/group
  groupadd -f -g 1000 elasticsearch && useradd elasticsearch -ou 1000 -g 1000

  ## Optional: Set timezone
  ## timedatectl set-timezone Asia/Taipei

  ## add docker repo to get correct docker-ce version
  yum install -y yum-utils
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

  ## Enable EPEL repository
  rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
  rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

  ## install related packages.
  ## yum install -y net-tools git wget curl python-pip telnet vim device-mapper-persistent-data lvm2 docker-ce ntp nfs-utils
  yum install -y net-tools bind-utils git wget curl python-pip \
  telnet vim device-mapper-persistent-data lvm2 docker-ce ntp nfs-utils \
  libxslt drbd90-utils kmod-drbd90 pygobject2 help2man

  ## install sshpass
  wget http://ftp.tu-chemnitz.de/pub/linux/dag/redhat/el7/en/x86_64/rpmforge/RPMS/rpmforge-release-0.5.3-1.el7.rf.x86_64.rpm
  rpm -Uvh rpmforge-release*rpm
  yum install -y sshpass

  ## install drbdmanage package
  git clone --recursive http://git.drbd.org/drbdmanage.git
  echo "# vgcreate drbdpool" > /etc/drbdmanaged.cfg
  cd drbdmanage
  # make
  make install
  cd ..
  rm -rf drbdmanage

  ## services
  systemctl start ntpd
  systemctl start nfs-server.service

  systemctl enable ntpd
  systemctl enable nfs-server.service

  # add /etc/hosts
  echo '192.168.100.60  drbd0 drbd0' >> /etc/hosts
  echo '192.168.100.61  drbd1 drbd1' >> /etc/hosts
  echo '192.168.100.62  drbd2 drbd2' >> /etc/hosts

  # create drbdpool
  vgcreate drbdpool /dev/sdb
  vgdisplay -c
  
  ## drdb cluster
  if [ "$(hostname)" == "drbd2" ];then

    ## install sshpass
    wget http://ftp.tu-chemnitz.de/pub/linux/dag/redhat/el7/en/x86_64/rpmforge/RPMS/rpmforge-release-0.5.3-1.el7.rf.x86_64.rpm
    rpm -Uvh rpmforge-release*rpm
    yum install -y sshpass

    # drbd2 -> drbd0/1
    echo | ssh-keygen -N ''
    ssh-keyscan drbd0 >> ~/.ssh/known_hosts
    ssh-keyscan drbd1 >> ~/.ssh/known_hosts
    sshpass -p vagrant ssh-copy-id -i ~/.ssh/id_rsa.pub drbd0
    sshpass -p vagrant ssh-copy-id -i ~/.ssh/id_rsa.pub drbd1

    ssh drbd0 "echo | ssh-keygen -N ''"
    ssh drbd0 "ssh-keyscan drbd1 >> ~/.ssh/known_hosts"
    ssh drbd0 "ssh-keyscan drbd2 >> ~/.ssh/known_hosts"
    ssh drbd0 "sshpass -p vagrant ssh-copy-id -i ~/.ssh/id_rsa.pub drbd1"
    ssh drbd0 "sshpass -p vagrant ssh-copy-id -i ~/.ssh/id_rsa.pub drbd2"

    ssh drbd1 "echo | ssh-keygen -N ''"
    ssh drbd1 "ssh-keyscan drbd0 >> ~/.ssh/known_hosts"
    ssh drbd1 "ssh-keyscan drbd2 >> ~/.ssh/known_hosts"
    ssh drbd1 "sshpass -p vagrant ssh-copy-id -i ~/.ssh/id_rsa.pub drbd0"
    ssh drbd1 "sshpass -p vagrant ssh-copy-id -i ~/.ssh/id_rsa.pub drbd2"

    ## init cluster
    echo "yes" | drbdmanage init 192.168.100.62
    echo "yes" | drbdmanage add-node drbd0 192.168.100.60
    echo "yes" | drbdmanage add-node drbd1 192.168.100.61

    drbdmanage update-pool
    drbdmanage list-nodes
    drbd-overview
    drbdadm status

    ## Add Cluster Resource
    drbdmanage add-volume esdata 1GB --deploy 3
    drbdmanage list-volumes
    # # drbdmanage add-resource esdata
    # # drbdmanage add-volume esdata 1GB
    # # drbdmanage deploy-resource esdata 3
    lsblk
  
    ## format disk
    mkfs.xfs /dev/drbd100
    mkdir -p /mnt/esdata
    mount /dev/drbd100 /mnt/esdata

    ## nfs server
    chown -R elasticsearch:elasticsearch /mnt/esdata
    chmod 755 /mnt/esdata
    echo "/mnt/esdata 192.168.100.0/24(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)" >> /etc/exports
    exportfs -a
  fi

  #debug
  # cat /etc/drbd.d/drbdctrl.res

  ## drbdmanage howto-join drbd1
  # drbdmanage join -p 6999 192.168.100.61 1 drbd0 192.168.100.60 0 OxIptyn+6hdVAu1R+Qy4
  # drbdmanage join -p 6999 192.168.100.62 2 drbd0 192.168.100.60 0 OxIptyn+6hdVAu1R+Qy4
  
SCRIPT

# This defines the version 2 of vagrant
Vagrant.configure(2) do |config|
  servers.each do |machine|
    config.vm.define machine[:hostname] do |node|
      node.vm.box = "bento/centos-7.3"
      node.vm.hostname = machine[:hostname]
      node.vm.network "private_network", ip: machine[:ip]
      config.vm.synced_folder ".", "/vagrant", type: "nfs", :linux__nfs_options => ["no_root_squash"], :map_uid => 0, :map_gid => 0
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 2048
        vb.cpus = 1
        unless File.exist?(machine[:disk])
          vb.customize ['createhd', '--filename', machine[:disk], '--variant', 'Fixed', '--size', 5 * 1024]
        end
        vb.customize ['storageattach', :id,  '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', machine[:disk]]
      end
      node.vm.provision "shell" do |s|
        s.inline = $script
      end
    end
  end
end
