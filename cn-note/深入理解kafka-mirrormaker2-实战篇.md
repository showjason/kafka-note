
# 折腾 Kafka MirrorMaker2 集群：一份个人安装手记

最近在跟 Kafka 死磕，想着搭一个跨机房的数据同步方案，MirrorMaker2 自然就成了首选。所以，我决定自己从头到尾摸索一遍，把整个过程记录下来，权当是写给未来自己的备忘录，也希望能给同样在折腾的你一点点启发。


## 一、把“家伙事儿”都备齐

开工前，先得把需要的软件包都下载下来。我用的是 Kafka 3.8.1 版本。另外，为了监控，还需要一个 Prometheus 的 JMX Exporter Agent。（3.9.0 版本的 mirrormaker2 还有 bug，社区已经在修复）。

- **Kafka 安装包**: [kafka_2.13-3.8.1.tgz](https://archive.apache.org/dist/kafka/3.8.1/kafka_2.13-3.8.1.tgz)
- **Prometheus JMX Exporter**: [jmx_prometheus_javaagent-1.0.1.jar](https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/1.0.1/jmx_prometheus_javaagent-1.0.1.jar)

我的计划是在三台机器上部署，组成一个 MirrorMaker2 集群，这样就算挂了一台，服务也能继续跑，稳！

## 二、部署其实很简单

拿到安装包后，事情就简单了。我把 `kafka_2.13-3.8.1.tgz` 分别上传到三台服务器的 `/opt/kafka` 目录下解压。

解压后，目录结构大概是这样：

```bash
/opt/kafka/kafka_2.13-3.8.1/
├── bin
├── config
├── libs
├── ... (其他文件)
```

接下来是关键一步，为了让 Prometheus 能抓到 MirrorMaker2 的监控指标，我把下载好的 `jmx_prometheus_javaagent-1.0.1.jar` 扔进了 `libs` 目录。三台机器都要做同样的操作。

```bash
# 假设 jar 包在当前目录
mv jmx_prometheus_javaagent-1.0.1.jar /opt/kafka/kafka_2.13-3.8.1/libs/
```

## 三、配置 MirrorMaker2

软件部署好了，但它还不知道要干嘛。接下来就得靠配置文件来告诉它。我主要修改了 `config/connect-mirror-maker.properties` 这个文件。

这是我的配置，里面加了一些注释来解释每个参数是干嘛的。

```properties
# 定义两个集群的别名，后面都用这个别名来引用
clusters = clusterA, clusterB

# --- 源集群 (clusterA) 和目标集群 (clusterB) 的基本信息 ---
# 这是 MirrorMaker2 内部消费者组的 ID，随便起个有意义的名字就行
clusterA.group.id = clusterA-clusterB-clusterAconsumer
clusterB.group.id = clusterA-clusterB-clusterBconsumer

# MirrorMaker2 实例的名字
name = clusterA-clusterB
# 定义谁是源，谁是目标
source.cluster.alias = clusterA
target.cluster.alias = clusterB

# 最大并发任务数，可以根据 Topic 分区数和机器性能来调整
tasks.max = 8
# ⚡ tasks.max 详解：
# 这个参数定义了该连接器可以创建的最大并发任务数，但实际任务数可能更少
# 影响因素：
# 1. 源Topic分区总数：实际任务数不会超过所有匹配topic的分区总和（这是最主要的限制因素）
# 注意：这里指的是所有匹配 prod.*.global 正则表达式的topic的分区数总和，不是单个topic的最大分区数
# 2. 集群节点数：任务会分配到不同节点执行
# 3. 系统资源：CPU核心数、内存大小、网络带宽
# 4. 连接器类型：不同连接器有不同的并行化能力
# 设置原则：建议先根据源topic分区数设置，然后根据实际性能表现调整
# 我这里设置8是因为：预估匹配的所有topic总共有30-50个分区，8个任务可以合理分配负载

# --- 集群连接信息 ---
# 两个集群的 broker 地址
clusterA.bootstrap.servers = brokerA-1:9092, brokerA-2:9092, brokerA-3:9092
clusterB.bootstrap.servers = brokerB-1:9092, brokerB-2:9092, brokerB-3:9092

# --- 核心：定义同步规则 ---
# 启用从 clusterA 到 clusterB 的同步
clusterA->clusterB.enabled = true

# 正则表达式，匹配需要同步的 topic，我这里同步所有以 .global 结尾的 prod 前缀的 topic
clusterA->clusterB.topics = prod.*.global$
# 正则表达式，排除掉一些不需要的 topic，比如 MirrorMaker 内部的心跳 topic（实际测试中，发现如果不加这段配置，heartbears topic会循环镜像）
clusterA->clusterB.topics.exclude = ^(clusterA|clusterB).*heartbeats$

# 正则表达式，匹配需要同步的消费者组，这样消费位点也能同步过去
clusterA->clusterB.groups = ^produser_consumer.*

# 我这里只做单向同步，所以把反向的关掉
clusterB->clusterA.enabled = false

# --- 内部 Topic 和可靠性配置 ---
# 在目标集群创建的 topic 的副本数，生产环境必须大于1，我设为3
clusterA->clusterB.replication.factor = 3
checkpoints.topic.replication.factor = 3
heartbeats.topic.replication.factor = 3
offset-syncs.topic.replication.factor = 3

# Connect 框架内部也需要一些 topic 来存配置、位点、状态，副本数同样设为3
offset.storage.replication.factor = 3
status.storage.replication.factor = 3
config.storage.replication.factor = 3
config.storage.topic = mm2-config.from-clusterA-to-clusterB.internal
offset.storage.topic = mm2-offset.from-clusterA-to-clusterB.internal
status.storage.topic = mm2-status.from-clusterA-to-clusterB.internal

# --- 其他高级同步选项 ---
# 启用 ACL 同步、位点同步等
clusterA->clusterB.sync.topic.acls.enabled = true
clusterA->clusterB.emit.checkpoints.enabled = true
clusterA->clusterB.emit.offset-syncs.enabled = true
clusterA->clusterB.sync.group.offsets.enabled = true
# 定期刷新 topic 和 group 列表的时间间隔
clusterA->clusterB.refresh.topics.interval.seconds = 300
clusterA->clusterB.refresh.groups.interval.seconds = 300
clusterA->clusterB.sync.topic.acls.interval.seconds = 300

# --- 安全配置 ---
# 如果你的 Kafka 集群启用了 SASL/PLAIN 认证，就需要加上这些
clusterA.sasl.mechanism=PLAIN
clusterA.security.protocol=SASL_PLAINTEXT
clusterA.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="your_user" password="your_password";

clusterB.sasl.mechanism=PLAIN
clusterB.security.protocol=SASL_PLAINTEXT
clusterB.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="your_user" password="your_password";
```

除了这个，JMX Exporter 也需要一个简单的配置文件 `config/jmx_exporter.yml`，我这里就用了个最基础的配置，让它把所有 MBean 都暴露出来。

```yaml
---
# 这个文件可以留空，或者只写一个空的 yaml 对象 {}
# 也可以使用官方提供的 JMX MBeans
```

## 四、修改启动脚本

万事俱备，只欠东风。这个“东风”就是启动脚本。我没有直接用官方的 `connect-mirror-maker.sh`，而是复制了一份，然后做了点修改，让它加载 JMX Exporter。

这是我的 `bin/connect-mirror-maker.sh` 脚本：

```bash
#!/bin/bash
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# ... (省略官方的版权信息) ...

if [ $# -lt 1 ];
then
        echo "USAGE: $0 [-daemon] mm2.properties"
        exit 1
fi

base_dir=$(dirname $0)

# 设置日志目录
if [ "x$LOG_DIR" = "x" ]; then
    export LOG_DIR="/var/log/kafka"
fi

# 设置 Log4j 配置文件
if [ "x$KAFKA_LOG4J_OPTS" = "x" ]; then
    export KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:$base_dir/../config/connect-log4j.properties"
fi

# 设置 JVM 堆大小，这个根据服务器内存来定
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
  export KAFKA_HEAP_OPTS="-Xmx14336m -Xms14336m"
fi

# !!! 最核心的修改在这里 !!!
# 通过 javaagent 参数加载 JMX Exporter，并指定端口 7070 和配置文件
export KAFKA_OPTS="$KAFKA_OPTS -javaagent:$base_dir/../libs/jmx_prometheus_javaagent-1.0.1.jar=7070:$base_dir/../config/jmx_exporter.yml"

EXTRA_ARGS=${EXTRA_ARGS-'-name mirrorMaker'}

COMMAND=$1
case $COMMAND in
  -daemon)
    EXTRA_ARGS="-daemon "$EXTRA_ARGS
    shift
    ;;
  *)
    ;;
esac

# 执行官方的启动类
exec $(dirname $0)/kafka-run-class.sh $EXTRA_ARGS org.apache.kafka.connect.mirror.MirrorMaker "$@"
```

最重要的就是 `export KAFKA_OPTS` 那一行，它在启动 MirrorMaker2 的 JVM 进程时，动态地挂载了 JMX Exporter Agent，这样 Prometheus 就能通过 `7070` 端口来抓取监控数据了。

## 五、集群启动与验证

所有准备工作都完成了，我在三台机器上都执行了同样的启动命令：

```bash
# 确保脚本有执行权限
chmod +x /opt/kafka/kafka_2.13-3.8.1/bin/connect-mirror-maker.sh

# 后台启动 MirrorMaker2
/opt/kafka/kafka_2.13-3.8.1/bin/connect-mirror-maker.sh /opt/kafka/kafka_2.13-3.8.1/config/connect-mirror-maker.properties
```

启动后，我做了几件事来确认它是不是真的在工作：

1.  **看日志**：`tail -f /var/log/kafka/connect.log`，看看有没有报错。
2.  **查 Topic**：去目标集群 `clusterB` 上看看，是不是出现了 `clusterA.prod.some-topic.global` 这样的 Topic。
3.  **查 Metrics**：在任意一台 MirrorMaker2 的机器上，用 `curl` 访问 JMX Exporter 的端口。

    ```bash
    curl http://localhost:7070
    ```

    如果能看到一大堆以 `kafka_` 开头的监控指标，那就说明 JMX Exporter 也正常工作了。接下来就可以在 Prometheus 里配置一个 Job，让它来抓取这三台机器的 `7070` 端口，实现监控和告警。

## 六、这集群“结实”吗？聊聊它的高可用

搭完三节点的集群后，我心里冒出一个问题：这玩意儿到底有多可靠？万一哪天运维手滑，搞挂了一两台机器，我的数据同步任务会歇菜吗？带着这个疑问，我深入研究了一下它的工作模式。

### 它没有“大脑”，但活得很好

我最开始有个误解，以为 MirrorMaker2 集群内部也得像 ZooKeeper 那样，搞个投票选举，选个“老大”出来发号施令。结果发现，**它根本没有这种“大脑”或者“法定人数（Quorum）”的设定**。

它的聪明之处在于，把自己完全“托管”给了 Kafka。它就是个“甩手掌柜”，把所有脏活累活都丢给了 Kafka Connect 框架和 Kafka 集群本身：

1.  **“我们是一个团队”**：我的三个 MirrorMaker2 节点在启动后，会向 Kafka 报到，说：“我们都是 `clusterA-clusterB` 这个组的成员”。于是 Kafka 的“小组长”（Group Coordinator）就把它们当成一个消费者团队来管理。
2.  **“工作日志”都记在 Kafka**：所有的工作状态，比如“我同步到哪里了？”（Offsets）、“团队的规章制度是什么？”（Configs），全都记录在 Kafka 的内部 Topic 里。只要 Kafka 集群本身是高可用的（这也是为什么我们把内部 Topic 的副本数设为3），这些工作日志就不会丢。

所以，MirrorMaker2 集群的可靠性，本质上依赖于你背后 Kafka 集群的可靠性。它自己是个“无状态”的执行者，非常灵活。

### 极限测试：干掉两个节点会怎样？

理论归理论，实践出真知。如果我的三节点集群真的挂了两个，会发生什么？

答案是：**只要还有一个节点在喘气，活儿就不会停！**

整个过程就像这样：

1.  **Kafka 小组长发现有人旷工**：Kafka 的 Group Coordinator 很快会发现：“咦，团队里有两个人没心跳了！”
2.  **紧急召开重组会议 (Rebalance)**：小组长立刻组织剩下的人（也就是那个幸存的节点）开会，说：“情况有变，工作得重新分配。”
3.  **幸存者扛起所有**：所有之前分配给那两个失败节点的工作任务，现在都会被打包，一股脑儿全部分配给这最后一个幸存的节点。
4.  **翻开工作日志，继续干**：这个“幸运儿”会先去内部 Topic 里翻看最新的工作日志，找到之前大家同步到的精确位置，然后接着干活，保证数据不重不漏。

**一句话总结：**

这种设计确保了服务的高可用性，哪怕是“伤亡惨重”到只剩一个节点，也能保证业务连续。当然，代价就是剩下的那个兄弟会压力山大，数据同步可能会因为性能跟不上而产生延迟（Lag）。所以，生产环境中还是得尽快把挂掉的节点救活，让团队恢复完整的战斗力。

## 写在最后

整个过程下来，感觉 MirrorMaker2 还是挺强大的，配置项虽然多，但都很灵活。最大的收获是把监控也一并搞定了，这样服务上线后心里才有底。从下载软件包到最后看到 Grafana 监控图上跳动的曲线，整个过程充满了解决问题的乐趣。希望这份记录能帮到有缘人。
