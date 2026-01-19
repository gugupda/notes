# NiceGUI app.storage.user 全面详细解析

`app.storage.user` 是 NiceGUI 内置的五种核心存储类型之一，专为**用户级持久化存储**设计，核心特点是 “服务器端存储、用户专属隔离、跨标签页 / 跨会话共享、长期持久化”，适用于存储用户个性化配置、长期会话数据、跨访问保留的用户状态（如偏好设置、登录状态、历史记录），是平衡 “用户数据隔离” 与 “持久化共享” 的核心方案。

## 一、核心定位与设计目标

### 1. 核心价值

- **用户专属隔离**：每个用户拥有独立的存储空间，数据仅当前用户可见，不与其他用户共享（通过浏览器会话 Cookie 中的唯一 ID 标识用户）；
- **跨场景共享**：支持跨浏览器标签页、跨页面重载、跨应用重启、跨服务器重启共享数据（只要用户未清除浏览器 Cookie）；
- **服务器端安全存储**：数据存储在服务器（文件 / Redis），相比浏览器端存储（`app.storage.browser`），容量更大、安全性更高（避免数据被客户端篡改）；
- **自动持久化**：数据自动序列化并持久化到存储介质（默认 JSON 文件，支持 Redis 分布式存储），无需手动管理持久化逻辑；
- **易用性**：API 完全模仿 Python 字典，支持键值对增删改查，学习成本低，无缝集成业务代码。

### 2. 设计目标

- 解决 “用户级个性化数据存储” 需求，避免用户数据混入全局存储或临时存储；
- 支持长期保留用户数据（如主题偏好、语言设置、登录令牌），提升用户体验；
- 兼顾安全性与灵活性：通过 `storage_secret` 加密用户标识 Cookie，防止身份伪造；支持分布式存储（Redis），适配多实例部署；
- 适配 “用户一次配置、长期生效” 的场景，减少用户重复操作。

## 二、核心特性与关键规则

### 1. 存储基础信息

| 特性     | 详细说明                                                     |
| -------- | ------------------------------------------------------------ |
| 存储位置 | 默认：服务器端 JSON 文件（`nicegui_user_storage/` 目录下，按用户 ID 分文件存储）；支持 Redis 分布式存储（需额外配置） |
| 数据支持 | 仅支持 JSON 可序列化类型（Python 基础类型：`str`、`int`、`float`、`list`、`dict`、`bool`、`None`），复杂对象需手动序列化 |
| 生命周期 | 用户级长期生命周期：从用户首次访问应用开始创建，直到用户清除浏览器 Cookie、手动删除存储数据或服务器清理过期数据为止 |
| 共享范围 | 同一用户跨浏览器标签页、跨页面重载、跨应用重启共享；不跨用户、不跨浏览器（不同浏览器的 Cookie 独立） |
| 依赖条件 | 必须在 `ui.run()` 中指定 `storage_secret` 参数（用于加密浏览器会话 Cookie，防止身份伪造），否则无法使用 |
| 身份标识 | 通过 `app.storage.browser['id']` 生成唯一用户 ID（存储在浏览器会话 Cookie 中），作为用户存储的唯一标识 |

### 2. 关键使用规则

- **`storage_secret` 必选**：`app.storage.user` 依赖加密的浏览器会话 Cookie 标识用户，必须在 `ui.run()` 中传入 `storage_secret`（任意字符串，建议使用复杂密钥），否则会抛出错误；
- **数据序列化限制**：仅支持 JSON 兼容类型，复杂对象（如 `datetime`、自定义类实例）需手动转换为基础类型（如 `datetime` 转为 ISO 字符串）；
- **用户标识稳定性**：用户标识存储在浏览器会话 Cookie 中，清除 Cookie 会导致用户身份重置（视为新用户，原存储数据无法访问）；浏览器关闭后重新打开，若 Cookie 未过期，仍可访问原数据；
- **持久化可靠性**：默认基于文件存储，多实例部署时会出现数据不一致（多个实例读写不同文件），需改用 Redis 存储；
- **自动清理**：默认无自动清理机制，需手动编写逻辑清理过期用户数据（如超过 30 天无访问的用户数据）。

## 三、核心配置与基础用法

### 1. 前置配置：指定 `storage_secret`

使用 `app.storage.user` 前，必须在 `ui.run()` 中设置 `storage_secret`（用于加密用户会话 Cookie），示例：

```python
from nicegui import app, ui

# 关键配置：传入 storage_secret（生产环境建议使用复杂随机字符串）
ui.run(storage_secret='your_secure_secret_key_123456')
```

### 2. 基础操作 API

`app.storage.user` 的 API 与 Python 字典完全一致，支持键值对的增、删、改、查，核心操作如下：

| 操作          | 语法                                                         | 功能描述                                                   | 示例                                             |
| ------------- | ------------------------------------------------------------ | ---------------------------------------------------------- | ------------------------------------------------ |
| 读取值        | `app.storage.user[key]` 或 `app.storage.user.get(key, default)` | 根据键读取用户数据，`get` 方法支持默认值（键不存在时返回） | `theme = app.storage.user.get('theme', 'light')` |
| 写入 / 更新值 | `app.storage.user[key] = value`                              | 写入新键值对或更新已有键的值（自动序列化并持久化）         | `app.storage.user['language'] = 'zh-CN'`         |
| 删除值        | `del app.storage.user[key]` 或 `app.storage.user.pop(key, default)` | 删除指定键，`pop` 方法支持默认值（键不存在时返回）         | `app.storage.user.pop('old_config', None)`       |
| 检查键存在    | `key in app.storage.user`                                    | 判断键是否存在于当前用户存储中                             | `if 'login_status' in app.storage.user: ...`     |
| 清空存储      | `app.storage.user.clear()`                                   | 清空当前用户的所有存储数据                                 | `app.storage.user.clear()`                       |
| 获取长度      | `len(app.storage.user)`                                      | 返回当前用户存储的键值对数量                               | `data_count = len(app.storage.user)`             |
| 遍历数据      | `app.storage.user.items()`                                   | 返回所有键值对视图，支持循环遍历                           | `for k, v in app.storage.user.items(): ...`      |

### 3. 存储格式与路径配置

- **默认存储路径**：用户数据默认存储在应用运行目录下的 `nicegui_user_storage/` 目录，每个用户对应一个 JSON 文件（文件名为用户 ID 的哈希值）；

- **自定义存储路径**：通过 `ui.run(user_storage_path='自定义路径')` 指定用户数据存储目录，示例：

  ```python
  ui.run(
      storage_secret='your_secret',
      user_storage_path='./data/user_storage/'  # 用户数据存储到指定目录
  )
  ```

- **JSON 缩进配置**：默认存储的 JSON 数据无缩进，可通过 `app.storage.user.indent = True` 开启 2 空格缩进（便于调试）：

  ```python
  app.storage.user.indent = True  # 存储的 JSON 文件带缩进，可读性更强
  ```

### 4. Redis 分布式存储配置（多实例部署）

默认文件存储不适用于多实例部署（如负载均衡），需改用 Redis 存储实现数据共享，配置步骤如下：

1. 安装依赖：`pip install nicegui[redis]`；
2. 设置环境变量 `NICEGUI_REDIS_URL`，指向 Redis 服务器（如 `redis://localhost:6379/0`）；
3. 启动 Redis 服务器（建议设置 `--timeout` 参数减少空闲连接）；
4. 运行应用，`app.storage.user` 自动使用 Redis 存储。

> 注意：Redis 存储会同步整个用户数据字典（而非仅变更部分），若存储大量数据，建议改用数据库（如 PostgreSQL）。

## 四、典型应用场景与完整示例

### 场景 1：用户个性化主题与语言设置（跨标签页 / 跨重启保留）

需求：用户设置主题（浅色 / 深色）和语言（中文 / 英文）后，跨标签页、页面重载、应用重启后仍保留设置，无需重复配置。

```python
from nicegui import app, ui

# 开启 JSON 缩进（调试用，生产环境可关闭）
app.storage.user.indent = True

@ui.page('/')
def index():
    # 初始化用户配置（键不存在时设默认值）
    default_theme = app.storage.user.get('theme', 'light')
    default_lang = app.storage.user.get('language', 'zh-CN')
    
    # 应用主题
    ui.query('body').classes(f'theme-{default_theme}')
    
    # 主题切换按钮
    def toggle_theme():
        new_theme = 'dark' if app.storage.user['theme'] == 'light' else 'light'
        app.storage.user['theme'] = new_theme
        ui.query('body').classes(f'theme-{new_theme}')
        ui.notify(f'主题已切换为：{"深色" if new_theme == "dark" else "浅色"}')
    
    # 语言切换按钮
    def switch_language():
        new_lang = 'en-US' if app.storage.user['language'] == 'zh-CN' else 'zh-CN'
        app.storage.user['language'] = new_lang
        ui.notify(f'Language switched to: {new_lang}')
    
    # 展示当前配置
    ui.label(f'当前主题：{"浅色" if default_theme == "light" else "深色"}')
    ui.label(f'当前语言：{default_lang}')
    
    # 操作按钮
    ui.button('切换主题', on_click=toggle_theme)
    ui.button('切换语言', on_click=switch_language)
    ui.button('重载页面', on_click=ui.navigate.reload)

# 必传 storage_secret
ui.run(storage_secret='your_secure_secret_789', reload=True)
```

**效果**：切换主题 / 语言后，打开新标签页、重载页面、重启应用，配置仍保留；不同浏览器访问视为不同用户，配置独立。

### 场景 2：用户页面访问次数统计（跨会话累计）

需求：统计每个用户的页面访问次数，即使用户关闭浏览器、应用重启，访问次数仍持续累计。

```python
from nicegui import app, ui
from datetime import datetime

@ui.page('/')
def index():
    # 初始化访问次数和首次访问时间（键不存在时设默认值）
    if 'visit_count' not in app.storage.user:
        app.storage.user['visit_count'] = 0
        app.storage.user['first_visit'] = datetime.now().isoformat()  # 首次访问时间（序列化存储）
    
    # 访问次数自增（每次页面加载时）
    app.storage.user['visit_count'] += 1
    
    # 读取并解析首次访问时间
    first_visit = datetime.fromisoformat(app.storage.user['first_visit'])
    current_visit = datetime.now()
    days_since_first_visit = (current_visit - first_visit).days
    
    # 展示用户访问信息
    ui.label(f'你是第 {app.storage.user["visit_count"]} 次访问本页面')
    ui.label(f'首次访问时间：{first_visit.strftime("%Y-%m-%d %H:%M:%S")}')
    ui.label(f'距离首次访问已过去 {days_since_first_visit} 天')

# 必传 storage_secret
ui.run(storage_secret='visit_count_secret_456')
```

**核心亮点**：通过 `datetime.isoformat()` 序列化日期对象，读取时反序列化，解决复杂类型存储问题；访问次数跨会话、跨重启累计。

### 场景 3：用户表单数据持久化（跨访问保留草稿）

需求：用户填写表单时，实时保存草稿到用户存储，跨标签页、页面重载后可恢复草稿，提交后清空。

```python
from nicegui import app, ui

@ui.page('/')
def index():
    # 初始化表单草稿（从用户存储读取，无则为空）
    form_draft = app.storage.user.get('form_draft', {
        'name': '',
        'email': '',
        'message': ''
    })
    
    # 创建表单组件并绑定草稿数据
    name_input = ui.input('姓名', value=form_draft['name'])
    email_input = ui.input('邮箱', value=form_draft['email'])
    msg_input = ui.textarea('留言', value=form_draft['message']).classes('w-full')
    
    # 实时保存草稿（输入时更新存储）
    def save_draft():
        app.storage.user['form_draft'] = {
            'name': name_input.value,
            'email': email_input.value,
            'message': msg_input.value
        }
        ui.notify('草稿已自动保存')
    
    # 绑定输入事件
    name_input.on('input', save_draft)
    email_input.on('input', save_draft)
    msg_input.on('input', save_draft)
    
    # 提交表单（清空草稿）
    def submit_form():
        app.storage.user.pop('form_draft', None)  # 提交后删除草稿
        ui.notify(f'表单提交成功！\n姓名：{name_input.value}\n邮箱：{email_input.value}')
        # 重置表单
        name_input.value = ''
        email_input.value = ''
        msg_input.value = ''
    
    # 清空草稿
    def clear_draft():
        app.storage.user.pop('form_draft', None)
        name_input.value = ''
        email_input.value = ''
        msg_input.value = ''
        ui.notify('草稿已清空')
    
    # 操作按钮
    ui.button('提交表单', on_click=submit_form).classes('positive')
    ui.button('清空草稿', on_click=clear_draft).classes('negative')
    ui.button('重载页面', on_click=ui.navigate.reload)

ui.run(storage_secret='form_draft_secret_123')
```

**效果**：输入过程中实时保存草稿，重载页面、打开新标签页后可恢复；提交或清空后，草稿数据删除。

## 五、与其他存储类型的核心区别

为明确 `app.storage.user` 的适用边界，以下对比其与 NiceGUI 其他四种存储类型的关键差异：

| 对比维度          | .user                   | .client              | .tab             | .browser                  | .general                |
| ----------------- | ----------------------- | -------------------- | ---------------- | ------------------------- | ----------------------- |
| 存储位置          | 服务器（文件 / Redis）  | 服务器内存           | 服务器内存       | 浏览器 Cookie             | 服务器（文件 / Redis）  |
| 跨标签页          | 是（共享）              | 否（独立）           | 否（独立）       | 是（共享）                | 是（共享）              |
| 跨页面重载        | 是（保留）              | 否（清空）           | 是（保留）       | 是（保留）                | 是（保留）              |
| 跨服务器重启      | 是（保留）              | 否                   | 是（标签页未关） | 否                        | 是（保留）              |
| 数据类型支持      | 序列化类型（JSON 兼容） | 任意 Python 对象     | 任意 Python 对象 | 序列化类型（Cookie 兼容） | 序列化类型（JSON 兼容） |
| 需 storage_secret | 是                      | 否                   | 否               | 是                        | 否                      |
| 共享范围          | 同一用户                | 同一客户端（标签页） | 同一标签页       | 同一用户（浏览器）        | 所有用户                |
| 核心生命周期      | 用户长期周期            | 页面访问周期         | 标签页会话周期   | 浏览器会话周期            | 应用生命周期            |

### 关键选型建议

- 需用户级个性化、跨访问保留 → 选 `app.storage.user`；
- 需客户端临时存储、重载清空 → 选 `app.storage.client`；
- 需标签页独立存储、重载保留 → 选 `app.storage.tab`；
- 需浏览器端存储、无服务器依赖 → 选 `app.storage.browser`（不推荐，优先 `user`）；
- 需全局所有用户共享 → 选 `app.storage.general`。

## 六、最佳实践与注意事项

### 1. 最佳实践

- **`storage_secret` 安全配置**：生产环境使用复杂随机字符串（如通过 `secrets.token_hex(16)` 生成），避免硬编码在代码中，建议通过环境变量传入：

  ```python
  import os
  from nicegui import ui
  
  # 从环境变量读取 storage_secret（生产环境推荐）
  storage_secret = os.getenv('NICEGUI_STORAGE_SECRET', 'fallback_secret')
  ui.run(storage_secret=storage_secret)
  ```

- **复杂类型序列化**：存储 `datetime`、`decimal` 等非 JSON 兼容类型时，手动转换为基础类型（如 `datetime` → ISO 字符串、`decimal` → 字符串）；

- **数据量控制**：避免存储过大数据（如超过 10MB），文件存储会导致读写性能下降，Redis 存储会增加网络开销，建议大文件 / 大数据用数据库；

- **过期数据清理**：定期清理长期无访问的用户数据（如超过 90 天），避免存储目录 / Redis 占用过多空间；

- **敏感数据加密**：存储用户令牌、密码哈希等敏感数据时，建议额外加密（如使用 `cryptography` 库），避免明文存储。

### 2. 注意事项

- **`storage_secret` 不可变更**：运行后若修改 `storage_secret`，会导致已有的用户 Cookie 失效（视为新用户），原用户数据无法访问；
- **Cookie 依赖风险**：用户清除浏览器 Cookie 后，身份标识丢失，原存储数据无法关联，需提醒用户谨慎清除 Cookie；
- **多实例部署限制**：默认文件存储不支持多实例部署（多个实例读写不同文件），需改用 Redis 存储；
- **序列化限制**：未序列化的复杂对象（如自定义类实例、函数）无法存储，会抛出 `TypeError`；
- **Redis 同步特性**：Redis 存储会同步整个用户数据字典，频繁更新大量数据会影响性能，需合理设计数据结构。

## 七、总结

`app.storage.user` 是 NiceGUI 针对 “用户级持久化存储” 场景的最优解，核心优势在于**用户专属隔离、跨场景共享、服务器端安全存储、自动持久化**。其适用场景集中在：

1. 用户个性化配置（主题、语言、显示密度）；
2. 跨会话保留的用户状态（登录状态、访问记录）；
3. 表单草稿、历史操作记录等需要长期保留的用户数据；
4. 无需数据库的轻量级用户数据存储需求。

与其他存储类型相比，`app.storage.user` 填补了 “用户级、长期化、跨场景共享” 的存储空白，既解决了浏览器存储的安全性和容量问题，又避免了数据库的复杂配置，是中小规模应用实现用户个性化的首选方案。使用关键在于：配置安全的 `storage_secret`、遵守数据序列化规则、根据部署规模选择存储介质（文件 / Redis）。