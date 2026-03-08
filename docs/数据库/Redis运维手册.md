# Redis 运维手册

> 版本：Redis 7.x  
> 适用对象：运维工程师、开发工程师  
> 最后更新：2026-03-08

---

## 目录

1. [Redis 安装部署](#一-redis-安装部署)
2. [配置管理](#二-配置管理)
3. [日常运维操作](#三-日常运维操作)
4. [性能监控与优化](#四-性能监控与优化)
5. [数据备份与恢复](#五-数据备份与恢复)
6. [高可用方案实施](#六-高可用方案实施)
7. [常见故障处理](#七-常见故障处理)
8. [安全加固](#八-安全加固)

---

## 一、Redis 安装部署

### 1.1 环境准备

#### 1.1.1 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | Linux (CentOS 7/8, Ubuntu 18.04/20.04/22.04) |
| 内存 | 建议 4GB 以上 |
| 磁盘 | SSD 推荐，用于持久化存储 |
| 网络 | 内网环境，避免公网暴露 |

#### 1.1.2 依赖安装

```bash
# CentOS/RHEL
yum install -y gcc gcc-c++ make tcl wget

# Ubuntu/Debian
apt-get update
apt-get install -y build-essential tcl wget
```

### 1.2 源码安装

#### 1.2.1 下载与编译

```bash
# 创建安装目录
mkdir -p /opt/redis
cd /opt/redis

# 下载 Redis 7.2.4（最新稳定版）
wget https://download.redis.io/releases/redis-7.2.4.tar.gz
tar xzf redis-7.2.4.tar.gz
cd redis-7.2.4

# 编译
make
make test
make install PREFIX=/usr/local/redis

# 创建软链接
ln -s /usr/local/redis/bin/redis-server /usr/local/bin/redis-server
ln -s /usr/local/redis/bin/redis-cli /usr/local/bin/redis-cli
```

#### 1.2.2 创建系统服务

```bash
# 创建 redis 用户
useradd -r -s /sbin/nologin redis

# 创建目录
mkdir -p /etc/redis
mkdir -p /var/lib/redis
mkdir -p /var/log/redis
chown -R redis:redis /var/lib/redis
chown -R redis:redis /var/log/redis

# 复制配置文件
cp /opt/redis/redis-7.2.4/redis.conf /etc/redis/redis.conf

# 创建 systemd 服务文件
cat > /etc/systemd/system/redis.service << 'EOF'
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 启动服务
systemctl daemon-reload
systemctl enable redis
systemctl start redis
systemctl status redis
```

### 1.3 Docker 安装

#### 1.3.1 单节点部署

```bash
# 拉取镜像
docker pull redis:7.2-alpine

# 运行容器
docker run -d \
  --name redis \
  --restart always \
  -p 6379:6379 \
  -v /data/redis/data:/data \
  -v /data/redis/redis.conf:/usr/local/etc/redis/redis.conf \
  redis:7.2-alpine \
  redis-server /usr/local/etc/redis/redis.conf
```

#### 1.3.2 Docker Compose 部署

```yaml
version: '3.8'
services:
  redis:
    image: redis:7.2-alpine
    container_name: redis
    restart: always
    ports:
      - "6379:6379"
    volumes:
      - /data/redis/data:/data
      - /data/redis/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    sysctls:
      - net.core.somaxconn=65535
    ulimits:
      nofile:
        soft: 65535
        hard: 65535
```

### 1.4 安装验证

```bash
# 检查版本
redis-server --version
redis-cli --version

# 连接测试
redis-cli ping
# 应返回 PONG

# 查看服务器信息
redis-cli info server
```

---

## 二、配置管理

### 2.1 核心配置参数

#### 2.1.1 网络配置

```conf
# 绑定地址，0.0.0.0 表示监听所有接口
bind 0.0.0.0

# 监听端口
port 6379

# TCP 连接队列长度
tcp-backlog 511

# 超时时间（秒），0 表示永不超时
timeout 0

# TCP keepalive
tcp-keepalive 300

# 保护模式
protected-mode yes
```

#### 2.1.2 内存配置

```conf
# 最大内存限制
maxmemory 4gb

# 内存淘汰策略
# volatile-lru: 从设置了过期时间的键中使用 LRU 淘汰
# allkeys-lru: 从所有键中使用 LRU 淘汰
# volatile-lfu: 从设置了过期时间的键中使用 LFU 淘汰
# allkeys-lfu: 从所有键中使用 LFU 淘汰
# volatile-random: 从设置了过期时间的键中随机淘汰
# allkeys-random: 从所有键中随机淘汰
# volatile-ttl: 淘汰即将过期的键
# noeviction: 不淘汰，写入时报错
maxmemory-policy allkeys-lru

# 采样数量，用于近似 LRU/LFU
maxmemory-samples 5
```

#### 2.1.3 持久化配置

**RDB 配置：**
```conf
# 保存策略：900秒内至少1个key变化则保存
save 900 1
save 300 10
save 60 10000

# 禁用 RDB
# save ""

# RDB 文件名
dbfilename dump.rdb

# RDB 文件保存路径
dir /var/lib/redis

# 压缩 RDB 文件
rdbcompression yes

# RDB 文件校验
rdbchecksum yes
```

**AOF 配置：**
```conf
# 启用 AOF
appendonly yes

# AOF 文件名
appendfilename "appendonly.aof"

# 同步策略
# always: 每次写入都同步（最安全，最慢）
# everysec: 每秒同步（推荐，平衡安全和性能）
# no: 由操作系统决定同步时机（最快，最不安全）
appendfsync everysec

# 重写时不执行 fsync
no-appendfsync-on-rewrite no

# 自动重写触发条件
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# AOF 加载时是否截断不完整文件
aof-load-truncated yes

# 开启混合持久化（Redis 4.0+）
aof-use-rdb-preamble yes
```

#### 2.1.4 安全配置

```conf
# 设置密码
requirepass your_strong_password

# 命令重命名（禁用危险命令）
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_9f3b2a1e"

# ACL 配置（Redis 6.0+）
aclfile /etc/redis/users.acl
```

### 2.2 配置文件模板

```conf
# Redis 7.x 配置文件模板
# ============================================

# 基础配置
bind 0.0.0.0
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300

# 守护进程
daemonize no
supervised systemd
pidfile /var/run/redis/redis-server.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16

# 持久化配置
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# AOF 配置
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes

# 内存管理
maxmemory 4gb
maxmemory-policy allkeys-lru
maxmemory-samples 5

# 安全配置
protected-mode yes
requirepass your_strong_password

# 客户端配置
maxclients 10000

# 慢查询日志
slowlog-log-slower-than 10000
slowlog-max-len 128

# 事件通知
notify-keyspace-events ""

# 高级配置
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
```

### 2.3 配置热加载

```bash
# 查看当前配置
redis-cli CONFIG GET *

# 修改单个配置（立即生效，重启后失效）
redis-cli CONFIG SET maxmemory 8gb
redis-cli CONFIG SET maxmemory-policy allkeys-lfu

# 将当前配置写入配置文件（持久化）
redis-cli CONFIG REWRITE
```

---

## 三、日常运维操作

### 3.1 连接与认证

```bash
# 本地连接
redis-cli

# 远程连接
redis-cli -h 192.168.1.100 -p 6379

# 带密码连接
redis-cli -h 192.168.1.100 -p 6379 -a password

# 或者连接后认证
redis-cli -h 192.168.1.100 -p 6379
AUTH password
```

### 3.2 键值操作

#### 3.2.1 Key 操作

```bash
# 查看所有 key（生产环境慎用）
KEYS pattern

# 更安全的扫描方式
SCAN 0 MATCH user:* COUNT 100

# 查看 key 是否存在
EXISTS key

# 查看 key 类型
TYPE key

# 查看 key 过期时间
TTL key        # 秒
PTTL key       # 毫秒

# 设置过期时间
EXPIRE key 60       # 60秒后过期
EXPIREAT key timestamp  # 指定时间戳过期
PERSIST key         # 移除过期时间

# 删除 key
DEL key
UNLINK key          # 异步删除（推荐大数据量）

# 重命名
RENAME key newkey
RENAMENX key newkey  # 仅当 newkey 不存在时
```

#### 3.2.2 String 操作

```bash
# 设置值
SET key value
SETNX key value          # 仅当 key 不存在时设置
SETEX key 60 value       # 设置并指定过期时间
MSET key1 val1 key2 val2 # 批量设置

# 获取值
GET key
MGET key1 key2

# 数值操作
INCR key
INCRBY key 10
DECR key
DECRBY key 5

# 其他操作
APPEND key value
STRLEN key
GETRANGE key 0 5
```

#### 3.2.3 Hash 操作

```bash
# 设置字段
HSET user:1001 name "张三"
HMSET user:1001 name "张三" age 25 city "北京"

# 获取字段
HGET user:1001 name
HMGET user:1001 name age
HGETALL user:1001
HKEYS user:1001
HVALS user:1001

# 其他操作
HLEN user:1001
HEXISTS user:1001 name
HDEL user:1001 age
HINCRBY user:1001 age 1
```

#### 3.2.4 List 操作

```bash
# 添加元素
LPUSH queue:task "task1"
RPUSH queue:task "task2"

# 获取元素
LRANGE queue:task 0 -1
LINDEX queue:task 0

# 弹出元素
LPOP queue:task
RPOP queue:task
BLPOP queue:task 30      # 阻塞式弹出，超时30秒

# 其他操作
LLEN queue:task
LREM queue:task 1 "task1"
LTRIM queue:task 0 99    # 只保留前100个
```

#### 3.2.5 Set 操作

```bash
# 添加元素
SADD tags:post:1001 redis
SADD tags:post:1001 mysql

# 获取元素
SMEMBERS tags:post:1001
SRANDMEMBER tags:post:1001 1

# 判断元素是否存在
SISMEMBER tags:post:1001 redis

# 删除元素
SREM tags:post:1001 mysql
SPOP tags:post:1001

# 集合运算
SINTER tags:post:1001 tags:post:1002    # 交集
SUNION tags:post:1001 tags:post:1002    # 并集
SDIFF tags:post:1001 tags:post:1002     # 差集
```

#### 3.2.6 Sorted Set 操作

```bash
# 添加元素
ZADD leaderboard 100 "player1"
ZADD leaderboard 200 "player2" 150 "player3"

# 获取元素（按分数排序）
ZRANGE leaderboard 0 -1 WITHSCORES          # 从小到大
ZREVRANGE leaderboard 0 -1 WITHSCORES       # 从大到小
ZRANGEBYSCORE leaderboard 100 200 WITHSCORES

# 获取排名
ZRANK leaderboard "player1"                 # 从小到大排名
ZREVRANK leaderboard "player1"              # 从大到小排名

# 获取分数
ZSCORE leaderboard "player1"

# 删除元素
ZREM leaderboard "player1"
ZREMRANGEBYRANK leaderboard 0 9             # 删除前10名

# 其他操作
ZCARD leaderboard                           # 元素数量
ZCOUNT leaderboard 100 200                  # 指定分数范围数量
ZINCRBY leaderboard 10 "player1"            # 增加分数
```

### 3.3 服务器管理

```bash
# 查看服务器信息
INFO                    # 所有信息
INFO server             # 服务器信息
INFO clients            # 客户端信息
INFO memory             # 内存信息
INFO persistence        # 持久化信息
INFO stats              # 统计信息
INFO replication        # 复制信息
INFO cpu                # CPU 信息
INFO commandstats       # 命令统计
INFO latencystats       # 延迟统计

# 查看客户端连接
CLIENT LIST
CLIENT INFO

# 关闭客户端连接
CLIENT KILL addr ip:port
CLIENT KILL ID client-id
CLIENT KILL TYPE normal

# 查看当前连接数
DBSIZE

# 查看慢查询
SLOWLOG GET 10
SLOWLOG LEN
SLOWLOG RESET

# 查看实时命令监控
MONITOR                   # 生产环境慎用

# 数据采样分析
--latency                 # 延迟测试
--latency-history         # 延迟历史
--stat                    # 实时统计

# 数据迁移
MIGRATE host port key destination-db timeout
MIGRATE host port "" destination-db timeout KEYS key1 key2
```

### 3.4 事务操作

```bash
# 事务示例
MULTI
SET balance:1001 1000
DECRBY balance:1001 100
INCRBY balance:1002 100
EXEC

# 乐观锁示例
WATCH balance:1001
GET balance:1001
MULTI
DECRBY balance:1001 100
EXEC
# 如果 balance:1001 被其他客户端修改，EXEC 返回 nil
```

### 3.5 发布订阅

```bash
# 订阅频道
SUBSCRIBE channel1
PSUBSCRIBE news.*

# 发布消息
PUBLISH channel1 "hello world"

# 查看订阅信息
PUBSUB CHANNELS
PUBSUB NUMSUB channel1
PUBSUB NUMPAT
```

---

## 四、性能监控与优化

### 4.1 监控指标

#### 4.1.1 关键性能指标

| 指标 | 说明 | 建议阈值 |
|------|------|----------|
| used_memory | 已使用内存 | < maxmemory 的 80% |
| used_memory_rss | 系统分配给 Redis 的内存 | 与 used_memory 差值 < 20% |
| mem_fragmentation_ratio | 内存碎片率 | 1.0 - 1.5 正常 |
| connected_clients | 连接客户端数 | < maxclients 的 80% |
| blocked_clients | 阻塞客户端数 | < 10 |
| instantaneous_ops_per_sec | 每秒操作数 | 根据业务评估 |
| keyspace_hits | 键空间命中次数 | - |
| keyspace_misses | 键空间未命中次数 | 命中率 > 95% |
| latest_fork_usec | 最后一次 fork 耗时 | < 1000ms |

#### 4.1.2 获取监控数据

```bash
# 内存信息
redis-cli INFO memory

# 统计信息
redis-cli INFO stats

# 计算命中率
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
# 命中率 = keyspace_hits / (keyspace_hits + keyspace_misses) * 100%

# 查看最大内存
redis-cli INFO memory | grep maxmemory

# 查看持久化状态
redis-cli INFO persistence
```

### 4.2 性能测试

```bash
# 基准测试
redis-benchmark -h 127.0.0.1 -p 6379 -c 50 -n 100000

# 测试特定命令
redis-benchmark -t set,get -n 100000

# 测试数据大小
redis-benchmark -d 1000 -n 100000

# 测试管道性能
redis-benchmark -P 16 -n 100000

# 测试集群模式
redis-benchmark -h 192.168.1.100 -p 6379 -c 100 -n 100000 --cluster
```

### 4.3 性能优化

#### 4.3.1 内存优化

```bash
# 1. 使用合适的数据结构
# String: 存储简单值
# Hash: 存储对象（字段数 < 512，值 < 64B 时使用 ziplist）
# List: 队列/栈
# Set: 去重集合
# Sorted Set: 排行榜

# 2. 设置合理的过期时间
EXPIRE key 3600

# 3. 使用内存淘汰策略
CONFIG SET maxmemory-policy allkeys-lru

# 4. 启用内存碎片整理（Redis 4.0+）
CONFIG SET activedefrag yes
CONFIG SET active-defrag-threshold-lower 10
CONFIG SET active-defrag-threshold-upper 100

# 5. 大 key 分析
redis-cli --bigkeys

# 6. 内存分析
redis-cli --memkeys
```

#### 4.3.2 命令优化

```bash
# 1. 使用管道批量操作
cat commands.txt | redis-cli --pipe

# 2. 使用 Lua 脚本减少网络往返
EVAL "return redis.call('set', KEYS[1], ARGV[1])" 1 mykey myvalue

# 3. 避免使用 KEYS 命令，使用 SCAN
SCAN 0 MATCH user:* COUNT 1000

# 4. 使用批量命令
MGET key1 key2 key3
MSET key1 val1 key2 val2
HMGET user:1001 name age city

# 5. 控制单个命令执行时间
# 避免 HGETALL 大 hash
# 避免 LRANGE 长 list
# 避免 ZRANGE 大 zset 返回全部数据
```

#### 4.3.3 配置优化

```conf
# 系统优化
# /etc/sysctl.conf
vm.overcommit_memory = 1
net.core.somaxconn = 65535

# Redis 配置优化
# 关闭透明大页
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 绑定 CPU（多实例时）
taskset -c 0 redis-server /etc/redis/redis.conf
```

### 4.4 延迟监控

```bash
# 内置延迟监控
CONFIG SET latency-monitor-threshold 100
LATENCY LATEST
LATENCY HISTORY command
LATENCY GRAPH command
LATENCY DOCTOR
LATENCY RESET

# 外部监控
redis-cli --latency
redis-cli --latency-history -i 1
redis-cli --latency-dist
```

---

## 五、数据备份与恢复

### 5.1 RDB 备份

#### 5.1.1 手动备份

```bash
# 触发 RDB 保存
redis-cli SAVE          # 同步保存，阻塞
redis-cli BGSAVE        # 异步保存，后台执行

# 查看保存状态
redis-cli INFO persistence | grep rdb

# 复制 RDB 文件
cp /var/lib/redis/dump.rdb /backup/redis/dump-$(date +%Y%m%d).rdb
```

#### 5.1.2 自动备份脚本

```bash
#!/bin/bash
# /opt/scripts/redis-backup.sh

BACKUP_DIR="/backup/redis"
DATE=$(date +%Y%m%d_%H%M%S)
REDIS_DATA="/var/lib/redis"

# 创建备份目录
mkdir -p $BACKUP_DIR

# 触发 BGSAVE
redis-cli BGSAVE

# 等待保存完成
while redis-cli INFO persistence | grep -q "rdb_bgsave_in_progress:1"; do
    sleep 1
done

# 检查保存是否成功
if redis-cli INFO persistence | grep -q "rdb_last_bgsave_status:ok"; then
    # 压缩备份
    gzip -c $REDIS_DATA/dump.rdb > $BACKUP_DIR/dump-$DATE.rdb.gz
    
    # 保留最近7天的备份
    find $BACKUP_DIR -name "dump-*.rdb.gz" -mtime +7 -delete
    
    echo "Backup completed: dump-$DATE.rdb.gz"
else
    echo "Backup failed!"
    exit 1
fi
```

#### 5.1.3 定时任务

```bash
# 添加定时任务
crontab -e

# 每天凌晨2点备份
0 2 * * * /opt/scripts/redis-backup.sh >> /var/log/redis-backup.log 2>&1
```

### 5.2 AOF 备份

```bash
# AOF 文件位置
/var/lib/redis/appendonly.aof

# AOF 重写（压缩）
redis-cli BGREWRITEAOF

# 查看 AOF 状态
redis-cli INFO persistence | grep aof

# 备份 AOF
cp /var/lib/redis/appendonly.aof /backup/redis/appendonly-$(date +%Y%m%d).aof
```

### 5.3 数据恢复

#### 5.3.1 RDB 恢复

```bash
# 1. 停止 Redis
systemctl stop redis

# 2. 备份当前数据（如果有）
mv /var/lib/redis/dump.rdb /var/lib/redis/dump.rdb.bak

# 3. 复制备份文件
cp /backup/redis/dump-20260308.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb

# 4. 启动 Redis
systemctl start redis

# 5. 验证数据
redis-cli DBSIZE
redis-cli INFO keyspace
```

#### 5.3.2 AOF 恢复

```bash
# 1. 停止 Redis
systemctl stop redis

# 2. 备份当前 AOF
mv /var/lib/redis/appendonly.aof /var/lib/redis/appendonly.aof.bak

# 3. 复制备份的 AOF
cp /backup/redis/appendonly-20260308.aof /var/lib/redis/appendonly.aof
chown redis:redis /var/lib/redis/appendonly.aof

# 4. 修复 AOF（如果需要）
redis-check-aof --fix /var/lib/redis/appendonly.aof

# 5. 启动 Redis
systemctl start redis
```

#### 5.3.3 混合持久化恢复

```bash
# Redis 4.0+ 支持 RDB-AOF 混合持久化
# 恢复方式与 AOF 相同
# 启动时会自动识别混合格式
```

### 5.4 数据迁移

```bash
# 单实例迁移
redis-cli --rdb backup.rdb
redis-cli -h new-host -p 6379 --pipe < backup.rdb

# 使用 MIGRATE
redis-cli MIGRATE new-host 6379 "" 0 5000 KEYS key1 key2

# 全量迁移（使用 redis-shake 或类似工具）
# redis-shake -conf redis-shake.conf -type sync
```

---

## 六、高可用方案实施

### 6.1 主从复制

#### 6.1.1 主节点配置

```conf
# /etc/redis/redis-master.conf
bind 0.0.0.0
port 6379
requirepass masterpassword

# 持久化配置
save 900 1
save 300 10
save 60 10000
appendonly yes
```

#### 6.1.2 从节点配置

```conf
# /etc/redis/redis-slave.conf
bind 0.0.0.0
port 6380
requirepass masterpassword

# 主从配置
replicaof 192.168.1.100 6379
masterauth masterpassword

# 从节点只读
replica-read-only yes

# 持久化配置
save 900 1
appendonly yes
```

#### 6.1.3 复制管理

```bash
# 查看复制状态
redis-cli INFO replication

# 手动切换主从
# 在从节点执行
REPLICAOF NO ONE          # 取消复制，变为主节点
REPLICAOF host port       # 切换新的主节点

# 强制同步
redis-cli SYNC
```

### 6.2 哨兵模式（Sentinel）

#### 6.2.1 哨兵配置

```conf
# /etc/redis/sentinel.conf
port 26379
daemonize yes
pidfile /var/run/redis/redis-sentinel.pid
logfile /var/log/redis/sentinel.log
dir /var/lib/redis

# 监控主节点
sentinel monitor mymaster 192.168.1.100 6379 2
sentinel auth-pass mymaster masterpassword

# 故障判定时间（毫秒）
sentinel down-after-milliseconds mymaster 5000

# 同步超时时间
sentinel failover-timeout mymaster 60000

# 并行同步的从节点数
sentinel parallel-syncs mymaster 1

# 通知脚本（可选）
sentinel notification-script mymaster /opt/scripts/notify.sh
sentinel client-reconfig-script mymaster /opt/scripts/reconfig.sh
```

#### 6.2.2 启动哨兵

```bash
# 启动哨兵
redis-sentinel /etc/redis/sentinel.conf

# 或
redis-server /etc/redis/sentinel.conf --sentinel

# 查看哨兵状态
redis-cli -p 26379 INFO sentinel
redis-cli -p 26379 SENTINEL masters
redis-cli -p 26379 SENTINEL slaves mymaster
```

### 6.3 Redis Cluster 集群

#### 6.3.1 集群节点配置

```conf
# /etc/redis/redis-7001.conf
port 7001
cluster-enabled yes
cluster-config-file nodes-7001.conf
cluster-node-timeout 5000
appendonly yes
requirepass clusterpassword
masterauth clusterpassword
```

#### 6.3.2 创建集群

```bash
# 1. 启动所有节点
redis-server /etc/redis/redis-7001.conf
redis-server /etc/redis/redis-7002.conf
redis-server /etc/redis/redis-7003.conf
redis-server /etc/redis/redis-7004.conf
redis-server /etc/redis/redis-7005.conf
redis-server /etc/redis/redis-7006.conf

# 2. 创建集群
redis-cli --cluster create \
  192.168.1.100:7001 \
  192.168.1.100:7002 \
  192.168.1.100:7003 \
  192.168.1.100:7004 \
  192.168.1.100:7005 \
  192.168.1.100:7006 \
  --cluster-replicas 1 \
  -a clusterpassword

# 3. 检查集群
redis-cli --cluster check 192.168.1.100:7001 -a clusterpassword

# 4. 查看集群信息
redis-cli -p 7001 -a clusterpassword CLUSTER INFO
redis-cli -p 7001 -a clusterpassword CLUSTER NODES
```

#### 6.3.3 集群管理

```bash
# 添加新节点
redis-cli --cluster add-node 192.168.1.100:7007 192.168.1.100:7001 -a clusterpassword

# 添加从节点
redis-cli --cluster add-node 192.168.1.100:7008 192.168.1.100:7001 --cluster-slave --cluster-master-id <node-id> -a clusterpassword

# 重新分片
redis-cli --cluster reshard 192.168.1.100:7001 -a clusterpassword

# 删除节点
redis-cli --cluster del-node 192.168.1.100:7001 <node-id> -a clusterpassword

# 故障转移
redis-cli -p 7001 CLUSTER FAILOVER      # 手动故障转移
redis-cli -p 7001 CLUSTER FAILOVER FORCE # 强制故障转移
```

### 6.4 高可用架构对比

| 方案 | 数据一致性 | 自动故障转移 | 扩展性 | 适用场景 |
|------|-----------|-------------|--------|----------|
| 主从复制 | 最终一致 | 不支持 | 读扩展 | 读多写少，数据量小 |
| 哨兵模式 | 最终一致 | 支持 | 读扩展 | 中小规模，高可用要求 |
| Cluster | 最终一致 | 支持 | 读写扩展 | 大规模，数据分片 |

---

## 七、常见故障处理

### 7.1 连接问题

#### 7.1.1 无法连接

```bash
# 1. 检查服务状态
systemctl status redis

# 2. 检查端口监听
netstat -tlnp | grep 6379
ss -tlnp | grep 6379

# 3. 检查防火墙
iptables -L -n | grep 6379
firewall-cmd --list-ports

# 4. 检查 bind 配置
grep "^bind" /etc/redis/redis.conf

# 5. 检查 protected-mode
grep "^protected-mode" /etc/redis/redis.conf
```

#### 7.1.2 认证失败

```bash
# 1. 检查密码配置
grep "^requirepass" /etc/redis/redis.conf

# 2. 检查 ACL 配置（Redis 6.0+）
redis-cli ACL LIST

# 3. 重置密码
redis-cli CONFIG SET requirepass newpassword
redis-cli CONFIG REWRITE
```

### 7.2 内存问题

#### 7.2.1 内存溢出

```bash
# 1. 查看内存使用
redis-cli INFO memory

# 2. 查看大 key
redis-cli --bigkeys

# 3. 分析内存使用
redis-cli --memkeys

# 4. 解决方案
# - 增加 maxmemory
CONFIG SET maxmemory 8gb

# - 调整淘汰策略
CONFIG SET maxmemory-policy allkeys-lru

# - 删除不必要的数据
redis-cli KEYS "temp:*" | xargs redis-cli DEL

# - 开启内存碎片整理
CONFIG SET activedefrag yes
```

#### 7.2.2 内存碎片率高

```bash
# 查看内存碎片率
redis-cli INFO memory | grep ratio

# 解决方案
# 1. 重启 Redis（最彻底）
# 2. 开启自动碎片整理
CONFIG SET activedefrag yes
CONFIG SET active-defrag-threshold-lower 10
```

### 7.3 持久化问题

#### 7.3.1 RDB 保存失败

```bash
# 1. 检查磁盘空间
df -h

# 2. 检查目录权限
ls -la /var/lib/redis/

# 3. 查看错误日志
tail -f /var/log/redis/redis-server.log

# 4. 手动测试
redis-cli SAVE
```

#### 7.3.2 AOF 文件损坏

```bash
# 1. 备份损坏的 AOF
cp /var/lib/redis/appendonly.aof /var/lib/redis/appendonly.aof.bak

# 2. 修复 AOF
redis-check-aof --fix /var/lib/redis/appendonly.aof

# 3. 重启 Redis
systemctl restart redis
```

### 7.4 性能问题

#### 7.4.1 响应慢

```bash
# 1. 查看慢查询
redis-cli SLOWLOG GET 10

# 2. 查看命令统计
redis-cli INFO commandstats

# 3. 延迟监控
redis-cli --latency-history

# 4. 检查是否有大 key
redis-cli --bigkeys

# 5. 检查 fork 耗时
redis-cli INFO persistence | grep fork
```

#### 7.4.2 主从复制延迟

```bash
# 1. 查看复制状态
redis-cli INFO replication

# 2. 查看复制积压缓冲区
redis-cli INFO replication | grep backlog

# 3. 解决方案
# - 增加复制缓冲区
CONFIG SET repl-backlog-size 256mb

# - 优化网络
# - 避免主节点高负载时全量同步
```

### 7.5 集群问题

#### 7.5.1 集群节点下线

```bash
# 1. 查看集群状态
redis-cli -p 7001 CLUSTER INFO
redis-cli -p 7001 CLUSTER NODES

# 2. 重新加入节点
redis-cli -p 7001 CLUSTER MEET 192.168.1.100 7002

# 3. 重新分配槽位
redis-cli --cluster reshard 192.168.1.100:7001
```

#### 7.5.2 脑裂问题

```bash
# 1. 查看节点状态
redis-cli -p 7001 CLUSTER NODES

# 2. 强制故障转移
redis-cli -p 7001 CLUSTER FAILOVER FORCE

# 3. 重置节点配置
redis-cli -p 7001 CLUSTER RESET SOFT
```

---

## 八、安全加固

### 8.1 网络安全

#### 8.1.1 绑定地址

```conf
# 只监听内网地址
bind 127.0.0.1 192.168.1.100

# 或只监听本地（如果需要远程，使用 SSH 隧道）
bind 127.0.0.1
```

#### 8.1.2 防火墙配置

```bash
# iptables
iptables -A INPUT -p tcp --dport 6379 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 6379 -j DROP

# firewalld
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port protocol="tcp" port="6379" accept'
firewall-cmd --reload
```

### 8.2 认证安全

#### 8.2.1 密码配置

```conf
# 设置强密码
requirepass YourStrongPassword123!

# 主从认证
masterauth YourStrongPassword123!
```

#### 8.2.2 ACL 配置（Redis 6.0+）

```bash
# 创建用户
ACL SETUSER admin on >adminpassword ~* +@all
ACL SETUSER readonly on >readonlypassword ~* +@read
ACL SETUSER appuser on >apppassword ~app:* +@all -@dangerous

# 查看用户
ACL LIST
ACL GETUSER admin

# 保存 ACL
ACL SAVE

# 加载 ACL
ACL LOAD
```

### 8.3 命令安全

```conf
# 禁用危险命令
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_9f3b2a1e"
rename-command DEBUG ""
rename-command KEYS ""
```

### 8.4 数据安全

#### 8.4.1 传输加密

```conf
# 启用 TLS（Redis 6.0+）
tls-port 6380
port 0
tls-cert-file /etc/redis/redis.crt
tls-key-file /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
```

#### 8.4.2 审计日志

```bash
# 启用命令日志（Redis 6.0+）
ACL LOG
ACL LOG RESET
```

### 8.5 系统安全

```bash
# 1. 使用专用用户运行
useradd -r -s /sbin/nologin redis

# 2. 限制文件权限
chmod 640 /etc/redis/redis.conf
chown root:redis /etc/redis/redis.conf

# 3. 禁用 THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 4. 系统限制
# /etc/security/limits.conf
redis soft nofile 65535
redis hard nofile 65535
```

### 8.6 安全 checklist

- [ ] 启用密码认证
- [ ] 禁用危险命令
- [ ] 绑定内网地址
- [ ] 配置防火墙规则
- [ ] 使用非 root 用户运行
- [ ] 启用持久化
- [ ] 定期备份数据
- [ ] 配置监控告警
- [ ] 启用 TLS 加密（可选）
- [ ] 配置审计日志（可选）

---

## 附录

### A. 常用命令速查表

| 命令 | 说明 |
|------|------|
| `INFO` | 查看服务器信息 |
| `CONFIG GET/SET` | 获取/设置配置 |
| `MONITOR` | 实时监控命令 |
| `SLOWLOG` | 查看慢查询 |
| `BGSAVE` | 异步保存 RDB |
| `BGREWRITEAOF` | 重写 AOF |
| `CLIENT LIST` | 查看客户端连接 |
| `LATENCY DOCTOR` | 延迟诊断 |

### B. 配置文件路径

| 系统 | 路径 |
|------|------|
| CentOS/RHEL | `/etc/redis.conf` |
| Ubuntu/Debian | `/etc/redis/redis.conf` |
| 源码安装 | `/usr/local/redis/etc/redis.conf` |

### C. 数据目录

| 类型 | 默认路径 |
|------|----------|
| RDB | `/var/lib/redis/dump.rdb` |
| AOF | `/var/lib/redis/appendonly.aof` |
| 日志 | `/var/log/redis/` |

## 九、Redis 完整卸载流程

### 9.1 卸载前准备

#### 9.1.1 备份重要数据

```bash
# 1. 确认数据备份
redis-cli SAVE
cp /var/lib/redis/dump.rdb /backup/redis/dump-final-$(date +%Y%m%d).rdb

# 2. 备份配置文件
cp /etc/redis/redis.conf /backup/redis/redis.conf.backup

# 3. 记录当前配置信息
redis-cli INFO > /backup/redis/redis-info-$(date +%Y%m%d).txt
```

#### 9.1.2 通知相关人员

- 通知开发团队 Redis 即将下线
- 确认无业务依赖此 Redis 实例
- 安排维护窗口时间

### 9.2 Linux 系统卸载

#### 9.2.1 停止 Redis 服务

```bash
# 1. 停止 Redis 服务
systemctl stop redis
systemctl stop redis-sentinel  # 如果有哨兵

# 2. 确认服务已停止
systemctl status redis

# 3. 检查是否还有 Redis 进程
ps aux | grep redis

# 4. 强制终止残留进程（如果有）
killall -9 redis-server
killall -9 redis-sentinel
killall -9 redis-cli

# 5. 再次确认进程已终止
ps aux | grep redis | grep -v grep
```

#### 9.2.2 移除系统服务

```bash
# 1. 禁用开机启动
systemctl disable redis
systemctl disable redis-sentinel

# 2. 删除 systemd 服务文件
rm -f /etc/systemd/system/redis.service
rm -f /etc/systemd/system/redis-sentinel.service

# 3. 重新加载 systemd
systemctl daemon-reload
systemctl reset-failed

# 4. 对于 SysVinit 系统（CentOS 6 等）
chkconfig --del redis
rm -f /etc/init.d/redis
```

#### 9.2.3 删除安装目录

```bash
# 1. 删除源码安装目录
rm -rf /usr/local/redis
rm -rf /usr/local/bin/redis-*
rm -rf /opt/redis

# 2. 删除包管理器安装的文件
# CentOS/RHEL
yum remove -y redis

# Ubuntu/Debian
apt-get remove --purge -y redis-server redis-tools
apt-get autoremove -y

# 3. 删除编译生成的文件
rm -rf /root/redis-*  # 如果是 root 用户编译
```

#### 9.2.4 清理配置文件

```bash
# 1. 删除主配置文件
rm -f /etc/redis.conf
rm -rf /etc/redis/
rm -rf /etc/redis-sentinel/

# 2. 删除其他配置文件
rm -f /usr/local/etc/redis.conf
rm -rf /usr/local/etc/redis/

# 3. 删除 ACL 文件（Redis 6.0+）
rm -f /etc/redis/users.acl
```

#### 9.2.5 清理数据目录

```bash
# 1. 删除数据文件
rm -rf /var/lib/redis/
rm -rf /var/lib/redis-sentinel/

# 2. 删除日志文件
rm -rf /var/log/redis/
rm -f /var/log/redis-server.log
rm -f /var/log/redis-sentinel.log

# 3. 删除 PID 文件
rm -f /var/run/redis/redis-server.pid
rm -f /var/run/redis/redis-sentinel.pid
rm -f /var/run/redis_*.pid

# 4. 删除其他数据目录
rm -rf /data/redis/
rm -rf /opt/redis-data/
```

#### 9.2.6 清理环境变量

```bash
# 1. 编辑系统环境变量
vi /etc/profile
vi /etc/bashrc

# 2. 删除 Redis 相关的 PATH 设置
# 删除类似以下内容：
# export PATH=$PATH:/usr/local/redis/bin
# export REDIS_HOME=/usr/local/redis

# 3. 编辑用户环境变量
vi ~/.bashrc
vi ~/.bash_profile
vi ~/.profile

# 4. 重新加载环境变量
source /etc/profile
source ~/.bashrc
```

#### 9.2.7 删除 Redis 用户

```bash
# 1. 删除 redis 用户
userdel -r redis

# 2. 删除 redis 用户组
groupdel redis 2>/dev/null

# 3. 确认用户已删除
id redis
```

#### 9.2.8 清理防火墙规则

```bash
# 1. 删除 iptables 规则
iptables -D INPUT -p tcp --dport 6379 -j ACCEPT
iptables -D INPUT -p tcp --dport 26379 -j ACCEPT

# 2. 删除 firewalld 规则
firewall-cmd --permanent --remove-port=6379/tcp
firewall-cmd --permanent --remove-port=26379/tcp
firewall-cmd --reload

# 3. 保存 iptables 规则
service iptables save
# 或
iptables-save > /etc/sysconfig/iptables
```

### 9.3 Docker 环境卸载

```bash
# 1. 停止并删除 Redis 容器
docker stop redis
docker rm redis

# 2. 删除 Redis 镜像
docker rmi redis:7.2-alpine

# 3. 删除数据卷
docker volume rm redis-data
docker volume prune

# 4. 清理主机上的数据目录
rm -rf /data/redis/

# 5. 如果使用 Docker Compose
docker-compose down -v
rm -f docker-compose.yml
```

### 9.4 集群环境卸载

#### 9.4.1 停止集群节点

```bash
# 1. 停止所有集群节点
for port in 7001 7002 7003 7004 7005 7006; do
    redis-cli -p $port SHUTDOWN
done

# 2. 确认所有节点已停止
ps aux | grep redis-server | grep cluster
```

#### 9.4.2 清理集群配置

```bash
# 1. 删除集群配置文件
rm -f /etc/redis/redis-700*.conf

# 2. 删除节点配置文件
rm -f /var/lib/redis/nodes-700*.conf

# 3. 删除集群数据目录
for port in 7001 7002 7003 7004 7005 7006; do
    rm -rf /var/lib/redis/$port
    rm -f /var/log/redis/redis-$port.log
done
```

### 9.5 哨兵环境卸载

```bash
# 1. 停止哨兵进程
redis-cli -p 26379 SHUTDOWN

# 2. 删除哨兵配置文件
rm -f /etc/redis/sentinel.conf

# 3. 删除哨兵日志和数据
rm -f /var/log/redis/sentinel.log
rm -f /var/lib/redis/sentinel.conf

# 4. 如果有多个哨兵
for port in 26379 26380 26381; do
    redis-cli -p $port SHUTDOWN
    rm -f /etc/redis/sentinel-$port.conf
done
```

### 9.6 验证卸载结果

#### 9.6.1 检查进程

```bash
# 1. 检查 Redis 进程
ps aux | grep redis | grep -v grep

# 2. 检查端口占用
netstat -tlnp | grep -E "6379|6380|7001|26379"
ss -tlnp | grep -E "6379|6380|7001|26379"
lsof -i :6379

# 3. 检查监听端口
netstat -an | grep LISTEN | grep 6379
```

#### 9.6.2 检查文件

```bash
# 1. 检查安装文件
ls -la /usr/local/bin/redis-*
ls -la /usr/local/redis/

# 2. 检查配置文件
ls -la /etc/redis.conf
ls -la /etc/redis/

# 3. 检查数据文件
ls -la /var/lib/redis/
ls -la /var/log/redis/

# 4. 全局搜索 Redis 相关文件
find / -name "*redis*" -type f 2>/dev/null
find / -name "*redis*" -type d 2>/dev/null
```

#### 9.6.3 检查服务

```bash
# 1. 检查 systemd 服务
systemctl list-unit-files | grep redis
ls -la /etc/systemd/system/redis*

# 2. 检查 SysV 服务
chkconfig --list | grep redis
ls -la /etc/init.d/redis*

# 3. 检查定时任务
crontab -l | grep redis
ls -la /etc/cron.d/*redis*
```

#### 9.6.4 检查用户和权限

```bash
# 1. 检查用户
id redis
grep redis /etc/passwd
grep redis /etc/group

# 2. 检查环境变量
echo $PATH | grep redis
grep -r "redis" /etc/profile /etc/bashrc ~/.bashrc 2>/dev/null
```

### 9.7 自动化卸载脚本

```bash
#!/bin/bash
# redis-uninstall.sh - Redis 完整卸载脚本
# 用法: ./redis-uninstall.sh [--force]

set -e

FORCE=false
BACKUP_DIR="/backup/redis-uninstall-$(date +%Y%m%d%H%M%S)"

# 解析参数
if [ "$1" == "--force" ]; then
    FORCE=true
fi

echo "========================================"
echo "Redis 完整卸载脚本"
echo "========================================"
echo ""

# 确认卸载
if [ "$FORCE" == false ]; then
    read -p "确定要卸载 Redis 吗？这将删除所有数据和配置！[y/N]: " confirm
    if [ "$confirm" != "y" ] && [ "$confirm" != "Y" ]; then
        echo "卸载已取消"
        exit 0
    fi
fi

# 创建备份目录
mkdir -p $BACKUP_DIR
echo "备份目录: $BACKUP_DIR"

# 1. 备份数据
echo "[1/9] 备份数据..."
if pgrep redis-server > /dev/null; then
    redis-cli SAVE 2>/dev/null || true
    cp /var/lib/redis/dump.rdb $BACKUP_DIR/ 2>/dev/null || true
fi
cp /etc/redis.conf $BACKUP_DIR/ 2>/dev/null || true
cp -r /etc/redis/ $BACKUP_DIR/ 2>/dev/null || true
echo "备份完成"

# 2. 停止服务
echo "[2/9] 停止 Redis 服务..."
systemctl stop redis 2>/dev/null || true
systemctl stop redis-sentinel 2>/dev/null || true
killall -9 redis-server 2>/dev/null || true
killall -9 redis-sentinel 2>/dev/null || true
sleep 2
echo "服务已停止"

# 3. 禁用开机启动
echo "[3/9] 禁用开机启动..."
systemctl disable redis 2>/dev/null || true
systemctl disable redis-sentinel 2>/dev/null || true
echo "开机启动已禁用"

# 4. 删除服务文件
echo "[4/9] 删除系统服务..."
rm -f /etc/systemd/system/redis*.service
rm -f /etc/init.d/redis*
systemctl daemon-reload
echo "服务文件已删除"

# 5. 删除安装文件
echo "[5/9] 删除安装文件..."
rm -rf /usr/local/redis
rm -f /usr/local/bin/redis-*
rm -rf /opt/redis
yum remove -y redis 2>/dev/null || true
apt-get remove --purge -y redis-server 2>/dev/null || true
echo "安装文件已删除"

# 6. 删除配置文件
echo "[6/9] 删除配置文件..."
rm -f /etc/redis.conf
rm -rf /etc/redis/
rm -rf /etc/redis-sentinel/
echo "配置文件已删除"

# 7. 删除数据文件
echo "[7/9] 删除数据文件..."
rm -rf /var/lib/redis/
rm -rf /var/log/redis/
rm -f /var/run/redis*.pid
echo "数据文件已删除"

# 8. 删除用户
echo "[8/9] 删除 Redis 用户..."
userdel -r redis 2>/dev/null || true
groupdel redis 2>/dev/null || true
echo "用户已删除"

# 9. 验证卸载
echo "[9/9] 验证卸载结果..."
echo ""
echo "检查残留进程:"
ps aux | grep redis | grep -v grep || echo "  无残留进程 ✓"

echo ""
echo "检查残留文件:"
find /usr/local/bin -name "redis-*" 2>/dev/null || echo "  无残留可执行文件 ✓"
ls /etc/redis.conf 2>/dev/null || echo "  无残留配置文件 ✓"

echo ""
echo "========================================"
echo "Redis 卸载完成!"
echo "备份位置: $BACKUP_DIR"
echo "========================================"
```

### 9.8 卸载后清理清单

| 检查项 | 命令 | 预期结果 |
|--------|------|----------|
| 进程检查 | `ps aux \| grep redis` | 无 Redis 进程 |
| 端口检查 | `netstat -tlnp \| grep 6379` | 无 6379 端口监听 |
| 安装目录 | `ls /usr/local/redis` | 目录不存在 |
| 配置文件 | `ls /etc/redis.conf` | 文件不存在 |
| 数据目录 | `ls /var/lib/redis` | 目录不存在 |
| 日志目录 | `ls /var/log/redis` | 目录不存在 |
| 系统服务 | `systemctl list-unit-files \| grep redis` | 无 Redis 服务 |
| 环境变量 | `echo $PATH \| grep redis` | 无 Redis 路径 |
| 系统用户 | `id redis` | 用户不存在 |

### 9.9 注意事项

⚠️ **重要提醒：**

1. **数据备份**：卸载前务必备份重要数据，一旦删除无法恢复
2. **业务影响**：确保没有业务依赖此 Redis 实例
3. **权限问题**：部分操作需要 root 权限
4. **集群环境**：集群卸载需要逐个节点清理
5. **Docker 环境**：注意区分容器内和主机上的数据

---

### D. 参考资源

- [Redis 官方文档](https://redis.io/documentation)
- [Redis 命令参考](https://redis.io/commands)
- [Redis 配置文件示例](https://raw.githubusercontent.com/redis/redis/unstable/redis.conf)

---

**文档版本**：v1.1  
**最后更新**：2026-03-08  
**维护人员**：运维团队
