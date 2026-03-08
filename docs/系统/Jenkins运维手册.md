# Jenkins 运维手册

> 版本：Jenkins 2.400+  
> 适用对象：运维工程师、DevOps工程师  
> 最后更新：2026-03-08

---

## 目录

1. [Jenkins 环境要求](#一-jenkins-环境要求)
2. [Jenkins 安装部署](#二-jenkins-安装部署)
3. [Jenkins 配置管理](#三-jenkins-配置管理)
4. [资源分配与优化](#四-资源分配与优化)
5. [备份与恢复策略](#五-备份与恢复策略)
6. [监控与告警](#六-监控与告警)
7. [安全加固措施](#七-安全加固措施)
8. [故障排查流程](#八-故障排查流程)
9. [日常维护任务](#九-日常维护任务)
10. [常见问题与解决方案](#十-常见问题与解决方案)

---

## 一、Jenkins 环境要求

### 1.1 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | Linux (Ubuntu 20.04/22.04, CentOS 7/8, RHEL 7/8) |
| CPU | 建议 4 核以上 |
| 内存 | 建议 8GB 以上 |
| 磁盘 | 建议 50GB 以上（SSD 推荐） |
| Java | JDK 11 或 OpenJDK 11 |
| 网络 | 稳定的网络连接 |
| 浏览器 | Chrome、Firefox、Safari、Edge |

### 1.2 依赖要求

| 依赖 | 版本 | 用途 |
|------|------|------|
| Java | 11+ | 运行环境 |
| Git | 2.0+ | 代码版本控制 |
| Maven | 3.6+ | Java 项目构建 |
| Gradle | 7.0+ | 构建工具 |
| Docker | 20.10+ | 容器化构建 |
| Node.js | 14.0+ | 前端项目构建 |

### 1.3 网络要求

| 端口 | 用途 |
|------|------|
| 8080 | Jenkins Web 界面 |
| 50000 | Jenkins Agent 通信 |
| 22 | SSH 访问 |
| 80/443 | HTTP/HTTPS 访问 |

---

## 二、Jenkins 安装部署

### 2.1 安装 Java

**Ubuntu/Debian：**

```bash
# 安装 OpenJDK 11
apt-get update
apt-get install -y openjdk-11-jdk

# 验证 Java 版本
java -version
```

**CentOS/RHEL：**

```bash
# 安装 OpenJDK 11
yum install -y java-11-openjdk-devel

# 验证 Java 版本
java -version
```

### 2.2 安装 Jenkins

**使用官方仓库安装：**

```bash
# 添加 Jenkins 仓库
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | apt-key add -
echo "deb https://pkg.jenkins.io/debian-stable binary/" > /etc/apt/sources.list.d/jenkins.list

# 安装 Jenkins
apt-get update
apt-get install -y jenkins

# 启动服务
systemctl enable jenkins
systemctl start jenkins
systemctl status jenkins
```

**使用 Docker 部署：**

```bash
# 拉取镜像
docker pull jenkins/jenkins:lts

# 运行容器
docker run -d \
  --name jenkins \
  --restart always \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /srv/jenkins/data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# 查看日志
docker logs -f jenkins
```

**使用 WAR 包部署：**

```bash
# 下载 WAR 包
wget https://get.jenkins.io/war-stable/latest/jenkins.war

# 运行 Jenkins
java -jar jenkins.war --httpPort=8080
```

### 2.3 初始化配置

1. **访问 Jenkins**：打开浏览器访问 `http://服务器IP:8080`
2. **解锁 Jenkins**：输入初始管理员密码
   - 密码位置：`/var/lib/jenkins/secrets/initialAdminPassword`
   - Docker 容器：`docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`
3. **安装插件**：选择推荐插件或自定义插件
4. **创建管理员用户**：设置用户名、密码、邮箱
5. **配置实例**：设置 Jenkins URL
6. **完成初始化**：点击 "Save and Finish"

### 2.4 安装验证

```bash
# 检查服务状态
systemctl status jenkins

# 检查端口
etstat -tlnp | grep 8080

# 访问验证
curl -I http://localhost:8080
```

---

## 三、Jenkins 配置管理

### 3.1 系统配置

**访问路径：** 管理 Jenkins → 系统配置

**主要配置项：**

| 配置项 | 说明 | 推荐值 |
|--------|------|--------|
| Jenkins URL | Jenkins 访问地址 | `https://jenkins.example.com` |
| 系统管理员邮件地址 | 管理员邮箱 | `admin@example.com` |
| 执行者数量 | 构建并发数 | CPU 核心数 |
| 环境变量 | 全局环境变量 | 按需配置 |
| 邮件通知 | 邮件服务器配置 | SMTP 服务器设置 |

### 3.2 全局工具配置

**访问路径：** 管理 Jenkins → 全局工具配置

**配置项：**

- **JDK**：配置 JDK 安装路径
- **Git**：配置 Git 路径
- **Maven**：配置 Maven 安装
- **Gradle**：配置 Gradle 安装
- **Node.js**：配置 Node.js 安装
- **Docker**：配置 Docker 路径

### 3.3 安全配置

**访问路径：** 管理 Jenkins → 全局安全配置

**认证设置：**

- **Jenkins 专有用户数据库**：使用 Jenkins 内置用户系统
- **LDAP**：使用 LDAP 认证
- **OAuth**：使用第三方认证（如 GitHub、Google）

**授权策略：**

- **安全矩阵**：基于用户/组的精细权限控制
- **项目矩阵授权策略**：基于项目的权限控制
- **角色策略**：基于角色的权限控制

### 3.4 插件管理

**访问路径：** 管理 Jenkins → 插件管理

**安装插件：**

1. **可用插件**：浏览并安装插件
2. **已安装**：查看已安装的插件
3. **更新中心**：更新插件
4. **高级**：上传插件

**推荐插件：**

- Git
- Maven Integration
- Pipeline
- Docker Pipeline
- Blue Ocean
- Credentials Binding
- Email Extension
- GitLab
- GitHub
- SonarQube Scanner
- OWASP Dependency-Check

### 3.5 凭据管理

**访问路径：** 管理 Jenkins → 凭据 → 系统 → 全局凭据

**凭据类型：**

- **用户名和密码**：用于认证
- **SSH 密钥**：用于 Git 访问
- **令牌**：用于 API 访问
- **Secret file**：用于存储配置文件
- **Secret text**：用于存储密钥

---

## 四、资源分配与优化

### 4.1 系统资源配置

**JVM 配置：**

```bash
# /etc/default/jenkins
JENKINS_JAVA_OPTIONS="-Xms4G -Xmx8G -XX:MaxPermSize=512M -XX:+UseG1GC"

# 或 Docker 容器
-e JAVA_OPTS="-Xms4G -Xmx8G -XX:MaxPermSize=512M -XX:+UseG1GC"
```

**系统限制：**

```bash
# /etc/security/limits.conf
jenkins soft nofile 65536
jenkins hard nofile 65536

# 重新加载
sysctl -p
```

### 4.2 构建执行器配置

**访问路径：** 管理 Jenkins → 系统配置 → 执行器数量

**配置建议：**

- **CPU 核心数**：执行器数量 ≤ CPU 核心数
- **内存**：每个执行器预留 2-4GB 内存
- **磁盘**：确保足够的构建空间

### 4.3 代理配置

**创建代理：**

1. 管理 Jenkins → 管理节点和云 → 新建节点
2. 填写节点名称和执行器数量
3. 选择启动方式：
   - **通过 Java Web Start**：手动启动
   - **通过 SSH**：自动启动
   - **通过云**：动态创建

**代理配置：**

- **标签**：用于指定构建任务运行的节点
- **工具位置**：配置代理节点的工具路径
- **环境变量**：配置代理节点的环境变量

### 4.4 性能优化

**构建优化：**

- 使用流水线并行构建
- 合理设置构建超时时间
- 清理构建历史
- 使用轻量级构建

**JVM 优化：**

- 调整内存参数
- 使用 G1GC 垃圾收集器
- 启用类数据共享

**系统优化：**

- 使用 SSD 存储
- 配置适当的交换空间
- 优化网络连接

---

## 五、备份与恢复策略

### 5.1 数据备份

**备份目录：**

- **Jenkins 主目录**：`/var/lib/jenkins` 或 `$JENKINS_HOME`
- **配置文件**：`/etc/default/jenkins` 或 `C:\Program Files\Jenkins\jenkins.xml`

**手动备份：**

```bash
# 停止 Jenkins
systemctl stop jenkins

# 备份数据
cp -r /var/lib/jenkins /backup/jenkins-$(date +%Y%m%d)

# 启动 Jenkins
systemctl start jenkins
```

**自动备份：**

```bash
# 编辑 crontab
crontab -e

# 添加定时任务（每天凌晨 2 点备份）
0 2 * * * /opt/scripts/jenkins-backup.sh

# 备份脚本示例
#!/bin/bash
BACKUP_DIR="/backup/jenkins"
DATE=$(date +%Y%m%d_%H%M%S)
JENKINS_HOME="/var/lib/jenkins"

mkdir -p $BACKUP_DIR
tar -czf $BACKUP_DIR/jenkins-$DATE.tar.gz $JENKINS_HOME

# 保留最近 7 天的备份
find $BACKUP_DIR -name "jenkins-*.tar.gz" -mtime +7 -delete
```

### 5.2 恢复策略

**恢复步骤：**

1. **停止 Jenkins**：`systemctl stop jenkins`
2. **备份当前数据**：`mv /var/lib/jenkins /var/lib/jenkins.bak`
3. **恢复备份**：`tar -xzf /backup/jenkins-20260308.tar.gz -C /var/lib/`
4. **修复权限**：`chown -R jenkins:jenkins /var/lib/jenkins`
5. **启动 Jenkins**：`systemctl start jenkins`
6. **验证恢复**：访问 Jenkins 界面，检查配置和任务

### 5.3 灾难恢复

**灾备方案：**

1. **定期备份**：每日自动备份
2. **异地存储**：将备份存储到不同地理位置
3. **多环境部署**：主备 Jenkins 服务器
4. **快速恢复**：预配置恢复脚本

**恢复演练：**

- 每月进行一次恢复测试
- 验证备份文件的完整性
- 更新恢复文档

---

## 六、监控与告警

### 6.1 内置监控

**系统信息：**

- 访问路径：管理 Jenkins → 系统信息
- 查看 Java 系统属性、环境变量、插件信息

**系统日志：**

- 访问路径：管理 Jenkins → 系统日志
- 查看 Jenkins 运行日志

**构建队列：**

- 访问路径：Jenkins 主页 → 构建队列
- 查看等待执行的构建任务

### 6.2 外部监控

**Prometheus 监控：**

1. **安装 Prometheus 插件**：管理 Jenkins → 插件管理 → 搜索 "Prometheus"
2. **配置 Prometheus**：添加 Jenkins 作为目标
   ```yaml
   scrape_configs:
     - job_name: 'jenkins'
       metrics_path: '/prometheus'
       static_configs:
         - targets: ['jenkins:8080']
   ```
3. **查看指标**：`http://jenkins:8080/prometheus`

**Grafana 仪表盘：**

1. **导入仪表盘**：ID 9964（Jenkins Overview）
2. **配置数据源**：指向 Prometheus
3. **监控指标**：构建成功率、执行时间、队列长度等

**Zabbix 监控：**

1. **安装 Zabbix 客户端**：`apt-get install zabbix-agent`
2. **配置监控项**：CPU、内存、磁盘、服务状态
3. **设置触发器**：当服务异常时告警

### 6.3 日志监控

**ELK Stack 集成：**

1. **安装 Filebeat**：`apt-get install filebeat`
2. **配置 Filebeat**：收集 Jenkins 日志
3. **发送到 Elasticsearch**：`output.elasticsearch: hosts: ["localhost:9200"]`
4. **在 Kibana 中查看**：创建日志仪表盘

**Graylog 集成：**

1. **安装 Graylog**：`docker run -d -p 9000:9000 graylog/graylog:latest`
2. **配置 GELF 输入**：在 Graylog 中创建 GELF 输入
3. **配置 Jenkins**：安装 Graylog 插件，配置 GELF 输出

### 6.4 告警配置

**邮件告警：**

1. **配置邮件服务器**：管理 Jenkins → 系统配置 → 邮件通知
2. **配置告警触发**：在构建任务中配置邮件通知
3. **测试邮件**：点击 "Test configuration"

**Slack 告警：**

1. **安装 Slack 插件**：管理 Jenkins → 插件管理
2. **配置 Slack**：管理 Jenkins → 系统配置 → Slack
3. **在任务中启用**：构建后操作 → Slack 通知

**Webhook 告警：**

1. **安装 Generic Webhook Trigger 插件**
2. **配置 webhook**：在构建任务中配置 webhook 通知
3. **接收端处理**：自定义系统接收告警

---

## 七、安全加固措施

### 7.1 系统安全

**更新系统：**

```bash
# Ubuntu/Debian
apt-get update && apt-get upgrade -y

# CentOS/RHEL
yum update -y
```

**配置防火墙：**

```bash
# 启用防火墙
ufw enable

# 允许 Jenkins 端口
ufw allow 8080/tcp
ufw allow 50000/tcp

# 允许 SSH
ufw allow 22/tcp
```

**禁用 root 远程登录：**

```bash
# /etc/ssh/sshd_config
PermitRootLogin no

# 重启 SSH
systemctl restart sshd
```

### 7.2 Jenkins 安全配置

**访问控制：**

- **启用安全**：管理 Jenkins → 全局安全配置 → 启用安全
- **选择认证方式**：Jenkins 专有用户数据库或 LDAP
- **配置授权策略**：安全矩阵或项目矩阵
- **启用 CSRF 保护**：防止跨站请求伪造

**用户权限：**

- **最小权限原则**：只授予必要的权限
- **角色管理**：使用 Role-based Strategy 插件
- **定期审查**：定期检查用户权限

**插件安全：**

- **使用官方插件**：只从 Jenkins 官方插件库安装
- **定期更新**：及时更新插件到最新版本
- **禁用未使用插件**：禁用或卸载未使用的插件

### 7.3 网络安全

**HTTPS 配置：**

1. **获取 SSL 证书**：使用 Let's Encrypt 或自签名证书
2. **配置 Jenkins**：管理 Jenkins → 系统配置 → Jenkins URL
3. **更新端口**：修改 Jenkins 配置使用 443 端口

**反向代理：**

```nginx
server {
    listen 443 ssl;
    server_name jenkins.example.com;
    
    ssl_certificate /etc/ssl/certs/jenkins.crt;
    ssl_certificate_key /etc/ssl/private/jenkins.key;
    
    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**网络隔离：**

- 将 Jenkins 部署在专用网络段
- 使用 VPN 访问 Jenkins
- 限制 Jenkins 访问 IP

### 7.4 凭证安全

**凭证管理：**

- **使用凭据插件**：存储密码、SSH 密钥等
- **加密存储**：Jenkins 自动加密存储凭据
- **定期轮换**：定期更新凭据

**敏感信息处理：**

- **避免硬编码**：使用环境变量或凭据
- **构建日志脱敏**：避免在日志中显示敏感信息
- **限制凭据访问**：只授予必要的用户访问权限

---

## 八、故障排查流程

### 8.1 服务启动失败

**排查步骤：**

1. **查看服务状态**：`systemctl status jenkins`
2. **查看日志**：`journalctl -u jenkins`
3. **检查 Java 环境**：`java -version`
4. **检查端口占用**：`netstat -tlnp | grep 8080`
5. **检查权限**：`ls -la /var/lib/jenkins`
6. **检查磁盘空间**：`df -h`

**常见问题：**

- Java 版本不兼容
- 端口被占用
- 权限不足
- 磁盘空间不足

### 8.2 构建失败

**排查步骤：**

1. **查看构建日志**：Jenkins 界面 → 构建历史 → 查看日志
2. **检查构建配置**：检查构建步骤和参数
3. **检查依赖**：检查 Git、Maven 等工具
4. **检查网络**：检查网络连接和代理设置
5. **检查权限**：检查文件和目录权限

**常见问题：**

- 代码编译错误
- 依赖下载失败
- 测试失败
- 网络超时

### 8.3 插件问题

**排查步骤：**

1. **查看插件状态**：管理 Jenkins → 插件管理 → 已安装
2. **查看日志**：管理 Jenkins → 系统日志
3. **检查版本兼容**：检查插件与 Jenkins 版本的兼容性
4. **重新安装插件**：卸载并重新安装插件

**常见问题：**

- 插件版本不兼容
- 插件冲突
- 插件依赖缺失

### 8.4 性能问题

**排查步骤：**

1. **查看系统资源**：`top`、`free -m`、`df -h`
2. **查看 Jenkins 队列**：Jenkins 主页 → 构建队列
3. **查看 JVM 状态**：`jstat -gc <pid> 1000`
4. **检查构建任务**：查看是否有长时间运行的任务

**常见问题：**

- 内存不足
- CPU 使用率高
- 磁盘 I/O 瓶颈
- 构建任务过多

---

## 九、日常维护任务

### 9.1 常规维护

**每日维护：**

- 检查 Jenkins 服务状态
- 查看构建失败的任务
- 监控系统资源使用情况
- 检查日志中的错误信息

**每周维护：**

- 更新 Jenkins 到最新版本
- 更新插件到最新版本
- 清理构建历史
- 检查磁盘空间
- 执行备份

**每月维护：**

- 审查用户权限
- 检查安全配置
- 进行恢复测试
- 更新维护文档

### 9.2 清理任务

**清理构建历史：**

1. **手动清理**：Jenkins 界面 → 项目 → 构建历史 → 删除
2. **自动清理**：使用 Build History Manager 插件
3. **配置保留策略**：项目配置 → 构建历史 → 保留构建

**清理工作空间：**

1. **手动清理**：Jenkins 界面 → 项目 → 工作空间 → 清理
2. **自动清理**：在构建后操作中添加 "Delete workspace when build is done"

**清理日志：**

1. **清理系统日志**：`find /var/log/jenkins -name "*.log" -mtime +7 -delete`
2. **清理构建日志**：使用 Log Rotator 插件

### 9.3 升级与迁移

**Jenkins 升级：**

1. **备份数据**：`tar -czf jenkins-backup.tar.gz /var/lib/jenkins`
2. **停止服务**：`systemctl stop jenkins`
3. **升级 Jenkins**：
   - Ubuntu/Debian：`apt-get update && apt-get upgrade jenkins`
   - CentOS/RHEL：`yum update jenkins`
4. **启动服务**：`systemctl start jenkins`
5. **验证升级**：访问 Jenkins 界面，检查插件兼容性

**Jenkins 迁移：**

1. **在原服务器备份**：`tar -czf jenkins-backup.tar.gz /var/lib/jenkins`
2. **在新服务器安装 Jenkins**：按照安装步骤安装
3. **停止新服务器 Jenkins**：`systemctl stop jenkins`
4. **恢复备份**：`tar -xzf jenkins-backup.tar.gz -C /var/lib/`
5. **修复权限**：`chown -R jenkins:jenkins /var/lib/jenkins`
6. **启动服务**：`systemctl start jenkins`
7. **更新 URL**：管理 Jenkins → 系统配置 → Jenkins URL

---

## 十、常见问题与解决方案

### 10.1 服务相关问题

**Jenkins 无法启动：**

- **原因**：端口被占用、Java 版本不兼容、配置文件错误
- **解决方案**：
  - 检查端口：`netstat -tlnp | grep 8080`
  - 检查 Java 版本：`java -version`
  - 检查配置文件：`/etc/default/jenkins`
  - 查看日志：`journalctl -u jenkins`

**Jenkins 响应缓慢：**

- **原因**：内存不足、CPU 使用率高、磁盘 I/O 瓶颈
- **解决方案**：
  - 增加内存：修改 JVM 配置
  - 增加 CPU：升级硬件或调整执行器数量
  - 使用 SSD：提高磁盘 I/O 性能
  - 清理构建历史：减少磁盘使用

### 10.2 构建相关问题

**构建超时：**

- **原因**：构建时间过长、网络问题、资源不足
- **解决方案**：
  - 设置构建超时时间：项目配置 → 构建超时
  - 优化构建脚本：减少构建时间
  - 检查网络连接：确保网络稳定
  - 增加资源：为构建分配更多资源

**构建失败：**

- **原因**：代码错误、依赖问题、配置错误
- **解决方案**：
  - 查看构建日志：分析失败原因
  - 检查代码：修复代码错误
  - 检查依赖：确保依赖可用
  - 检查配置：验证构建配置

### 10.3 插件相关问题

**插件安装失败：**

- **原因**：网络问题、版本不兼容、磁盘空间不足
- **解决方案**：
  - 检查网络：确保网络连接正常
  - 检查版本：确保插件与 Jenkins 版本兼容
  - 检查磁盘空间：`df -h`
  - 手动安装：下载插件文件并上传

**插件冲突：**

- **原因**：多个插件功能冲突
- **解决方案**：
  - 禁用冲突插件：管理 Jenkins → 插件管理 → 已安装
  - 升级插件：更新到最新版本
  - 替换插件：使用功能相似的插件

### 10.4 安全相关问题

**权限错误：**

- **原因**：用户权限不足、权限配置错误
- **解决方案**：
  - 检查用户权限：管理 Jenkins → 管理用户
  - 检查授权策略：管理 Jenkins → 全局安全配置
  - 重新配置权限：根据需要调整权限

**访问被拒绝：**

- **原因**：认证失败、IP 限制、CSRF 保护
- **解决方案**：
  - 检查凭据：确保用户名和密码正确
  - 检查 IP 限制：确保 IP 被允许
  - 检查 CSRF 令牌：确保请求包含有效的 CSRF 令牌

### 10.5 其他问题

**备份失败：**

- **原因**：磁盘空间不足、权限不足、Jenkins 运行中
- **解决方案**：
  - 检查磁盘空间：`df -h`
  - 检查权限：确保备份目录可写
  - 停止 Jenkins：在备份前停止服务

**恢复失败：**

- **原因**：备份文件损坏、权限错误、版本不兼容
- **解决方案**：
  - 检查备份文件：验证备份文件完整性
  - 修复权限：`chown -R jenkins:jenkins /var/lib/jenkins`
  - 确保版本兼容：使用相同或兼容的 Jenkins 版本

---

**修订记录**

| 版本 | 日期 | 作者 | 修改内容 |
|------|------|------|----------|
| v1.0 | 2026-03-08 | 运维团队 | 初始版本 |
