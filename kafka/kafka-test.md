# 1. kafka配置

## 1. 配置docker

```yml
version: "3"
services:
  zookeeper:
    image: zookeeper:test
    hostname: vm-zookeeper
    ports:
      - "2181:2181"
    container_name: zookeeper
    restart: always
  kafka:
    image: kafka:test
    hostname: vm-kafka
    ports:
      - "9902:9902"
    container_name: kafka
    depends_on:
      - zookeeper
    environment:
      - KAFKA_BROKER_ID=0
      - KAFKA_ZOOKEEPER_CONNECT=192.168.13.142:2181
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.13.142:9902
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9902
    volumes:
      - $PWD/docker.sock:/var/run/docker.sock
    restart: always

```
1. 启动容器
2. 创建topic
```sh
#进入容器内部
docker exec -it kafka /bin/bash
#进入命令工具目录
cd /opt/kafka/bin/
#创建名称为mytest的topic
kafka-topics.sh --create --topic mytest --partitions 4 --zookeeper 192.168.13.142:2181 --replication-factor 1
```
3. 查看topic
```sh
kafka-topics.sh --zookeeper 192.168.13.142:2181 --describe --topic mytest
```
4. 生产消息
```sh
kafka-console-producer.sh --topic=mytest --broker-list 192.168.13.142:9902
```
5. 消费消息
```sh
kafka-console-consumer.sh --bootstrap-server 192.168.13.142:9902 --from-beginning --topic mytest
```