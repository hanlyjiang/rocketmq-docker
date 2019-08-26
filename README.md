# RocketMQ-Docker [![License](https://img.shields.io/badge/license-Apache%202-4EB1BA.svg)](https://www.apache.org/licenses/LICENSE-2.0.html)

This is the Git repo of the Docker Image for Apache RocketMQ. You could run it through the following ways: 

1. Generate a RocketMQ Docker image
2. Run the docker image with the below modes:
   1. Single Node.
   2. Cluster with docker-compose.
   3. Cluster on Kubernetes.


## Prerequisites

The Docker images in this repository should support Docker version 1.12+, and Kubernetes version 1.9+.


## Quick start

### A. Generate a RocketMQ docker image

Note: This is an experimented code to allow users to build docker image locally according to a given RocketMQ version. Actually the formal images have been generated by RocketMQ official maintainer and stored in docker hub. Suggest common users to use these remote images directly.

```
cd image-build
sh build-image.sh RMQ-VERSION BASE-IMAGE
```

> Tip: The supported RMQ-VERSIONs can be obtained from [here](https://dist.apache.org/repos/dist/release/rocketmq/). The supported BASE-IMAGEs are [centos, alpine]. For example: ```sh build-image.sh 4.5.0 alpine```

### B. Stage a specific version

Users can generate a runtime (stage) directory based on a specific version and docker style operate the RocketMQ cluster/server/nameserver beneath the directory.

``` 
sh stage.sh RMQ-VERSION
```

After executing the above shell script, (e.g.  sh stage.sh 4.5.0), it will generate a stage directory (./stages/4.5.0).  User can do the following works under the directory, assuming the RMQ-version is defined with 4.5.0.

#### 1. Single Node

Run: 

```
cd stages/4.5.0 

./play-docker.sh alpine

```
> NOTE:
Some Linux Systems (e.g. Ubuntu) may generate path 
```stages/4.5.0/template```, please adjust the command accordingly.


#### 2. Cluster with docker-compose

Run:

```
cd stages/4.5.0 

./play-docker-compose.sh

```


#### 3. Cluster on Kubernetes

Run:

```
cd stages/4.5.0 

./play-kubernetes.sh

```

#### 4. Cluster of Deledger storage 

Run: (Note: This feature needs RMQ version is 4.4.0 or above)

```
cd stages/4.5.0 

./play-docker-deledger.sh

```

## 5. TLS support 

Run:  (It will startup nameserver and broker with SSL enabled style. The client will not invoke nameserver or broker until related SSL client is configurated. ) 

You can see detailed TLS config instruction from [here](templates/ssl/README.md) 

```
cd stages/4.5.0 

./play-docker-tls.sh

# Once nameserver and broker startup correctly, you still can use the following script to test produce/consume in SSL mode, why, due to they still use the SSL setting which exists in JAVA-OPT of the docker rmqbroker container. 
./play-producer.sh
./play-consumer.sh
```

### How to update RocketMQ image repository using update.sh
Run:

```
cd image-build
./update.sh 
```

This script will get the latest release version of RocketMQ and build the docker images based on ```alpine``` and ```centos``` respectively, then push the new images to the current official repository ```rocketmqinc/rocketmq```.

### How to verify RocketMQ works well

#### Verify with Docker and docker-compose

1. Use `docker ps|grep rmqbroker` to find your RocketMQ broker container id.

2. Use `docker exec -it {container_id} ./mqadmin clusterList -n {nameserver_ip}:9876` to verify if RocketMQ broker works, for example:
```
root$ docker exec -it 63950574b491 ./mqadmin clusterList -n 192.168.43.56:9876
OpenJDK 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0
OpenJDK 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0
#Cluster Name     #Broker Name            #BID  #Addr                  #Version                #InTPS(LOAD)       #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE
DefaultCluster    63950574b491            0     172.17.0.3:10911       V4_3_0                   0.00(0,0ms)         0.00(0,0ms)          0 429398.92 -1.0000

```

#### Verify with Kubernetes

1. Use `kubectl get pods|grep rocketmq` to find your RocketMQ broker Pod id, for example:
```
[root@k8s-master rocketmq]# kubectl get pods |grep rocketmq
rocketmq-7697d9d574-b5z7g             2/2       Running       0          2d
```

2. Use `kubectl -n {namespace} exec -it {pod_id} -c broker bash` to login the broker pod, for example:
```
[root@k8s-master rocketmq]# kubectl -n default exec -it  rocketmq-7697d9d574-b5z7g -c broker bash
[root@rocketmq-7697d9d574-b5z7g bin]# 
```

3. Use `mqadmin clusterList -n {nameserver_ip}:9876` to verify if RocketMQ broker works, for example:
```
[root@rocketmq-7697d9d574-b5z7g bin]# ./mqadmin clusterList -n localhost:9876
OpenJDK 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0
OpenJDK 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0
#Cluster Name     #Broker Name            #BID  #Addr                  #Version                #InTPS(LOAD)       #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE
DefaultCluster    rocketmq-7697d9d574-b5z7g  0     192.168.196.14:10911   V4_3_0                   0.00(0,0ms)         0.00(0,0ms)          0 429399.44 -1.0000

```

So you will find it works, enjoy !

### C. Product level configuration

The project also provides a usage reference for product level cluster docker configuration and startup. Please see the [README.md](product/README.md) details in /product directory.


## FAQ

#### 1. If I want the broker container to load my customized configuration file (which means `broker.conf`) when it starts, how can I achieve this? 

First, create the customized `broker.conf`, like below:
```
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
#set `brokerIP1` if you want to set physical IP as broker IP.
brokerIP1=10.10.101.80 #change you own physical IP Address
```

And put the customized `broker.conf` file at a specific path, like "`pwd`/data/broker/conf/broker.conf". 

Then we can modify the `play-docker.sh` and volume this file to the broker container when it starts. For example: 

```
docker run -d -p 10911:10911 -p 10909:10909 -v `pwd`/data/broker/logs:/root/logs -v `pwd`/data/broker/store:/root/store -v `pwd`/data/broker/conf/broker.conf:/opt/rocketmq-4.4.0/conf/broker.conf --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" rocketmqinc/rocketmq:4.4
.0 sh mqbroker -c /opt/rocketmq-4.4.0/conf/broker.conf

```

Finally we can find the customized `broker.conf` has been used in the broker container. For example:

```
huandeMacBook-Pro:4.4.0 huan$ docker ps |grep mqbroker
a32c67aed6dd        rocketmqinc/rocketmq:4.4.0   "sh mqbroker"       20 minutes ago      Up 20 minutes       0.0.0.0:10909->10909/tcp, 9876/tcp, 0.0.0.0:10911->10911/tcp   rmqbroker
huandeMacBook-Pro:4.4.0 huan$ docker exec -it a32c67aed6dd cat /opt/rocketmq-4.4.0/conf/broker.conf
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
#set `brokerIP1` if you want to set physical IP as broker IP.
brokerIP1=10.10.101.80 #change you own physical IP Address

```

In the case of docker-compose, change the docker-compose.yml like following:
```
version: '2'
services:
  namesrv:
    image: rocketmqinc/rocketmq:4.4.0
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    volumes:
      - ./data/namesrv/logs:/home/rocketmq/logs
    command: sh mqnamesrv
  broker:
    image: rocketmqinc/rocketmq:4.4.0
    container_name: rmqbroker
    ports:
      - 10909:10909
      - 10911:10911
      - 10912:10912
    volumes:
      - ./data/broker/logs:/home/rocketmq/logs
      - ./data/broker/store:/home/rocketmq/store
      - ./data/broker/conf/broker.conf:/opt/rocketmq-4.4.0/conf/broker.conf
    #command: sh mqbroker -n namesrv:9876
    command: sh mqbroker -n namesrv:9876 -c ../conf/broker.conf
    depends_on:
      - namesrv

```
