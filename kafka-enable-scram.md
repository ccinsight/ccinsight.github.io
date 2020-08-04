---
title: "Kafka Enable SCRAM"
date: 2020-08-04T13:15:55+08:00
summary: "Kafka 启用 SCRAM 权限认证"
isCJKLanguage: true
toc: true
tags: ["kafka"]
categories: ["tech"]
comment: true
draft: false
---

# Kafka 启用 SCRAM 权限认证



## 目标&环境

### 环境信息

- CentOS 7.6
- 内网IP：10.0.2.15（模拟）
- 外网IP：192.168.56.11（模拟）
- Kafka：2.3.1

### 目标

- 内网访问不启用 SCRAM
- 外网访问启用 SCRAM

## 配置

### 创建 SCRAM Principal

```bash
# 管理员账户
bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-256=[password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' --entity-type users --entity-name admin

# 普通用户，稍后用这个用户验证权限
bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=alice-secret],SCRAM-SHA-512=[password=alice-secret]' --entity-type users --entity-name alice
```

> 信息是保存在`ZooKeeper`里的，`Kafka`不启动也能执行。

### 配置 Kafka Broker 

```properties
sasl.enabled.mechanisms=SCRAM-SHA-256
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
security.inter.broker.protocol=SASL_PLAINTEXT
listeners=PLAINTEXT://10.0.2.15:9092,SASL_PLAINTEXT://192.168.56.11:9093
advertised.listeners=PLAINTEXT://10.0.2.15:9092,SASL_PLAINTEXT://192.168.56.11:9093

listener.name.sasl_plaintext.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
   username="admin" \
   password="admin-secret";

super.users=User:admin
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
```

重启`Kafka Broker`。

### 授权

要实现内网不需要认证授权，需要给匿名用户授权。

```bash
# 给`Anonymous`用户授权
bin/kafka-acls.sh --add --authorizer-properties zookeeper.connect=localhost:2181 --allow-principal User:ANONYMOUS --allow-host '*' --operation All --topic '*' --group '*'
```

> 信息是保存在`ZooKeeper`里的，`Kafka`不启动也能执行。



## 验证

### 准备

```bash
# 创建一个`topic`用于后续验证
bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic test-topic --partitions 1  --replication-factor 1
```

### 内网无密码访问

```bash
# 写入
bin/kafka-console-producer.sh --broker-list 10.0.2.15:9092 --topic test-topic
# 读取
bin/kafka-console-consumer.sh --bootstrap-server 10.0.2.15:9092 --topic test-topic --from-beginning
```

![Access-Plaintext](/images/kafka_plain_1.png)



### 外网无密码访问

预期会报错，要求认证。

```bash
# 写入
bin/kafka-console-producer.sh --broker-list 192.168.56.11:9093 --topic test-topic
# 读取
bin/kafka-console-consumer.sh --bootstrap-server 192.168.56.11:9093 --topic test-topic --from-beginning
```

![Access-SASL-without-principal](/images/kafka_plain_2.png)



### 外网访问

#### 授权

```bash
# 授权`alice`用户作为生产者
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:alice --producer --topic test-topic

# 授权`alice`用户作为消费者
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:alice --consumer --topic test-topic --group '*'
```



#### 创建配置文件

使用`Kafka`命令行工具生产和消费时需要几个配置文件。

```bash
tee kafka_client_jaas_alice.conf >/dev/null <<EOF
KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="alice"
  password="alice-secret";
};
EOF

tee client_sasl.properties >/dev/null <<EOF
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
EOF
```

#### 提供凭证连接 Kafka

```bash
# 写入
KAFKA_OPTS=-Djava.security.auth.login.config=$(pwd)/kafka_client_jaas_alice.conf \
  bin/kafka-console-producer.sh --broker-list 192.168.56.11:9093 --producer.config $(pwd)/client_sasl.properties --topic test-topic

# 读取
KAFKA_OPTS=-Djava.security.auth.login.config=$(pwd)/kafka_client_jaas_alice.conf \
  bin/kafka-console-consumer.sh --bootstrap-server 192.168.56.11:9093 --consumer.config $(pwd)/client_sasl.properties --topic test-topic --from-beginning
```

![Access-SASL-with-principal](/images/kafka_sasl_1.png)

### 使用客户端连接

把`Kafka`客户端的连接的`properties`，加上认证信息就行：

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="alice" \
    password="alice-secret";
```

可以参考这个：

https://www.instaclustr.com/support/documentation/kafka/accessing-and-using-kafka/use-kafka-with-java/

