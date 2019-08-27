### config section

# number of performance module machines to set up
$num_pm = 2

# number of user module machines to set up
$num_um = 2

# network address base, make sure it does not collide with other VM
# setups on the same machine
$net_base = "10.42.23"

### config section ends, no changes should be needed to the below

## preparations

# create /etc/hosts entries for all VMs 
# UM ip addresses start with .101
# PM ip addresses start with .201

$hostfile = ""

(1..$num_um).each do |i|
  $hostfile += "#{$net_base}.#{i+100} um-#{i}\n"
end

(1..$num_pm).each do |i|
  $hostfile += "#{$net_base}.#{i+200} pm-#{i}\n"
end

## actual VM config setup


Vagrant.configure("2") do |config|
  # base box is hard coded for now
  # TODO: make configurable?
  config.vm.box = "ubuntu/bionic64"

  # machine resources are also hard coded for now
  # TODO: make configurable?
  config.vm.provider "virtualbox" do |v|
    v.memory = 4096
    v.cpus   = 1
  end

  # execute basic provisioning for all VMs
  config.vm.provision "shell", inline: $common_script  

  # set up UM nodes
  (1..$num_um).each do |i|
    config.vm.define "um-#{i}" do |config|
      config.vm.hostname = "um-#{i}"
      config.vm.network "private_network", ip: "#{$net_base}.#{i+100}"
      config.vm.provision "shell", 
        env: {
          "NET_BASE": "#{$net_base}",
          "NUM_UM":   "#{$num_um}",
          "NUM_PM":   "#{$num_pm}",
          "MY_NAME":  "um-#{i}",
          "MY_IP":    "#{$net_base}.#{i+100}"
        },
        inline: $um_script
    end
  end

  # set up PM nodes
  # doing this in reverse order so that postConfigure on pm-1
  # always runs last
  (1..$num_pm).reverse_each do |i|
    config.vm.define "pm-#{i}" do |config|
      config.vm.hostname = "pm-#{i}"
      config.vm.network "private_network", ip: "#{$net_base}.#{i+200}"
      config.vm.provision "shell", 
        env: {
          "NET_BASE": "#{$net_base}",
          "NUM_UM":   "#{$num_um}",
          "NUM_PM":   "#{$num_pm}",
          "MY_NAME":  "pm-#{i}",
          "MY_IP":    "#{$net_base}.#{i+100}"
        },
        inline: $pm_script
    end
  end

end

$common_script= <<END
  # add hosts entries we prepared earlier
  echo "#{$hostfile}" >> /etc/hosts

  # set up inter node passwordless SSH access for users 'vagrant' and 'root'
  # using the same predefined key pair for all
  for user in vagrant root
  do
    dir=$(eval realpath ~$user)/.ssh
    mkdir -p $dir
    cp /vagrant/files/ssh/config $dir
    cp /vagrant/files/ssh/id_rsa $dir
    cp /vagrant/files/ssh/id_rsa.pub $dir
    cat /vagrant/files/ssh/id_rsa.pub >> $dir/authorized_keys
    chown -R $user:$user $dir
  done

  # don't let apt-get / dpkg ask config questions, just use defaults
  export DEBIAN_FRONTEND=noninteractive

  # fetch and add packet signing key
  wget -qO - https://downloads.mariadb.com/MariaDB/mariadb-columnstore/MariaDB-ColumnStore.gpg.key | sudo apt-key add -

  # set up columnstore repository
  # TODO: so far hardcoded to "bionic"
  cp /vagrant/files/apt/mariadb-columnstore.list /etc/apt/sources.list.d/
  apt-get update

  # install columnstore packages required by all nodes
  apt-get install -y mariadb-columnstore-client mariadb-columnstore-common mariadb-columnstore-shared
  apt-get install -y mariadb-columnstore-server mariadb-columnstore-storage-engine
  apt-get install -y mariadb-columnstore-libs mariadb-columnstore-platform
END

$um_script= <<END
END

$pm_script= <<END
  # special final touches to be run on pm-1 only
  if test "$MY_NAME" = "pm-1"
  then
    # postConfigure wants to see all .deb packages in /root
    cp /var/cache/apt/archives/mariadb-columnstore-* /root

    # run postConfigure to finish columnstore setup
    # TODO answers file needs to be created dynamically
    #      to support different UM/PM counts
    /usr/local/mariadb/columnstore/bin/postConfigure -d < /vagrant/files/postConfigure-answers.txt
  fi
END



