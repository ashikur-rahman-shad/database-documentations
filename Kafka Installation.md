Remember, if you get permission errors, use `sudo` in shell
## Pre-requisites
Java Development Kit 
We used `java -version`
   ```bash
openjdk version "21.0.9" 2025-10-21
OpenJDK Runtime Environment (build 21.0.9+10-Ubuntu-124.04)
OpenJDK 64-Bit Server VM (build 21.0.9+10-Ubuntu-124.04, mixed mode, sharing)
   ```


## Download Kafka from website and Extract it

```bash
cd Downloads/
wget https://downloads.apache.org/kafka/4.1.1/kafka_2.13-4.1.1.tgz 
sudo tar -xzf kafka_2.13-4.1.1.tgz -C /opt 
sudo mv /opt/kafka_2.13-4.1.1 /opt/kafka
cd /opt/kafka
```
You can install it on any folder literally. But if you 
## Configure server properties for each server

Go to the Kafka installation folder
```bash
cd /opt/kafka
```

Edit the following file:
```bash
nano config/server.properties
```

Update these lines:
```
node.id= set your unique number
controller.quorum.voters=1@10.11.200.99:9093,2@10.11.205.205:9093,3@10.11.205.206:9093
listeners=PLAINTEXT://<CURRENT SERVER IP>:9092,CONTROLLER://<CURRENT SERVER IP>:9093
advertised.listeners=PLAINTEXT://<CURRENT SERVER IP>:9092,CONTROLLER://<CURRENT SERVER IP>:9093

## Comment this line
#controller.quorum.bootstrap.servers=<IP_1>:9093,<IP_2>:9093,<IP_3>:9093

## The followings may already be fine
process.roles=broker,controller
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
controller.listener.names=CONTROLLER 
```


## Configure KRaft
For bash/zsh
```bash
KAFKA_CLUSTER_ID=$(/opt/kafka/bin/kafka-storage.sh random-uuid)
```
For fish shell
```fish
set KAFKA_CLUSTER_ID $(/opt/kafka/bin/kafka-storage.sh random-uuid)
```
Check the Cluster ID (Optional)
```bash
echo "Cluster ID: $KAFKA_CLUSTER_ID"
```

Stop server if running
```bash
sudo /opt/kafka/bin/kafka-server-stop.sh
```
Delete older logs if exists
```bash
sudo rm -rf /tmp/kraft-combined-logs/
sudo rm -rf /tmp/kafka-logs
```

Format the log directory (multi-nodes). Make sure all clusters use the SAME CLUSTER ID
```bash
sudo /opt/kafka/bin/kafka-storage.sh format -t <KAFKA_CLUSTER_ID> -c /opt/kafka/config/server.properties
```

### Allow firewall
```
sudo firewall-cmd --add-port=9092/tcp --permanent
sudo firewall-cmd --add-port=9093/tcp --permanent
sudo firewall-cmd --reload
```

## Start Kafka Server
```bash
sudo bin/kafka-server-start.sh config/server.properties
```

To Stop Server:
```bash
# if running in background
sudo bin/kafka-server-stop.sh config/server.properties

# to force stop, find the process, 
sudo lsof -i :9093
# then force kill it
sudo kill -9 <PID>
```

View cluster ID in future:
```bash
/opt/kafka/bin/kafka-cluster.sh cluster-id --bootstrap-server localhost:9092
```
*You can also use broker address instead of  localhost*

# Kafka UI

## Pre-requisites

### Install docker
```bash
sudo apt install -y docker docker-compose
```

Docker version we used
```bash
docker --version
Docker version 28.5.1, build e180ab8
```

To run docker without sudo:
```bash
sudo usermod -aG docker (whoami)
```


## Installing Kafka-UI (using docker)

Create docker-compose.yaml file
```bash
mkdir ~/kafka-ui; cd ~/kafka-ui
editor docker-compose.yaml
```

Add your configuration to the docker-compose.yaml file. Fill up `YOUR_IP`!
```yaml
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    network_mode: "host"
    environment:
      KAFKA_CLUSTERS_0_NAME: local-dgx
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: 10.11.200.99:9092
      DYNAMIC_CONFIG_ENABLED: 'true'
```

Install Kafka-UI
```bash
sudo docker compose up -d
```


# Testing

Check if cluster is reachable or not (optional)
```bash
/opt/kafka/bin/kafka-broker-api-versions.sh \
--bootstrap-server 10.11.200.99:9092
```

### Create Topic
```bash
/opt/kafka/bin/kafka-topics.sh \
--create \
--topic test-topic \
--bootstrap-server 10.11.200.99:9092 \
--partitions 3 \
--replication-factor 3
```
Here, 
- you can use any bootstrap-server address from your cluster's broker list
- Partition allows to deal with consumers parallel increasing performance, but takes more memory and space
- Replication will allow you to replicate the data across multiple brokers. Even if the leader server fail, next server will become the leader. Max limit: Number of brokers.

###  Create Consumer
```bash
/opt/kafka/bin/kafka-console-consumer.sh \
--topic test-topic \
--from-beginning \
--bootstrap-server 10.11.205.205:9092
```

### Create Producer
```bash
/opt/kafka/bin/kafka-console-producer.sh \
--topic test-topic \
--bootstrap-server 10.11.200.99:9092
```

Test Multiple Consumer
```bash
/opt/kafka/bin/kafka-console-consumer.sh \
--topic test-topic \
--group test-group \
--bootstrap-server 10.11.200.99:9092 \
--property print.partition=true \
--property print.offset=true
```