# Introduction

MySQL InnoDB Cluster delivers an integrated, native, HA solution for your databases. MySQL InnoDB Cluster consists of:
 * MySQL Servers with Group Replication to replicate data to all members of the cluster while providing fault tolerance, automated failover, and elasticity.
 * MySQL Router to ensure client requests are load balanced and routed to the correct servers in case of any database failures.
 * MySQL Shell to create and administer InnoDB Clusters using the built-in AdminAPI.

For more information, see the [official product page](https://www.mysql.com/products/enterprise/high_availability.html) and the [official user guide](https://dev.mysql.com/doc/refman/5.7/en/mysql-innodb-cluster-userguide.html). 

## Container Usage

You can either use the example shell scripts to create a cluster, or you can do it manually.

### Security

A secure method of password generation and management is available using the auto-generated random password that's stored in a file within each container.
All that's necessary to use the secure method is to replace all instances of ```MYSQL_ROOT_PASSWORD=root``` with ```MYSQL_ROOT_PASSWORD=$(cat secretpassword.txt)``` in each example docker command.

**Note:** the scripted method now uses the secure password management facilities by default.

### Scripted Method

Helper scripts can be used to either create a cluster, or to tear one down.

#### Create a cluster

To create a three node cluster that includes MySQL Router and MySQL Shell, and connect to the cluster with MySQL Shell:

  ```./start_three_node_cluster.sh```

**Note:** if you want to use a different image (for example when you have built a local variant of the image) you can run the following *before* invoking start_three_node_cluster.sh:

  ```export INNODB_CLUSTER_IMG=your_username/your_image_name```

#### Tear down (remove) a cluster

  ```./cleanup_cluster.sh```

### Manual Method
This manual process essentially documents what the `start_three_node_cluster.sh` helper script performs. The main difference is that the examples contain a simple and non-secure password value of 'root'. You can leverage the built-in secure means of password management by using ```MYSQL_ROOT_PASSWORD=$(cat secretpassword.txt)``` instead.

1. Create a private network for the containers

  ```
  docker network create --driver bridge grnet
  ```

2. Bootstrap the cluster

  ```
  docker run --name=mysqlgr1 --hostname=mysqlgr1 --network=grnet -e MYSQL_ROOT_PASSWORD=root -e BOOTSTRAP=1 -itd mattalord/innodb-cluster && docker logs mysqlgr1 | grep GROUP_NAME
  ```

  This will spit out the `GROUP_NAME` to use for subsequent nodes. For example,
  the output will contain something similar to:

  ```
  You will need to specify GROUP_NAME=a94c5c6a-ecc6-4274-b6c1-70bd759ac27f 
  if you want to add another node to this cluster
  ```

  You will use this variable when adding additional nodes below. In other words, replace the example value `a94c5c6a-ecc6-4274-b6c1-70bd759ac27f` below with yours.

3. Add a second node to the cluster via a seed node

   ```
   docker run --name=mysqlgr2 --hostname=mysqlgr2 --network=grnet -e MYSQL_ROOT_PASSWORD=root -e GROUP_NAME="a94c5c6a-ecc6-4274-b6c1-70bd759ac27f" -e GROUP_SEEDS="mysqlgr1:6606" -itd mattalord/innodb-cluster
   ```

4. Add a third node to the cluster via a seed node

   ```
   docker run --name=mysqlgr3 --hostname=mysqlgr3 --network=grnet -e MYSQL_ROOT_PASSWORD=root -e GROUP_NAME="a94c5c6a-ecc6-4274-b6c1-70bd759ac27f" -e GROUP_SEEDS="mysqlgr1:6606" -itd mattalord/innodb-cluster
   ```

5. Optionally add additional nodes via a seed node using the same process ...

6. Add a router for the cluster 

   ```
   docker run --name=mysqlrouter1 --hostname=mysqlrouter1 --network=grnet -e NODE_TYPE=router -e MYSQL_HOST=mysqlgr1 -e MYSQL_ROOT_PASSWORD=root -itd mattalord/innodb-cluster
   ```

7. Connect to the cluster via the mysql command-line client or MySQL Shell on one of the nodes

  To use the classic mysql command-line client:

  ```docker exec -it mysqlgr1 mysql -hmysqlgr1 -uroot -proot```

  There you can view the cluster membership status from the mysql console:

  ```SELECT * from performance_schema.replication_group_members;```

  To use the MySQL Shell:

  ```docker exec -it mysqlgr1 mysqlsh --uri=root:root@mysqlgr1:3306```

  There you can view the cluster status with:

  ```dba.getCluster().status()```

### Testing the MySQL Router instance

  ```docker exec -it mysqlrouter1 bash```

To test the `RW` port, which always goes to the PRIMARY node:

  ```mysql -u root -proot -h localhost --protocol=tcp -P6446 -e 'SELECT @@global.server_uuid'```

To test the `RO` port, which is round-robin load balanced to the SECONDARY nodes:

  ```mysql -u root -proot -h localhost --protocol=tcp -P6447 -e 'SELECT @@global.server_uuid'```

---

## Testing out myarbitratord 
  If you'd also like to test out [my example arbitrator](https://github.com/mattlord/myarbitratord), you can do so easily once you have a working cluster. For example, if you're using ```./start_three_node_cluster.sh``` then you can run a myarbitratord container this way:

  ```
  docker run --rm --name=myarbitratord --network=grnet --hostname=myarbitratord --entrypoint=/bin/bash -itd golang -c "go get github.com/mattlord/myarbitratord && /go/bin/myarbitratord -mysql_password '$(cat secretpassword.txt)' -seed_host myinnodbcluster"
  ```   

  You can then see the runtime stats with:
  ```
  docker exec -it myarbitratord curl localhost:8099/stats
  ```

---

## Health Checks
  Docker 1.12 added support for custom health check commands which can be defined in the [Dockerfile](https://docs.docker.com/engine/reference/builder/#healthcheck) or on the command-line with [docker run](https://docs.docker.com/engine/reference/run/#healthcheck). For Group Replication nodes you could use a healthcheck command like this so that the container will end/exit when the node is in the [ERROR or OFFLINE state](https://dev.mysql.com/doc/refman/5.7/en/group-replication-server-states.html):

  ```
  docker run --name=mysqlgr3 --hostname=mysqlgr3 --network=grnet -e MYSQL_ROOT_PASSWORD=root -e GROUP_NAME="a94c5c6a-ecc6-4274-b6c1-70bd759ac27f" -e GROUP_SEEDS="mysqlgr1:6606" --health-cmd='mysql -nsLNE -e "select member_state from performance_schema.replication_group_members where member_id=@@server_uuid;" 2>/dev/null | grep -v "*" | egrep -v "ERROR|OFFLINE" || exit 1' --health-start-period=60s --health-timeout=10s --health-interval=5s --health-retries=1 -itd mattalord/innodb-cluster
  ```

  The start period of 60 seconds gives the container plenty of time to initialize and reach a normal state. The timeout of 10 seconds is more than ample to ensure that we're able to connect to mysql, execute the query, and get the results. The interval of 5 seconds means that we only want to run this healthcheck command every 5 seconds, and the interval of 1 means that we consider the container unhealthy after 1 failed healthcheck. 

---

### macOS tip (and some Windows too)
  If you're like me and you use Docker on macOS, it's helpful to know that Docker actually executes the containers inside an [Alpine Linux](https://alpinelinux.org) VM which in turn runs inside of a native [xhyve](http://www.pagetable.com/?p=831) hypervisor. You can access the console for that VM using:
  ```
  screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
  ```

From there you can see the docker networking, volumes (/var/lib/docker), etc. Knowing how this all works "under the hood" will certainly come in handy sooner or later. Whenever you want to detach and close your console session just use:
```CTRL-A-\```

FWIW, Docker on Windows (assuming you're not using the fully native windows-only version available in Windows Server 2016) works in a similar way, but uses [Hyper-V](https://en.wikipedia.org/wiki/Hyper-V) as the native hypervisor.
