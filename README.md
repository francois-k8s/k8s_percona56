k8s_percona
===========
kubernetes replication controller file to create a percona cluster

# Requirements
- A 3 nodes (at least) kubernetes cluster

# Usage
Launch the replication controller . It will spin up 3 containers.
```
core@master:~/k8s_percona56$ git clone https://github.com/francois-k8s/k8s_percona56.git
core@master:~/k8s_percona56$ cd k8s_percona56
core@master:~/k8s_percona56$ kubecfg -c percona-rc.yaml create replicationControllers
core@master:~/k8s_percona56$ kubecfg list pods
ID                                     Image(s)                                   Host                Labels                                                             Status
----------                             ----------                                 ----------          ----------                                                         ----------
42236103-6ddb-11e4-9c2f-525400e93647   francois/percona:0.0.1   192.168.23.111/     name=percona,replicationController=percona-rc,service=percona-rc   Running
d8aed68a-6ddb-11e4-9c2f-525400e93647   francois/percona:0.0.1   192.168.23.112/     name=percona,replicationController=percona-rc,service=percona-rc   Running
4b8994f1-6ddc-11e4-9c2f-525400e93647   francois/percona:0.0.1   192.168.23.113/     name=percona,replicationController=percona-rc,service=percona-rc   Running
```

Connect to each container and get its ipaddress:
```
ssh -p 33001 root@192.168.23.111
root@d8aed68a-6ddb-11e4-9c2f-525400e93647:~# ip a | grep inet | grep eth0 | awk '{ print $2; }' | cut -d "/" -f 1
10.244.0.3
```

```
ssh -p 33001 root@192.168.23.112
root@d8aed68a-6ddb-11e4-9c2f-525400e93647:~# ip a | grep inet | grep eth0 | awk '{ print $2; }' | cut -d "/" -f 1
10.244.1.4
```

```
ssh -p 33001 root@192.168.23.113
root@4b8994f1-6ddc-11e4-9c2f-525400e93647:~# ip a | grep inet | grep eth0 | awk '{ print $2; }' | cut -d "/" -f 1
10.244.2.3
```

In each container, modify the file /etc/mysql/my.cnf
```
sed -e "s@gcomm://@gcomm://10.244.0.3,10.244.1.4,10.244.2.3@"
sed -e "s@wsrep_node_address=@wsrep_node_address=IP_ADDRESS_Of_THE_NODE@"
```

Once the 3 nodes have the correct configuration, it's time to bootstrap the cluster.
Connect to the first node
```
/etc/init.d/mysql bootstrap-pxc
mysql -uroot -p
mysql> CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 's3cretPass';
mysql> GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
mysql> FLUSH PRIVILEGES;
mysql> show status like 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | b598af3e-ace3-11e2-0800-3e90eb9cd5d3 |
...
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
...
| wsrep_cluster_size         | 1                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
...
| wsrep_ready                | ON                                   |
+---------------------------

```
Then connect to the 2 others nodes and restart the mysql service
```
service mysql restart
 * Stopping MySQL (Percona XtraDB Cluster) mysqld
   ...done.
 * Starting MySQL (Percona XtraDB Cluster) database server mysqld
 * State transfer in progress, setting sleep higher mysqld
   ...done.

```

And now
```
mysql> show status like 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | b598af3e-ace3-11e2-0800-3e90eb9cd5d3 |
...
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
...
| wsrep_cluster_size         | 3                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
...
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
40 rows in set (0.01 sec)
```

You can now restart the mysql service on the first node.

Congrats ! You have multi-master percona cluster \o/
