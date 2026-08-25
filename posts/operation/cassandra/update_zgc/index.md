# Cassandra 运维 -- 升级ZGC


## 检查机器使用的编码集为 UTF-8
使用 `locale` 命令检查：
- `LANG=en_US.UTF-8`
- `LC_CTYPE="en_US.UTF-8"`

---

## 更改 Apollo 配置，下掉相应的节点
- `CassandraConfigV3` 、`CassandraKeyspaceMappingV2` 两个配置都要下节点

---

## 在服务器上进行抓包看是否还有请求
```bash
tcpdump -s 0 -i bond0 port 9160
```

---

## 相应节点加维护
- 地址：https://starry.beisen-inc.com/#/monitor/maintenanceManagement

---

## disable 节点
```bash
/usr/local/cassandra-1.2.4/bin/nodetool disablegossip

#4xx
/usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/bin/nodetool disablegossip

#recruitment snapshot
/usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/bin/nodetool disablegossip
```
- 检查其他机器的 ring，都认为这个机器 down 了
- 检查异常

---

## 抓包没有请求连接了，停止 Cassandra 服务
```bash
/usr/local/cassandra-1.2.4/bin/stop-server

#4xx
/usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/bin/stop-server

#recruitment snapshot
/usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/bin/stop-server
```

---

## 关注 CassandraV2 异常，及时通知业务

---

## 关闭命令检查
```bash
echo OFFLINE > /beisen/server_status 
cat /beisen/server_status 
```

---

## 关闭预读

### 检查磁盘
```bash
df -h
```

### 针对 nvme0n1
```bash
#检查预读是否已经关闭
blockdev --getra /dev/nvme0n1
#如果没有则关闭
blockdev --setra 0 /dev/nvme0n1
blockdev --getra /dev/nvme0n1

#检查rc.local是否添加关闭预读命令
cat /etc/rc.local

#如果没有则添加
echo "blockdev --setra 0 /dev/nvme0n1" >> /etc/rc.local
cat /etc/rc.local
```

### 针对 vbda
```bash
#检查预读是否已经关闭
blockdev --getra /dev/vbda
#如果没有则关闭
blockdev --setra 0 /dev/vbda
blockdev --getra /dev/vbda

#检查rc.local是否添加关闭预读命令
cat /etc/rc.local

#如果没有则添加
echo "blockdev --setra 0 /dev/vbda" >> /etc/rc.local
cat /etc/rc.local
```

---

## 切换到 JDK 17

### 同步 JDK 包
```bash
rsync -avP easyops@10.23.40.32:/usr/local/java/jdk-17.0.2 /usr/local/java/
```

### 注释 `/etc/profile` 下 JDK 8 相关配置
```bash
vim /etc/profile

#export JAVA_HOME=/usr/local/java/jdk1.8.0_66
#export PATH=$PATH:$JAVA_HOME/bin
#export CLASSPATH=.:/usr/local/java/jdk1.8.0_66\lib/dt.jar:/usr/local/java/jdk1.8.0_66/lib/tools.jar
```

### 添加 JDK 17 配置
```bash
export JAVA_HOME=/usr/local/java/jdk-17.0.2
export CLASSPATH=.:$JAVA_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin

rm /usr/bin/java
ln -s /usr/local/java/jdk-17.0.2/bin/java /usr/bin/java

#重新打开一个xshell窗口检查
java -version
```

---

## 切换 ZGC 配置

### 备份原有 `cassandra-env.sh`
```bash
mv /usr/local/cassandra-1.2.4/conf/cassandra-env.sh /usr/local/cassandra-1.2.4/conf/cassandra-env.sh.$(date +%Y%m%d)

#4xx
mv /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh.$(date +%Y%m%d)

#recruitment snapshot
mv /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh.$(date +%Y%m%d)
```

### 传输新的 env 文件
```bash
rsync -avP easyops@10.23.40.131:/home/easyops/zgc/cassandra-env.sh /usr/local/cassandra-1.2.4/conf/

chown beisen:beisen /usr/local/cassandra-1.2.4/conf/cassandra-env.sh

#4xx
chown beisen:beisen /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh

#recruitment snapshot
rsync -avP easyops@10.23.40.131:/home/easyops/zgc/cassandra-env.sh /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/

chown beisen:beisen /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh
```

### 将 env 文件里的 IP 改成当前机器 IP
```bash
#sed -i "s#10.23.41.31#`hostname -I | sed 's/ *$//'`#g" /usr/local/cassandra-1.2.4/conf/cassandra-env.sh
sed -i "s#10.23.41.31#`hostname -I | awk '{print $1}'`#g" /usr/local/cassandra-1.2.4/conf/cassandra-env.sh
cat /usr/local/cassandra-1.2.4/conf/cassandra-env.sh | grep hostname

#4xx
sed -i "s#10.23.41.31#`hostname -I`#g" /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh
cat /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh | grep hostname

#recruitment snapshot
sed -i "s#10.23.41.31#`hostname -I | awk '{print $1}'`#g" /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh
cat /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh | grep hostname
```

### 检查 log 目录是否正确
```bash
#4xx
sed -i "s#/opt/data/cassandra/logs/gc.log#/opt/data/cassandra/cassandra-1.2.4/logs/gc.log#g" /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh

#recruitment snapshot
sed -i "s#/opt/data/cassandra/logs/gc.log#/opt/data/cassandra/cassandra-1.2.4-recruitment-snapshot/logs/gc.log#g" /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh

cat /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh | grep logs
```

### 临时将内存配置切换为 70G / 50G
```bash
sed -i "s#80G#65G#g" /usr/local/cassandra-1.2.4/conf/cassandra-env.sh
cat /usr/local/cassandra-1.2.4/conf/cassandra-env.sh | grep MAX_HEAP_SIZE

sed -i "s#60g#50g#g" /usr/local/cassandra-1.2.4/conf/cassandra-env.sh
cat /usr/local/cassandra-1.2.4/conf/cassandra-env.sh | grep SoftMaxHeapSize

#BeisenCloud
#File
sed -i "s#80G#60G#g" /usr/local/cassandra-1.2.4/conf/cassandra-env.sh
cat /usr/local/cassandra-1.2.4/conf/cassandra-env.sh | grep MAX_HEAP_SIZE

sed -i "s#60g#40g#g" /usr/local/cassandra-1.2.4/conf/cassandra-env.sh
cat /usr/local/cassandra-1.2.4/conf/cassandra-env.sh | grep SoftMaxHeapSize

#4xx
sed -i "s#80G#70G#g" /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh
cat /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh | grep MAX_HEAP_SIZE

sed -i "s#60g#50g#g" /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh
cat /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/conf/cassandra-env.sh | grep SoftMaxHeapSize

#recruitment snapshot
sed -i "s#80G#65G#g" /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh
cat /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh | grep MAX_HEAP_SIZE

sed -i "s#60g#50g#g" /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh
cat /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/conf/cassandra-env.sh | grep SoftMaxHeapSize
```

---

## 更换 jar 包

### 备份原有 jar 包
```bash
mv /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)
mv /usr/local/cassandra-1.2.4/lib/libthrift-0.7.0.jar /usr/local/cassandra-1.2.4/lib/libthrift-0.7.0.jar.$(date +%Y%m%d)

#4xx
mv /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)
mv /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/libthrift-0.7.0.jar /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/libthrift-0.7.0.jar.$(date +%Y%m%d)

#recruitment snapshot
mv /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar.$(date +%Y%m%d)
mv /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/libthrift-0.7.0.jar /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/libthrift-0.7.0.jar.$(date +%Y%m%d)
```

### 传输新 jar 包
```bash
rsync -avP easyops@10.23.40.131:/home/easyops/zgc/libthrift-0.7.0.jar easyops@10.23.40.131:/home/easyops/zgc/apache-cassandra-1.2.4.jar /usr/local/cassandra-1.2.4/lib/

#rsync -avP easyops@10.23.40.131:/home/easyops/zgc/libthrift-0.7.0.jar  /usr/local/cassandra-1.2.4/lib/
#rsync -avP easyops@10.23.40.131:/home/easyops/zgc/apache-cassandra-1.2.4.jar /usr/local/cassandra-1.2.4/lib/

#recruitment snapshot
rsync -avP easyops@10.23.40.131:/home/easyops/zgc/libthrift-0.7.0.jar easyops@10.23.40.131:/home/easyops/zgc/apache-cassandra-1.2.4.jar /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/
```

### 修改权限
```bash
chown beisen:beisen /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar
chown beisen:beisen /usr/local/cassandra-1.2.4/lib/libthrift-0.7.0.jar

#确认jar包大小和权限
#2995967
ll /usr/local/cassandra-1.2.4/lib/apache-cassandra-1.2.4.jar
#302205
ll /usr/local/cassandra-1.2.4/lib/libthrift-0.7.0.jar

#4xx
chown beisen:beisen /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar
chown beisen:beisen /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/libthrift-0.7.0.jar

#确认jar包大小和权限
ll /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/apache-cassandra-1.2.4.jar
ll /usr/local/cassandra/cassandra-1.2.4-steppecluster4xx/lib/libthrift-0.7.0.jar

#recruitment snapshot
chown beisen:beisen /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar
chown beisen:beisen /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/libthrift-0.7.0.jar

#确认jar包大小和权限
ll /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/apache-cassandra-1.2.4.jar
ll /usr/local/cassandra/cassandra-1.2.4-recruitment-snapshot/lib/libthrift-0.7.0.jar
```

---

## 停止现有 zabbix 监控
```bash
systemctl stop zabbix-agent
/etc/zabbix/scripts/cassandra_monitor_manager.sh --stop all

ps -ef | grep zabbix
#如果有未停止的脚本则人工停止
```

---

## 添加 ZGC 监控脚本和配置文件

### 备份历史脚本
```bash
mv /etc/zabbix/scripts/cassandra_monitor_manager.sh /etc/zabbix/scripts/cassandra_monitor_manager.sh.$(date +%Y%m%d)
```

### 传输脚本
```bash
rsync -avP easyops@10.23.40.131:/home/easyops/zgc/java_services_discovery_zgc.conf /etc/zabbix/zabbix_agentd.d/

rsync -avP easyops@10.23.40.131:/home/easyops/zgc/java_services_discovery_zgc.sh easyops@10.23.40.131:/home/easyops/zgc/java_gc_parser_zgc.sh easyops@10.23.40.131:/home/easyops/zgc/cassandra_monitor_manager.sh easyops@10.23.40.131:/home/easyops/zgc/log_file_monitor.sh /etc/zabbix/scripts/
#rsync -avP easyops@10.23.40.131:/home/easyops/zgc/java_services_discovery_zgc.sh /etc/zabbix/scripts/
#rsync -avP easyops@10.23.40.131:/home/easyops/zgc/java_gc_parser_zgc.sh /etc/zabbix/scripts/
#rsync -avP easyops@10.23.40.131:/home/easyops/zgc/cassandra_monitor_manager.sh /etc/zabbix/scripts/
```

### 检查 `log_file_monitor` 是否正确
```bash
cat /etc/zabbix/scripts/log_file_monitor.sh

#如果不正确
rsync -avP easyops@10.23.41.212:/etc/zabbix/scripts/log_file_monitor.sh /etc/zabbix/scripts/
```

### 赋予执行权限
```bash
chmod +x /etc/zabbix/scripts/cassandra_monitor_manager.sh /etc/zabbix/scripts/java_gc_parser_zgc.sh /etc/zabbix/scripts/java_services_discovery_zgc.sh
ll /etc/zabbix/scripts/

#chmod +x /etc/zabbix/scripts/cassandra_monitor_manager.sh
#chmod +x /etc/zabbix/scripts/java_gc_parser_zgc.sh
#chmod +x /etc/zabbix/scripts/java_services_discovery_zgc.sh
```

### zabbix 挂载模板
- Template Java ZGC

### 启动 zabbix
```bash
systemctl start zabbix-agent
```
- 观察 zabbix 是否出现 ZGC 监控

### 启动监控脚本
```bash
/etc/zabbix/scripts/cassandra_monitor_manager.sh --start all
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

## 检查 Cassandra 队列
```bash
ss -tln | grep 9160 | grep 1024
```

---

## 打开命令检查
```bash
echo ONLINE > /beisen/server_status 
cat /beisen/server_status 
```

---

## 关注 CassandraV2 异常，及时通知业务

---

## zabbix 观察 ZGC 监控数据是否正常，观察 GC 状况是否正常

---

## 检查 zabbix 脚本日志是否有错误，ZGC 相关日志
```bash
tail -10 /var/log/zabbix/java_gc_monitor.log
```

---

## 移除维护

