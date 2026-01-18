# Git Stash 命令详解

Git stash 是 Git 中一个非常实用的功能，用于临时保存工作区和暂存区的修改，让你可以在不提交当前工作的情况下切换到其他分支或执行其他操作。

## 1 Git Stash 的基本概念

**什么是 Stash？**

- Stash 是一个临时存储区域
- 用于保存未完成的工作
- 可以随时恢复这些修改
- 不会创建新的提交

**使用场景：**

- 需要切换分支但当前工作未完成
- 需要紧急修复 bug 但不想提交当前修改
- 临时需要拉取或合并其他代码
- 想清理工作区但不想丢失当前修改

## 2 常用 Stash 命令

### 2.1 基本命令

```bash
# 创建 stash
git stash

# 创建带消息的 stash
git stash save "WIP: 功能开发中"

# 查看 stash 列表
git stash list

# 恢复最近的 stash（不删除）
git stash apply

# 恢复特定的 stash
git stash apply stash@{1}

# 恢复最近的 stash 并删除
git stash pop

# 恢复特定的 stash 并删除
git stash pop stash@{1}

# 查看 stash 的内容
git stash show

# 查看 stash 的详细差异
git stash show -p
git stash show stash@{1} -p
```

### 2.2 高级命令

```bash
# 包含未跟踪的文件
git stash -u
git stash --include-untracked

# 包含忽略的文件
git stash -a
git stash --all

# 创建新分支并恢复 stash
git stash branch new-branch

# 删除最近的 stash
git stash drop

# 删除特定的 stash
git stash drop stash@{1}

# 清空所有 stash
git stash clear

# 查看 stash 的具体内容
git stash show stash@{0}
```

## 3 实际操作演示

### 场景 1：基本使用流程

```bash
# 1 在工作区有修改
echo "正在开发新功能..." >> feature.txt
git status

# 2 创建 stash
git stash save "新功能开发中"

# 3 工作区变干净
git status

# 4 切换到其他分支处理紧急任务
git checkout hotfix
# ... 处理紧急任务 ...

# 5 切回原分支
git checkout feature-branch

# 6 恢复 stash
git stash pop

# 7 继续开发
git add feature.txt
git commit -m "完成新功能"
```

### 场景 2：多个 stash 管理

```bash
# 创建第一个 stash
git stash save "UI 优化"

# 继续开发并创建第二个 stash
git stash save "API 集成"

# 查看所有 stash
git stash list
# stash@{0}: On feature: API 集成
# stash@{1}: On feature: UI 优化

# 恢复特定的 stash
git stash apply stash@{1}  # 恢复 UI 优化

# 完成 UI 优化后提交
git add .
git commit -m "完成 UI 优化"

# 恢复 API 集成
git stash pop  # 默认恢复最近的
```

### 场景 3：包含未跟踪文件

```bash
# 创建新文件（未跟踪）
echo "新配置文件" > config.json

# 修改已跟踪的文件
echo "新功能" >> feature.txt

# 只 stash 已跟踪的修改
git stash
# config.json 仍然在工作区

# stash 所有修改（包括未跟踪）
git stash -u

# 查看 stash 内容
git stash show -p
```

## 4 Stash 的内部原理

### Stash 的存储结构

- Stash 实际上是一个特殊的提交对象
- 包含工作区和暂存区的状态
- 存储在 `.git/refs/stash` 引用中

### Stash 的内容

```bash
# 查看 stash 的详细信息
git stash list --pretty=format:"%h %s"

# 查看 stash 的完整对象信息
git log --oneline --graph stash@{0}
```

## 5 最佳实践

### 5.1 使用建议

1 **添加有意义的消息**

```bash
git stash save "用户认证模块开发中"
```

2 **及时清理不需要的 stash**

```bash
# 定期清理
git stash list
git stash drop stash@{2}
```

3 **避免长时间存放**

- Stash 适合短期存储
- 长期未完成的工作建议创建分支

4 **使用分支替代 stash**

```bash
# 对于长期工作，推荐使用分支
git checkout -b feature-in-progress
git add .
git commit -m "WIP: 功能开发中"
```

### 5.2 常见问题解决

**问题 1：Stash 冲突**

```bash
# 恢复 stash 时发生冲突
git stash apply

# 解决冲突
# 1 编辑冲突文件
# 2 git add 冲突文件
# 3 继续工作
```

**问题 2：找回删除的 stash**

```bash
# 如果不小心删除了 stash，可以尝试找回
git fsck --no-reflog | awk '/dangling commit/ {print $3}'
git show <commit-hash>
```

**问题 3：stash 太大**

```bash
# 避免 stash 大型二进制文件
# 使用 .gitignore 忽略不需要版本控制的文件
```

## 6 高级技巧

### 6.1 自定义 stash 行为

```bash
# 设置 stash 的默认行为
git config --global stashincludeuntracked true

# 配置 stash 消息格式
git config --global alias.stash-save '!git stash save "$(date +\"%Y-%m-%d %H:%M:%S\") - $1"'
```

### 6.2 与其他 Git 命令结合

```bash
# stash 后立即切换分支
git stash && git checkout other-branch

# stash 后拉取更新
git stash && git pull && git stash pop

# 清理工作区但保留 stash
git checkout . && git clean -fd && git stash apply
```

Git stash 是一个强大的工具，可以帮助你在不中断当前工作的情况下处理其他任务。合理使用 stash 可以让你的 Git 工作流程更加灵活和高效。