### 背景

RabbitMQ是遵从AMQP协议的， 换句话说 ，RabbitMQ就是 AMQP协议的 Erlang 的实现(当然 RabbitMQ 还支持 STOMP2、 MQTT3 等协议 )，RabbitMQ 中的交换器、交换器类型、队列、绑定、路由键等都是遵循的 AMQP 协议中相应的概念。目前 RabbitMQ 最新版本默认支持的是 AMQP 0-9-1



### 环境搭建

拉取RabbitMQ镜像

```bash
docker pull rabbitmq
```

创建RabbitMQ容器

```bash
docker run -d -p 1883:1883 -p 5671:5671 -p 5672:5672 -p 15672:15672 -p 15671:15671 -p 25672:25672 --name rabbitmq_test rabbitmq
# 暂未指定文件目录挂载
# 端口映射需要包括提前指定MQTT所需映射的端口1883，端口15672为web管理页面，5671和5672为AMQP 0-9-1 without and with TLS，其中端口5672为RabbitMQ消费者读取端口和mqtt发送端口，25672为server间内部通信口，15671为management监听端口
```

端口解释：

```bash
4369 (epmd，即Erlang Port Mapper Daemon，Erlang 端口映射守护进程), 
25672 (Erlang distribution，即server间内部通信端口)
5672, 5671 (AMQP 0-9-1 without and with TLS，即带有或不带有TLS的AMQP协议通信端口)
15672 (if management plugin is enabled，即web管理页面)
61613, 61614 (if STOMP is enabled)
1883, 8883 (if MQTT is enabled)
```

进入RabbitMQ容器

```bash
docker exec -it rabbitmq_test bash
```

加载mqtt插件以及RabbitMQ web管理插件

```bash
rabbitmq-plugins enable rabbitmq_mqtt
rabbitmq-plugins enable rabbitmq_management
```

修改RabbitMQ配置文件

```bash
apt update
apt install vim
vim /etc/rabbitmq/rabbiqmt.conf
```

```bash
# /etc/rabbitmq/rabbiqmt.conf


#loopback_users.guest = false
#listeners.tcp.default = 5672
#hipe_compile = false
# added

loopback_users = none
listeners.tcp.default = 5672
mqtt.listeners.tcp.default = 1883
## Default MQTT with TLS port is 8883
# mqtt.listeners.ssl.default = 8883

# anonymous connections, if allowed, will use the default
# credentials specified here
mqtt.allow_anonymous  = false
# mqtt.default_user     = guest
# mqtt.default_pass     = guest
# mqtt的默认用户为guest/guest，如果开启匿名允许，mqtt发送消息时不使用身份认证则会使用这个默认用户，建议修改这个默认用户名和密码或直接删除，删除后若开启允许匿名，并匿名发送则会报以下错误：
# MQTT login failed for "guest" auth_failure: Refused

mqtt.vhost            = /
mqtt.exchange         = amq.topic
# 24 hours by default
mqtt.subscription_ttl = 86400000
mqtt.prefetch         = 10 
mqtt.listeners.ssl = none

mqtt.tcp_listen_options.backlog = 128
mqtt.tcp_listen_options.nodelay = true
```

退出并重启容器

```bash
docker restart rabbitmq_test
```

此时RabbitMQ服务器搭建完成

然后创建RabbitMQ消费者（从队列中读取消息）

```python
# rabbitmq_consumer.py


#!/usr/bin/env python3
#-*- coding:utf-8 -*-

import pika
# ########################## 消费者 ##########################
# 创建连接到rabbitmq服务器的管道
credentials = pika.PlainCredentials('guest', 'guest')  # 已修改为admin2018, sscyanfa@2018
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost', 5672, '/', credentials))

channel = connection.channel()
channel.queue_declare(queue='chat')  # 需要先声明queue，不然读取时会报错没有该队列
channel.queue_bind(exchange='amq.topic', queue='chat')

# 定义一个回调函数来处理，这边的回调函数就是将信息打印出来。后期修改消息处理就修改此函数
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)

# 告诉rabbitmq使用callback来接收信息
channel.basic_consume(callback,
                      queue='chat',
                      no_ack=True)
 # no_ack=True表示在回调函数中不需要发送确认标识

print(' [*] Waiting for messages. To exit press CTRL+C')

# 开始接收信息，并进入阻塞状态，队列里有信息才会调用callback进行处理。按ctrl+c退出。

channel.start_consuming()
```

创建MQTT生产者

```python
# mqtt_client.py 


#!/usr/bin/env python3
# -*- coding:utf-8 -*-

import paho.mqtt.client as mqtt
#HOST = "172.16.0.12"
HOST = "127.0.0.1"
PORT = 1883
def test():
    client = mqtt.Client()
    client.username_pw_set('cccc', 'cccc')  # 需要在connect之前设定用户
    client.connect(HOST, PORT, 60)
    client.publish("chat","hello test check!", 2) #chat topic用于收集该业务的日志
    client.loop_forever()

if __name__ == '__main__':
    test()
```

可通过docker logs查看RabbitMQ容器日志

```bash
docker logs -f --tail 20 rabbitmq_test
```

先运行`rabbitmq_consumer.py`，再运行`mqtt_client.py`即可，其中`mqtt_client.py `运行一次只发送一个信息。

后期修改RabbitMQ消费者中的`callback`函数以进行数据处理。



#### **RabbitMQ相关用户配置：**

```bash
rabbitmqctl add_user test1 test1  # 添加用户名test1 密码test1
rabbitmqctl set_user_tags test1 none  # 此处将用户tag设置为none或者management似乎并不影响对虚拟机vhost的访问（对虚拟机vhost的访问权限在下一行权限设置中）
rabbitmqctl set_permissions -p / test1 '.*' '.*' '.*'  # conf/write/read
# 此处'/'即代表虚拟机vhost，而.*一定要用引号扩起来才能生效，否则报错命令参数过多，此处设置的权限是用户对于虚拟机vhost的权限（具体看下文权限及用户管理）
```

**注：RabbitMQ创建的用户是属于RabbitMQ自己的用户，只是同时可以提供给MQTT做身份验证，匿名登录只限制mqtt发送到RabbitMQ，并不能限制从RabbitMQ中读取，因为读取都是需要身份验证的。另外提到的用emqtt搭建的MQTT服务器提供的用户应该也只针对于emqtt使用，MQTT协议本身应该不包含用户的概念。以下有用户测试：**

**用户测试1**

此时将`rabbitmq_consumer.py`中的`credentials = pika.PlainCredentials('guest', 'guest')`一句修改为`credentials = pika.PlainCredentials('test1', 'test1')`也可正常读取消息。

将RabbitMQ配置中的允许匿名改为false后， mqtt_client.py中如果没有声明用户名和密码，则发送时RabbitMQ会报错"登录失败:没有提供凭证信息"：

```bash
2019-02-26 08:27:19.212 [error] <0.769.0> MQTT login failed: no credentials provided~n
2019-02-26 08:27:19.212 [info] <0.769.0> MQTT protocol error connect_expected for connection 172.30.0.1:56046 -> 172.30.0.2:1883
2019-02-26 08:27:20.218 [error] <0.772.0> MQTT login failed: no credentials provided~n
2019-02-26 08:27:22.223 [error] <0.778.0> MQTT login failed: no credentials provided~n
2019-02-26 08:27:26.229 [error] <0.781.0> MQTT login failed: no credentials provided~n
```

声明用户名和密码`    client.username_pw_set('test1', 'test1')`后则可正常运行。

**用户测试2**

关闭匿名发送的同时将`    client.username_pw_set('test1', 'test1')`中的用户名密码改为错误用户名密码`    client.username_pw_set('test333', 'test333')`后RabbitMQ会报错用户验证失败，但不会报连接错误，后期判断读取未成功时需要查看RabbitMQ的日志信息以判断是否为用户名、密码验证失败：

```bash
2019-02-26 08:31:48.604 [info] <0.1009.0> MQTT vhost picked using plugin configuration or default
2019-02-26 08:31:48.606 [error] <0.1009.0> MQTT login failed for "test3" auth_failure: Refused
2019-02-26 08:31:50.612 [info] <0.1015.0> MQTT vhost picked using plugin configuration or default
2019-02-26 08:31:50.613 [error] <0.1015.0> MQTT login failed for "test3" auth_failure: Refused
2019-02-26 08:31:54.619 [info] <0.1024.0> MQTT vhost picked using plugin configuration or default
2019-02-26 08:31:54.621 [error] <0.1024.0> MQTT login failed for "test3" auth_failure: Refused
```

### RabbbitMQ处理流程为：

`MQTT(携带身份信息) --> RabbitMQ(身份验证) --> 同一vhost --> 同一exchange -->不同身份信息对应的不同业务(即不同topic) --> bloomfilter --> mongo`

其中身份信息以及身份信息对应的业务topic由前端（或者后端或者终端）生成，只需考虑身份信息与topic的绑定问题。



### **权限及用户管理：**

#### 权限：

>AMQP 协议中并没有指 定权限在 vhost级别还是在服务器级别实现，由具体的应用自定义。在 RabbitMQ 中，权限控制则 是以 vhost 为单位的 。当创建一个用户时，用户通常会被指派给至少一个 vhost，并且只能访问被指派的 vhost 内的队列、交换器和绑定关系等。因此， RabbitMQ中的授予权限是指在 vhost 级别对用户而言的权限授予 。

>授予root用户可访问虚拟主机vhost1，并在所有资源上都具备可配置、可写及可读的权限，示例如下: 
>
>[root@nodel -]# rabbitmqctl set_permissions -p vhost1 root ".\*" ".\*" ".\*" 
>
>Setting permissions for user "root" in vhost "vhost1" 
>
>授予root用户可访问虚拟主机vhost2， 在以 "queue" 开头的资源上具备可配置权限， 并在所有资源上拥有可写、可读 的权限， 示 例如下: 
>
>[root@nodel -]# rabbitmqctl set_permi ssions -p vhost2 root "^queue.\*" ".\*" ".\*"
>
>Setting permissions for user "root" in vhost "vhost2" 
>
>清除权限也是在vhost级别对用户而言的。清除权限的命令为rabbitmqctl clear_permissions [-p vhost] {username}。其中vhost用于设置禁止用户访问的虚拟主机的名称，默认为"/"；username表示禁止访问特定虚拟主机的用户名称 。 
>
>示例如下: 
>
>[root@nodel -]# rabbitmqctl clear_permissions -p vhost1 root 
>
>Clearing permissions for user "root" in vhost "vhost1" 

**实际操作：**

```bash
# 先修改（这一步应该是保证test1对vhost有足够的权限）：
rabbitmqctl set_permissions -p / test1 '.*' '.*' '.*'  # 设定用户对于vhost：/的权限，分别为配置、写入、读取
# 再修改（这一步设定了test1只能访问名为queue1的消息队列，同时如果mqtt用此用户名发送到名为queue2的队列则不会发送成功，RabbitMQ也不会报错（待定））
# 使用'^()$'完全匹配来做队列的权限管理
rabbitmqctl set_topic_permissions -p / test1 amq.topic "^(queue1)$" "^(queue1)$"
```

**命令解释：**

```bash
rabbitmqctl set_permissions [-p <vhost>] <username> <conf> <write> <read>
# 注:可配置指的是队列和交换器的创建及删除之类的操作;可写指的是发布消息;可读指与消息有关的操作，包括读取消息及清空整个队列等。
# rabbitmqctl set_permissions参数：
vhost: 授予用户访问权限的 vhost名称，可以设置为默认值，即 vhost为 "/"。
user: 可以访问指定vhost 的用户名。
conf: 一个用于匹配用户在哪些资源上拥有可配置权限的正则表达式。
write: 一个用于匹配用户在哪些资源上拥有可写权限的正则表达式。
read: 一个用于匹配用户在哪些资源上拥有可读权限的正则表达式 。
```

```bash
rabbitmqctl set_topic_permissions [-p <vhost>] <username> <exchange> <write_pattern><read_pattern>
# vhost:授予用户访问权的虚拟主机的名称，默认为“/”。
user:目标虚拟主机中的权限适用的用户的名称。 
exchange:主题交换授权检查名称将应用于。 
write:与发布的消息的路由键匹配的正则表达式。 
read:与消费消息的路由键匹配的正则表达式。 
设置用户主题权限示例：
sudo rabbitmqctl set_topic_permissions -p /test_vhost tester amq.topic "^tester-.*" "^tester-.*"
该命令指示RabbitMQ代理让名为“tonyg”的用户发布和使用以“tonyg-”开头的路由密钥通过“/ myvhost”虚拟主机的“amp.topic”交换消息.
sudo rabbitmqctl set_topic_permissions -p /test_vhost tester amq.topic "^{username}-.*" "^{username}-.*"
主题权限支持以下变量的变量扩展：username，vhost和client_id。 
请注意，client_id仅在使用MQTT时才展开。前面的例子可以通过使用“^ {username} - 。*”来增加通用性。
```

> **下表展示了不同AMQP命令的列表和对应的权限：**

| AMQP命令                  | 可配置   | 可写                  | 可读             |
| ------------------------- | -------- | --------------------- | ---------------- |
| Exchange.Declare          | exchange |                       |                  |
| Exchange.Declare(with AE) | exchange | exchange(AX)          | exchange         |
| Exchange.Delete           | exchange |                       |                  |
| Queue.Declare             | queue    |                       |                  |
| Queue.Declare(with DLX)   | queue    | exchange(DLX)         | queue            |
| Queue.Delete              | queue    |                       |                  |
| Exchange.Bind             |          | exchange(destination) | exchange(source) |
| Exchange.Unbind           |          | exchange(destination) | exchange(source) |
| Queue.Bind                |          | queue                 | exchange         |
| Queue.Unbind              |          | queue                 | exchange         |
| Basic.Publish             |          | exchange              | exchange         |
| Basic.Get                 |          |                       | queue            |
| Basic.Consume             |          |                       | queue            |
| Queue.Purge               |          |                       | queue            |

#### 用户管理：

用户的角色分为 5 种类型：

- none: 无任何角色类型。新创建的用户的角色默认为 none。
- management: 可以访问 Web 管理页面。
- policymaker: 包含management的所有权限，并且可以管理策略(Policy)和参数(Parameter)。
- monitoring: 包含management的所有权限，并且可以看到所有连接、信道及节点相关的信息
- administrator: 包含 monitoring 的所有权限，井且可以管理用户、虚拟主机、权限、策略、参数等。administrator 代表了最高的权限。

#### 创建队列

```
Erlang:
channel.queueDeclare("myqueue", false, false, false, args);
Python:
channel.queue_declare(queue=topic)
# 以下使用python
```

RabbitMQ Consumer和Procuder都可以通过queue.declare创建queue，对于某一个channel来说，consumer不能declare一个queue，却订阅其他的queue。当然也可以创建私有的queue。这样只有APP本身才可以使用这个queue。queue也可以自动删除，被标为auto-delete的queue在最后一个Consumer unsubscribe后就会被自动删除。





#### TTL存活时间

RabbitMQ 会确保在过期时间到达后将队列删除，但是不保障删除的动作有多及时 。在RabbitMQ 重启后 ， 持久化的队列的过期时间会被重新计算。
