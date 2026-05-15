# Linux运维常用命令


## 系统信息与状态查看

``` bash
# 查看CPU信息
lscpu
cat /proc/cpuinfo

# 查看内存信息
free -h
# 显示：
#               total        used        free      shared  buff/cache   available
# Mem:           7.7G        2.1G        3.2G        123M        2.4G        5.2G
# Swap:          2.0G          0B        2.0G

# 查看磁盘空间
df -h
# 显示：
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   15G   33G  32% /
# /dev/sda2       200G   45G  145G  24% /data

# 查看磁盘inode使用情况
df -i
```

---

## 文件与目录操作

### 基本文件操作

``` bash
# 递归创建目录
mkdir -p /opt/apps/{bin,logs,conf}

# 复制并保留属性
cp -rp source_dir/ destination_dir/

# 查看大文件（支持搜索和翻页）
less /var/log/syslog

# 显示文件最后100行
tail -n 100 /var/log/messages

# 显示文件开头50行
head -n 50 /var/log/secure
```

### 文件查找

``` bash
# 按文件名查找
find / -name "nginx.conf" -type f 2>/dev/null

# 按文件大小查找（大于100MB）
find /var -size +100M -type f

# 按修改时间查找（7天内修改的文件）
find /home -mtime -7 -type f

# 查找并处理（压缩7天前的日志）
find /var/log -name "*.log" -mtime +7 -exec gzip {} \;
```

---

## 性能监控与进程管理

### 实时监控

``` bash
# 动态查看进程和资源使用
top

# 监控IO状态
iostat -x 1

# 监控CPU和内存
vmstat 1 5
```

### 系统资源监控

``` bash
# 监控网络连接
ss -tln -p
ss -uln -p

# 查看内存使用详情
cat /proc/meminfo

# 监控系统负载
sar -u 1 3             # CPU使用率
sar -r 1 3             # 内存使用率
sar -b 1 3             # IO使用率
```

---

## 网络管理

### 端口与连接管理

``` bash
# 查看监听端口
netstat -tlnp
ss -tln -p

# 查看所有连接
netstat -an
ss -an

# 查看特定端口占用
lsof -i :80
netstat -tulnp | grep :80

# 检查远程端口是否开放
telnet 192.168.1.100 22
nc -zv 192.168.1.100 22
```

### 网路服务

``` bash
# DNS查询
nslookup google.com
dig google.com A

# 下载文件
wget http://example.com/file.tar.gz
curl -O http://example.com/file.tar.gz

# 测试HTTP服务
curl -I http://example.com
```

---

## 系统服务管理（Systemd）

### 服务管理

``` bash
# 服务状态管理
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx      # 重载配置
systemctl enable nginx      # 开机自启
systemctl disable nginx     # 禁用开机自启

# 查看所有服务
systemctl list-units --type=service
```

### 日志管理

``` bash
# 查看系统日志
journalctl -f
tail -f /var/log/messages
```

---

## 文本处理三剑客

### grep 文本搜索

``` bash
# 基本搜索
grep "error" /var/log/syslog
grep -i "warning" /var/log/messages    # 忽略大小写
grep -v "debug" /var/log/app.log       # 反向匹配

# 递归搜索
grep -r "Connection refused" /var/log/

# 显示上下文
grep -A 2 -B 2 "critical" /var/log/application.log
```

### sed 流编辑器

``` bash
# 文本替换
sed 's/foo/bar/g' file.txt
sed -i 's/old/new/g' file.conf         # 原地修改

# 删除行
sed '/^#/d' config.conf                # 删除注释行
sed '/^$/d' file.txt                   # 删除空行

# 提取特定行
sed -n '10,20p' large_file.log         # 显示10-20行
```

### awk 文本分析

``` bash
# 提取字段
awk '{print $1}' access.log            # 打印第一列
awk -F: '{print $1, $6}' /etc/passwd   # 指定分隔符

# 条件处理
awk '$3 > 100 {print $0}' data.txt     # 第三列大于100的行
awk '/error/ {count++} END {print count}' app.log  # 统计错误次数

# 数值计算
awk '{sum += $1} END {print sum}' numbers.txt
```

---

## 磁盘管理

### 磁盘使用分析

``` bash
# 查看目录大小
du -sh /var/log/
du -h --max-depth=1 /home/

# 查找大文件
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# 磁盘IO监控
iostat -dx 1
```

### 文件系统操作

``` bash
# 挂载文件系统
mount /dev/sdb1 /mnt/data
umount /mnt/data

# 查看挂载信息
mount | grep /dev/sd
findmnt

# 检查磁盘错误
fsck /dev/sda1
```

---

## 备份与压缩

### 打包压缩

``` bash
# 创建压缩包
tar -czvf backup.tar.gz /path/to/backup/

# 解压缩
tar -xzvf backup.tar.gz

# 查看压缩包内容
tar -tzf backup.tar.gz
```

### 数据同步

``` bash
# 文件同步
rsync -avz /source/ user@remote:/destination/
rsync -avz --delete /src/ /backup/    # 同步时删除目标多余文件

# 增量备份
tar -czvf backup-$(date +%Y%m%d).tar.gz --newer-mtime="2023-10-01" /data/
```

---

