$haproxy = <<SCRIPT
echo "instalando nfs haproxy"
apt-get update
apt-get install -y haproxy
echo 'frontend http-revolver

frontend varnish
  bind 0.0.0.0:80
  default_backend varnish-server

frontend nginx
  bind 0.0.0.0:8080
  default_backend public-servers

backend varnish-server
  server s1 127.0.0.1:6081 check

backend public-servers
  server s1 10.100.199.201:80 check
  server s2 10.100.199.202:80 check

listen stats
  bind  0.0.0.0:1984
  stats enable
  stats scope http-revolver
  stats scope public-servers
  stats scope admin-servers
  stats uri /
  stats realm Haproxy\ Statistics
  stats auth user:password '>>  /etc/haproxy/haproxy.cfg

service haproxy restart

SCRIPT

$varnish =  <<SCRIPT

apt-get install -y varnish
cp /tmp/default.vcl /etc/varnish/default.vcl
sed -i "s/localhost:6082/:6082/" /lib/systemd/system/varnish.service
systemctl daemon-reload
service varnish restart
cat /etc/varnish/secret

SCRIPT

$nfsserver = <<SCRIPT
echo "instalando nfs server"
apt-get update
apt-get install -y nfs-kernel-server
echo "/var/www 10.100.199.202(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports
service nfs-kernel-server restart
SCRIPT


$nfsclient = <<SCRIPT
echo "instalando nfs cliente"
apt-get update
apt-get install -y nfs-common
echo "10.100.199.201:/var/www/ /var/www/ nfs defaults 0 0" >> /etc/fstab
mount -a

SCRIPT

$wordpress = <<SCRIPT

cd /var/www
#####wget https://wordpress.org/latest.tar.gz
#####tar -xzvf latest.tar.gz
#####chown www-data. -R wordpress
git clone https://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@github.com/pps-ciber/wordpress-pclementeciber.git wordpress
chown www-data. -R wordpress
cd wordpress
mysql -uciber -psupersegura1 -h 10.100.199.203 < wordpress.dump

SCRIPT

$nginx = <<SCRIPT
echo "instalando nginx"
apt-get update
apt-get install -y nginx mysql-client 

echo 'server {
  listen 80;
  #server_name www.ejemplo.com ejemplo.com;

  root /var/www/wordpress;
  index index.php;

  # log files
  access_log /var/log/nginx/ejemplo.com.access.log;
  error_log /var/log/nginx/ejemplo.com.error.log;

  location = /favicon.ico {
      log_not_found off;
      access_log off;
  }

  location = /robots.txt {
      allow all;
      log_not_found off;
      access_log off;
  }

  location / {
      try_files \$uri \$uri/ /index.php?\$args;
  }

  location ~ \\.php\$ {
      include snippets/fastcgi-php.conf;
      ##fastcgi_pass 127.0.0.1:9000;
      fastcgi_pass unix:/run/php/php7.4-fpm.sock;
  }

  location ~* \\.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
      expires max;
      log_not_found off;
  }
}' >> /etc/nginx/conf.d/ejemplo.conf

rm /etc/nginx/sites-enabled/default

service nginx restart

SCRIPT

$php = <<SCRIPT
apt-get update
apt-get install -y php-fpm php-mysql php-curl php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip


mkdir /var/www/

##sed -i "s/^listen =.*/listen = 9000/" /etc/php/7.4/fpm/pool.d/www.conf

systemctl restart php7.4-fpm
SCRIPT

$mysql = <<SCRIPT
apt-get update
apt-get install -y mysql-server

sed -i "s/^bind-address.*/bind-address = 10.100.199.203/" /etc/mysql/mysql.conf.d/mysqld.cnf

##SET GLOBAL validate_password.policy=LOW;
mysql -e "CREATE USER 'ciber'@'%' IDENTIFIED BY 'supersegura1';"
mysql -e "create database wordpress;"
mysql -e "Grant all privileges on wordpress.* TO 'ciber'@'%'"

systemctl restart mysql
SCRIPT

$redisserver = <<SCRIPT
apt-get update
apt-get install -y redis
SCRIPT

$redisclient = <<SCRIPT
apt-get install -y redis-tools
SCRIPT

$jenkins = <<SCRIPT
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install -y jenkins openjdk-11-jre
SCRIPT

Vagrant.configure("2") do |config|
  
  config.vm.box = "ubuntu/focal64"

  config.vm.define :haproxy, primary: true do |haproxy|
    haproxy.vm.box = "ubuntu/focal64"
    haproxy.vm.box_check_update = true

    haproxy.vm.hostname = "ubuntu-haproxy"

    haproxy.vm.network :private_network, ip: "10.100.199.200"
    haproxy.vm.network "forwarded_port", guest: 80, host: 80
    haproxy.vm.network "forwarded_port", guest: 8080, host: 8080
    haproxy.vm.network "forwarded_port", guest: 1984, host: 1984
    haproxy.vm.provision "shell", inline: $haproxy, privileged: true
    haproxy.vm.provision "file" , source: "./default.vcl" , destination: "/tmp/default.vcl"
    haproxy.vm.provision "shell", inline: $varnish, privileged: true

    haproxy.vm.provider "virtualbox" do |vb|
      vb.name = "ubuntu-haproxy-vb"
      vb.memory = "2048"
      vb.cpus = "1"
    end
  end   

  config.vm.define "mysql" do |mysql|
    mysql.vm.box = "ubuntu/focal64"
    mysql.vm.box_check_update = true

    mysql.vm.hostname = "ubuntu-mysql"

    mysql.vm.disk :disk, size: "50GB", primary: true
    mysql.vm.network :private_network, ip: "10.100.199.203"
    mysql.vm.provision "shell", inline: $mysql, privileged: true
    mysql.vm.provider "virtualbox" do |vb|
      vb.name = "ubuntu-mysql-vb"
      vb.memory = "2048"
      vb.cpus = "1"
    end
  end

  config.vm.define "front1" do |front1|
    front1.vm.box = "ubuntu/focal64"
    front1.vm.box_check_update = true

    front1.vm.hostname = "ubuntu-front1"

    front1.vm.disk :disk, size: "50GB", primary: true
    front1.vm.network :private_network, ip: "10.100.199.201"
    front1.vm.provision "shell", inline: $nginx, privileged: true
    front1.vm.provision "shell", inline: $php, privileged: true
    front1.vm.provision "shell", inline: $nfsserver, privileged: true
    front1.vm.provision "shell", inline: $wordpress, privileged: true
    front1.vm.provider "virtualbox" do |vb|
      vb.name = "ubuntu-front1-vb"
      vb.memory = "2048"
      vb.cpus = "1"
    end
  end   

  config.vm.define "front2" do |front2|
    front2.vm.box = "ubuntu/focal64"
    front2.vm.box_check_update = true

    front2.vm.hostname = "ubuntu-front2"

    front2.vm.disk :disk, size: "50GB", primary: true
    front2.vm.network :private_network, ip: "10.100.199.202"
    front2.vm.provision "shell", inline: $nginx, privileged: true
    front2.vm.provision "shell", inline: $php, privileged: true
    front2.vm.provision "shell", inline: $nfsclient, privileged: true
    front2.vm.provider "virtualbox" do |vb|
      vb.name = "ubuntu-front2-vb"
      vb.memory = "2048"
      vb.cpus = "1"
    end
  end  

  config.vm.define "redis" do |redis|
    redis.vm.box = "ubuntu/focal64"
    redis.vm.box_check_update = true

    redis.vm.hostname = "ubuntu-redis"

    redis.vm.disk :disk, size: "50GB", primary: true
    redis.vm.network :private_network, ip: "10.100.199.204"
    redis.vm.provision "shell", inline: $redisserver, privileged: true
    redis.vm.provider "virtualbox" do |vb|
      vb.name = "ubuntu-redis-vb"
      vb.memory = "2048"
      vb.cpus = "1"
    end
  end  

  config.vm.define "jenkins" do |jenkins|
    jenkins.vm.box = "ubuntu/focal64"
    jenkins.vm.box_check_update = true

    jenkins.vm.hostname = "ubuntu-jenkins"

    ###jenkins.vm.disk :disk, size: "50GB", primary: true
    jenkins.vm.network "forwarded_port", guest: 8080, host: 8080
    jenkins.vm.network :private_network, ip: "10.100.199.205"
    jenkins.vm.provision "shell", inline: $jenkins, privileged: true
    jenkins.vm.provider "virtualbox" do |vb|
      vb.name = "ubuntu-jenkins"
      vb.memory = "2048"
      vb.cpus = "1"
    end
  end  

end
