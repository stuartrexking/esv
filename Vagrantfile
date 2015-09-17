# -*- mode: ruby -*-
# vi: set ft=ruby :
$script = <<SCRIPT

echo Installing dependencies...
sudo apt-get install -y unzip

echo Fetching Consul...
cd /tmp/
wget https://dl.bintray.com/mitchellh/consul/0.5.2_linux_amd64.zip -O consul.zip

echo Installing Consul...
unzip consul.zip
sudo chmod +x consul
sudo mv consul /usr/bin/consul

sudo mkdir /etc/consul.d
sudo chmod a+w /etc/consul.d

echo Configuring Consul...
cat <<EOF > /etc/consul.d/config.json
{
  "bind_addr": "BIND_ADDR",
  "bootstrap_expect": 2,
  "data_dir": "/tmp/consul",
  "retry_join": [
    "172.20.20.10",
    "172.20.20.11",
    "172.20.20.12"
  ],
  "retry_interval": "5s",
  "server": true
}
EOF

echo '{"service": {"name": "eventstore", "port": 1113}}' >/etc/consul.d/eventstore.json

ipadr=$(ifconfig eth1 2>/dev/null|awk '/inet addr:/ {print $2}'|sed 's/addr://')
sed -i "s/BIND_ADDR/$ipadr/g" /etc/consul.d/config.json

cat <<EOF > /etc/init/consul.conf
description "Consul agent"

start on runlevel [2345]
stop on runlevel [!2345]

respawn

script
  export GOMAXPROCS=`nproc`

  exec /usr/bin/consul agent \
    -config-dir="/etc/consul.d" \
    >>/var/log/consul.log 2>&1
end script
EOF

echo Starting Consul...
service consul start

echo Installing dnsmasq...
sudo apt-get install -y dnsmasq
echo "server=/eventstore.service.consul/127.0.0.1#8600" > /etc/dnsmasq.d/10-consul
sudo service dnsmasq restart
echo Setup complete

echo Installing Event Store...
curl https://apt-oss.geteventstore.com/eventstore.key | sudo apt-key add -
echo "deb [arch=amd64] https://apt-oss.geteventstore.com/ubuntu/ trusty main" | sudo tee /etc/apt/sources.list.d/eventstore.list
sudo apt-get update
sudo apt-get install -y eventstore-oss

cat <<EOF > /etc/eventstore/eventstore.conf
---
Log: "/var/log/eventstore"
MemDb: True
IntIp: "$ipadr"
ExtIp: "$ipadr"
ClusterSize: 3
ClusterDns: "eventstore.service.consul"
ClusterGossipPort: 2112
---
EOF

sudo service eventstore start
SCRIPT

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/trusty64"

  config.vm.provision "shell", inline: $script

  (1..3).each do |i|
      config.vm.define "n#{i}" do |node|
          node.vm.hostname = "n#{i}"
          node.vm.network "private_network", ip: "172.20.20.1#{i-1}"
          node.vm.network "forwarded_port", guest: 2113, host: 10079+i
      end
  end
end
