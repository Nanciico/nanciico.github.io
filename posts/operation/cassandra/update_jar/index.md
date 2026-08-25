# Cassandra 运维 -- 更换jar包


## 前提
需要先在服务器上准备好要更换的jar包

---

## 更改 Apollo 配置，下掉相应的节点
- `CassandraConfigV3` 、`CassandraKeyspaceMappingV2` 两个配置都要下节点

---

## 相应节点加维护
- 地址：https://starry.beisen-inc.com/#/monitor/maintenanceManagement

---

## 在服务器上进行抓包看是否还有请求
```bash
tcpdump -s 0 -i bond0 port 9160
```

---

## 抓包没有请求连接了，disable gossip
```bash
/usr/local/cassandra-1.2.4/bin/nodetool disablegossip

#4xx
/usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/bin/nodetool disablegossip

#recruitment snapshot
/usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/bin/nodetool disablegossip
```

---

## 集群其他机器检查节点处于 DOWN 状态

- 检查其他机器的 ring，都认为这个机器 down 了
- 检查异常

---

## 停止 Cassandra 服务
```bash
/usr/local/cassandra-1.2.4/bin/stop-server

#4xx
/usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/bin/stop-server

#recruitment snapshot
/usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/bin/stop-server
```

---

## 关注 CassandraV2 异常，及时通知业务
- 日志查询地址：[Tara 平台错误日志](https://tara-platform.beisen-inc.com/#/logQuery/errorLog?s=exprs:!(a),queries:(a:(filter:(must:!((disable:!f,field:time,operator:relative,type:SIMPLE,values:!(last%2012%20hours)),(disable:!f,field:productName,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:service,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:clusterName,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:applicationName,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:status,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:exceptionFullName,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:tenantId,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:traceId,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:message,operator:match,type:SIMPLE,values:!()),(disable:!f,field:exceptionStackTrace,operator:match,type:SIMPLE,values:!(Beisen.CassandraV2)),(disable:!f,field:exceptionMessage,operator:match,type:SIMPLE,values:!())),type:COMPLEX),from:ErrorLogAgg))&o=time|desc&_v=2)

---

## 更换 jar 包（做备份，以便于必要时回滚）
```bash
mv /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)
ll /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)

#4xx
mv /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)
ll /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)

#recruitment snapshot
mv /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)
ll /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)
```

更换jar包

修改权限：
```bash
chown beisen:beisen /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar

#4xx
chown beisen:beisen /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar

#recruitment snapshot
chown beisen:beisen /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar
```

检查jar包字节数、权限是否符合预期：2995608
```bash
ll /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar

#4xx
ll /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar

#recruitment snapshot
ll /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar
```

---

## 如果是 ZGC，检查 ZGC add opens 是否最新（如果有新增，文档需同步更新）
```bash
cat /usr/local/cassandra-1.2.4/conf/cassandra-env.sh | grep 'add-opens' | wc -l

#4xx
cat /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh | grep 'add-opens' | wc -l

#recruitment snapshot
cat /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh | grep 'add-opens' | wc -l
```

若不是最新，替换以下 JVM_OPTS
```bash
JVM_OPTS="$JVM_OPTS --add-opens java.base/java.io=ALL-UNNAMED \
                    --add-opens java.base/java.lang=ALL-UNNAMED \
                    --add-opens java.base/java.lang.ref=ALL-UNNAMED \
                    --add-opens java.base/java.lang.reflect=ALL-UNNAMED\
                    --add-opens java.base/java.math=ALL-UNNAMED \
                    --add-opens java.base/java.net=ALL-UNNAMED \
                    --add-opens java.base/java.nio=ALL-UNNAMED \
                    --add-opens java.base/java.nio.channels=ALL-UNNAMED \
                    --add-opens java.base/java.security=ALL-UNNAMED \
                    --add-opens java.base/sun.net.dns=ALL-UNNAMED \
                    --add-opens java.base/sun.nio.ch=ALL-UNNAMED \
                    --add-opens java.base/sun.security.provider=ALL-UNNAMED \
                    --add-opens java.base/sun.security.ssl=ALL-UNNAMED \
                    --add-opens java.base/java.util=ALL-UNNAMED \
                    --add-opens java.base/java.util.concurrent=ALL-UNNAMED \
                    --add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED \
                    --add-opens java.base/java.util.concurrent.locks=ALL-UNNAMED \
                    --add-opens java.management/sun.management=ALL-UNNAMED \
                    --add-exports java.base/sun.nio.ch=ALL-UNNAMED \
                    --add-exports java.base/jdk.internal.ref=ALL-UNNAMED"
```

---

## 启动服务
```bash
/usr/local/cassandra-1.2.4/bin/start-server

#4xx
/usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/bin/start-server

#recruitment snapshot
/usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/bin/start-server
```

---

## 检查服务是否启动
```bash
ps -ef | grep cassandra 
/usr/local/cassandra-1.2.4/bin/nodetool ring

#4xx
/usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/bin/nodetool ring

#recruitment snapshot
/usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/bin/nodetool ring
```
（当前机器和集群内其他机器分别检查）

---

## 查看服务端日志
```bash
cat /opt/data/cassandra/logs/system.log | grep -i error | grep -v 'Read a frame of size'

#4xx
cat /opt/data/cassandra/cassandra-1.2.4-steppecluster4xx/logs/system.log | grep -i error | grep -v 'Read a frame of size'

#recruitment snapshot
cat /opt/data/cassandra/cassandra-1.2.4-recruitment-snapshot/logs/system.log | grep -i error | grep -v 'Read a frame of size'
```

---

## 关注 CassandraV2 异常，及时通知业务

---

## 如果存在大小为 0 的文件，对相应 Keyspace 执行 Repair（不做）
```bash
/usr/local/cassandra/cassandra-1.2.4-steppecluster001/bin/nodetool repair [keyspace]
```

---

## 远程配置改成 true

---

## 移除维护

