# .gitignore 文件全解析

`.gitignore` 是 Git 中用于**指定无需追踪（忽略）的文件 / 目录**的核心配置文件，其作用是让 Git 自动忽略匹配规则的文件，避免这些文件被意外提交到版本库（如编译产物、日志、依赖包、本地配置等）。合理配置 `.gitignore` 能大幅简化版本管理，减少仓库体积，避免敏感信息泄露。

### 一、核心概念

1. **作用范围**：`.gitignore` 仅对**未被 Git 追踪**的文件生效（若文件已被 `git add`/`git commit` 纳入追踪，`.gitignore` 无法忽略，需先移除追踪）。
2. **生效优先级**：Git 会按以下顺序读取忽略规则（优先级从高到低）：
   - 本地仓库配置：`.git/info/exclude`（仅当前仓库生效，不提交到远程）；
   - 全局配置：`~/.gitignore_global`（需通过 `git config --global core.excludesfile` 指定，所有仓库生效）；
   - 仓库内 `.gitignore`（可提交到远程，团队共享规则，最常用）；
   - 命令行参数：`git add --exclude`（临时生效）。
3. **匹配规则本质**：基于「glob 模式」（简化的正则表达式），区分大小写（除非系统 / 仓库配置不区分）。

### 二、基本语法规则

`.gitignore` 的规则遵循简洁的 glob 语法，核心规则如下：

| 规则类型        | 示例             | 说明                                                         |
| --------------- | ---------------- | ------------------------------------------------------------ |
| 匹配文件 / 目录 | `node_modules/`  | 以 `/` 结尾表示匹配目录；无 `/` 则匹配文件和目录（如 `log` 匹配文件 `log` 和目录 `log/`） |
| 通配符 `*`      | `*.log`          | 匹配任意字符（不含路径分隔符 `/`），如 `error.log`、`app.log` |
| 通配符 `**`     | `**/dist`        | 匹配任意层级的目录，如 `dist`、`src/dist`、`src/app/dist`    |
| 通配符 `?`      | `app?.js`        | 匹配单个字符，如 `app1.js`、`app2.js`，不匹配 `app10.js`     |
| 字符范围 `[]`   | `[0-9].txt`      | 匹配单个指定范围字符，如 `1.txt`、`9.txt`，不匹配 `10.txt`   |
| 否定规则 `!`    | `!important.log` | 覆盖前面的忽略规则，即「不忽略该文件」（需放在匹配规则之后） |
| 注释 `#`        | `# 忽略日志文件` | 以 `#` 开头的行是注释，不会生效                              |
| 路径分隔符 `/`  | `/build`         | 以 `/` 开头表示匹配仓库根目录下的文件 / 目录，如 `/build` 不匹配 `src/build` |

#### 语法示例（直观理解）

```gitignore
# 注释：忽略所有日志文件
*.log

# 忽略根目录下的 node_modules 目录
/node_modules/

# 忽略任意层级的 dist 目录
**/dist/

# 忽略 src 目录下的 .env 文件
src/.env

# 匹配 src/app 下的任意 .js 文件（不含子目录）
src/app/*.js

# 匹配 src 下所有子目录的 .vue 文件
src/**/*.vue

# 忽略所有 .txt 文件，但保留 important.txt
*.txt
!important.txt

# 忽略 app 目录下的所有文件，但保留 app/config.json
app/*
!app/config.json
```

### 三、配置与使用步骤

#### 1. 创建 .gitignore 文件

- **仓库根目录创建**（推荐）：适用于整个仓库的忽略规则，可提交到远程供团队共享。

  ```bash
  touch .gitignore  # Linux/macOS
  # 或在 Windows 右键新建文本文档，重命名为 .gitignore（注意前缀点）
  ```

- **局部 .gitignore**：可在仓库子目录创建 `.gitignore`，仅对该子目录生效（规则无需加子目录路径）。

#### 2. 常见忽略场景配置示例

##### 前端项目（Vue/React）

```gitignore
# 依赖包
node_modules/

# 编译产物
dist/
build/
out/

# 环境变量
.env
.env.local
.env.development.local
.env.production.local

# 日志
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# IDE 配置
.idea/
.vscode/
*.swp
*.swo
.DS_Store  # macOS 隐藏文件

# 缓存
.cache/
```

##### 后端项目（Java）

```gitignore
# 编译产物
target/
*.class

# 依赖包
lib/
*.jar

# IDE 配置
.idea/
*.iml
.project
.classpath

# 日志
logs/
*.log

# 本地配置
application-local.yml
```

##### Python 项目

```gitignore
# 虚拟环境
venv/
env/
*.env

# 编译产物
__pycache__/
*.py[cod]
*.so

# 依赖清单（按需忽略）
# requirements.txt （若需共享则不忽略）
pip-wheel-metadata/

# 日志
*.log
```

#### 3. 关键操作：处理已被追踪的文件

若文件已被 `git add`/`git commit` 纳入追踪，即使在 `.gitignore` 中添加规则，Git 仍会继续追踪。需执行以下步骤停止追踪（保留本地文件）：

```bash
# 停止追踪单个文件（如 .env）
git rm --cached .env

# 停止追踪整个目录（如 node_modules）
git rm --cached -r node_modules

# 提交修改（让忽略规则生效）
git commit -m "feat: 停止追踪 .env 和 node_modules"
```

- `--cached`：仅移除 Git 追踪，保留本地文件；若不加该参数，会同时删除本地文件（谨慎使用）。

#### 4. 验证忽略规则是否生效

可通过以下命令检查文件是否被忽略：

```bash
# 检查单个文件（如 .env）
git check-ignore -v .env
# 输出示例：.gitignore:5:.env   .env （表示匹配 .gitignore 第5行规则）

# 列出所有被忽略的文件
git ls-files --ignored --exclude-standard
```

- `git check-ignore -v`：显示文件匹配的具体忽略规则（便于调试）；
- `git ls-files --ignored`：列出所有被忽略的文件。

### 四、进阶用法

#### 1. 全局 .gitignore 配置

若希望所有本地 Git 仓库共享忽略规则（如 IDE 配置、系统临时文件），可配置全局忽略文件：

```bash
# 1. 创建全局忽略文件
touch ~/.gitignore_global

# 2. 配置 Git 识别该文件
git config --global core.excludesfile ~/.gitignore_global

# 3. 编辑 ~/.gitignore_global，添加全局规则（如）：
# .idea/
# .vscode/
# .DS_Store
# *.swp
```

#### 2. 动态忽略：.git/info/exclude

该文件位于仓库的 `.git/info` 目录下，作用与 `.gitignore` 一致，但**不会被提交到远程**，适用于本地临时忽略（如个人测试文件）：

```bash
# 编辑 exclude 文件
vim .git/info/exclude

# 添加本地临时忽略规则（如）：
# test-temp.js
# local-config.json
```

#### 3. 忽略规则的「例外」高级用法

否定规则 `!` 需注意**顺序**：Git 按规则从上到下匹配，后定义的规则覆盖先定义的。

示例：忽略 `src/` 下所有 `.js` 文件，但保留 `src/main.js` 和 `src/utils/` 下的所有 `.js`：

```gitignore
# 先忽略 src 下所有 .js
src/*.js

# 例外：保留 main.js
!src/main.js

# 先忽略 src/utils 下所有 .js
src/utils/*.js

# 例外：保留 src/utils 下所有 .js
!src/utils/*.js
```

#### 4. 忽略已提交的目录下的新文件

若目录已被提交（如 `src/`），但希望忽略该目录下新增的特定文件（如 `src/temp.js`），直接在 `.gitignore` 中添加：

```gitignore
src/temp.js
```

### 五、最佳实践

1. **按项目类型标准化**：
   - 使用开源模板：GitHub 提供了各语言 / 框架的 `.gitignore` 模板（https://github.com/github/gitignore），可直接复用；
   - 团队统一：将 `.gitignore` 提交到远程仓库，确保团队成员使用相同的忽略规则。
2. **避免忽略必要文件**：
   - 不忽略核心配置模板（如 `env.example`），仅忽略本地环境配置（如 `.env.local`）；
   - 不忽略构建脚本（如 `webpack.config.js`）、依赖清单（如 `package.json`/`pom.xml`）。
3. **规则简洁化**：
   - 优先使用通配符减少重复规则（如 `*.log` 代替逐个列出日志文件）；
   - 避免过度嵌套路径（如用 `**/dist` 代替 `src/dist`、`src/app/dist`）。
4. **敏感信息必忽略**：
   - 严格忽略包含密码、密钥、Token 的文件（如 `config/secrets.json`、`.env`）；
   - 若需共享配置，可提供模板文件（如 `config.example.json`），让开发者本地复制修改。
5. **定期维护**：
   - 项目迭代中新增的临时文件 / 编译产物，及时补充到 `.gitignore`；
   - 清理过时规则（如废弃的日志目录、旧依赖包）。

### 六、常见问题与解决方案

1. **.gitignore 规则不生效**：
   - 原因 1：文件已被 Git 追踪 → 解决方案：执行 `git rm --cached <文件/目录>` 并提交；
   - 原因 2：规则语法错误（如路径分隔符、通配符使用不当）→ 解决方案：用 `git check-ignore -v <文件>` 调试规则；
   - 原因 3：规则优先级问题（如全局规则覆盖仓库规则）→ 解决方案：检查 `~/.gitignore_global` 和 `.git/info/exclude`。
2. **误忽略了需要提交的文件**：
   - 解决方案：使用否定规则 `!` 排除该文件（确保 `!` 规则在匹配规则之后）；
   - 示例：已忽略 `*.txt`，需恢复 `readme.txt` → 添加 `!readme.txt`。
3. **Windows 下创建 .gitignore 失败**：
   - 原因：Windows 不允许直接创建以 `.` 开头的文件 → 解决方案：
     - 命令行创建：`echo. > .gitignore`；
     - 记事本保存：文件名输入 `.gitignore.`（末尾加一个点），保存后会自动变为 `.gitignore`。
4. **忽略符号链接**：
   - 直接在 `.gitignore` 中添加符号链接的路径即可（如 `link-to-config`）。

### 总结

`.gitignore` 是 Git 版本管理的基础配置，核心价值是「过滤非核心文件，聚焦源码管理」。掌握其语法规则（通配符、否定规则、路径匹配），结合项目类型标准化配置，同时规避「已追踪文件无法忽略」等常见问题，能显著提升 Git 使用效率，保证仓库的整洁性和安全性。