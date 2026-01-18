# Git Stash 详细解析

`git stash` 是 Git 中用于**临时保存工作区和暂存区未提交修改**的核心命令，适用于需要切换分支、拉取远程代码、修复紧急 Bug 等场景 —— 既不想提交不完整的代码，又要清空工作区 / 暂存区时，`stash` 是最优选择。

### 一、核心概念

- **Stash 本质**：将工作区（已修改但未暂存）和暂存区（已暂存但未提交）的修改保存到一个**临时存储区**（栈结构），同时将工作区 / 暂存区恢复到当前 HEAD 提交的干净状态。
- **Stash 内容**：默认包含：
  - 工作区的修改（未 `git add` 的文件）；
  - 暂存区的修改（已 `git add` 的文件）；
  - 不包含：未被 Git 追踪的新文件（需显式指定）、被忽略的文件（`.gitignore` 中的）。
- **栈结构**：多次 `stash` 会按顺序压栈，默认取出最新的 stash（后进先出）。

### 二、基本用法

#### 1. 保存当前修改（基础）

```bash
git stash
# 等价于 git stash push
```

- 执行后，工作区 / 暂存区恢复为干净状态，修改被保存到 stash 栈；

- 若需要给 stash 加备注（便于区分），推荐：

  ```bash
  git stash push -m "修复订单模块 Bug 未完成"  # 备注式保存，推荐
  ```

#### 2. 查看 stash 列表

```bash
git stash list
```

- 输出示例：

  ```plaintext
  stash@{0}: On feature/user: 修复订单模块 Bug 未完成
  stash@{1}: On master: 临时调整配置文件
  ```

  - `stash@{n}` 是 stash 的唯一标识（n 从 0 开始，0 是最新的）；
  - 备注会显示在列表中，便于识别用途。

#### 3. 恢复 stash 内容

恢复分为两种方式：**保留 stash 记录** 和 **删除 stash 记录**。

##### 方式 1：恢复最新 stash（保留记录）

```bash
git stash apply  # 恢复最新的 stash@{0}
# 恢复指定 stash
git stash apply stash@{1}
```

- 恢复后，stash 仍保留在栈中，可重复恢复；
- 若恢复时存在冲突（如当前工作区已有同名文件修改），Git 会提示冲突，需手动解决。

##### 方式 2：恢复最新 stash（删除记录）

```bash
git stash pop  # 恢复并删除最新的 stash@{0}
# 恢复并删除指定 stash
git stash pop stash@{1}
```

- 推荐场景：确认恢复后不再需要该 stash，避免栈中堆积无用记录。

#### 4. 删除 stash 记录

##### 删除指定 stash

```bash
git stash drop stash@{1}
```

##### 删除所有 stash

```bash
git stash clear
```

- 谨慎使用 `clear`，删除后无法恢复。

### 三、进阶用法

#### 1. 保存未追踪的新文件

默认 `git stash` 不保存未被 Git 追踪的新文件（即从未执行过 `git add` 的文件），需加 `-u`/`--include-untracked`：

```bash
git stash push -u -m "包含新文件的临时修改"
```

- 若要保存被忽略的文件（如 `node_modules`、`dist`），需加 `-a`/`--all`：

  ```bash
  git stash push -a -m "包含忽略文件的临时修改"
  ```

#### 2. 仅保存暂存区 / 仅保存工作区

- 仅保存暂存区（工作区修改保留）：

  ```bash
  git stash push --staged  # 等价于 --index
  ```

- 仅保存工作区（暂存区修改保留）：无直接命令，可先 `git stash push` 再 `git reset HEAD` 恢复暂存区。

#### 3. 查看 stash 内容详情

```bash
# 查看最新 stash 的修改内容
git stash show -p
# 查看指定 stash 的修改内容
git stash show -p stash@{1}
```

- `-p`（`--patch`）：显示具体的代码修改（diff），不加则仅显示文件列表。

#### 4. 从 stash 创建分支

若 stash 的修改与当前分支冲突，或需要基于 stash 单独开发，可直接创建分支：

```bash
git stash branch <分支名> [stash标识]
# 示例：基于 stash@{1} 创建分支 fix-stash
git stash branch fix-stash stash@{1}
```

- 执行后，会切换到新分支，并自动应用该 stash，应用成功后删除该 stash 记录。

#### 5. 部分应用 stash（Git 2.23+）

若只需恢复 stash 中的部分文件 / 部分修改，可使用 `git stash apply --patch`（交互式选择）：

```bash
git stash apply --patch stash@{0}
```

- 执行后，Git 会逐块显示 stash 中的修改，让你选择是否应用（`y`/`n`/`s`/`q` 等）。

### 四、工作原理（补充）

- Stash 数据存储在本地仓库的 `.git/refs/stash` 和 `.git/objects` 目录中，是一个**提交对象（commit）**，包含：
  1. 工作区的修改（对应 `HEAD` 的 diff）；
  2. 暂存区的修改（对应 index 的 diff）；
  3. 原始 HEAD 提交的引用。
- Stash 是本地操作，不会推送到远程仓库，需手动备份或提交到远程分支。

### 五、常见场景与最佳实践

#### 场景 1：临时切换分支

```bash
# 正在 feature 分支开发，需切换到 master 修复 Bug
git stash push -m "feature: 未完成的支付功能"
git checkout master
# 修复 Bug 后切回 feature，恢复修改
git checkout feature
git stash pop
```

#### 场景 2：拉取远程代码前清理工作区

```bash
# 本地有未提交修改，拉取远程代码可能冲突
git stash push -u
git pull origin master
git stash pop  # 拉取后恢复修改，解决冲突（若有）
```

#### 场景 3：放弃未完成的修改

```bash
# 临时修改后不想保留，直接清空工作区
git stash push  # 先保存（防止误删）
git stash drop  # 确认不需要后删除 stash
# 或直接 git checkout . && git reset HEAD（但 stash 更安全）
```

#### 最佳实践：

1. **必加备注**：使用 `-m` 标注 stash 的用途（分支 + 功能），避免后续无法识别；
2. **及时清理**：定期执行 `git stash list` 清理无用的 stash（`drop`/`clear`）；
3. **避免滥用**：stash 仅用于临时保存，长期未完成的修改建议创建「草稿分支」（如 `feature/draft-xxx`）提交，而非依赖 stash；
4. **冲突处理**：恢复 stash 冲突时，先解决冲突，再 `git add` 标记为已解决，无需执行 `git commit`（stash 恢复不是提交操作）。

### 六、常见问题

1. **恢复 stash 提示冲突**：
   - 解决：手动编辑冲突文件（标记 `<<<<<<<`/`=======`/`>>>>>>>` 的部分），然后 `git add` 冲突文件，无需 commit；
2. **误删 stash 能否恢复**：
   - 若未执行 `git gc`（垃圾回收），可通过 `git fsck --no-reflogs` 查找 stash 的 commit ID，再 `git stash apply <commit ID>` 恢复；
3. **stash 列表乱码**：
   - 解决：设置 Git 字符编码 `git config --global core.quotepath false`。

### 总结

`git stash` 是 Git 处理临时修改的核心工具，核心价值是「临时保存、干净切换、按需恢复」。掌握其基础用法（`push`/`list`/`apply`/`pop`/`drop`）和进阶技巧（分支创建、部分应用），能大幅提升多场景下的开发效率，同时注意规范使用备注和及时清理，避免 stash 堆积导致管理混乱。

