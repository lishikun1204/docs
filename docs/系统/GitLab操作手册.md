# GitLab 操作手册

> 版本：GitLab 16.0+  
> 适用对象：开发工程师、产品经理、测试人员  
> 最后更新：2026-03-08

---

## 目录

1. [GitLab 平台介绍](#一-gitlab-平台介绍)
2. [项目创建与管理](#二-项目创建与管理)
3. [代码仓库操作](#三-代码仓库操作)
4. [分支管理与合并请求](#四-分支管理与合并请求)
5. [CI/CD 流水线配置](#五-cicd-流水线配置)
6. [用户与权限管理](#六-用户与权限管理)
7. [Issue 与里程碑管理](#七-issue-与里程碑管理)
8. [Wiki 与文档管理](#八-wiki-与文档管理)
9. [项目设置与配置](#九-项目设置与配置)
10. [常见问题与解决方案](#十-常见问题与解决方案)

---

## 一、GitLab 平台介绍

### 1.1 GitLab 概述

GitLab 是一个基于 Git 的代码托管平台，提供代码仓库管理、CI/CD 流水线、问题追踪、Wiki 文档等功能，支持团队协作开发。

### 1.2 核心功能

| 功能模块 | 说明 |
|---------|------|
| 代码仓库 | Git 版本控制、分支管理、合并请求 |
| CI/CD | 持续集成、持续部署、自动化测试 |
| 问题追踪 | Issue 管理、里程碑、标签 |
| Wiki | 项目文档管理、知识库 |
| 容器镜像库 | 容器镜像管理 |
| 包注册表 | 软件包管理 |
| 安全扫描 | 代码安全分析、依赖扫描 |

### 1.3 访问方式

- **Web 界面**：通过浏览器访问 GitLab 服务器
- **Git 客户端**：使用 Git 命令行工具
- **API**：通过 REST API 或 GraphQL API 访问
- **GitLab CLI**：使用 `glab` 命令行工具

### 1.4 术语定义

| 术语 | 说明 |
|------|------|
| 项目 | GitLab 中的代码仓库及其相关资源 |
| 组 | 项目的集合，用于管理多个相关项目 |
| 分支 | 代码的独立开发线 |
| 提交 | 代码的修改记录 |
| 合并请求 | 将代码从一个分支合并到另一个分支的请求 |
| Issue | 问题、任务或 bug 的追踪项 |
| 里程碑 | 项目的阶段性目标 |
| 流水线 | CI/CD 的自动化流程 |
| Runner | 执行 CI/CD 任务的代理 |

---

## 二、项目创建与管理

### 2.1 创建新项目

**Web 界面操作：**

1. 登录 GitLab
2. 点击顶部导航栏的 "New project" 按钮
3. 选择项目创建方式：
   - **Create blank project**：创建空项目
   - **Import project**：导入现有项目
   - **Create from template**：从模板创建
4. 填写项目信息：
   - **Project name**：项目名称
   - **Project slug**：项目路径
   - **Project description**：项目描述
   - **Visibility level**：可见性级别（Private、Internal、Public）
5. 配置高级选项：
   - 初始化 README
   - 添加 .gitignore 模板
   - 添加许可证
6. 点击 "Create project" 按钮

**命令行操作：**

```bash
# 克隆空项目
git clone https://gitlab.example.com/username/project.git
cd project

# 创建 README.md
echo "# Project" > README.md
git add README.md
git commit -m "Initial commit"
git push origin main
```

### 2.2 项目设置

**基本设置：**

1. 进入项目 → Settings → General
2. 修改项目名称、描述、可见性
3. 配置项目功能开关（Issues、Merge Requests、CI/CD 等）
4. 设置项目 avatar

**集成与服务：**

1. 进入项目 → Settings → Integrations
2. 配置外部服务：
   - JIRA
   - Slack
   - Webhook
   - 其他集成

**高级设置：**

1. 进入项目 → Settings → Advanced
2. 配置：
   - 项目归档
   - 项目转移
   - 项目删除
   - 导入/导出

### 2.3 项目导入与导出

**导出项目：**

1. 进入项目 → Settings → Advanced → Export project
2. 选择导出内容
3. 点击 "Export project" 按钮
4. 下载导出文件

**导入项目：**

1. 进入 GitLab 主页 → New project → Import project
2. 选择 "GitLab export"
3. 上传导出文件
4. 填写项目信息
5. 点击 "Import project" 按钮

### 2.4 项目模板

**使用内置模板：**

1. 进入 GitLab 主页 → New project → Create from template
2. 选择模板类型：
   - Ruby
   - Node.js
   - React
   - Python
   - 其他
3. 填写项目信息
4. 点击 "Create project" 按钮

**创建自定义模板：**

1. 创建模板项目
2. 进入项目 → Settings → General → Visibility
3. 设置为 "Public"
4. 其他项目可以通过导入模板项目创建

---

## 三、代码仓库操作

### 3.1 克隆仓库

**HTTPS 克隆：**

```bash
git clone https://gitlab.example.com/username/project.git
```

**SSH 克隆：**

```bash
git clone git@gitlab.example.com:username/project.git
```

**克隆指定分支：**

```bash
git clone -b develop https://gitlab.example.com/username/project.git
```

**克隆子模块：**

```bash
git clone --recursive https://gitlab.example.com/username/project.git
```

### 3.2 提交代码

**基本流程：**

```bash
# 查看状态
git status

# 添加修改
git add .

# 提交修改
git commit -m "feat: 添加新功能"

# 推送代码
git push origin main
```

**提交规范：**

```bash
# 格式
<type>(<scope>): <subject>

<body>

<footer>

# 类型说明
feat:     新功能
fix:      修复bug
docs:     文档更新
style:    代码格式
refactor: 代码重构
perf:     性能优化
test:     测试相关
chore:    构建/工具

# 示例
feat(用户模块): 添加用户登录功能

- 实现用户名密码登录
- 添加验证码支持

Closes #123
```

### 3.3 拉取代码

```bash
# 拉取最新代码
git pull origin main

# 拉取并变基
git pull --rebase origin main

# 拉取特定分支
git pull origin develop
```

### 3.4 查看历史

```bash
# 查看提交历史
git log

# 单行显示
git log --oneline

# 图形化显示
git log --oneline --graph

# 查看文件历史
git log -- filename.txt

# 查看特定提交
git show commit_hash
```

### 3.5 代码比较

**Web 界面：**

1. 进入项目 → Repository → Compare
2. 选择比较的分支或提交
3. 查看差异

**命令行：**

```bash
# 比较分支
git diff main..develop

# 比较提交
git diff commit1 commit2

# 比较文件
git diff main..develop -- filename.txt
```

---

## 四、分支管理与合并请求

### 4.1 分支创建

**Web 界面：**

1. 进入项目 → Repository → Branches
2. 点击 "New branch" 按钮
3. 填写分支名称
4. 选择源分支
5. 点击 "Create branch" 按钮

**命令行：**

```bash
# 创建分支
git branch feature-login

# 创建并切换分支
git checkout -b feature-login

# 基于特定提交创建分支
git checkout -b feature-login commit_hash
```

### 4.2 分支管理

**查看分支：**

```bash
# 查看本地分支
git branch

# 查看所有分支
git branch -a

# 查看远程分支
git branch -r
```

**删除分支：**

```bash
# 删除本地分支
git branch -d feature-login

# 强制删除
git branch -D feature-login

# 删除远程分支
git push origin --delete feature-login
```

**分支重命名：**

```bash
# 重命名本地分支
git branch -m old-name new-name

# 推送新分支
git push origin new-name

# 删除旧分支
git push origin --delete old-name
```

### 4.3 合并请求

**创建合并请求：**

1. 进入项目 → Merge Requests
2. 点击 "New merge request" 按钮
3. 选择源分支和目标分支
4. 填写标题和描述
5. 选择 assignee、reviewer
6. 关联 Issue
7. 点击 "Create merge request" 按钮

**合并请求审核：**

1. 进入合并请求页面
2. 查看代码差异
3. 进行评论
4. 批准或请求修改
5. 点击 "Merge" 按钮完成合并

**合并策略：**

- **Fast-forward merge**：直接快进合并
- **Merge commit**：创建合并提交
- **Squash commit**：压缩为单个提交
- **Rebase**：变基后合并

### 4.4 分支保护

**设置分支保护：**

1. 进入项目 → Settings → Repository → Protected branches
2. 选择分支规则
3. 配置保护设置：
   - 谁可以合并
   - 谁可以推送
   - 是否需要代码审查
   - 是否需要 CI 成功
4. 点击 "Protect" 按钮

**常用保护规则：**

- **main/master 分支**：只允许维护者合并，需要代码审查
- **develop 分支**：允许开发者推送，需要 CI 成功
- **release 分支**：只允许维护者推送和合并

---

## 五、CI/CD 流水线配置

### 5.1 CI/CD 配置文件

**创建 .gitlab-ci.yml 文件：**

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

### 5.2 流水线触发

**自动触发：**

- 推送代码到分支
- 创建合并请求
- 定时触发（cron）

**手动触发：**

1. 进入项目 → CI/CD → Pipelines
2. 点击 "Run pipeline" 按钮
3. 选择分支
4. 填写变量
5. 点击 "Run pipeline" 按钮

### 5.3 流水线管理

**查看流水线：**

1. 进入项目 → CI/CD → Pipelines
2. 查看流水线状态和历史

**查看作业：**

1. 进入流水线详情
2. 查看各个作业的执行状态
3. 点击作业查看日志

**取消流水线：**

1. 进入流水线详情
2. 点击 "Cancel pipeline" 按钮

### 5.4 环境管理

**配置环境：**

1. 进入项目 → CI/CD → Environments
2. 查看环境状态
3. 配置环境变量

**部署策略：**

- **蓝绿部署**：同时运行两个环境，切换流量
- **金丝雀部署**：逐步将流量引导到新版本
- **滚动部署**：逐个更新实例

### 5.5 GitLab Runner 管理

**查看 Runner：**

1. 进入项目 → Settings → CI/CD → Runners
2. 查看可用的 Runner

**注册 Runner：**

```bash
# 注册 Runner
gitlab-runner register

# 输入 GitLab URL
# 输入注册令牌
# 输入 Runner 描述
# 输入标签
# 选择执行器（shell、docker、kubernetes 等）
```

---

## 六、用户与权限管理

### 6.1 用户角色

| 角色 | 权限 | 适用场景 |
|------|------|----------|
| Guest | 只读权限，可创建 Issue | 外部协作者 |
| Reporter | 查看代码、创建 Issue | 测试人员、产品经理 |
| Developer | 开发权限，可推送代码 | 开发人员 |
| Maintainer | 管理权限，可配置项目 | 项目负责人 |
| Owner | 完全控制权限 | 项目所有者 |

### 6.2 添加项目成员

**Web 界面：**

1. 进入项目 → Members
2. 点击 "Invite members" 按钮
3. 输入用户名或邮箱
4. 选择角色
5. 设置过期日期（可选）
6. 点击 "Invite" 按钮

**批量添加：**

1. 进入项目 → Members
2. 点击 "Import members" 按钮
3. 从其他项目导入成员

### 6.3 组管理

**创建组：**

1. 进入 GitLab 主页 → Groups → New group
2. 填写组名称和路径
3. 设置可见性级别
4. 点击 "Create group" 按钮

**添加组成员：**

1. 进入组 → Members
2. 点击 "Invite members" 按钮
3. 输入用户名或邮箱
4. 选择角色
5. 点击 "Invite" 按钮

**组层级：**

- 支持嵌套组
- 子组继承父组的成员权限
- 可设置组级别的 CI/CD 变量

### 6.4 权限设置

**项目权限：**

1. 进入项目 → Settings → Members
2. 管理项目成员及其权限

**分支权限：**

1. 进入项目 → Settings → Repository → Protected branches
2. 配置分支保护规则

**标签权限：**

1. 进入项目 → Settings → Repository → Protected tags
2. 配置标签保护规则

**环境权限：**

1. 进入项目 → CI/CD → Environments
2. 配置环境访问权限

---

## 七、Issue 与里程碑管理

### 7.1 创建 Issue

**Web 界面：**

1. 进入项目 → Issues
2. 点击 "New issue" 按钮
3. 填写标题和描述
4. 配置：
   - Assignee：负责人
   - Milestone：里程碑
   - Labels：标签
   - Due date：截止日期
5. 点击 "Submit issue" 按钮

**快捷创建：**

- 在提交信息中使用 `Closes #123` 关联 Issue
- 使用 Issue 模板

### 7.2 Issue 管理

**查看 Issue：**

1. 进入项目 → Issues
2. 使用过滤器和搜索
3. 按状态、标签、里程碑等筛选

**更新 Issue：**

1. 进入 Issue 详情页
2. 修改状态、 assignee、标签等
3. 添加评论
4. 关闭或重新打开 Issue

**批量操作：**

1. 进入项目 → Issues
2. 选择多个 Issue
3. 执行批量操作（关闭、分配、添加标签等）

### 7.3 里程碑管理

**创建里程碑：**

1. 进入项目 → Issues → Milestones
2. 点击 "New milestone" 按钮
3. 填写标题、描述、开始和结束日期
4. 点击 "Create milestone" 按钮

**管理里程碑：**

1. 进入项目 → Issues → Milestones
2. 查看里程碑进度
3. 编辑或删除里程碑
4. 关联 Issue 到里程碑

### 7.4 标签管理

**创建标签：**

1. 进入项目 → Issues → Labels
2. 点击 "New label" 按钮
3. 填写名称、描述、颜色
4. 点击 "Create label" 按钮

**使用标签：**

- 为 Issue 添加标签
- 为合并请求添加标签
- 使用标签进行筛选和管理

**常用标签：**

- `bug`：缺陷
- `feature`：新功能
- `documentation`：文档
- `enhancement`：增强
- `critical`：关键
- `wontfix`：不会修复

---

## 八、Wiki 与文档管理

### 8.1 创建 Wiki 页面

**Web 界面：**

1. 进入项目 → Wiki
2. 点击 "New page" 按钮
3. 填写标题和内容
4. 选择格式（Markdown、HTML 等）
5. 点击 "Create page" 按钮

**上传文件：**

1. 进入 Wiki 页面编辑
2. 点击 "Attach a file" 按钮
3. 选择文件上传
4. 在页面中引用文件

### 8.2 Wiki 导航

**创建侧边栏：**

1. 进入项目 → Wiki
2. 点击 "Sidebar"
3. 编辑侧边栏内容
4. 点击 "Save changes" 按钮

**创建目录：**

1. 使用层级标题
2. 利用侧边栏组织页面
3. 创建索引页面

### 8.3 文档规范

**Markdown 格式：**

```markdown
# 标题 1

## 标题 2

### 标题 3

- 列表项 1
- 列表项 2

```bash
# 代码块
echo "Hello World"
```

| 表头 1 | 表头 2 |
|--------|--------|
| 内容 1 | 内容 2 |

> 引用内容

![图片描述](path/to/image.png)
```

**文档结构：**

- 首页：项目概述
- 安装指南：环境搭建
- 快速开始：基本使用
- API 文档：接口说明
- 常见问题：FAQ

---

## 九、项目设置与配置

### 9.1 集成设置

**Webhook 配置：**

1. 进入项目 → Settings → Integrations → Webhooks
2. 填写 URL 和 Secret Token
3. 选择触发事件
4. 点击 "Add webhook" 按钮

**服务集成：**

1. 进入项目 → Settings → Integrations
2. 选择服务：
   - JIRA
   - Slack
   - Microsoft Teams
   - GitHub
   - 其他
3. 填写配置信息
4. 点击 "Save changes" 按钮

### 9.2 CI/CD 设置

**变量配置：**

1. 进入项目 → Settings → CI/CD → Variables
2. 点击 "Add variable" 按钮
3. 填写变量名称和值
4. 选择变量类型（变量或文件）
5. 选择保护和掩码选项
6. 点击 "Add variable" 按钮

**Runner 设置：**

1. 进入项目 → Settings → CI/CD → Runners
2. 管理 Runner 配置
3. 启用或禁用 Runner

### 9.3 仓库设置

**分支设置：**

1. 进入项目 → Settings → Repository → Branches
2. 配置默认分支
3. 管理分支保护规则

**标签设置：**

1. 进入项目 → Settings → Repository → Tags
2. 管理标签保护规则

**推送规则：**

1. 进入项目 → Settings → Repository → Push rules
2. 配置推送规则：
   - 提交消息规则
   - 分支名称规则
   - 文件大小限制

### 9.4 安全设置

**Protected environments：**

1. 进入项目 → Settings → CI/CD → Protected environments
2. 配置环境访问权限

**Security scanning：**

1. 进入项目 → Security & Compliance → Configuration
2. 启用安全扫描：
   - SAST（静态应用安全测试）
   - DAST（动态应用安全测试）
   - Dependency scanning（依赖扫描）
   - Container scanning（容器扫描）

---

## 十、常见问题与解决方案

### 10.1 代码提交问题

**推送被拒绝：**

```bash
# 原因：远程分支有新提交
# 解决方案：
git pull --rebase origin main
git push origin main
```

**权限不足：**

- 检查用户权限
- 确保 SSH 密钥正确配置
- 检查分支保护规则

**提交冲突：**

```bash
# 解决冲突
git pull --rebase origin main
# 手动解决冲突
git add .
git rebase --continue
git push origin main
```

### 10.2 合并请求问题

**CI 失败：**

- 查看 CI 日志
- 修复构建错误
- 重新运行 CI

**代码审查：**

- 查看代码差异
- 添加评论和建议
- 等待审查通过

**合并冲突：**

- 解决冲突
- 重新推送
- 再次尝试合并

### 10.3 CI/CD 问题

**Runner 未运行：**

```bash
# 检查 Runner 状态
gitlab-runner status

# 重启 Runner
systemctl restart gitlab-runner
```

**流水线卡住：**

- 检查 Runner 状态
- 查看作业日志
- 取消并重新运行流水线

**环境变量问题：**

- 检查变量配置
- 确保变量正确传递
- 检查变量权限

### 10.4 Issue 管理问题

**Issue 重复：**

- 搜索现有 Issue
- 合并重复 Issue
- 关闭重复 Issue

**里程碑延期：**

- 更新里程碑日期
- 重新分配任务
- 调整项目计划

**标签管理：**

- 标准化标签使用
- 定期清理标签
- 建立标签使用指南

### 10.5 权限问题

**访问被拒绝：**

- 检查用户权限
- 检查项目可见性
- 联系项目管理员

**权限变更：**

- 进入项目 → Members
- 更新用户角色
- 保存变更

**组权限继承：**

- 检查组层级关系
- 确认成员权限设置
- 调整组权限

---

**修订记录**

| 版本 | 日期 | 作者 | 修改内容 |
|------|------|------|----------|
| v1.0 | 2026-03-08 | 开发团队 | 初始版本 |
