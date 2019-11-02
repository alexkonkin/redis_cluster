VAGRANTFILE_API_VERSION = "2"

$install_mysql_cluster1 = <<-SCRIPT
  nod_nam=$1
  nod_ip=$2
  is_first_node=$3

  echo $nod_nam
  echo $nod_ip
  echo $is_first_node

  echo "Substitution with sed..."
  sed -i 's/bind-address\t\t= 127.0.0.1/#bind-address\t\t= 127.0.0.1\\nskip-bind-address\\nskip-networking=0\\n/g' /etc/mysql/my.cnf
  echo $?
SCRIPT

$install_keepalived_haproxy = <<-SCRIPT
  state=$1
  priority=$2
  this_ip=$3
  peer_ip=$4

  echo $nod_nam
  echo $nod_ip
  echo $is_first_node
  sudo apt install keepalived haproxy mc -y
  echo -e "192.168.1.11       srv1\n192.168.1.12       srv2\n172.16.94.11       redis-node1\n172.16.94.12       redis-node2\n172.16.94.13       redis-node3" >> /etc/hosts

cat > /etc/haproxy/haproxy.cfg << "END"
global
    log 127.0.0.1 local0 notice
    user haproxy
    group haproxy


defaults
    log     global
    option  dontlognull
    retries 3
    option redispatch
    timeout connect  5000
    timeout client  50000
    timeout server  50000
    stats enable
    stats uri /haproxy/stats
    stats auth admin:admin


frontend http
    bind *:80
    mode http
    option httplog
    default_backend webservers

backend webservers
    mode http
    stats enable
    stats uri /haproxy/stats
    stats auth admin:admin
    stats hide-version
    balance roundrobin
    option httpclose
    option forwardfor
    cookie SRVNAME insert
    server srv1 192.168.1.11:80 check cookie srv1
    server srv2 192.168.1.12:80 check cookie srv2


frontend ft_redis
    bind 192.168.1.100:6379 name redis
    default_backend bk_redis

backend bk_redis
    balance roundrobin
    option tcp-check
    tcp-check connect
    tcp-check send PING\r\n
    tcp-check expect string +PONG
    tcp-check send QUIT\r\n
    tcp-check expect string +OK
    server redis-node1 172.16.94.11:6379 check inter 1s
    server redis-node2 172.16.94.12:6380 check inter 1s
    server redis-node3 172.16.94.13:6381 check inter 1s

END

cat > /etc/keepalived/keepalived.conf << "END"
vrrp_script chk_haproxy {
    script "killall -0 haproxy"     # cheaper than pidof
    interval 2                      # check every 2 seconds
}

vrrp_instance VI_1 {
    state STATE
    interface eth1
    virtual_router_id 51
    priority PRIORITY
    vrrp_unicast_bind THIS_IP   # Internal IP of this machine
    vrrp_unicast_peer PEER_IP   # Internal IP of peer
    virtual_ipaddress {
        192.168.1.100 dev eth1 label eth1:vip1
    }
    track_script {
        chk_haproxy weight 2
    }
}
END

#allow binding to virtual IP of keepalived and apply this adjustment
echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
sysctl -p

#configue master/slave keepalived instances
sed -i 's/STATE/'$state'/g' /etc/keepalived/keepalived.conf
sed -i 's/PRIORITY/'$priority'/g' /etc/keepalived/keepalived.conf
sed -i 's/THIS_IP/'$this_ip'/g' /etc/keepalived/keepalived.conf
sed -i 's/PEER_IP/'$peer_ip'/g' /etc/keepalived/keepalived.conf

systemctl start keepalived
systemctl restart haproxy

SCRIPT

$install_redis_cluster = <<-SCRIPT
   nod_nam=$1
   nod_ip=$2
   is_first_node=$3
   echo -e "192.168.1.11       srv1\n192.168.1.12       srv2\n172.16.94.11       redis-node1\n172.16.94.12       redis-node2\n172.16.94.13       redis-node3" >> /etc/hosts

   apt-get update && sudo apt-get upgrade
   apt install make gcc libc6-dev tcl mc -y
   mkdir -p /var/lib/redis/
   touch /var/log/redis_master.log
   touch /var/log/redis_slave.log
SCRIPT


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.define "srv1" do |srv1|
    srv1.vm.box = "bento/ubuntu-18.04"
    srv1.vm.hostname = "srv1"
    srv1.vm.network :private_network, ip: "192.168.1.11"
    config.vm.provision "shell" do |script|
       script.inline = $install_keepalived_haproxy
       script.args = ["MASTER","101","192.168.1.11","192.168.1.12"]
    end
  end

  config.vm.define "srv2" do |srv2|
    srv2.vm.box = "bento/ubuntu-18.04"
    srv2.vm.hostname = "srv2"
    srv2.vm.network :private_network, ip: "192.168.1.12"
    config.vm.provision "shell" do |script|
       script.inline = $install_keepalived_haproxy
       script.args = ["BACKUP","100","192.168.1.12","192.168.1.11"]
    end
  end

  config.vm.define "redis1" do |redis1|
    redis1.vm.box = "bento/ubuntu-18.04"
    redis1.vm.hostname = "redis-node1"
    redis1.vm.network :private_network, ip: "172.16.94.11"
    config.vm.provision "shell" do |script|
       script.inline = $install_redis_cluster
       script.args = ["redis-node1","172.16.94.11","true"]
    end
  end

  config.vm.define "redis2" do |redis2|
    redis2.vm.box = "bento/ubuntu-18.04"
    redis2.vm.hostname = "redis-node2"
    redis2.vm.network :private_network, ip: "172.16.94.12"
    config.vm.provision "shell" do |script|
       script.inline = $install_redis_cluster
       script.args = ["redis-node2","172.16.94.12","false"]
    end
  end

  config.vm.define "redis3" do |redis3|
    redis3.vm.box = "bento/ubuntu-18.04"
    redis3.vm.hostname = "redis-node3"
    redis3.vm.network :private_network, ip: "172.16.94.13"
    #redis3.vm.provision "shell", path: 'provisioners/install_k8s.sh', privileged: false
    config.vm.provision "shell" do |script|
       script.inline = $install_redis_cluster
       script.args = ["redis-node3","172.16.94.13","false"]
    end
   end

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "320"]
    vb.customize ["modifyvm", :id, "--cpus", "1"]
  end
end
