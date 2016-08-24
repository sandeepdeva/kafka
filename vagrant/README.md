Vagrantfile is modified a little to simulate a 3 node cluster along with master, scheduler. You can use your local machine as CLI for executing different Kafka commands on Mesos.

# Vagrant VMs for Mesos cluster
Vagrantfile creates mesos cluster with following nodes:
- master;
- slave0..slave(N-1) (N is specified in vagrantfile); (specifed as 3 slaves)
- scheduler

Master provides web ui listening on http://master:5050
Both master and slave nodes runs mesos slave daemons.

Every node has pre-installed docker. Master node has pre-installed
marathon scheduler.

Host's public key is copied to `authorized_hosts`,
so direct access like `ssh vagrant@master|scheduler|slaveX` should work.

For general mesos overview please refer to
http://mesos.apache.org/documentation/latest/mesos-architecture/

## Node Names
During first run vagrantfile creates `hosts` file which
contains host names for cluster nodes. It is recommended
to append its content to `/etc/hosts` (or other OS-specific
location) of the running (hosting) OS to be able to refer
master and slaves by logical names.

For example on my Mac I have following in {{/etc/hosts}}
```sh
$ cat /etc/hosts
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost

# kafka cluster nodes - vagrant
192.168.3.5	master
192.168.3.6	scheduler
192.168.3.7	slave0
192.168.3.8	slave1
192.168.3.9	slave2
```

## Startup
Run `vagrant up`, this will start the VMs for master, scheduler and slaves. This will take a while to provision.

Mesos master and slaves daemons are started automatically.

Each slave node runs 'mesos-slave' daemon while master runs both
'mesos-master' and 'mesos-slave' daemons.

Daemons could be controlled by using:
`/etc/init.d/mesos-{master|slave} {start|stop|status|restart}`

- SSH into scheduler node.
  ```
  vagrant ssh scheduler
  ```
- We need to build kafka-mesos jar, navigate to vagrant directory and run gradle command.
  ```
  cd /vagrant
  ./gradlew jar
  ```
  > This can also be run on your local machine. If the gradle command fails then run it again.

- Now we need Kafka jar
  ```
  wget https://archive.apache.org/dist/kafka/0.8.2.2/kafka_2.10-0.8.2.2.tgz
  ```
- Add MESOS_NATIVE_JAVA_LIBRARY env variable
  ```
  export MESOS_NATIVE_JAVA_LIBRARY=/usr/local/lib/libmesos.so
  ```
- Start the Kafka scheduler
  ```
  ./kafka-mesos.sh scheduler
  ```
- On your local machine install [kafkacat](https://github.com/edenhill/kafkacat) to easily produce and consume using CLI.
  ```
  brew install kafkacat
  ```
- From your local machine follow the commands mentioned in kafka-mesos' [README](https://github.com/mesos/kafka#starting-and-using-1-broker) and [kafkacat](https://github.com/edenhill/kafkacat) to play around.

## Simulate rebalance
If you want to test rebalance try the following steps
- Add 3 brokers and start them.
  
  ```
  ./kafka-mesos.sh broker add 0..2 --mem 200
  ./kafka-mesos.sh start 0..2
  ```
- Add a topic t1 with 2 partitions and replication factor of 2 on brokers 0 & 1.
  ```
  ./kafka-mesos.sh topi add t1 --broker 0,1 --partitions 2 --replicas 2
  ```
- Produce some messages to topic t1 on broker 0 using kafkacat (you need to get hostname of broker 0).
  
  ```
  echo "some text for topic t1" | kafkacat -b "slave0:31000" -t t1 -e
  ```
- Consume the message.
  
  ```
  kafkacat -C -b "slave0:31000" -t t1 -e
  ```
- List the partitions for topic t1.
  
  ```
  ./kafka-mesos.sh topic partitions t1
  ```
- Take down the broker 0 by stoping it and removing it.
  
  ```
  ./kafka-mesos.sh broker stop 0
  ./kafka-mesos.sh broker remove 0
  ```
- Check the partitions listing for topic t1, the broker 0 should have \*not-isr.
  
  ```
  ./kafka-mesos.sh topic partitions t1
  ```
- Now rebalance topic t1 on brokers 1 and 2.
  
  ```
  ./kafka-mesos.sh topic rebalance t1 --broker 1,2
  ```
- Check the status of rebalance.
  
  ```
  ./kafka-mesos.sh topic rebalance status
  ```  
- Finally check that partitions are rebalanced on brokers 1 and 2.
  
  ```
  ./kafka-mesos.sh topic partitions t1
  ```

## Configuration
Configuration is read from the following locations:
- `/etc/mesos`, `/etc/mesos-{master|slave}`
  for general or master|slave specific CLI options;
- `/etc/default/mesos`, `/etc/default/mesos-{master|slave}`
  for general or master|slave specific environment vars;

Please refer to CLI of 'mesos-master|slave' daemons and `/usr/bin/mesos-init-wrapper`
for details.

## Logs
Logs are written to `/var/log/mesos/mesos-{master|slave}.*`
