# GitHub 远程仓库专属操作详细指南

结合之前的 Git 核心操作，你需要明确：**本地 Git 仅负责本地版本控制，而远程仓库的专属配置、权限管理、仓库架构等核心操作，必须在 GitHub 平台（网页端 / 官方 CLI）完成**。以下按操作类型分类，详细阐述 GitHub 远程仓库的专属动作，以及对应的本地 Git 配合操作：

## 一、仓库初始化：创建空白远程仓库（GitHub 专属）

本地项目要关联远程仓库，**第一步必须在 GitHub 网页端创建空白远程仓库**（本地 `git init` 仅初始化本地仓库，无法生成远程仓库），这是所有远程操作的基础。

### 操作步骤（GitHub 网页端）

1. 登录 GitHub 账号，点击页面右上角「+」图标 → 选择「New repository」（新建仓库）。
2. 填写仓库核心信息（均为 GitHub 远程仓库专属配置）：
   - **Repository name**：仓库名称（唯一标识，如 `vue-project-demo`）。
   - **Description**：仓库描述（可选，说明仓库用途）。
   - **Visibility**：仓库可见性（专属配置，两种类型）：
     - Public（公共仓库）：所有人可查看，免费版支持，适合开源项目。
     - Private（私有仓库）：仅你和授权合作者可查看，免费版支持，适合企业项目 / 个人私密项目。
   - **初始化可选配置**（可选，建议空仓库，便于本地关联）：
     - 取消勾选「Initialize this repository with a README」（避免本地仓库与远程仓库初始提交冲突）。
     - 取消勾选「Add .gitignore」「Choose a license」。
3. 点击「Create repository」按钮，完成 GitHub 远程空白仓库创建。
4. 创建后，GitHub 会提供仓库 URL（HTTPS/SSH 格式），用于本地 Git 关联（对应本地 `git remote add` 操作）。

### 本地配合操作

```bash
# 本地项目已初始化 Git 后，关联 GitHub 新建的远程仓库
git remote add origin git@github.com:你的用户名/仓库名.git
# 首次推送本地分支到 GitHub 远程仓库
git push -u origin main
```

## 二、权限与协作管理：GitHub 专属配置（本地无法完成）

团队协作时，远程仓库的访问权限、协作人员管理完全由 GitHub 管控，本地 Git 无任何对应操作，核心动作如下：

### 1. 添加 / 移除仓库合作者（Collaborators）

适用于私有仓库，给指定 GitHub 用户分配仓库访问权限（读写 / 只读等），操作仅在 GitHub 网页端完成：

- 操作路径：进入对应仓库 → 点击「Settings」（设置）→ 左侧导航栏「Collaborators and teams」→ 点击「Add people」。
- 输入合作者的 GitHub 用户名 / 邮箱，点击搜索并选择目标用户。
- 选择权限等级（GitHub 专属权限划分）：
  - Write（读写权限）：可推送代码、创建分支、提交 PR 等（核心协作人员）。
  - Read（只读权限）：仅可克隆、拉取代码，无法推送更改（查看者）。
  - Admin（管理员权限）：拥有仓库全部操作权限（仓库所有者 / 核心负责人）。
- 移除合作者：在同一页面，找到对应用户，点击「⋯」→ 选择「Remove access」即可。

### 2. 管理组织 / 团队权限（Organizations）

适用于企业 / 大型团队，通过「组织」统一管理多个仓库的团队权限，完全为 GitHub 专属功能：

- 操作路径：先创建组织（GitHub 首页「+」→「New organization」）→ 组织页面添加团队 / 成员 → 进入对应仓库 →「Settings」→「Collaborators and teams」→「Add a team」，给团队分配仓库权限。

### 3. 设置仓库访问保护（Branch Protection Rules）

为核心分支（如 `main`/`dev`）设置保护规则，防止误推送、强制覆盖等操作，仅 GitHub 网页端可配置：

- 操作路径：仓库 →「Settings」→「Branches」→「Add rule」。
- 核心保护规则（GitHub 专属）：
  - 勾选「Require a pull request before merging」（合并代码前必须提交 PR 并审核通过）。
  - 勾选「Require approvals」（需要指定数量的审核者批准，如 1 人 / 2 人）。
  - 勾选「Deny force pushes」（禁止强制推送，防止覆盖分支历史）。
  - 勾选「Require status checks to pass before merging」（要求 CI/CD 检查通过后才能合并）。

## 三、代码合并与审核：Pull Request（PR）/Merge Request（MR）

本地 Git 仅能执行 `git merge` 本地分支合并，而**远程仓库的代码审核、分支合并流程，必须通过 GitHub 的 Pull Request（PR）完成**，这是团队协作的核心流程，完全为 GitHub 专属操作。

### 完整操作流程（GitHub 网页端）

1. **前提**：开发者本地创建功能分支，开发完成后推送到 GitHub 远程仓库（本地操作：`git push -u origin feature/login`）。
2. **创建 PR**：
   - 进入 GitHub 对应仓库，会看到「Compare & pull request」提示按钮，点击进入 PR 创建页面。
   - 配置 PR 信息：选择源分支（如 `feature/login`）和目标分支（如 `main`）→ 填写 PR 标题（如「完成登录功能开发」）→ 填写 PR 描述（详细说明更改内容、测试情况等）→ 点击「Create pull request」。
3. **代码审核**：
   - PR 创建后，仓库管理员 / 指定审核者会收到通知，进入 PR 页面查看代码更改（Files changed 标签页）。
   - 审核者可对单行代码添加评论、提出修改意见，开发者根据意见在本地修改后，重新推送代码（本地 `git commit` + `git push`），PR 会自动同步最新更改。
4. **合并 PR**：
   - 当 PR 满足所有条件（审核通过、CI 检查通过、无冲突），审核者 / 开发者可点击「Merge pull request」按钮，选择合并方式（GitHub 专属 3 种合并方式）：
     - Create a merge commit（创建合并提交）：保留分支历史，生成一条合并记录（最常用）。
     - Squash and merge（压缩合并）：将功能分支的所有提交压缩为一条提交，保持 `main` 分支历史整洁。
     - Rebase and merge（变基合并）：将功能分支的提交变基到目标分支，无额外合并记录。
5. **删除远程功能分支**：
   - PR 合并后，GitHub 会提示「Delete branch」按钮，点击即可删除远程无用分支（对应本地 `git push origin --delete feature/login`，但网页端操作更便捷）。

## 四、仓库配置与维护：GitHub 专属操作

本地 Git 无法修改远程仓库的基础配置、清理远程资源等，这些动作需在 GitHub 网页端完成：

### 1. 修改仓库基础信息

- 操作路径：仓库 →「Settings」→「General」。
- 可修改内容（均为 GitHub 远程仓库专属）：
  - 仓库名称、描述、可见性（Public/Private 切换，需满足账号权限）。
  - 仓库默认分支（如将 `main` 改为 `dev`，本地需配合 `git checkout` 切换分支 + `git pull` 拉取更新）。
  - 关闭 / 归档仓库（归档后仓库只读，无法推送更改，适合已下线项目）。

### 2. 管理仓库文件（补充 / 修改基础文件）

部分核心文件可直接在 GitHub 网页端编辑，无需本地操作，适合快速修改：

- 创建 / 编辑 `README.md`：仓库首页点击「Add file」→「Create new file」，编写仓库说明（支持 Markdown 格式），提交后直接同步到远程仓库（本地可通过 `git pull` 拉取该更改）。
- 创建 / 编辑 `.gitignore`：若本地遗漏该文件，可在 GitHub 网页端创建，选择对应语言框架的模板（如 Node.js、Vue 等），快速生成忽略规则。
- 编辑 LICENSE：给开源项目添加许可证（如 MIT、Apache 2.0），明确代码使用权限，仅 GitHub 网页端提供便捷模板选择。

### 3. 查看远程仓库提交历史 / 分支

虽然本地可通过 `git log`/`git branch -a` 查看远程相关信息，但 GitHub 网页端提供更直观的可视化界面：

- 提交历史：仓库 →「Commits」标签页，可查看所有远程提交记录、提交人、更改文件，支持按作者 / 日期筛选。
- 分支管理：仓库 →「Branches」标签页，可查看所有远程分支、分支最后提交时间，快速切换 / 删除远程分支。

### 4. 释放版本（Releases）

为项目标记正式版本（如 v1.0.0、v2.1.0），并提供编译后的产物下载，是 GitHub 远程仓库专属功能，用于项目版本发布：

- 操作路径：仓库 →「Releases」→「Draft a new release」。
- 配置版本信息：
- 选择标签（Tag version，如 `v1.0.0`，可新建标签）。
- 填写版本标题（如「v1.0.0 正式版：支持登录 / 注册功能」）。
- 填写版本说明（新增功能、修复 Bug 等）。
- 上传产物（如 `dist.zip`、安装包等，可选）。
- 点击「Publish release」，完成版本发布。

## 五、SSH 密钥配置：GitHub 网页端 + 本地配合（核心权限配置）

本地使用 SSH 格式 URL 与 GitHub 远程仓库交互（免密登录），**SSH 密钥的添加与验证必须在 GitHub 网页端完成**，本地仅负责生成密钥。

### 1. 本地操作：生成 SSH 密钥

```bash
# 生成 SSH 密钥，邮箱填写 GitHub 绑定邮箱
ssh-keygen -t ed25519 -C "你的GitHub邮箱@xxx.com"
# 按回车默认保存路径（Windows：C:\Users\用户名\.ssh；Mac/Linux：~/.ssh），无需设置密码
```

### 2. GitHub 网页端专属操作：添加 SSH 公钥

- 本地获取公钥内容：
- Windows：打开 `C:\Users\用户名\.ssh\id_ed25519.pub` 文件，复制全部内容。
- Mac/Linux：执行 `cat ~/.ssh/id_ed25519.pub`，复制输出的全部内容。
- GitHub 操作路径：点击右上角头像 →「Settings」→ 左侧「SSH and GPG keys」→「New SSH key」。
- 配置公钥信息：
- Title：密钥名称（自定义，如「My Windows PC」「MacBook Pro」）。
- Key type：选择「Authentication key」（认证密钥）。
- Key：粘贴本地复制的公钥内容。
- 点击「Add SSH key」，完成配置（本地可通过 `ssh -T git@github.com` 验证是否成功）。

## 六、仓库迁移 / 地址变更：GitHub 专属操作

当需要将 GitHub 仓库迁移（如转移所有权、复制仓库），或修改仓库 URL，核心操作在 GitHub 网页端完成：

### 1. 仓库转移所有权

- 操作路径：仓库 →「Settings」→「General」→ 拉到最下方「Danger Zone」→「Transfer ownership」。
- 输入目标 GitHub 用户名 / 组织名称，验证仓库名称，点击「Transfer」即可（目标用户需接受转移邀请）。

### 2. 复制仓库（Fork/Import）

- Fork 仓库：适用于开源项目，点击仓库首页「Fork」按钮，可将他人公共仓库复制到自己的 GitHub 账号下，拥有独立操作权限（后续可通过 PR 向原仓库提交更改）。
- Import 仓库：适用于从其他平台（如 Gitee、GitLab）迁移仓库到 GitHub，操作路径：GitHub 首页「+」→「Import repository」，填写原仓库 URL、目标仓库名称，完成导入。

### 3. 修改仓库 URL（本地配合）

若仓库名称 / 所有者变更，导致远程 URL 变化，可在 GitHub 确认新 URL 后，本地修改关联：

```bash
# 本地修改远程仓库 URL，配合 GitHub 仓库变更
git remote set-url origin git@github.com:新用户名/新仓库名.git
```

## 七、常见 GitHub 专属功能（本地无对应操作）

1. **Issues 管理**：用于追踪 Bug、需求任务，仓库 →「Issues」标签页，可创建 Issue、分配负责人、添加标签 / 里程碑，是项目管理的核心工具（本地 Git 无对应功能）。
2. **GitHub Actions**：自动化 CI/CD 流程（如代码提交后自动测试、打包、部署），仓库 →「Actions」标签页，选择模板或自定义 workflow 配置文件（`.github/workflows/xxx.yml`），本地仅需推送配置文件，运行与触发由 GitHub 远程管控。
3. **Git Pages**：免费托管静态网站（如项目文档、个人博客），仓库 →「Settings」→「Pages」，配置源分支（如 `gh-pages`）和根目录，GitHub 会分配专属域名（`用户名.github.io/仓库名`），本地仅需推送静态文件到对应分支。

## 总结

GitHub 远程仓库的专属操作（无法在本地 Git 完成）可归纳为 6 大类核心动作：

1. 仓库初始化：创建空白远程仓库（配置公共 / 私有可见性）。
2. 权限管理：添加合作者、配置团队权限、设置分支保护规则。
3. 代码协作：创建 / 审核 / 合并 Pull Request（PR），实现远程分支规范合并。
4. 仓库维护：修改仓库基础信息、编辑核心文件、释放版本、删除远程分支。
5. 权限配置：添加 SSH 公钥（实现免密登录）。
6. 专属功能：Issues 任务追踪、GitHub Actions 自动化、Git Pages 静态网站托管。

这些远程操作是本地 Git 功能的补充与管控，确保团队协作的规范性、代码的安全性，以及项目的高效管理，是使用 GitHub 进行版本控制的核心必备技能。