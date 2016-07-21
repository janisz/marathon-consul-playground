# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu/trusty64"
  config.vm.network "private_network", ip: "10.10.10.10"
  config.vm.hostname = "marathon-consul"
  config.vm.provider "virtualbox" do |vb|
    vb.name = "marathon-consul"
  end
  config.vm.post_up_message = """

      * Mesos:    http://10.10.10.10:5050
      * Marathon: http://10.10.10.10:8080
      * Consul:   http://10.10.10.10:8500
      * HAProxy:  http://10.10.10.10:8900

   """

  config.vm.provision "shell", inline: <<-SHELL

  curl -s https://bintray.com/user/downloadSubjectPublicKey?username=allegro | sudo apt-key add -
  sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E56151BF

  DISTRO=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
  CODENAME=$(lsb_release -cs)-unstable
  echo "deb http://repos.mesosphere.io/${DISTRO} ${CODENAME} main" | \
    sudo tee /etc/apt/sources.list.d/mesosphere.list
  echo "deb http://dl.bintray.com/v1/content/allegro/deb /" | \
    sudo tee /etc/apt/sources.list.d/marathon-consul.list
  sudo add-apt-repository ppa:webupd8team/java
  sudo add-apt-repository ppa:vbernat/haproxy-1.6

  sudo apt-get -y update

  apt-get install -y haproxy

  echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | sudo /usr/bin/debconf-set-selections
  sudo apt-get -qy install curl unzip oracle-java8-set-default zookeeperd mesos marathon marathon-consul

  echo "10.10.10.10" > /etc/mesos-slave/hostname
  mkdir -p /etc/marathon/conf
  echo "http_callback" > /etc/marathon/conf/event_subscriber
  echo "http://10.10.10.10:4000/events" > /etc/marathon/conf/http_endpoints

  mkdir -p /usr/share/consul
  mkdir -p /etc/consul.d/server
  mkdir -p /var/consul
  CONSUL_VERSION=0.6.3
  curl -OLs https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip
  unzip -o consul_${CONSUL_VERSION}_linux_amd64.zip -d /usr/local/bin && rm consul_${CONSUL_VERSION}_linux_amd64.zip
  curl -OLs https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_web_ui.zip
  unzip -o consul_${CONSUL_VERSION}_web_ui.zip -d /usr/share/consul/ui && rm consul_${CONSUL_VERSION}_web_ui.zip
  CONSUL_TEMPLATE_VERSION=0.14.0
  curl -OLs https://releases.hashicorp.com/consul-template/${CONSUL_TEMPLATE_VERSION}/consul-template_${CONSUL_TEMPLATE_VERSION}_linux_amd64.zip
  unzip -o consul-template_${CONSUL_TEMPLATE_VERSION}_linux_amd64.zip -d /usr/local/bin
  rm consul-template_${CONSUL_TEMPLATE_VERSION}_linux_amd64.zip

cat > /etc/consul.d/server/config.json <<EOF
{
    "bootstrap": true,
    "server": true,
    "datacenter": "dc1",
    "data_dir": "/var/consul",
    "advertise_addr": "10.10.10.10",
    "client_addr": "10.10.10.10",
    "log_level": "INFO",
    "enable_syslog": true,
    "ui_dir": "/usr/share/consul/ui"
}
EOF

cat > /etc/init/consul.conf <<EOF
description "Consul server process"

start on (local-filesystems and net-device-up IFACE=eth0)
stop on runlevel [!12345]

respawn

exec consul agent -config-dir /etc/consul.d/server
EOF

cat > /etc/init/consul-template.conf <<EOF
description "Consul Template Jobs"

start on runlevel [2345]
stop on runlevel [!2345]
start on filesystem and started consul

setuid root
setgid root

exec consul-template \
    -consul 10.10.10.10:8500 \
    -template "/vagrant/haproxy.cfg.ctmpl:/etc/haproxy/haproxy.cfg:/etc/init.d/haproxy restart" \
    -template "/vagrant/hosts.ctmpl:/etc/hosts"
EOF

  service zookeeper restart
  service mesos-slave restart
  service mesos-master restart
  service marathon restart
  service consul restart
  service consul-template restart
  service marathon-consul restart

  until curl -sf http://10.10.10.10:8080/v2/apps -o /dev/null; do sleep 1; echo "Waiting for Marathon"; done
  curl -X PUT -H "Content-Type: application/json" --data @/vagrant/ok.json http://10.10.10.10:8080/v2/apps
  until curl -sf http://ok.test.disco/ -o /dev/null; do sleep 1; echo "Waiting for app"; done
  curl http://ok.test.disco/

  SHELL
end
