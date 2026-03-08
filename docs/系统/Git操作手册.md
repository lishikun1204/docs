# Git 操作手册

> 版本：Git 2.40+  
> 适用对象：开发工程师、运维工程师  
> 最后更新：2026-03-08

---

## 目录

1. [Git 环境搭建与配置](#一-git-环境搭建与配置)
2. [仓库初始化与克隆](#二-仓库初始化与克隆)
3. [基础操作](#三-基础操作)
4. [分支管理](#四-分支管理)
5. [远程仓库操作](#五-远程仓库操作)
6. [标签管理](#六-标签管理)
7. [冲突解决](#七-冲突解决)
8. [撤销与回滚](#八-撤销与回滚)
9. [Git 工作流规范](#九-git-工作流规范)
10. [常见问题与解决方案](#十-常见问题与解决方案)

---

## 一、Git 环境搭建与配置

### 1.1 安装 Git

#### 1.1.1 Linux 安装

```bash
# CentOS/RHEL
yum install -y git

# Ubuntu/Debian
apt-get update
apt-get install -y git

# 从源码编译
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.40.0.tar.gz
tar xzf git-2.40.0.tar.gz
cd git-2.40.0
make prefix=/usr/local all
make prefix=/usr/local install
```

#### 1.1.2 Windows 安装

1. 下载安装包：https://git-scm.com/download/win
2. 运行安装程序，按向导完成安装
3. 安装完成后，右键菜单会出现 "Git Bash Here" 选项

#### 1.1.3 macOS 安装

```bash
# 使用 Homebrew
brew install git

# 使用 Xcode Command Line Tools
xcode-select --install
```

### 1.2 验证安装

```bash
# 查看版本
git --version

# 查看安装路径
which git
where git    # Windows
```

### 1.3 初始配置

#### 1.3.1 用户信息配置

```bash
# 设置用户名
git config --global user.name "Your Name"

# 设置邮箱
git config --global user.email "your.email@example.com"

# 查看配置
git config --list
git config --global --list

# 查看特定配置项
git config user.name
git config user.email
```

#### 1.3.2 编辑器配置

```bash
# 设置默认编辑器
git config --global core.editor vim

# 设置默认编辑器为 VS Code
git config --global core.editor "code --wait"

# 设置默认编辑器为 Notepad++（Windows）
git config --global core.editor "'C:/Program Files/Notepad++/notepad++.exe' -multiInst -notabbar -nosession -noPlugin"
```

#### 1.3.3 换行符配置

```bash
# Windows（提交时转换为LF，检出时转换为CRLF）
git config --global core.autocrlf true

# Linux/macOS（提交时转换为LF，检出时不转换）
git config --global core.autocrlf input

# 禁用自动转换
git config --global core.autocrlf false
```

#### 1.3.4 常用配置项

```bash
# 设置默认分支名为 main
git config --global init.defaultBranch main

# 设置中文文件名显示
git config --global core.quotepath false

# 设置日志输出格式
git config --global format.pretty "%h - %an, %ar : %s"

# 设置别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"

# 设置颜色输出
git config --global color.ui auto
git config --global color.status auto
git config --global color.branch auto
git config --global color.diff auto
```

### 1.4 SSH 密钥配置

```bash
# 生成 SSH 密钥
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"

# 或使用 Ed25519 算法（推荐）
ssh-keygen -t ed25519 -C "your.email@example.com"

# 查看公钥
cat ~/.ssh/id_rsa.pub
cat ~/.ssh/id_ed25519.pub

# 测试 SSH 连接
ssh -T git@github.com
ssh -T git@gitee.com

# 配置多个 SSH 密钥
# ~/.ssh/config
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_github

Host gitee.com
    HostName gitee.com
    User git
    IdentityFile ~/.ssh/id_rsa_gitee
```

### 1.5 凭据管理

```bash
# 缓存凭据（默认15分钟）
git config --global credential.helper cache

# 缓存凭据（指定时间，单位秒）
git config --global credential.helper 'cache --timeout=3600'

# 永久存储凭据
git config --global credential.helper store

# Windows 凭据管理器
git config --global credential.helper manager

# macOS 钥匙串
git config --global credential.helper osxkeychain
```

---

## 二、仓库初始化与克隆

### 2.1 创建仓库

#### 2.1.1 初始化本地仓库

```bash
# 创建目录并初始化
mkdir my-project
cd my-project
git init

# 在现有目录初始化
cd existing-project
git init

# 初始化时指定分支名
git init -b main
git init --initial-branch=main

# 初始化裸仓库（用于服务器）
git init --bare project.git
```

#### 2.1.2 克隆远程仓库

```bash
# HTTPS 克隆
git clone https://github.com/user/project.git

# SSH 克隆
git clone git@github.com:user/project.git

# 克隆到指定目录
git clone https://github.com/user/project.git my-project

# 克隆指定分支
git clone -b develop https://github.com/user/project.git

# 浅克隆（只克隆最近一次提交）
git clone --depth 1 https://github.com/user/project.git

# 克隆单个分支
git clone --single-branch --branch main https://github.com/user/project.git

# 克隆指定深度和分支
git clone --depth 1 --branch v1.0.0 https://github.com/user/project.git
```

### 2.2 仓库结构

```
my-project/
├── .git/                  # Git 仓库目录
│   ├── HEAD               # 当前分支引用
│   ├── config             # 仓库配置
│   ├── description        # 仓库描述
│   ├── hooks/             # 钩子脚本
│   ├── index              # 暂存区
│   ├── info/              # 额外信息
│   ├── objects/           # 对象数据库
│   └── refs/              # 引用
│       ├── heads/         # 本地分支
│       ├── tags/          # 标签
│       └── remotes/       # 远程分支
├── .gitignore             # 忽略规则
├── .gitattributes         # 属性配置
└── 项目文件...
```

### 2.3 .gitignore 配置

```bash
# .gitignore 文件示例

# 操作系统文件
.DS_Store
Thumbs.db
Desktop.ini

# IDE 配置
.idea/
.vscode/
*.swp
*.swo
*~

# 编译产物
*.class
*.jar
*.war
target/
build/
dist/

# 依赖目录
node_modules/
vendor/
venv/

# 日志文件
*.log
logs/

# 环境配置
.env
.env.local
*.local

# 敏感信息
*.key
*.pem
credentials.json

# 临时文件
*.tmp
*.temp
*.bak

# 但排除特定文件
!important.log
```

### 2.4 .gitattributes 配置

```bash
# .gitattributes 文件示例

# 自动检测文本文件并转换换行符
* text=auto

# 指定文件类型的换行符
*.txt text
*.sh text eol=lf
*.bat text eol=crlf

# 二进制文件
*.png binary
*.jpg binary
*.pdf binary

# 语言特定属性
*.java text diff=java
*.py text diff=python
*.js text diff=javascript
```

---

## 三、基础操作

### 3.1 工作区域概念

```
┌─────────────┐    git add    ┌─────────────┐   git commit   ┌─────────────┐
│  工作目录    │ ────────────> │   暂存区     │ ─────────────> │   本地仓库   │
│ (Working    │               │  (Staging   │                │  (Local     │
│  Directory) │ <──────────── │    Area)    │ <───────────── │ Repository) │
└─────────────┘   git restore └─────────────┘  git restore   └─────────────┘
                  --staged                    --staged --worktree
```

### 3.2 查看状态

```bash
# 查看工作区状态
git status

# 简洁模式
git status -s
git status --short

# 显示分支信息
git status -sb

# 输出说明
# ??  未跟踪文件
# A   新添加到暂存区
# M   已修改
# MM  暂存区有修改，工作区也有修改
# D   已删除
# R   已重命名
```

### 3.3 添加文件

```bash
# 添加单个文件
git add filename.txt

# 添加多个文件
git add file1.txt file2.txt

# 添加所有文件
git add .
git add --all

# 添加所有 .txt 文件
git add *.txt

# 添加目录
git add src/

# 交互式添加
git add -p
git add --patch

# 添加时忽略空白变化
git add -w
git add --ignore-whitespace

# 更新已跟踪文件（不包括新文件）
git add -u
git add --update

# 查看暂存区内容
git status
git diff --staged
git diff --cached
```

### 3.4 提交更改

```bash
# 提交暂存区内容
git commit -m "提交说明"

# 提交并打开编辑器
git commit

# 提交所有已跟踪文件的修改
git commit -a -m "提交说明"
git commit --all -m "提交说明"

# 修改上次提交
git commit --amend -m "新的提交说明"

# 修改上次提交（不修改说明）
git commit --amend --no-edit

# 提交时添加作者信息
git commit --author="Name <email@example.com>" -m "提交说明"

# 提交时添加日期
git commit --date="2026-03-08 10:00:00" -m "提交说明"

# 签名提交
git commit -S -m "提交说明"
git commit --gpg-sign -m "提交说明"

# 允许空提交
git commit --allow-empty -m "空提交"
```

### 3.5 提交信息规范

```bash
# 推荐格式
<type>(<scope>): <subject>

<body>

<footer>

# 类型说明
feat:     新功能
fix:      修复bug
docs:     文档更新
style:    代码格式（不影响代码运行）
refactor: 重构（既不是新功能也不是修复bug）
perf:     性能优化
test:     测试相关
chore:    构建过程或辅助工具变动
revert:   回滚
build:    构建系统或外部依赖变更
ci:       CI配置文件和脚本变更

# 示例
feat(用户模块): 添加用户登录功能

- 实现用户名密码登录
- 添加验证码支持
- 添加登录日志记录

Closes #123
```

### 3.6 查看历史

```bash
# 查看提交历史
git log

# 单行显示
git log --oneline

# 图形化显示
git log --oneline --graph --all

# 显示最近 N 次提交
git log -n 5
git log -5

# 显示文件变更统计
git log --stat

# 显示每次提交的差异
git log -p

# 按作者筛选
git log --author="张三"

# 按提交信息筛选
git log --grep="关键词"

# 按时间筛选
git log --since="2026-01-01"
git log --until="2026-03-08"
git log --since="2 weeks ago"

# 按文件筛选
git log -- filename.txt
git log -p -- filename.txt

# 自定义格式
git log --pretty=format:"%h - %an, %ar : %s"
git log --pretty=format:"%h %ad | %s%d [%an]" --graph --date=short

# 格式占位符
# %H  完整哈希值
# %h  短哈希值
# %an 作者名字
# %ae 作者邮箱
# %ad 作者日期
# %ar 作者日期（相对时间）
# %cn 提交者名字
# %ce 提交者邮箱
# %cd 提交日期
# %s  提交说明
# %d  引用名称
```

### 3.7 查看差异

```bash
# 工作区与暂存区差异
git diff

# 暂存区与最新提交差异
git diff --staged
git diff --cached

# 工作区与最新提交差异
git diff HEAD

# 两个提交之间差异
git diff commit1 commit2

# 两个分支之间差异
git diff main develop

# 只显示文件名
git diff --name-only

# 显示统计信息
git diff --stat

# 查看特定文件差异
git diff -- filename.txt

# 忽略空白变化
git diff -w
git diff --ignore-all-space

# 显示差异上下文行数
git diff -U3
git diff --unified=3
```

### 3.8 删除与移动

```bash
# 删除文件（从工作区和暂存区）
git rm filename.txt

# 只从暂存区删除（保留工作区）
git rm --cached filename.txt

# 删除目录
git rm -r directory/

# 强制删除（未提交的修改也会删除）
git rm -f filename.txt

# 移动/重命名文件
git mv oldname.txt newname.txt

# 移动文件到目录
git mv file.txt newdir/

# 手动重命名后更新
mv oldname.txt newname.txt
git add newname.txt
git rm oldname.txt
```

---

## 四、分支管理

### 4.1 分支概念

```
                    main
                      │
                      ▼
A ─── B ─── C ─── D ─── E
              │
              └───── F ─── G  ◀── develop
```

### 4.2 创建分支

```bash
# 查看所有分支
git branch

# 查看远程分支
git branch -r

# 查看所有分支（本地+远程）
git branch -a

# 创建新分支
git branch feature-login

# 基于指定提交创建分支
git branch feature-login commit_hash

# 基于远程分支创建本地分支
git branch feature-login origin/feature-login

# 创建并切换分支
git checkout -b feature-login

# 创建并切换分支（新语法）
git switch -c feature-login

# 基于指定提交创建并切换
git checkout -b feature-login commit_hash

# 创建空分支（无提交历史）
git checkout --orphan new-branch
```

### 4.3 切换分支

```bash
# 切换分支
git checkout develop

# 切换分支（新语法）
git switch develop

# 切换到上一个分支
git checkout -
git switch -

# 切换并创建分支
git checkout -b feature-login
git switch -c feature-login

# 强制切换（丢弃未提交的修改）
git checkout -f develop
git switch -f develop
```

### 4.4 合并分支

```bash
# 合并指定分支到当前分支
git merge feature-login

# 合并并生成合并提交
git merge --no-ff feature-login

# 合并但不提交
git merge --no-commit feature-login

# 压缩合并（所有提交压缩为一个）
git merge --squash feature-login

# 合并时指定提交信息
git merge -m "合并登录功能" feature-login

# 中止合并
git merge --abort

# 查看合并状态
git status
```

### 4.5 删除分支

```bash
# 删除已合并的分支
git branch -d feature-login

# 强制删除分支（未合并也删除）
git branch -D feature-login

# 删除远程分支
git push origin --delete feature-login

# 删除本地已不存在的远程分支引用
git fetch -p
git fetch --prune

# 批量删除本地分支
git branch | grep 'feature/' | xargs git branch -d
```

### 4.6 分支重命名

```bash
# 重命名当前分支
git branch -m new-name

# 重命名指定分支
git branch -m old-name new-name

# 重命名远程分支（需要删除旧的并推送新的）
git branch -m old-name new-name
git push origin --delete old-name
git push origin new-name
```

### 4.7 分支查看

```bash
# 查看分支及其最后一次提交
git branch -v

# 查看分支及其上游分支
git branch -vv

# 查看已合并到当前分支的分支
git branch --merged

# 查看未合并到当前分支的分支
git branch --no-merged

# 查看包含指定提交的分支
git branch --contains commit_hash

# 查看不包含指定提交的分支
git branch --no-contains commit_hash
```

### 4.8 分支变基

```bash
# 变基到目标分支
git rebase main

# 交互式变基
git rebase -i main
git rebase -i HEAD~3

# 变基时保留合并提交
git rebase -p main

# 中止变基
git rebase --abort

# 继续变基
git rebase --continue

# 跳过当前提交
git rebase --skip

# 变基冲突解决后
git add .
git rebase --continue

# 黄金法则：不要对已推送的提交进行变基
```

### 4.9 Cherry-pick

```bash
# 选择特定提交应用到当前分支
git cherry-pick commit_hash

# 选择多个提交
git cherry-pick commit1 commit2

# 选择提交范围
git cherry-pick commit1..commit2

# 只应用但不提交
git cherry-pick -n commit_hash
git cherry-pick --no-commit commit_hash

# 继续cherry-pick
git cherry-pick --continue

# 放弃cherry-pick
git cherry-pick --abort
```

---

## 五、远程仓库操作

### 5.1 查看远程仓库

```bash
# 查看远程仓库
git remote

# 查看远程仓库详细信息
git remote -v

# 查看特定远程仓库信息
git remote show origin

# 查看远程仓库URL
git remote get-url origin
```

### 5.2 添加远程仓库

```bash
# 添加远程仓库
git remote add origin https://github.com/user/project.git

# 添加多个远程仓库
git remote add github https://github.com/user/project.git
git remote add gitee https://gitee.com/user/project.git

# 添加远程仓库并指定默认推送分支
git remote add -t main -m main origin https://github.com/user/project.git
```

### 5.3 修改远程仓库

```bash
# 修改远程仓库URL
git remote set-url origin https://github.com/user/new-project.git

# 修改远程仓库名称
git remote rename origin upstream

# 删除远程仓库
git remote remove origin
```

### 5.4 拉取更新

```bash
# 拉取远程仓库更新（不合并）
git fetch origin

# 拉取所有远程仓库更新
git fetch --all

# 拉取并清理无效的远程分支引用
git fetch -p
git fetch --prune

# 拉取特定分支
git fetch origin main

# 拉取并合并（pull = fetch + merge）
git pull origin main

# 拉取并变基
git pull --rebase origin main

# 拉取所有分支
git pull --all

# 设置上游分支
git branch --set-upstream-to=origin/main main
git branch -u origin/main
```

### 5.5 推送更新

```bash
# 推送到远程仓库
git push origin main

# 推送并设置上游分支
git push -u origin main
git push --set-upstream origin main

# 推送所有分支
git push --all origin

# 推送所有标签
git push --tags

# 强制推送（覆盖远程历史）
git push -f origin main
git push --force origin main

# 安全强制推送（如果远程有新提交则拒绝）
git push --force-with-lease origin main

# 删除远程分支
git push origin --delete feature-login

# 推送标签
git push origin v1.0.0

# 推送单个提交
git push origin commit_hash:refs/heads/branch-name
```

### 5.6 远程分支操作

```bash
# 查看远程分支
git branch -r

# 查看所有分支
git branch -a

# 创建本地分支跟踪远程分支
git checkout -b feature-login origin/feature-login
git checkout --track origin/feature-login

# 查看分支跟踪关系
git branch -vv

# 设置已有分支跟踪远程分支
git branch -u origin/feature-login

# 取消跟踪
git branch --unset-upstream
```

---

## 六、标签管理

### 6.1 创建标签

```bash
# 创建轻量标签
git tag v1.0.0

# 创建附注标签（推荐）
git tag -a v1.0.0 -m "版本 1.0.0 发布"

# 为特定提交创建标签
git tag -a v0.9.0 commit_hash -m "版本 0.9.0"

# 创建签名标签
git tag -s v1.0.0 -m "版本 1.0.0 发布"

# 验证签名标签
git tag -v v1.0.0
```

### 6.2 查看标签

```bash
# 查看所有标签
git tag

# 查看匹配模式的标签
git tag -l "v1.*"

# 查看标签信息
git show v1.0.0

# 查看标签列表及信息
git tag -n
```

### 6.3 推送标签

```bash
# 推送单个标签
git push origin v1.0.0

# 推送所有标签
git push origin --tags

# 推送时同时推送标签
git push --follow-tags
```

### 6.4 删除标签

```bash
# 删除本地标签
git tag -d v1.0.0

# 删除远程标签
git push origin --delete v1.0.0
git push origin :refs/tags/v1.0.0
```

### 6.5 检出标签

```bash
# 检出标签（分离HEAD状态）
git checkout v1.0.0

# 基于标签创建分支
git checkout -b version1 v1.0.0
git switch -c version1 v1.0.0
```

---

## 七、冲突解决

### 7.1 冲突产生场景

```bash
# 合并时产生冲突
git merge feature-branch

# 变基时产生冲突
git rebase main

# 拉取时产生冲突
git pull origin main

# Cherry-pick时产生冲突
git cherry-pick commit_hash
```

### 7.2 冲突文件状态

```bash
# 查看冲突文件
git status

# 冲突标记说明
# <<<<<<< HEAD
# 当前分支的内容
# =======
# 要合并进来的内容
# >>>>>>> feature-branch
```

### 7.3 解决冲突

```bash
# 方法一：手动编辑冲突文件
# 1. 打开冲突文件
# 2. 找到冲突标记
# 3. 修改为正确内容
# 4. 删除冲突标记

# 方法二：使用合并工具
git mergetool

# 方法三：选择一方版本
# 使用当前分支版本
git checkout --ours filename.txt

# 使用合并进来的版本
git checkout --theirs filename.txt

# 解决后添加到暂存区
git add filename.txt

# 继续合并/变基
git commit                    # 合并时
git rebase --continue         # 变基时
git cherry-pick --continue    # cherry-pick时
```

### 7.4 中止操作

```bash
# 中止合并
git merge --abort

# 中止变基
git rebase --abort

# 中止cherry-pick
git cherry-pick --abort

# 重置到操作前状态
git reset --hard HEAD
```

### 7.5 预览冲突

```bash
# 预览合并可能产生的冲突
git merge --no-commit --no-ff feature-branch

# 查看差异
git diff

# 取消预览
git merge --abort
```

---

## 八、撤销与回滚

### 8.1 撤销工作区修改

```bash
# 撤销单个文件修改
git checkout -- filename.txt
git restore filename.txt

# 撤销所有文件修改
git checkout -- .
git restore .

# 撤销特定目录修改
git restore path/to/directory/
```

### 8.2 撤销暂存区

```bash
# 取消暂存单个文件
git reset HEAD filename.txt
git restore --staged filename.txt

# 取消暂存所有文件
git reset HEAD
git restore --staged .

# 同时撤销暂存和工作区
git restore --staged --worktree filename.txt
```

### 8.3 撤销提交

```bash
# 撤销最后一次提交（保留修改）
git reset --soft HEAD~1

# 撤销最后一次提交（保留工作区修改，取消暂存）
git reset --mixed HEAD~1
git reset HEAD~1

# 撤销最后一次提交（丢弃所有修改）
git reset --hard HEAD~1

# 撤销多次提交
git reset --hard HEAD~3

# 撤销到指定提交
git reset --hard commit_hash
```

### 8.4 revert 回滚

```bash
# 回滚指定提交（创建新提交）
git revert commit_hash

# 回滚多个提交
git revert commit1 commit2

# 回滚提交范围
git revert commit1..commit2

# 回滚但不自动提交
git revert -n commit_hash

# 回滚合并提交
git revert -m 1 merge_commit_hash
# -m 1 表示保留第一个父提交的版本

# 取消正在进行的revert
git revert --abort

# 继续revert
git revert --continue
```

### 8.5 reset vs revert

| 操作 | reset | revert |
|------|-------|--------|
| 原理 | 移动HEAD指针 | 创建新提交来撤销 |
| 历史 | 会改变历史 | 不改变历史 |
| 安全性 | 已推送的提交不建议使用 | 安全，可用于已推送的提交 |
| 使用场景 | 本地未推送的提交 | 已推送的提交 |

### 8.6 恢复误删提交

```bash
# 查看操作历史
git reflog

# 恢复到指定状态
git reset --hard HEAD@{n}
git reset --hard commit_hash

# 查看悬空对象
git fsck --lost-found

# 恢复悬空提交
git cherry-pick commit_hash
```

### 8.7 清理未跟踪文件

```bash
# 查看将被删除的文件
git clean -n

# 删除未跟踪的文件
git clean -f

# 删除未跟踪的文件和目录
git clean -fd

# 同时删除被忽略的文件
git clean -fdx

# 只删除被忽略的文件
git clean -fdX

# 交互式删除
git clean -i
```

---

## 九、Git 工作流规范

### 9.1 Git Flow 工作流

```
                    ┌───────────────────────────────────── main
                    │                           │
                    │                     ┌─────┴─────┐
                    │                     │           │
                    └─────────────────────┘           │
                              │                       │
                    ┌─────────┴─────────┐             │
                    │                   │             │
              ┌─────┴─────┐       ┌─────┴─────┐       │
              │           │       │           │       │
         ┌────┴────┐ ┌────┴────┐ ┌┴───┐ ┌────┴───┐   │
         │         │ │         │ │    │ │        │   │
     feature/A feature/B feature/C hotfix  release  │
                                                    │
                                              develop
```

**分支说明：**

| 分支 | 说明 | 来源 | 合并到 |
|------|------|------|--------|
| main | 生产环境代码 | - | - |
| develop | 开发主分支 | main | main |
| feature/* | 功能开发分支 | develop | develop |
| release/* | 发布准备分支 | develop | main, develop |
| hotfix/* | 紧急修复分支 | main | main, develop |

**操作流程：**

```bash
# 1. 初始化
git checkout -b develop main

# 2. 开发新功能
git checkout -b feature/login develop
# ... 开发 ...
git checkout develop
git merge --no-ff feature/login
git branch -d feature/login

# 3. 准备发布
git checkout -b release/1.0.0 develop
# ... 修复bug、更新版本号 ...
git checkout main
git merge --no-ff release/1.0.0
git tag -a v1.0.0 -m "版本 1.0.0"
git checkout develop
git merge --no-ff release/1.0.0
git branch -d release/1.0.0

# 4. 紧急修复
git checkout -b hotfix/bug-123 main
# ... 修复 ...
git checkout main
git merge --no-ff hotfix/bug-123
git tag -a v1.0.1 -m "版本 1.0.1"
git checkout develop
git merge --no-ff hotfix/bug-123
git branch -d hotfix/bug-123
```

### 9.2 GitHub Flow 工作流

```
main ─── A ─── B ─── C ─── D ─── E
              │           │
              └───── F ───┘
               feature/xxx
```

**操作流程：**

```bash
# 1. 从main创建分支
git checkout -b feature/new-feature main

# 2. 开发并提交
git add .
git commit -m "feat: 添加新功能"

# 3. 推送到远程
git push -u origin feature/new-feature

# 4. 创建Pull Request

# 5. 代码审查后合并到main

# 6. 部署
```

### 9.3 GitLab Flow 工作流

```
production ─── A ─── B ─── C
              │
pre-production┴─── D ─── E
              │
main ─────────┴─── F ─── G ─── H
              │           │
feature ──────┴─── I ────┴───
```

### 9.4 提交规范

```bash
# 提交信息格式
<type>(<scope>): <subject>

<body>

<footer>

# 示例
feat(用户模块): 添加用户注册功能

- 实现邮箱验证
- 添加密码加密
- 添加注册日志

Closes #123

# 常用类型
feat:     新功能
fix:      修复bug
docs:     文档更新
style:    代码格式
refactor: 代码重构
perf:     性能优化
test:     测试相关
chore:    构建/工具
revert:   回滚
```

### 9.5 分支命名规范

```bash
# 功能分支
feature/user-login
feature/payment-integration

# 修复分支
bugfix/login-error
fix/null-pointer

# 发布分支
release/1.0.0
release/2026.03.08

# 热修复分支
hotfix/security-patch
hotfix/critical-bug

# 个人分支
dev/zhangsan/feature-name
```

---

## 十、常见问题与解决方案

### 10.1 提交相关

#### 10.1.1 修改最后一次提交信息

```bash
# 修改提交信息
git commit --amend -m "新的提交信息"

# 修改作者信息
git commit --amend --author="Name <email@example.com>"

# 添加遗漏的文件到最后一次提交
git add forgotten-file.txt
git commit --amend --no-edit
```

#### 10.1.2 撤销已推送的提交

```bash
# 使用 revert（推荐）
git revert commit_hash
git push origin main

# 强制回退（不推荐，仅适用于个人分支）
git reset --hard HEAD~1
git push -f origin main
```

#### 10.1.3 合并多个提交

```bash
# 交互式变基合并最近3个提交
git rebase -i HEAD~3

# 在编辑器中将 pick 改为 squash
# pick commit1
# squash commit2
# squash commit3
```

### 10.2 分支相关

#### 10.2.1 恢复误删分支

```bash
# 查看操作历史
git reflog

# 找到分支删除前的提交
git checkout -b recovered-branch commit_hash
```

#### 10.2.2 分支未合并但需要删除

```bash
# 强制删除
git branch -D feature-branch

# 先查看未合并的更改
git diff main..feature-branch
```

#### 10.2.3 解决分支分叉

```bash
# 使用变基
git checkout feature-branch
git rebase main

# 或使用合并
git checkout feature-branch
git merge main
```

### 10.3 远程相关

#### 10.3.1 推送被拒绝

```bash
# 原因：远程有新提交
# 解决方案一：拉取后合并
git pull origin main
git push origin main

# 解决方案二：拉取后变基
git pull --rebase origin main
git push origin main

# 解决方案三：强制推送（谨慎使用）
git push -f origin main
git push --force-with-lease origin main
```

#### 10.3.2 清理远程已删除的分支

```bash
# 清理本地引用
git fetch -p
git remote prune origin

# 查看可清理的分支
git remote prune origin --dry-run
```

#### 10.3.3 多个远程仓库同步

```bash
# 添加多个远程仓库
git remote add github https://github.com/user/project.git
git remote add gitee https://gitee.com/user/project.git

# 同时推送到多个仓库
git push github main
git push gitee main

# 或设置同一个远程仓库多个URL
git remote set-url --add --push origin https://github.com/user/project.git
git remote set-url --add --push origin https://gitee.com/user/project.git
git push origin main
```

### 10.4 文件相关

#### 10.4.1 追踪已忽略的文件

```bash
# 强制添加
git add -f ignored-file.txt

# 或修改 .gitignore
```

#### 10.4.2 停止追踪但保留文件

```bash
# 从暂存区移除
git rm --cached filename.txt

# 提交更改
git commit -m "停止追踪 filename.txt"
```

#### 10.4.3 恢复单个文件的历史版本

```bash
# 查看文件历史
git log -- filename.txt

# 恢复到指定版本
git checkout commit_hash -- filename.txt
git restore --source=commit_hash filename.txt
```

### 10.5 性能相关

#### 10.5.1 仓库体积过大

```bash
# 查看仓库大小
du -sh .git

# 查看大文件
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  awk '/^blob/ {print substr($0,6)}' | \
  sort -n -k2 | \
  tail -20

# 清理大文件（使用 git-filter-repo）
pip install git-filter-repo
git filter-repo --path large-file.zip --invert-paths

# 清理后压缩仓库
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

#### 10.5.2 加速克隆

```bash
# 浅克隆
git clone --depth 1 https://github.com/user/project.git

# 单分支克隆
git clone --single-branch --branch main https://github.com/user/project.git

# 后续获取完整历史
git fetch --unshallow
```

### 10.6 其他问题

#### 10.6.1 中文文件名乱码

```bash
git config --global core.quotepath false
```

#### 10.6.2 换行符问题

```bash
# Windows
git config --global core.autocrlf true

# Linux/macOS
git config --global core.autocrlf input

# 转换文件换行符
git add --renormalize .
```

#### 10.6.3 权限问题

```bash
# 忽略文件权限变化
git config core.fileMode false

# 恢复权限变化
git diff --summary | grep --color 'mode change'
```

#### 10.6.4 子模块问题

```bash
# 添加子模块
git submodule add https://github.com/user/lib.git libs/lib

# 初始化子模块
git submodule init
git submodule update

# 克隆时自动初始化子模块
git clone --recursive https://github.com/user/project.git

# 更新子模块
git submodule update --remote

# 删除子模块
git submodule deinit libs/lib
git rm libs/lib
rm -rf .git/modules/libs/lib
```

---

**修订记录**

| 版本 | 日期 | 作者 | 修改内容 |
|------|------|------|----------|
| v1.0 | 2026-03-08 | 运维团队 | 初始版本 |
