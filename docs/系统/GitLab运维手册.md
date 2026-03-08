# GitLab 运维手册

> 版本：GitLab 16.0+  
> 适用对象：运维工程师、DevOps工程师  
> 最后更新：2026-03-08

---

## 目录

1. [GitLab 环境搭建](#一-gitlab-环境搭建)
2. [配置管理规范](#二-配置管理规范)
3. [日常运维操作](#三-日常运维操作)
4. [备份与恢复策略](#四-备份与恢复策略)
5. [性能监控方法](#五-性能监控方法)
6. [常见故障排查](#六-常见故障排查)
7. [安全加固措施](#七-安全加固措施)
8. [CI/CD 配置](#八-cicd-配置)
9. [用户与权限管理](#九-用户与权限管理)
10. [GitLab 与其他工具集成](#十-gitlab-与其他工具集成)

---

## 一、GitLab 环境搭建

### 1.1 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | Ubuntu 20.04/22.04 LTS, CentOS 7/8, RHEL 7/8 |
| CPU | 建议 4 核以上 |
| 内存 | 建议 8GB 以上 |
| 磁盘 | 建议 100GB 以上（SSD 推荐） |
| 数据库 | PostgreSQL 14+ |
| Redis | 7.0+ |
| Docker | 20.10.0+（如果使用 Docker 部署） |

### 1.2 安装方式

#### 1.2.1 官方包安装（推荐）

**Ubuntu/Debian：**

```bash
# 安装依赖
apt-get update
apt-get install -y curl openssh-server ca-certificates tzdata perl

# 安装邮件服务（可选）
apt-get install -y postfix

# 添加 GitLab 仓库
curl -fsSL https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | bash

# 安装 GitLab EE（企业版）
EXTERNAL_URL="https://gitlab.example.com" apt-get install gitlab-ee

# 或安装 GitLab CE（社区版）
EXTERNAL_URL="https://gitlab.example.com" apt-get install gitlab-ce
```

**CentOS/RHEL：**

```bash
# 安装依赖
yum install -y curl policycoreutils-python openssh-server perl

# 启用 SSH 服务
systemctl enable sshd
systemctl start sshd

# 安装邮件服务（可选）
yum install -y postfix
systemctl enable postfix
systemctl start postfix

# 添加 GitLab 仓库
curl -fsSL https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | bash

# 安装 GitLab EE
external_url="https://gitlab.example.com" yum install -y gitlab-ee

# 或安装 GitLab CE
external_url="https://gitlab.example.com" yum install -y gitlab-ce
```

#### 1.2.2 Docker 部署

```bash
# 拉取镜像
docker pull gitlab/gitlab-ee:latest

# 运行容器
docker run -d \
  --hostname gitlab.example.com \
  --name gitlab \
  --restart always \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ee:latest

# 查看容器状态
docker ps

# 查看日志
docker logs -f gitlab
```

#### 1.2.3 源码安装（高级）

```bash
# 安装依赖
# PostgreSQL, Redis, Ruby, Git 等

# 克隆 GitLab 源码
git clone https://gitlab.com/gitlab-org/gitlab-foss.git
cd gitlab-foss

# 安装依赖
bundle install

# 配置数据库
# 配置 Redis
# 运行安装脚本
```

### 1.3 初始化配置

```bash
# 重新配置 GitLab
gitlab-ctl reconfigure

# 启动服务
gitlab-ctl start

# 查看服务状态
gitlab-ctl status

# 查看日志
gitlab-ctl tail

# 查看版本
gitlab-rake gitlab:env:info
```

### 1.4 首次访问

1. 打开浏览器访问：`https://gitlab.example.com`
2. 首次登录会提示设置 root 密码
3. 使用 root 账户登录
4. 完成初始设置向导

### 1.5 升级 GitLab

```bash
# 备份数据
gitlab-rake gitlab:backup:create

# 升级 GitLab
# Ubuntu/Debian
apt-get update
apt-get install gitlab-ee

# CentOS/RHEL
yum update gitlab-ee

# 重新配置
gitlab-ctl reconfigure

# 重启服务
gitlab-ctl restart

# 检查状态
gitlab-rake gitlab:check SANITIZE=true
```

---

## 二、配置管理规范

### 2.1 配置文件结构

```
/etc/gitlab/
├── gitlab.rb              # 主配置文件
├── gitlab-secrets.json    # 密钥文件
├── ssh_host_*             # SSH 主机密钥
└── gitlab.rb.template     # 配置模板
```

### 2.2 主配置文件详解

```ruby
# /etc/gitlab/gitlab.rb

# 外部 URL
external_url 'https://gitlab.example.com'

# Nginx 配置
nginx['enable'] = true
nginx['client_max_body_size'] = '250m'
nginx['redirect_http_to_https'] = true
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.example.com.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.example.com.key"

# PostgreSQL 配置
postgresql['enable'] = true
postgresql['shared_buffers'] = '256MB'
postgresql['work_mem'] = '16MB'
postgresql['maintenance_work_mem'] = '128MB'
postgresql['max_connections'] = 200

# Redis 配置
redis['enable'] = true
redis['maxmemory'] = '256mb'
redis['maxmemory_policy'] = 'volatile-lru'

# GitLab 应用配置
gitlab_rails['gitlab_email_from'] = 'gitlab@example.com'
gitlab_rails['gitlab_email_reply_to'] = 'noreply@example.com'
gitlab_rails['time_zone'] = 'Asia/Shanghai'
gitlab_rails['backup_keep_time'] = 604800  # 7 天

# 存储配置
git_data_dirs({
  "default" => { "path" => "/var/opt/gitlab/git-data" }
})

# 备份配置
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"

# 日志配置
logging['logrotate_frequency'] = "daily"
logging['logrotate_maxsize'] = "200M"
logging['logrotate_rotate'] = 30

# 监控配置
prometheus['enable'] = true

# 高级配置
unicorn['worker_processes'] = 4
sidekiq['concurrency'] = 16
```

### 2.3 配置修改流程

1. 备份当前配置：`cp /etc/gitlab/gitlab.rb /etc/gitlab/gitlab.rb.bak`
2. 编辑配置文件：`vim /etc/gitlab/gitlab.rb`
3. 重新配置：`gitlab-ctl reconfigure`
4. 重启服务：`gitlab-ctl restart`
5. 验证配置：`gitlab-rake gitlab:check`

### 2.4 SSL 证书配置

```bash
# 创建 SSL 目录
mkdir -p /etc/gitlab/ssl
chmod 700 /etc/gitlab/ssl

# 复制证书文件
cp fullchain.pem /etc/gitlab/ssl/gitlab.example.com.crt
cp privkey.pem /etc/gitlab/ssl/gitlab.example.com.key
chmod 600 /etc/gitlab/ssl/*.key

# 重新配置
gitlab-ctl reconfigure
```

### 2.5 环境变量配置

```bash
# /etc/gitlab/gitlab.rb
gitlab_rails['env'] = {
  'GITLAB_OMNIBUS_CONFIG' => 'external_url \"https://gitlab.example.com\"',
  'RAILS_ENV' => 'production',
  'NODE_ENV' => 'production'
}
```

---

## 三、日常运维操作

### 3.1 服务管理

```bash
# 启动所有服务
gitlab-ctl start

# 停止所有服务
gitlab-ctl stop

# 重启所有服务
gitlab-ctl restart

# 重载配置
gitlab-ctl reload

# 查看服务状态
gitlab-ctl status

# 查看单个服务状态
gitlab-ctl status nginx
gitlab-ctl status postgresql
gitlab-ctl status redis

# 启动单个服务
gitlab-ctl start sidekiq
gitlab-ctl start puma

# 停止单个服务
gitlab-ctl stop nginx
```

### 3.2 日志管理

```bash
# 查看所有日志
gitlab-ctl tail

# 查看特定服务日志
gitlab-ctl tail nginx
gitlab-ctl tail postgresql
gitlab-ctl tail gitlab-rails
gitlab-ctl tail sidekiq

# 查看错误日志
gitlab-ctl tail gitlab-rails/production.log | grep ERROR

# 实时监控日志
gitlab-ctl tail -f gitlab-rails/production.log

# 查看日志大小
du -sh /var/log/gitlab/*

# 清理日志
gitlab-ctl rotate-logs

# 配置日志轮转
# /etc/gitlab/gitlab.rb
logging['logrotate_frequency'] = "daily"
logging['logrotate_maxsize'] = "200M"
logging['logrotate_rotate'] = 30
```

### 3.3 用户管理

```bash
# 创建用户
gitlab-rake gitlab:user:add[username,email@example.com,password,true]

# 列出所有用户
gitlab-rake gitlab:user:list

# 禁用用户
gitlab-rake gitlab:user:disable[username]

# 启用用户
gitlab-rake gitlab:user:enable[username]

# 重置用户密码
gitlab-rake gitlab:user:password[username,new_password]

# 删除用户
gitlab-rake gitlab:user:remove[username]
```

### 3.4 项目管理

```bash
# 列出所有项目
gitlab-rake gitlab:projects:list

# 清理未使用的项目
gitlab-rake gitlab:cleanup:projects

# 导入项目
gitlab-rake gitlab:import:repos[source_directory]

# 导出项目
gitlab-rake gitlab:backup:create[SKIP=db,uploads]
```

### 3.5 仓库管理

```bash
# 清理仓库数据
gitlab-rake gitlab:cleanup:repos

# 重新计算仓库大小
gitlab-rake gitlab:git:fsck

# 优化仓库
gitlab-rake gitlab:git:gc

# 检查仓库完整性
gitlab-rake gitlab:git:check
```

### 3.6 系统维护

```bash
# 运行健康检查
gitlab-rake gitlab:check SANITIZE=true

# 检查数据库
gitlab-rake gitlab:db:check

# 清理 Redis 缓存
gitlab-rake cache:clear

# 清理 Sidekiq 队列
gitlab-rake sidekiq:clear

# 重新索引搜索
gitlab-rake gitlab:elastic:index

# 检查系统状态
gitlab-rake gitlab:status
```

---

## 四、备份与恢复策略

### 4.1 备份配置

```ruby
# /etc/gitlab/gitlab.rb

# 备份目录
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"

# 备份保留时间（秒）
gitlab_rails['backup_keep_time'] = 604800  # 7 天

# 备份权限
gitlab_rails['backup_archive_permissions'] = 0644

# 备份策略
gitlab_rails['backup_pg_schema'] = true
```

### 4.2 手动备份

```bash
# 创建完整备份
gitlab-rake gitlab:backup:create

# 备份指定内容
gitlab-rake gitlab:backup:create[SKIP=db]
# SKIP=db,uploads,artifacts,lfs,registry,pages

# 备份并压缩
gitlab-rake gitlab:backup:create[COMPRESSION_LEVEL=6]

# 查看备份文件
ls -la /var/opt/gitlab/backups/
```

### 4.3 自动备份

```bash
# 编辑 crontab
crontab -e

# 添加定时任务（每天凌晨 2 点备份）
0 2 * * * /opt/gitlab/bin/gitlab-rake gitlab:backup:create CRON=1

# 查看定时任务
crontab -l
```

### 4.4 恢复备份

```bash
# 停止服务
gitlab-ctl stop puma
gitlab-ctl stop sidekiq

# 确保备份文件在正确位置
cp 1234567890_2026_03_08_16.0.0_gitlab_backup.tar /var/opt/gitlab/backups/
chmod 600 /var/opt/gitlab/backups/1234567890_2026_03_08_16.0.0_gitlab_backup.tar

# 恢复备份
gitlab-rake gitlab:backup:restore BACKUP=1234567890_2026_03_08_16.0.0

# 启动服务
gitlab-ctl start

# 验证恢复
gitlab-rake gitlab:check
```

### 4.5 备份验证

```bash
# 检查备份文件完整性
tar -tvf /var/opt/gitlab/backups/1234567890_2026_03_08_16.0.0_gitlab_backup.tar

# 测试恢复流程（在测试环境）
# 1. 部署测试环境
# 2. 恢复备份
# 3. 验证数据完整性

# 备份异地存储
# 使用 rsync 或 S3 等存储服务
rsync -avz /var/opt/gitlab/backups/ user@backup-server:/backup/gitlab/
```

### 4.6 灾难恢复计划

1. **定期测试**：每月至少测试一次恢复流程
2. **备份验证**：每次备份后验证备份文件完整性
3. **异地存储**：将备份存储到不同地理位置
4. **恢复演练**：每季度进行一次完整的灾难恢复演练
5. **文档更新**：及时更新恢复文档和流程

---

## 五、性能监控方法

### 5.1 内置监控

#### 5.1.1 Prometheus 监控

```ruby
# /etc/gitlab/gitlab.rb
prometheus['enable'] = true
prometheus['listen_address'] = 'localhost:9090'

# 启用节点监控
node_exporter['enable'] = true

# 启用 PostgreSQL 监控
postgres_exporter['enable'] = true

# 启用 Redis 监控
redis_exporter['enable'] = true
```

#### 5.1.2 GitLab 状态页面

- 访问：`https://gitlab.example.com/admin/health_check`
- 查看系统健康状态
- 检查各组件运行情况

#### 5.1.3 监控指标

| 指标 | 说明 | 建议值 |
|------|------|--------|
| CPU 使用率 | 系统 CPU 使用率 | < 70% |
| 内存使用率 | 系统内存使用率 | < 80% |
| 磁盘使用率 | 存储磁盘使用率 | < 85% |
| PostgreSQL 连接数 | 数据库连接数 | < 80% 最大连接数 |
| Redis 内存使用率 | Redis 内存使用 | < 70% 最大内存 |
| 响应时间 | API 响应时间 | < 500ms |
| 队列长度 | Sidekiq 队列长度 | < 100 |

### 5.2 外部监控

#### 5.2.1 集成 Grafana

```bash
# 安装 Grafana
docker run -d --name grafana -p 3000:3000 grafana/grafana

# 配置 Prometheus 数据源
# URL: http://gitlab-server:9090

# 导入 GitLab 监控面板
# ID: 12633 (GitLab Overview)
```

#### 5.2.2 集成 Zabbix

```bash
# 安装 Zabbix 客户端
apt-get install zabbix-agent

# 配置 Zabbix 客户端
# /etc/zabbix/zabbix_agentd.conf
Server=zabbix-server-ip

# 启动服务
systemctl restart zabbix-agent

# 添加 GitLab 监控模板
# 监控项：CPU、内存、磁盘、服务状态等
```

#### 5.2.3 日志监控

```bash
# 集成 ELK Stack
# 1. 安装 Elasticsearch、Logstash、Kibana
# 2. 配置 Logstash 收集 GitLab 日志
# 3. 创建 Kibana 仪表盘

# 或使用 Graylog
# 1. 安装 Graylog
# 2. 配置 GitLab 日志输出
# 3. 创建告警规则
```

### 5.3 性能分析

```bash
# 查看性能统计
gitlab-rake gitlab:usage_data:dump

# 分析慢查询
gitlab-rake gitlab:db:slow_queries

# 查看数据库索引
gitlab-rake gitlab:db:indexes:check

# 分析 Redis 使用
gitlab-rake gitlab:redis:info

# 分析 Sidekiq 队列
gitlab-rake sidekiq:queue:info
```

### 5.4 告警配置

```ruby
# /etc/gitlab/gitlab.rb

# 电子邮件告警
gitlab_rails['health_check_email_notification'] = true
gitlab_rails['health_check_notification_email'] = 'admin@example.com'

# Prometheus 告警
prometheus['alertmanager_config'] = {
  'global' => {
    'resolve_timeout' => '5m'
  },
  'route' => {
    'group_by' => ['alertname'],
    'receiver' => 'email'
  },
  'receivers' => [{
    'name' => 'email',
    'email_configs' => [{
      'to' => 'admin@example.com'
    }]
  }]
}
```

---

## 六、常见故障排查

### 6.1 服务启动失败

```bash
# 查看服务状态
gitlab-ctl status

# 查看错误日志
gitlab-ctl tail gitlab-rails/production.log
gitlab-ctl tail nginx/error.log

# 检查配置
gitlab-ctl reconfigure

# 检查端口占用
netstat -tlnp | grep 80
tcping -p 80 gitlab.example.com

# 检查权限
ls -la /var/opt/gitlab/
chown -R git:git /var/opt/gitlab/

# 检查磁盘空间
df -h
```

### 6.2 502 错误

```bash
# 检查 Puma 服务
gitlab-ctl status puma

# 检查 Puma 日志
gitlab-ctl tail puma/current

# 检查 Nginx 配置
gitlab-ctl tail nginx/error.log

# 检查内存使用
free -m
top

# 重启服务
gitlab-ctl restart puma
gitlab-ctl restart nginx
```

### 6.3 数据库连接失败

```bash
# 检查 PostgreSQL 服务
gitlab-ctl status postgresql

# 检查数据库日志
gitlab-ctl tail postgresql/current

# 连接数据库
sudo -u gitlab-psql /opt/gitlab/embedded/bin/psql -h /var/opt/gitlab/postgresql -d gitlabhq_production

# 检查数据库连接数
gitlab-rake gitlab:db:check

# 增加连接数
# /etc/gitlab/gitlab.rb
postgresql['max_connections'] = 200
```

### 6.4 Redis 连接失败

```bash
# 检查 Redis 服务
gitlab-ctl status redis

# 检查 Redis 日志
gitlab-ctl tail redis/current

# 连接 Redis
/opt/gitlab/embedded/bin/redis-cli -s /var/opt/gitlab/redis/redis.socket

# 检查 Redis 内存
gitlab-rake gitlab:redis:info

# 增加 Redis 内存
# /etc/gitlab/gitlab.rb
redis['maxmemory'] = '512mb'
```

### 6.5 CI/CD 流水线失败

```bash
# 查看 CI 日志
# 在 GitLab Web 界面查看具体作业日志

# 检查 Runner 状态
gitlab-runner status

# 检查 Runner 日志
journalctl -u gitlab-runner

# 检查 CI 配置
# .gitlab-ci.yml 文件验证

# 检查 Docker 服务（如果使用 Docker Runner）
systemctl status docker
```

### 6.6 备份失败

```bash
# 查看备份日志
gitlab-ctl tail gitlab-rails/production.log | grep backup

# 检查磁盘空间
df -h

# 检查权限
ls -la /var/opt/gitlab/backups/
chmod 700 /var/opt/gitlab/backups/

# 检查数据库连接
gitlab-rake gitlab:db:check

# 测试备份命令
gitlab-rake gitlab:backup:create[SKIP=uploads,artifacts]
```

### 6.7 性能问题

```bash
# 查看系统负载
top
iostat -x 1

# 查看数据库性能
explain analyze SELECT * FROM projects;

# 查看 Redis 性能
redis-cli --stat

# 查看 GitLab 进程
gitlab-ctl status
ps aux | grep gitlab

# 优化配置
# 增加内存、CPU
# 调整 PostgreSQL 配置
# 调整 Redis 配置
```

---

## 七、安全加固措施

### 7.1 系统安全

```bash
# 更新系统
apt-get update && apt-get upgrade -y

# 配置防火墙
ufw enable
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp
ufw allow 9090/tcp  # Prometheus

# 禁用 root 远程登录
# /etc/ssh/sshd_config
PermitRootLogin no

# 启用 Fail2ban
apt-get install fail2ban
# 配置 Fail2ban 规则

# 启用 SELinux（CentOS/RHEL）
setenforce 1
# 配置 SELinux 策略
```

### 7.2 GitLab 安全配置

```ruby
# /etc/gitlab/gitlab.rb

# 强制 HTTPS
external_url 'https://gitlab.example.com'
nginx['redirect_http_to_https'] = true

# 禁用未使用的功能
gitlab_rails['gitlab_default_projects_features_issues'] = true
gitlab_rails['gitlab_default_projects_features_merge_requests'] = true
gitlab_rails['gitlab_default_projects_features_wiki'] = true
gitlab_rails['gitlab_default_projects_features_snippets'] = false
gitlab_rails['gitlab_default_projects_features_builds'] = true
gitlab_rails['gitlab_default_projects_features_container_registry'] = false
gitlab_rails['gitlab_default_projects_features_pages'] = false

# 会话配置
gitlab_rails['session_expire_delay'] = 3600  # 1 小时
gitlab_rails['session_ttl'] = 86400          # 24 小时

# 密码策略
gitlab_rails['password_authentication_enabled_for_web'] = true
gitlab_rails['minimum_password_length'] = 12
gitlab_rails['password_complexity'] = 'strong'

# 2FA 配置
gitlab_rails['two_factor_grace_period'] = 48  # 48 小时
gitlab_rails['two_factor_remember_period'] = 30  # 30 天
```

### 7.3 用户权限管理

```bash
# 启用管理员审批
gitlab-rails['admin_mode'] = true

# 限制用户注册
gitlab_rails['gitlab_signup_enabled'] = false

# 限制项目可见性
gitlab_rails['gitlab_default_projects_visibility'] = 'private'

# 配置 LDAP 认证
# /etc/gitlab/gitlab.rb
gitlab_rails['ldap_enabled'] = true
gitlab_rails['ldap_servers'] = {
  'main' => {
    'label' => 'LDAP',
    'host' => 'ldap.example.com',
    'port' => 389,
    'uid' => 'sAMAccountName',
    'bind_dn' => 'cn=gitlab,ou=service,dc=example,dc=com',
    'password' => 'password',
    'base' => 'dc=example,dc=com'
  }
}

# 配置 SSO 认证
# OAuth、SAML 等
```

### 7.4 网络安全

```ruby
# /etc/gitlab/gitlab.rb

# 限制 SSH 访问
# 配置 SSH 密钥策略

# 限制 API 访问
gitlab_rails['rate_limit_requests_per_period'] = 10

# 配置 CSP
gitlab_rails['content_security_policy'] = {
  'enabled' => true,
  'directives' => {
    'default_src' => "'self'",
    'script_src' => "'self' 'unsafe-inline' 'unsafe-eval'",
    'style_src' => "'self' 'unsafe-inline'",
    'img_src' => "'self' data: https:",
    'connect_src' => "'self' https:"
  }
}

# 配置 HSTS
nginx['hsts_max_age'] = 31536000
nginx['hsts_include_subdomains'] = true
```

### 7.5 审计与监控

```bash
# 启用审计日志
gitlab-ctl set-gitlab-config gitlab-rails audit_log_enabled true

# 查看审计日志
tail -f /var/log/gitlab/gitlab-rails/audit_json.log

# 配置入侵检测系统
# 安装 OSSEC 或 Wazuh

# 定期安全扫描
# 使用 Nessus 或 OpenVAS 进行漏洞扫描

# 安全更新
# 定期更新 GitLab 版本
```

---

## 八、CI/CD 配置

### 8.1 GitLab Runner 安装

```bash
# 下载 Runner
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | bash

# 安装 Runner
apt-get install gitlab-runner

# 或使用 Docker
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest

# 注册 Runner
gitlab-runner register
# 输入 GitLab URL 和注册令牌
# 选择执行器（shell、docker、kubernetes 等）
```

### 8.2 CI/CD 配置文件

```yaml
# .gitlab-ci.yml

stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2

cache:
  paths:
    - node_modules/
    - vendor/

build:
  stage: build
  script:
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/
  tags:
    - docker

test:
  stage: test
  script:
    - npm test
    - npm run lint
  tags:
    - docker

deploy:
  stage: deploy
  script:
    - rsync -avz dist/ user@server:/var/www/html/
  environment:
    name: production
  only:
    - main
  tags:
    - docker
```

### 8.3 CI/CD 变量管理

1. **项目级变量**：在项目设置 → CI/CD → 变量中配置
2. **组级变量**：在组设置 → CI/CD → 变量中配置
3. **环境变量**：在 .gitlab-ci.yml 中定义
4. **文件变量**：用于存储证书、密钥等敏感文件

### 8.4 流水线管理

```bash
# 查看流水线状态
gitlab-runner list

# 查看 Runner 状态
gitlab-runner status

# 重启 Runner
systemctl restart gitlab-runner

# 清理 Runner 缓存
gitlab-runner cache-cleanup

# 查看流水线日志
# 在 GitLab Web 界面查看
```

### 8.5 部署策略

- **蓝绿部署**：同时运行两个环境，切换流量
- **金丝雀部署**：逐步将流量引导到新版本
- **滚动部署**：逐个更新实例
- **环境管理**：开发、测试、预生产、生产环境

---

## 九、用户与权限管理

### 9.1 用户管理

| 角色 | 权限 | 适用场景 |
|------|------|----------|
| Guest | 只读权限 | 外部协作者 |
| Reporter | 查看、创建问题 | 测试人员 |
| Developer | 开发、提交代码 | 开发人员 |
| Maintainer | 管理项目设置 | 项目负责人 |
| Owner | 完全控制 | 项目所有者 |

### 9.2 组管理

```bash
# 创建组
gitlab-rake gitlab:group:add[name,path,owner_email]

# 添加用户到组
gitlab-rake gitlab:group:add_member[group_path,user_email,access_level]
# access_level: 10(Guest), 20(Reporter), 30(Developer), 40(Maintainer), 50(Owner)

# 查看组信息
gitlab-rake gitlab:group:list
```

### 9.3 权限配置

```bash
# 项目权限设置
# 在项目 → 设置 → 成员中配置

# 分支保护
# 在项目 → 设置 → 仓库 → 分支保护中配置

# 标签保护
# 在项目 → 设置 → 仓库 → 标签保护中配置

# 环境权限
# 在项目 → CI/CD → 环境中配置
```

### 9.4 访问控制

```ruby
# /etc/gitlab/gitlab.rb

# 限制 IP 访问
gitlab_rails['rack_attack_git_basic_auth'] = {
  'enabled' => true,
  'ip_whitelist' => ['192.168.1.0/24'],
  'maxretry' => 10,
  'findtime' => 60,
  'bantime' => 3600
}

# 限制 API 访问
gitlab_rails['rate_limit_requests_per_period'] = 10

# 禁用公共访问
gitlab_rails['gitlab_default_projects_visibility'] = 'private'
```

---

## 十、GitLab 与其他工具集成

### 10.1 与 JIRA 集成

```ruby
# /etc/gitlab/gitlab.rb
gitlab_rails['jira_issue_tracker'] = {
  'url' => 'https://jira.example.com',
  'username' => 'gitlab',
  'password' => 'password'
}
```

### 10.2 与 LDAP/AD 集成

```ruby
# /etc/gitlab/gitlab.rb
gitlab_rails['ldap_enabled'] = true
gitlab_rails['ldap_servers'] = {
  'main' => {
    'label' => 'LDAP',
    'host' => 'ldap.example.com',
    'port' => 389,
    'uid' => 'sAMAccountName',
    'bind_dn' => 'cn=gitlab,ou=service,dc=example,dc=com',
    'password' => 'password',
    'base' => 'dc=example,dc=com',
    'verify_certificates' => false
  }
}
```

### 10.3 与 Jenkins 集成

1. **安装 GitLab 插件**：在 Jenkins 中安装 GitLab 插件
2. **配置 GitLab 集成**：在 Jenkins 项目中配置 GitLab 连接
3. **设置 Webhook**：在 GitLab 项目中设置 Webhook 指向 Jenkins
4. **触发构建**：通过 GitLab 事件触发 Jenkins 构建

### 10.4 与 Docker Registry 集成

```ruby
# /etc/gitlab/gitlab.rb
registry['enable'] = true
registry['port'] = 5000
registry['path'] = "/var/opt/gitlab/registry"
registry['storage'] = {
  's3' => {
    'bucket' => 'gitlab-registry',
    'region' => 'us-east-1',
    'accesskey' => 'accesskey',
    'secretkey' => 'secretkey'
  }
}
```

### 10.5 与监控工具集成

```bash
# 集成 Prometheus
# 配置 Prometheus 数据源指向 GitLab Prometheus 端点

# 集成 Grafana
# 导入 GitLab 监控面板

# 集成 Alertmanager
# 配置告警规则和通知渠道
```

---

**修订记录**

| 版本 | 日期 | 作者 | 修改内容 |
|------|------|------|----------|
| v1.0 | 2026-03-08 | 运维团队 | 初始版本 |
