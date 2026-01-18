# 添加远程仓库完整演示

### 步骤 1：创建本地仓库

```bash
# 创建项目目录
mkdir git-demo
cd git-demo

# 初始化Git仓库
git init

# 配置用户信息
git config user.name "Demo User"
git config user.email "demo@example.com"

# 创建初始文件
echo "Initial content" > README.md
git add README.md
git commit -m "Initial commit"
```

### 步骤 2：添加远程仓库

```bash
# 添加远程仓库（使用示例URL）
git remote add origin https://github.com/username/repository.git

# 查看远程仓库配置
git remote -v
```

### 步骤 3：实际使用示例

#### 情况 1：使用真实的 GitHub 仓库

```bash
# 1 首先在GitHub上创建一个空仓库
# 2 复制仓库URL
# 3 添加远程仓库
git remote add origin https://github.com/yourusername/your-repo.git

# 4 推送代码
git push -u origin master
```

#### 情况 2：修改远程仓库 URL

```bash
# 查看当前远程仓库
git remote -v

# 修改远程仓库URL
git remote set-url origin https://github.com/newusername/new-repo.git

# 确认修改
git remote -v
```

#### 情况 3：添加多个远程仓库

```bash
# 添加主要远程仓库
git remote add origin https://github.com/username/main-repo.git

# 添加备用远程仓库
git remote add backup https://github.com/username/backup-repo.git

# 查看所有远程仓库
git remote -v

# 推送到不同的远程仓库
git push origin master
git push backup master
```

### 常见操作示例

```bash
# 从远程仓库拉取代码
git pull origin master

# 获取远程更新（不合并）
git fetch origin

# 推送代码到远程仓库
git push origin master

# 推送所有分支
git push --all origin

# 推送标签
git push --tags origin
```

### 错误处理

```bash
# 常见错误：远程仓库不存在
# 错误信息：fatal: repository 'https://github.com/username/repository.git/' not found

# 解决方案：
# 1 检查URL是否正确
# 2 确保远程仓库已创建
# 3 检查网络连接

# 常见错误：权限问题
# 错误信息：remote: Permission to username/repository.git denied to yourusername

# 解决方案：
# 1 确保你有仓库的访问权限
# 2 检查SSH密钥或HTTPS凭证是否正确配置
```

### 查看远程分支

```bash
# 查看远程分支
git branch -r

# 查看所有分支（本地+远程）
git branch -a

# 跟踪远程分支
git checkout -b local-branch origin/remote-branch
```

这个演示展示了添加远程仓库的完整流程，包括创建本地仓库、添加远程仓库、推送代码以及常见的远程仓库操作。

