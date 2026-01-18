# Git 的工作区、暂存区和版本库详解

Git 有三个核心区域，它们共同构成了 Git 的工作流程：

## 1 工作区 (Working Directory)

**工作区**是你当前看到和编辑文件的地方，也就是项目的实际文件目录。

### 特点：

- 包含项目的实际文件
- 可以直接编辑修改
- 未被 Git 跟踪的修改都在这里
- 相当于 "草稿纸"

### 相关命令：

```bash
# 查看工作区状态
git status

# 查看工作区与暂存区的差异
git diff

# 放弃工作区的修改
git checkout -- filename.txt
git restore filename.txt  # Git 223+ 新语法
```

## 2 暂存区 (Staging Area/Index)

**暂存区**是 Git 中一个特殊的区域，用于临时保存你的修改，相当于一个 "待提交清单"。

### 特点：

- 存储在 `.git/index` 文件中
- 保存了下次要提交的文件快照
- 可以选择性地将工作区修改添加到暂存区
- 相当于 "购物车"

### 相关命令：

```bash
# 添加文件到暂存区
git add filename.txt
git add .  # 添加所有修改
git add *.txt  # 添加特定类型文件

# 从暂存区移除文件
git reset HEAD filename.txt
git restore --staged filename.txt  # Git 223+ 新语法

# 查看暂存区与版本库的差异
git diff --staged
git diff --cached

# 交互式添加
git add -i
```

## 3 版本库 (Repository)

**版本库**包含了项目的完整历史记录，存储在 `.git` 目录中。

### 特点：

- 存储在 `.git` 目录中
- 包含所有提交的历史记录
- 存储了所有文件的完整版本
- 包含分支、标签等引用
- 相当于 "仓库"

### 相关命令：

```bash
# 提交到版本库
git commit -m "commit message"

# 查看提交历史
git log

# 查看版本库中的文件内容
git show HEAD:filename.txt

# 回滚到之前的版本
git reset --hard commit-hash
```

## 4 工作流程演示

```bash
# 1 在工作区修改文件
echo "New content" >> file.txt

# 2 查看状态
git status
# 显示：Changes not staged for commit

# 3 添加到暂存区
git add file.txt

# 4 再次查看状态
git status
# 显示：Changes to be committed

# 5 提交到版本库
git commit -m "Add new content"

# 6 最终状态
git status
# 显示：nothing to commit, working tree clean
```

## 5 三个区域的关系

```plaintext
┌─────────────────┐    git add    ┌─────────────────┐    git commit    ┌─────────────────┐
│                 │  ───────────> │                 │  ────────────>  │                 │
│  工作区         │                │  暂存区         │                 │  版本库         │
│ (Working Dir)   │  <─────────── │ (Staging Area)  │  <───────────   │ (Repository)    │
│                 │   git restore │                 │   git reset     │                 │
└─────────────────┘               └─────────────────┘                 └─────────────────┘
```

## 6 实际操作示例

### 示例 1：基本工作流程

```bash
# 创建文件
echo "Hello Git" > hello.txt

# 工作区 -> 暂存区
git add hello.txt

# 暂存区 -> 版本库
git commit -m "Add hello.txt"
```

### 示例 2：修改文件

```bash
# 修改文件（工作区）
echo "Hello World" >> hello.txt

# 查看差异
git diff

# 添加到暂存区
git add hello.txt

# 查看暂存区差异
git diff --staged

# 提交到版本库
git commit -m "Update hello.txt"
```

### 示例 3：撤销操作

```bash
# 撤销工作区修改
git checkout -- hello.txt
# 或
git restore hello.txt

# 撤销暂存区修改
git reset HEAD hello.txt
# 或
git restore --staged hello.txt

# 撤销提交（软重置）
git reset --soft HEAD~1
```

## 7 深入理解

### 暂存区的作用：

- **选择性提交**：可以只提交部分修改
- **变更预览**：在提交前确认要包含的修改
- **分阶段提交**：将复杂的修改分解为多个小的提交
- **冲突处理**：在合并时帮助处理冲突

### 版本库的内部结构：

- **objects/**：存储所有 Git 对象（blob、tree、commit）
- **refs/**：存储分支和标签的引用
- **HEAD**：指向当前所在分支的引用

通过理解这三个区域的概念和关系，你可以更好地掌握 Git 的工作原理，从而更高效地使用 Git 进行版本控制。