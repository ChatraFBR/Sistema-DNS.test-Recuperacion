Vagrant.configure("2") do |config|

  config.vm.box = "debian/bookworm64"

  config.vbguest.auto_update = false

  config.vm.define "earth" do |earth|
    earth.vm.network "private_network", ip: "192.168.57.103"

    earth.vm.hostname = "tierra.sistema.test"

    earth.vm.provision "shell", inline: <<-SHELL
        apt update -y
        apt-get install -y dnsutils bind9
    SHELL

    earth.vm.provision "shell", inline: <<-SHELL
      cp -v /vagrant/config/resolution/resolv.conf /etc/ 
      cp -v /vagrant/config/default/named /etc/default/named   
      cp -v /vagrant/config/options/named.conf.options /etc/bind/named.conf.options 
      cp -v /vagrant/config/earth/named.conf.local /etc/bind/named.conf.local
      mkdir -p /etc/bind/zones 
      cp -v /vagrant/config/earth/db.sistema.test /etc/bind/zones/db.sistema.test
      cp -v /vagrant/config/earth/db.192.168.57 /etc/bind/zones/db.192.168.57
      sudo systemctl restart bind9 
    SHELL
  end

  config.vm.define "venus" do |venus|
    venus.vm.network "private_network", ip: "192.168.57.102"

    venus.vm.hostname = "venus.sistema.test"

    venus.vm.provision "shell", inline: <<-SHELL
      apt update -y
      apt-get install -y dnsutils bind9
    SHELL

    
    venus.vm.provision "shell", inline: <<-SHELL
      cp -v /vagrant/config/resolution/resolv.conf /etc/
      cp -v /vagrant/config/default/named /etc/default/ 
      cp -v /vagrant/config/venus/named.conf.local /etc/bind/
      cp -v /vagrant/config/options/named.conf.options /etc/bind/
      mkdir -p /etc/bind/zones 
      touch /etc/bind/zones/db.sistema.test 
      touch /etc/bind/zones/db.192.168.57  
      sudo systemctl restart bind9 
    SHELL

  end

end