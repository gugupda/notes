# Git 详细使用指南

## 1 Git 基础概念

Git 是一个分布式版本控制系统，用于跟踪文件的变化。

### 核心概念

- **仓库 (Repository)**: 存储项目文件和版本历史的地方
- **提交 (Commit)**: 保存对文件的修改
- **分支 (Branch)**: 独立的开发线
- **合并 (Merge)**: 将不同分支的修改整合到一起
- **远程仓库 (Remote)**: 存储在网络上的仓库

## 2 Git 安装与配置

### 安装 Git

```bash
# Ubuntu/Debian
sudo apt-get install git

# macOS (使用 Homebrew)
brew install git

# Windows (下载安装包)
# https://git-scm.com/download/win
```

### 配置 Git

```bash
# 设置用户名
git config --global user.name "Your Name"

# 设置邮箱
git config --global user.email "your.email@example.com"

# 查看配置
git config --list

# 设置默认编辑器
git config --global core.editor "code --wait"  # VS Code
```

## 3 基本操作命令

### 初始化仓库

```bash
# 在当前目录初始化
git init

# 克隆远程仓库
git clone https://github.com/username/repository.git

# 克隆到指定目录
git clone https://github.com/username/repository.git my-project
```

### 查看状态

```bash
# 查看当前状态
git status

# 简洁输出
git status -s
```

### 添加文件到暂存区

```bash
# 添加单个文件
git add filename.txt

# 添加所有修改的文件
git add .

# 添加特定类型的文件
git add *.txt

# 交互式添加
git add -i
```

### 提交修改

```bash
# 基本提交
git commit -m "Initial commit"

# 提交所有修改的文件（跳过暂存区）
git commit -am "Update files"

# 修改上次提交信息
git commit --amend -m "Updated commit message"

# 添加新文件到上次提交
git commit --amend --no-edit
```

### 查看历史

```bash
# 查看提交历史
git log

# 简洁输出
git log --oneline

# 查看指定数量的提交
git log -n 5

# 查看包含修改内容的历史
git log -p

# 查看文件的修改历史
git log -- filename.txt

# 图形化显示分支关系
git log --graph --oneline --all
```

## 4 分支管理

### 查看分支

```bash
# 查看本地分支
git branch

# 查看所有分支（包括远程）
git branch -a

# 查看分支详细信息
git branch -v
```

### 创建和切换分支

```bash
# 创建分支
git branch feature-branch

# 切换分支
git checkout feature-branch

# 创建并切换分支
git checkout -b feature-branch

# 使用新语法
git switch feature-branch
git switch -c new-branch  # 创建并切换
```

### 合并分支

```bash
# 切换到目标分支
git checkout main

# 合并其他分支
git merge feature-branch

# 处理合并冲突后
git add conflicted-file.txt
git commit -m "Resolve merge conflicts"
```

### 删除分支

```bash
# 删除已合并的分支
git branch -d feature-branch

# 强制删除分支
git branch -D feature-branch

# 删除远程分支
git push origin --delete feature-branch
```

## 5 远程仓库操作

### 添加远程仓库

```bash
# 添加远程仓库
git remote add origin https://github.com/username/repository.git

# 查看远程仓库
git remote -v

# 修改远程仓库URL
git remote set-url origin https://github.com/username/new-repo.git
```

### 推送和拉取

```bash
# 推送到远程仓库
git push origin main

# 推送到指定分支
git push origin feature-branch

# 推送所有分支
git push --all origin

# 拉取更新
git pull origin main

# 仅获取远程更新（不合并）
git fetch origin
```

### 分支跟踪

```bash
# 设置本地分支跟踪远程分支
git branch --set-upstream-to=origin/main main

# 推送并设置上游分支
git push -u origin feature-branch
```

## 6 高级操作

### 暂存工作区

```bash
# 暂存当前修改
git stash

# 查看暂存列表
git stash list

# 恢复最近的暂存
git stash apply

# 恢复并删除暂存
git stash pop

# 删除暂存
git stash drop
git stash clear  # 清除所有暂存
```

### 重置操作

```bash
# 软重置（保留工作区和暂存区）
git reset --soft HEAD~1

# 混合重置（保留工作区，清空暂存区）
git reset --mixed HEAD~1
git reset HEAD~1  # 默认是 --mixed

# 硬重置（清空工作区和暂存区）
git reset --hard HEAD~1

# 重置到特定提交
git reset --hard commit-hash
```

### 标签管理

```bash
# 创建轻量级标签
git tag v10

# 创建带注释的标签
git tag -a v10 -m "Version 10 release"

# 查看标签
git tag

# 推送标签
git push origin v10
git push origin --tags

# 删除标签
git tag -d v10
git push origin :refs/tags/v10
```

### 查看差异

```bash
# 查看工作区与暂存区的差异
git diff

# 查看暂存区与最近提交的差异
git diff --staged

# 查看两个提交之间的差异
git diff commit1 commit2

# 查看两个分支之间的差异
git diff branch1 branch2

# 仅显示文件名
git diff --name-only
```

## 7 实际使用场景示例

### 场景 1：日常开发流程

```bash
# 1 拉取最新代码
git pull origin main

# 2 创建新功能分支
git checkout -b feature-login

# 3 开发过程中
git add .
git commit -m "Add login form"
git add .
git commit -m "Implement login validation"

# 4 推送到远程
git push -u origin feature-login

# 5 完成开发后合并
git checkout main
git pull origin main
git merge feature-login
git push origin main

# 6 删除分支
git branch -d feature-login
git push origin --delete feature-login
```

### 场景 2：处理合并冲突

```bash
# 冲突发生时
git status  # 查看冲突文件

# 手动编辑冲突文件，解决冲突后
git add conflicted-file.txt
git commit -m "Resolve merge conflicts"
```

### 场景 3：撤销修改

```bash
# 撤销工作区的修改
git checkout -- filename.txt

# 撤销暂存区的修改
git reset HEAD filename.txt

# 撤销提交（但保留修改）
git reset --soft HEAD~1

# 完全撤销提交
git reset --hard HEAD~1
```

## 8 实用技巧

### .gitignore 文件

```bash
# 创建 .gitignore 文件
touch .gitignore

# 常用忽略规则
# .gitignore 内容示例
*.log
*.tmp
node_modules/
dist/
.env
.DS_Store
```

### 别名设置

```bash
# 设置常用命令别名
git config --global alias.st status
git config --global aliasci commit
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --all"
```

### 查看文件内容

```bash
# 查看文件的特定版本
git show commit-hash:filename.txt

# 查看文件的历史修改
git blame filename.txt
```

### 查找和修复

```bash
# 查找包含特定内容的提交
git log -S "search text"

# 查找特定作者的提交
git log --author="Your Name"

# 查找特定日期范围的提交
git log --since="2024-01-01" --until="2024-01-31"
```

## 9 常见问题解决

### 撤销错误的推送

```bash
# 本地重置
git reset --hard HEAD~1

# 强制推送到远程
git push -f origin main
```

### 恢复删除的文件

```bash
# 恢复删除的文件
git checkout HEAD -- deleted-file.txt
```

### 清理未跟踪的文件

```bash
# 查看未跟踪的文件
git clean -n

# 删除未跟踪的文件
git clean -f

# 删除未跟踪的文件和目录
git clean -fd
```

Git 的强大之处在于它的分布式特性和灵活的分支管理，掌握这些命令将大大提高你的开发效率！





