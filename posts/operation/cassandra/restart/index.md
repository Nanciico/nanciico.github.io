# Cassandra 运维 -- 重启节点


---

## 更改 Apollo 配置，下掉相应的节点
- `CassandraConfigV3`、`CassandraKeyspaceMappingV2` 两个配置都要下节点

---

## 在服务器上进行抓包看是否还有请求
```bash
tcpdump -s 0 -i bond0 port 9160
```

---

## 相应节点加维护
- 地址：https://starry.beisen-inc.com/#/monitor/maintenanceManagement

---

## 执行 `disablegossip`
```bash
/usr/local/cassandra-1.2.4/bin/nodetool disablegossip
```
- 检查其他机器的 ring，都认为这个机器 down 了
- 检查异常

---

## 抓包没有请求连接了，停止 Cassandra 服务
```bash
/usr/local/cassandra-1.2.4/bin/stop-server
```

---

## 关注 CassandraV2 异常，及时通知业务
- 日志查询地址：[Tara 平台错误日志](https://tara-platform.beisen-inc.com/#/logQuery/errorLogDetails?s=exprs:!(a),queries:(a:(filter:(must:!((disable:!f,field:time,operator:relative,type:SIMPLE,values:!('last%201%20day')),(disable:!f,field:customHash,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:productName,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:service,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:clusterName,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:applicationName,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:status,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:traceId,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:exceptionFullName,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:message,operator:match,type:SIMPLE,values:!()),(disable:!f,field:exceptionMessage,operator:match,type:SIMPLE,values:!()),(disable:!f,field:tenantId,operator:equal,type:SIMPLE,values:!()),(disable:!f,field:exceptionStackTrace,operator:match,type:SIMPLE,values:!(beisen.cassandrav2))),type:COMPLEX),from:ErrorLog))&o=time|desc&_v=1)

---

## 重启机器
```bash
reboot
```

---

## 等待机器启动

---

## 启动监控脚本
```bash
/etc/zabbix/scripts/cassandra_monitor_manager.sh --start all
```

---

## 启动服务
```bash
/usr/local/cassandra-1.2.4/bin/start-server
```

---

## 检查服务是否启动
```bash
ps -ef | grep cassandra
/usr/local/cassandra-1.2.4/bin/nodetool ring   # 当前机器和集群内其他机器分别检查
```

---

## 查看服务端日志
```bash
cat /opt/data/cassandra/logs/system.log | grep -i error | grep -v 'Read a frame of size'
```

---

## 关注 CassandraV2 异常，及时通知业务

---

## 如果存在大小为 0 的文件，对相应 Keyspace 执行 Repair
```bash
/usr/local/cassandra-1.2.4/bin/nodetool repair [keyspace]
```

---

## zabbix 观察操作机器监控数据是否正常

---

## 移除维护
