# GitHub Repository 全面管理操作详解

从**仓库创建、基础配置、代码提交与同步、分支管理、协作管控、仓库维护、高级操作**等核心维度展开，详细了解 GitHub 仓库（Repository，简称 Repo）的管理操作，覆盖命令行（Git）和 GitHub 网页端两种操作方式。

## 一、 仓库创建（Repository Creation）

仓库是存放项目代码、资源文件和版本记录的核心容器，分为两种创建方式：

### 1. 网页端创建（直观便捷，推荐新手）

1. 登录 GitHub 账号，点击右上角「+」号，选择「New repository」。
2. 填写核心配置项：
   - **Repository name**：仓库名称（唯一标识，如 `my-project-demo`，不能包含特殊字符）。
   - **Description**：仓库描述（可选，简要说明项目用途，便于他人理解）。
   - **Visibility**：仓库可见性（核心选项）：
     - Public：公共仓库，所有人可见，开源项目首选（免费）。
     - Private：私有仓库，仅你和授权协作者可见（个人版 / 团队版支持，部分需付费）。
   - **Initialize this repository with**：初始化配置（可选，推荐新手勾选）：
     - Add a README file：自动生成 `README.md`（项目说明文档，必备）。
     - Add .gitignore：选择项目对应的语言 / 框架（如 Python、Java），自动生成忽略文件（排除无需版本控制的文件，如日志、编译产物）。
     - Choose a license：选择开源许可证（如 MIT、Apache 2.0，规定他人使用项目的权限）。
3. 点击「Create repository」完成创建。

### 2. 本地创建后推送到 GitHub（适用于已有本地项目）

1. 本地终端进入项目目录：`cd /path/to/your/local/project`。

2. 初始化本地 Git 仓库：`git init`（生成隐藏的 `.git` 目录，用于存储版本信息）。

3. 关联远程 GitHub 仓库（需先在 GitHub 网页端创建**空仓库**，无初始化文件）：

   ```bash
   # 替换 <username> 为你的 GitHub 用户名，<repo-name> 为仓库名称
   git remote add origin https://github.com/<username>/<repo-name>.git
   # 示例：git remote add origin https://github.com/zhangsan/my-local-project.git
   ```

4. 后续通过提交和推送操作，将本地代码同步到远程仓库（见下文「代码提交与同步」）。

## 二、 仓库基础配置（Core Configuration）

创建后需完善基础配置，保障仓库可用性和规范性：

### 1. 核心文件维护

- **README.md**：项目门面，需包含项目介绍、安装步骤、使用方法、贡献指南等，支持 Markdown 语法，可在网页端直接编辑（点击文件右上角「✏️」）或本地编辑后提交。
- **.gitignore**：自定义忽略规则，如需新增忽略文件（如 `node_modules/`、`*.log`），可直接编辑该文件，规则支持通配符（`*` 匹配所有、`/` 匹配目录、`!` 排除指定文件）。
- **LICENSE**：若未在创建时勾选，可后续手动添加（网页端点击「Add file」→「Create new file」，文件名填写 `LICENSE`，GitHub 会提供许可证模板选择）。

### 2. 仓库信息修改（网页端）

进入仓库主页，点击「Settings」（设置），在「General」标签下可修改：

- 仓库名称、描述、可见性（公共仓库转私有需满足账号权限，部分版本需付费）。
- 仓库默认分支（默认 `main`，原 `master`，可自定义）。
- 归档仓库（Archive this repository）：标记仓库不再维护，只读状态，不可修改。

### 3. 远程仓库关联管理（命令行）

```bash
# 查看当前关联的远程仓库
git remote -v
# 移除已关联的远程仓库
git remote remove origin
# 修改远程仓库地址（如更换仓库域名、仓库名称）
git remote set-url origin https://github.com/<new-username>/<new-repo-name>.git
```

## 三、 代码提交与同步（Code Commit & Synchronization）

这是仓库管理的核心日常操作，实现本地代码与远程仓库的版本同步，遵循「本地暂存 → 本地提交 → 远程推送」流程，拉取远程更新则为「远程拉取 → 本地合并」。

### 1. 本地代码提交（Git 命令行）

```bash
# 1. 查看本地文件状态（已修改、未跟踪、已暂存的文件）
git status
# 2. 将指定文件添加到暂存区（跟踪文件变更）
git add <file1> <file2>
# 或添加所有修改/未跟踪的文件（除 .gitignore 忽略的文件）
git add .
# 3. 将暂存区文件提交到本地仓库，填写提交说明（必填，清晰描述变更内容）
git commit -m "feat: 新增用户登录功能 | fix: 修复订单查询bug"
# 若提交后需修改提交说明（未推送到远程时）
git commit --amend -m "新的提交说明"
```

### 2. 本地代码推送到远程仓库

```bash
# 首次推送（需指定分支，关联本地分支与远程分支）
git push -u origin <branch-name>
# 示例：首次推送 main 分支
git push -u origin main
# 非首次推送（已关联分支，直接推送）
git push
# 强制推送（谨慎使用！会覆盖远程仓库对应分支的历史版本，仅个人分支或确需覆盖时使用）
git push -f origin <branch-name>
```

### 3. 拉取远程仓库更新

```bash
# 方式1：拉取远程更新并自动合并到本地当前分支（常用）
git pull origin <branch-name>
# 示例：拉取远程 main 分支更新
git pull origin main
# 方式2：先拉取远程更新（不合并），再手动合并（灵活度更高，便于处理冲突）
git fetch origin
git merge origin/<branch-name>
```

### 4. 网页端直接编辑 / 提交

无需本地环境时，可在 GitHub 仓库主页直接编辑文件：

1. 点击目标文件，右上角点击「✏️」进入编辑模式。
2. 修改完成后，下拉到页面底部，填写「Commit changes」信息：
   - 提交说明（Commit message）。
   - 可选填写详细描述（Extended description）。
   - 选择提交到当前分支或新建分支。
3. 点击「Commit changes」完成提交（直接同步到远程仓库）。

## 四、 分支管理（Branch Management）

分支是 Git 的核心特性，用于实现并行开发（如功能开发、bug 修复、发布准备），避免直接修改主分支（`main`/`master`）的稳定代码，核心流程为「创建分支 → 分支开发 → 合并分支 → 删除无用分支」。

### 1. 分支核心操作（命令行）

```bash
# 1. 查看所有分支（本地+远程，* 标记当前所在分支）
git branch -a
# 2. 创建本地分支
git branch <new-branch-name>
# 示例：创建功能分支 feat-user
git branch feat-user
# 3. 切换到指定分支
git checkout <branch-name>
# 4. 创建并切换到新分支（常用，一步到位）
git checkout -b <new-branch-name>
# 示例：创建并切换到 fix-order 分支
git checkout -b fix-order
# 5. 推送本地分支到远程仓库（创建远程对应分支）
git push origin <branch-name>
# 6. 删除本地无用分支（已合并到主分支的分支）
git branch -d <branch-name>
# 强制删除本地分支（未合并的分支，需确认无用）
git branch -D <branch-name>
# 7. 删除远程无用分支
git push origin --delete <branch-name>
# 示例：删除远程 feat-user 分支
git push origin --delete feat-user
```

### 2. 分支合并（两种核心方式）

#### （1） 命令行合并（本地合并后推送）

```bash
# 1. 先切换到目标分支（如合并到 main 分支，先切到 main）
git checkout main
# 2. 拉取 main 分支最新远程更新（避免冲突）
git pull origin main
# 3. 合并指定分支到当前分支（如合并 feat-user 到 main）
git merge feat-user
# 4. 若合并时出现冲突，手动编辑冲突文件（标记 <<<<<<< HEAD 到 >>>>>>> feat-user 之间为冲突内容）
#    解决冲突后，重新暂存并提交
git add .
git commit -m "merge: 合并 feat-user 分支到 main"
# 5. 推送合并后的更新到远程 main 分支
git push origin main
```

#### （2） 网页端 Pull Request（PR）/ Merge Request（MR）（协作开发首选）

这是团队协作的核心流程，便于代码审查和版本管控：

1. 开发者推送功能分支到远程后，在 GitHub 仓库主页点击「Pull requests」→「New pull request」。
2. 选择「base repository」（目标仓库）、「base」（目标分支，如 main）和「compare」（待合并的功能分支，如 feat-user）。
3. 填写 PR 标题和描述，说明分支变更内容、关联的任务 / BUG 等。
4. 点击「Create pull request」提交 PR。
5. 团队成员可在 PR 页面进行代码审查（Comment 评论、Review 批准 / 驳回）。
6. 若无冲突，点击「Merge pull request」选择合并方式（Merge commit/ Squash and merge/ Rebase and merge），完成合并。
7. 合并后可勾选「Delete branch」直接删除远程功能分支，本地分支手动删除即可。

## 五、 仓库协作管控（Collaboration & Permission Control）

GitHub 仓库支持多用户协作，通过权限分配和保护规则，保障仓库安全性和规范性。

### 1. 协作者添加（网页端，适用于小型团队）

1. 仓库主页 →「Settings」→「Collaborators」→「Add people」。
2. 输入协作者的 GitHub 用户名 / 邮箱，点击搜索并选择目标用户。
3. 分配权限等级（核心权限）：
   - Read：只读权限，可克隆、查看仓库，无法修改。
   - Write：读写权限，可推送代码、创建分支、提交 PR，无法修改仓库设置。
   - Admin：管理员权限，拥有仓库所有操作权限（包括删除仓库、修改权限）。
   - Maintain：维护者权限，介于 Write 和 Admin 之间，可管理仓库设置但无法删除仓库。
4. 协作者接收邮件邀请并确认后，即可参与协作。

### 2. 团队管理（适用于大型团队 / 组织）

1. 先创建 GitHub 组织（Organization）：右上角「+」→「New organization」。
2. 组织内创建团队（Teams）：组织主页 →「Teams」→「New team」，添加团队成员。
3. 将仓库关联到组织，并为团队分配权限：仓库主页 →「Settings」→「Access」→「Manage access」→「Add people or team」，选择组织团队并分配权限。
4. 优势：批量管理成员权限，无需逐个添加，便于团队层级管控。

### 3. 分支保护规则（核心，保障主分支稳定）

针对核心分支（如 main），设置保护规则防止误操作：

1. 仓库主页 →「Settings」→「Branches」→「Branch protection rules」→「Add rule」。
2. 填写「Branch name pattern」（如 `main`，匹配主分支）。
3. 勾选核心保护选项：
   - Require a pull request before merging：必须通过 PR 才能合并，禁止直接推送主分支。
   - Require approvals：PR 合并前需要指定数量的协作者批准（如 1 人 / 2 人批准）。
   - Dismiss stale pull request approvals when new commits are pushed：当 PR 有新提交时，旧的批准自动失效，需重新审查。
   - Require status checks to pass before merging：要求 CI/CD 检查（如代码规范、单元测试）通过后才能合并。
   - Do not allow force pushes：禁止强制推送该分支，防止历史版本被覆盖。
   - Do not allow deletions：禁止删除该分支。
4. 点击「Create rule」生效。

## 六、 仓库维护（Repository Maintenance）

### 1. 仓库清理

- 清理无用分支：定期删除已合并的功能分支、废弃分支（本地 + 远程）。
- 清理大文件：若仓库包含大文件（如压缩包、视频），会导致克隆速度变慢，可使用 `git filter-repo` 工具移除历史中的大文件（谨慎操作，需全员同步）。
- 归档旧项目：对不再维护的项目，使用「Archive repository」功能标记为只读，便于后续查阅，不占用活跃仓库空间。

### 2. 版本标签（Tag）管理（用于发布版本）

标记重要的版本节点（如 v1.0.0、v1.1.0），便于回溯和发布：

```bash
# 1. 创建轻量标签（仅标记版本号，无附加信息）
git tag <tag-name>
# 示例：git tag v1.0.0
# 2. 创建附注标签（包含作者、日期、描述，推荐发布版本使用）
git tag -a <tag-name> -m "v1.0.0 正式版，支持用户管理和订单功能"
# 3. 查看所有标签
git tag
# 4. 推送标签到远程仓库
git push origin <tag-name>
# 推送所有本地标签到远程
git push origin --tags
# 5. 删除本地标签
git tag -d <tag-name>
# 6. 删除远程标签
git push origin --delete <tag-name>
# 7. 切换到指定标签版本（查看对应代码）
git checkout <tag-name>
```

### 3. 仓库备份

- 克隆备份：`git clone --mirror https://github.com/<username>/<repo-name>.git`（创建镜像克隆，包含所有分支和标签）。
- GitHub 自动备份：GitHub 会默认备份仓库数据，但建议定期手动备份重要项目。

## 七、 仓库高级操作

### 1. Fork 仓库（开源协作常用）

- 操作：在他人公共仓库主页，点击右上角「Fork」，将仓库复制到自己的 GitHub 账号下，拥有完全读写权限。

- 用途：为开源项目贡献代码时，先 Fork 到自己账号，在自己的仓库中开发，再通过 PR 向原仓库提交修改。

- 同步原仓库更新：Fork 后的仓库不会自动同步原仓库更新，需手动关联上游仓库：

  ```bash
  # 关联原仓库（命名为 upstream，区别于自己的 origin 仓库）
  git remote add upstream https://github.com/<original-username>/<original-repo-name>.git
  # 拉取原仓库最新更新
  git fetch upstream
  # 合并到本地 main 分支
  git merge upstream/main
  # 推送更新到自己的远程仓库
  git push origin main
  ```

### 2. 仓库转移（Transfer Repository）

将仓库转移到其他 GitHub 账号或组织，保留所有历史记录、分支、PR 等：

1. 仓库主页 →「Settings」→ 拉到页面底部「Danger Zone」→「Transfer」。
2. 输入目标账号 / 组织名称和当前仓库名称，确认转移（目标账号 / 组织需接收邀请）。

### 3. 删除仓库（谨慎操作！不可逆）

1. 仓库主页 →「Settings」→ 拉到页面底部「Danger Zone」→「Delete this repository」。
2. 输入仓库名称进行验证，确认删除（删除后无法恢复，需谨慎）。

## 总结

GitHub 仓库管理的核心要点可归纳为：

1. 基础流程：「创建仓库 → 配置基础文件 → 分支开发 → 代码提交同步 → 合并分支 → 维护清理」。
2. 核心操作：命令行（Git）负责本地版本管控，网页端负责协作配置和可视化操作，两者互补。
3. 协作关键：通过「分支保护 + PR 审查 + 权限分配」保障仓库稳定和代码质量。
4. 进阶技巧：Fork 用于开源贡献，Tag 用于版本发布，仓库转移用于组织调整，备份用于数据安全。

遵循以上操作规范，可高效、安全地管理个人或团队的 GitHub 仓库，适配小型项目到大型开源项目的不同需求。

