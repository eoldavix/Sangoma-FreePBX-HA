# Sangoma-FreePBX-HA
High availability cluster for Sangoma's FreePBX platform

Disclamer: This configuration is for **testing purpose** only!

## Files

### ocf:heartbeat modules changes
- asterisk --> fwconsole implementations
- apache --> call fail2ban after apache start
- galera --> Workarounded version for issue: https://github.com/ClusterLabs/resource-agents/issues/940

This modules goes to /usr/lib/ocf/resource.d/heartbeat/

### config files
- my.cnf.d/galera.cnf --> Galera configuration for MariaDB
- httpd/conf/httpd.conf --> Added "Listen <floating_ip>:80"
- httpd/conf.d/status.conf --> Configuration for /server-status

## Procedure

### hosts file configuration
```
echo -e "10.0.0.1\tha1.freepbx.local\n10.0.0.2\tha2.freepbx.local" >> /etc/hosts
```

### disabling services
```
systemctl disable httpd
systemctl disable freepbx
chkconfig mysql off
```

### galera configuration

#### installation

- First of all, we need to add the MariaDB repo:
```
cat <<EOF > /etc/yum.repos.d/MariaDB.repo
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
EOF
```
- Next, let's update and install the MariaDB package and it's dependencies:
```
yum update -y
yum install -y MariaDB-server MariaDB-client MariaDB-compat galera socat jemalloc
```

#### configuration

- We need to create the file `/etc/sysconfig/clustercheck`, used by the galera heartbeat script:
```
cat > /etc/sysconfig/clustercheck << EOF
MYSQL_USERNAME="clustercheck"
MYSQL_PASSWORD="redhat"
MYSQL_HOST="localhost"
MYSQL_PORT="3306"
EOF
mysql -e "CREATE USER 'clustercheck'@'localhost' IDENTIFIED BY 'redhat';"
```
- Let's securize the MariaDB installation with `mysql_secure_install`, providing a password to root user, and answering yes to all questions.

- We create a user who will use Galera for synchronization:
```
mysql> DELETE FROM mysql.user WHERE user='';
mysql> GRANT ALL ON *.* TO 'root'@'%' IDENTIFIED BY 'securepass';
mysql> GRANT USAGE ON *.* to sst_user@'%' IDENTIFIED BY 'securepass';
mysql> GRANT ALL PRIVILEGES on *.* to sst_user@'%';
mysql> FLUSH PRIVILEGES;
mysql> quit
```
- The ODBC connector included in the distro doesn't work with MariaDB. For fix this issue:
```
cd /tmp
wget https://downloads.mariadb.com/enterprise/r8ex-kwwe/connectors/odbc/connector-odbc-2.0.9/mariadb-connector-odbc-2.0.9-beta-linux-x86_64.tar.gz
tar zxvf mariadb-connector-odbc-2.0.9-beta-linux-x86_64.tar.gz
mv mariadb-connector-odbc-2.0.9-beta-linux-x86_64/lib/libmaodbc.so /usr/lib/x86_64-linux-gnu/odbc/
mv mariadb-connector-odbc-2.0.9-beta-linux-x86_64/lib/libmaodbc.so /usr/lib64/
chown root:root /usr/lib64/libmaodbc.so
sed -i s/libmyodbc5/libmaodbc/ /etc/odbcinst.ini
ln -s /usr/lib64/libodbcinst.so.2 /usr/lib64/libodbcinst.so.1
ldd /usr/lib64/libmaodbc.so
```

#### first time execution

- Once the configuration file `/etc/my.cnf.d/galera.cnf` is putted in his place, and modified the specific node configuration in all nodes, we need to stop MariaDB (`systemctl stop mariadb`). Once stopped in all nodes, in the first node we execute `galera_new_cluster`, and in the rest of nodes, `systemctl start mariadb`

- After the cluster has been initialized, we proceed to stop it with `systemctl stop mariadb` in all nodes.


### pacemaker configuration
- After all ocf heartbeat scripts are copied in `/usr/lib/ocf/resource.d/heartbeat`, to create the pacemaker/corosync cluster, we need to execute:
```
pcs cluster auth ha1.freepbx.local ha2.freepbx.local
pcs cluster setup --name test_asterisk ha1.freepbx.local ha2.freepbx.local \
  --transport udpu
pcs cluster start --all --wait=60
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
pcs resource create db ocf:heartbeat:galera \
  enable_creation=true \
  wsrep_cluster_address=gcomm://ha1.freepbx.local,ha2.freepbx.local \
  log=/var/log/mysqld.log \
  op start interval=0s timeout=120 stop interval=0s timeout=120 monitor \
  interval=20 timeout=30 monitor interval=10 role=Master timeout=30 monitor \
  interval=30 role=Slave timeout=30 promote interval=0s timeout=300 demote \
  interval=0s timeout=120 meta master-max=2
pcs resource create my_Floating_IP ocf:heartbeat:IPaddr2 \
  ip=10.0.0.20 cidr_netmask=24 nic=eth0:1 \
  op start interval=0s timeout=20s stop interval=0s timeout=20s monitor \
  interval=10s
pcs resource create webserver ocf:heartbeat:apache \
  configfile=/etc/httpd/conf/httpd.conf statusurl=http://127.0.0.1/server-status \
  op start interval=60s timeout=60s stop interval=0s timeout=60s monitor \
  interval=1min
pcs resource create asterisk ocf:heartbeat:asterisk \
  user=asterisk group=asterisk \
  op start interval=60s timeout=30s stop interval=0s timeout=20 monitor \
  interval=60s timeout=30
pcs resource create pause ocf:heartbeat:Delay \
  startdelay=15 stopdelay=0 mondelay=0 \
  op start interval=0s timeout=60s stop interval=0s timeout=30 monitor \
  interval=10 timeout=30
pcs resource group add afterDB webserver asterisk
pcs resource master db-master db master-max=2
pcs \
  constraint colocation add my_Floating_IP with webserver \
  id=colocation-my_Floating_IP-webserver-INFINITY
pcs \
  constraint colocation add my_Floating_IP with asterisk \
  id=colocation-my_Floating_IP-asterisk-INFINITY
pcs \
  constraint colocation add my_Floating_IP with pause \
  id=colocation-my_Floating_IP-pause-INFINITY
pcs constraint order my_Floating_IP \
  then db-master kind=Serialize id=order-my_Floating_IP-db-master-Serialize
pcs constraint order db-master \
  then pause kind=Serialize id=order-db-master-pause-Serialize
pcs constraint order pause \
  then webserver kind=Serialize id=order-pause-webserver-Serialize
pcs constraint order pause \
  then asterisk kind=Serialize id=order-pause-asterisk-Serialize
```
