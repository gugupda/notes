# NiceGUI 中`ui.request`的两个同名核心 API 深度解析

NiceGUI 中存在两个**同名但功能、场景、底层实现完全不同**的 `ui.request` API，是新手最易混淆的核心点。本文从定义、用途、参数、示例、底层逻辑等维度彻底区分二者，同时明确配合使用的最佳实践。

## 一、核心结论（先明确差异）

| 维度         | `ui.request`（HTTP 请求 API）                         | `ui.request`（UI 线程调度 API）                            |
| ------------ | ----------------------------------------------------- | ---------------------------------------------------------- |
| **核心定位** | 前端发起 HTTP/HTTPS 请求（替代浏览器`fetch`/`axios`） | 调度 UI 操作到主线程执行（解决线程安全问题）               |
| **入参核心** | URL、请求方法、请求体 / 头（接口调用参数）            | 无参 UI 操作函数、优先级、超时（UI 调度参数）              |
| **返回值**   | `Response`对象（包含状态码、响应体等）                | `asyncio.Future`对象（调度结果 / 执行状态）                |
| **执行方式** | 必须`await`（异步 HTTP 请求）                         | 可`await`（等待执行）或直接调用（异步调度）                |
| **底层实现** | 浏览器端`fetch` API（前端 JS）                        | NiceGUI 事件循环调度器（后端 Python）                      |
| **核心场景** | 前端→后端 / 第三方接口调用（如查数据、提交表单）      | 非 UI 线程 / 异步任务中更新 UI（如后台线程改标签、进度条） |
| **异常类型** | 网络错误、接口 4xx/5xx、超时（HTTP 层面）             | 调度超时、函数执行异常（UI 操作层面）                      |

## 二、API 1：`ui.request`（HTTP 请求 API）

### 1. 官方定义与函数签名

这是 NiceGUI 封装的**前端 HTTP 请求工具**，本质是浏览器`fetch` API 的 Python 封装，仅在异步上下文（如`async def`函数）中可用。

```python
async def request(
    url: str,                  # 请求地址（相对/绝对URL）
    method: str = 'GET',       # 请求方法：GET/POST/PUT/DELETE等
    headers: dict | None = None, # 请求头（如Content-Type、Token）
    data: Any | None = None,   # 请求体（POST/PUT时传参，自动序列化）
    params: dict | None = None,# URL查询参数（GET时拼接）
    timeout: float = 30.0,     # 请求超时时间（秒）
    verify_ssl: bool = True,   # 是否验证SSL证书
) -> Response:
    """前端发起HTTP请求，返回响应对象"""
```

### 2. 核心特性

- **前端侧执行**：请求由用户浏览器发起（而非 NiceGUI 后端），避免后端代理导致的性能损耗；
- **异步必选**：必须用`await`调用，否则会返回`coroutine`对象而非响应；
- **自动序列化**：`data`参数自动转为 JSON（Content-Type 默认`application/json`）；
- **响应对象**：返回的`Response`包含以下核心属性 / 方法：
  - `status`：HTTP 状态码（如 200/404/500）；
  - `ok`：布尔值，`status`在 200-299 时为`True`；
  - `json()`：异步方法，解析 JSON 响应体；
  - `text()`：异步方法，获取文本响应体；
  - `headers`：响应头字典。

### 3. 典型使用场景与示例

#### 场景 1：GET 请求获取后端数据（最常用）

```python
from nicegui import ui

# 后端API（FastAPI原生路由，NiceGUI基于FastAPI）
@ui.page('/api/users')
async def get_users_api():
    return [{'id': 1, 'name': '张三'}, {'id': 2, 'name': '李四'}]

# 前端UI + HTTP请求逻辑
user_list = ui.column()

async def load_users():
    """异步加载用户列表（调用HTTP请求API）"""
    user_list.clear()  # 清空旧数据
    try:
        # 调用相对URL的后端接口（GET请求）
        response = await ui.request('/api/users', method='GET')
        if response.ok:  # 等价于response.status >=200 and response.status <300
            users = await response.json()  # 解析JSON响应
            for user in users:
                user_list.add(ui.label(f'ID：{user["id"]}，姓名：{user["name"]}'))
        else:
            user_list.add(ui.label(f'接口错误：{response.status}').classes('text-red-500'))
    except Exception as e:
        user_list.add(ui.label(f'请求失败：{str(e)}').classes('text-red-500'))

ui.button('加载用户列表', on_click=load_users)
ui.run()
```

#### 场景 2：POST 请求提交表单数据

```python
from nicegui import ui

# 表单UI
name_input = ui.input('姓名')
age_input = ui.number('年龄')
submit_result = ui.label('')

async def submit_form():
    """提交表单（POST请求）"""
    # 构造请求体
    post_data = {
        'name': name_input.value,
        'age': age_input.value
    }
    try:
        # POST请求 + JSON请求体
        response = await ui.request(
            url='/api/submit-user',
            method='POST',
            headers={'Authorization': 'Bearer token123'},  # 自定义请求头
            data=post_data  # 自动转为JSON
        )
        if response.ok:
            result = await response.json()
            submit_result.set_text(f'提交成功：{result["msg"]}').classes('text-green-500')
        else:
            submit_result.set_text(f'提交失败：{response.status}').classes('text-red-500')
    except Exception as e:
        submit_result.set_text(f'网络错误：{str(e)}').classes('text-red-500')

ui.button('提交', on_click=submit_form)

# 后端接收POST请求的接口
@ui.page('/api/submit-user', methods=['POST'])
async def submit_user_api(request):
    data = await request.json()  # 获取前端提交的JSON数据
    return {'code': 200, 'msg': f'用户{data["name"]}已提交'}

ui.run()
```

### 4. 关键注意事项

- **跨域问题**：若请求第三方接口（非 NiceGUI 后端），需确保目标接口开启 CORS；
- **相对 URL**：请求 NiceGUI 自身后端时，用相对 URL（如`/api/users`）即可，无需拼接域名；
- **异常捕获**：必须捕获`TimeoutError`、`NetworkError`等，避免前端无响应；
- **数据解析**：`response.json()`/`response.text()`是异步方法，需加`await`。

## 三、API 2：`ui.request`（UI 线程调度 API）

### 1. 官方定义与函数签名

这是 NiceGUI 解决**非 UI 线程更新 UI**的核心工具，用于将 UI 操作调度到主线程（UI 线程）执行，避免线程安全问题。

```python
def request(
    func: Callable[[], None],  # 无参UI操作函数（核心：要执行的UI逻辑）
    *,
    priority: int = 0,         # 执行优先级（数值越大越先执行）
    timeout: float | None = None,  # 调度超时时间（秒），None无超时
) -> asyncio.Future:
    """将UI操作函数调度到UI主线程执行"""
```

### 2. 核心特性

- **后端侧执行**：调度逻辑在 NiceGUI 后端的事件循环中运行，确保 UI 操作在主线程；
- **线程安全**：无论在哪个线程调用，`func`都会被转发到 UI 主线程执行；
- **批量优化**：短时间内多个`ui.request`会合并为一次 UI 刷新，提升性能；
- **返回值**：`asyncio.Future`可通过`await`等待执行完成，或`result()`同步阻塞（不推荐）。

### 3. 典型使用场景与示例

#### 场景 1：后台线程更新 UI（核心场景）

```python
from nicegui import ui
import threading
import time

progress = ui.progress(0).props('indeterminate=False')
status = ui.label('未开始')

def background_task():
    """后台线程执行耗时任务"""
    for i in range(1, 101):
        time.sleep(0.1)  # 模拟耗时操作（如计算、IO）
        # 关键：用UI调度API更新UI（避免线程安全问题）
        ui.request(
            lambda i=i: (  # 闭包传参，避免循环变量陷阱
                progress.set_value(i/100),
                status.set_text(f'进度：{i}%')
            )
        )

# 启动后台线程
threading.Thread(target=background_task, daemon=True).start()

ui.run()
```

#### 场景 2：异步任务中安全更新 UI

```python
from nicegui import ui
import asyncio

counter = ui.label('0')

async def async_worker():
    """异步任务（非UI线程风险场景）"""
    for i in range(1, 101):
        await asyncio.sleep(0.05)
        # 用UI调度API确保UI操作在主线程
        await ui.request(lambda i=i: counter.set_text(str(i)))

ui.button('启动异步任务', on_click=async_worker)
ui.run()
```

#### 场景 3：高优先级 UI 更新（紧急提示）

```python
from nicegui import ui
import threading
import time

alert = ui.label('').classes('text-red-500 text-xl')
normal = ui.label('普通更新')

def background_job():
    # 低优先级更新（默认0）
    ui.request(lambda: normal.set_text('普通更新中...'))
    time.sleep(0.1)
    # 高优先级更新（优先执行）
    ui.request(lambda: alert.set_text('⚠️ 紧急警告！'), priority=10)

threading.Thread(target=background_job, daemon=True).start()
ui.run()
```

### 4. 关键注意事项

- **闭包变量陷阱**：循环中传参需用`lambda i=i: ...`，否则所有回调共享最后一个变量值；
- **避免耗时操作**：`func`中仅放 UI 更新逻辑（如`set_text`、`add`），耗时操作放后台线程；
- **无需嵌套调用**：若`func`本身在 UI 线程执行（如按钮点击回调），无需再用`ui.request`；
- **异步等待**：异步场景下用`await ui.request(...)`，确保 UI 操作完成后再执行后续逻辑。

## 四、两个 API 的配合使用（实战最佳范式）

实际项目中，**HTTP 请求 API**和**UI 调度 API**通常配合使用（前端调用接口→响应后安全更新 UI），这是 NiceGUI 开发的核心范式：

### 完整示例：接口调用 + 线程安全 UI 更新

```python
from nicegui import ui, app

# 步骤1：后端API（模拟耗时接口）
@app.get('/api/statistics')
async def get_statistics():
    import asyncio
    await asyncio.sleep(1)  # 模拟接口耗时
    return {'total_users': 120, 'total_orders': 350}

# 步骤2：前端UI
stats_container = ui.card().tight().style('width: 300px')
with stats_container:
    ui.label('数据统计').classes('text-lg font-bold')
    stats_label = ui.label('未加载').classes('text-gray-500')
load_btn = ui.button('加载统计数据', on_click=load_statistics)

# 步骤3：核心逻辑（HTTP请求 + UI调度）
async def load_statistics():
    # 1. 前端状态更新（UI线程直接操作，无需调度）
    load_btn.disable()
    stats_label.set_text('加载中...').classes('text-blue-500')

    try:
        # 2. 调用HTTP请求API，获取后端数据
        response = await ui.request('/api/statistics', method='GET')
        if not response.ok:
            raise Exception(f'接口错误：{response.status}')
        
        stats = await response.json()

        # 3. 用UI调度API安全更新UI（兜底线程安全）
        await ui.request(
            lambda stats=stats: (
                stats_label.set_text(
                    f'总用户：{stats["total_users"]}\n总订单：{stats["total_orders"]}'
                ).classes('text-green-500'),
                load_btn.enable()
            )
        )

    except Exception as e:
        # 异常场景的UI更新（同样用UI调度API）
        await ui.request(
            lambda: (
                stats_label.set_text(f'加载失败：{str(e)}').classes('text-red-500'),
                load_btn.enable()
            )
        )

ui.run(port=8080)
```

### 配合逻辑拆解

1. **触发阶段**：按钮点击→执行`load_statistics`（异步函数，UI 线程）；
2. **请求阶段**：`await ui.request('/api/statistics')`（HTTP 请求 API）调用后端接口；
3. **响应阶段**：接口返回后，`await ui.request(lambda: ...)`（UI 调度 API）将 UI 更新逻辑调度到主线程；
4. **兜底阶段**：无论成功 / 失败，都通过 UI 调度 API 恢复按钮状态、更新提示，确保线程安全。

## 五、总结

| 最终记忆要点    | HTTP 请求 API                | UI 线程调度 API                    |
| --------------- | ---------------------------- | ---------------------------------- |
| 一句话用途      | “前端发请求拿数据”           | “后台更 UI 保安全”                 |
| 入参识别        | 第一个参数是 URL             | 第一个参数是函数                   |
| 是否必加`await` | 是（异步 HTTP 请求）         | 异步场景建议加，同步场景可直接调用 |
| 核心风险点      | 跨域、接口超时、数据解析错误 | 闭包变量陷阱、耗时操作阻塞 UI      |
| 最佳搭档        | 后端 API 接口                | 后台线程 / 异步任务                |

理解两个`ui.request`的核心差异，是掌握 NiceGUI“前端交互 + 后端接口 + 线程安全 UI 更新” 的关键。记住：**见 URL 则是 HTTP 请求，见函数则是 UI 调度**。

# NiceGUI 中`ui.request`（HTTP 请求 API）深度解析

`ui.request`（HTTP 请求 API）是 NiceGUI 封装的**前端侧异步 HTTP 请求工具**，本质是浏览器`fetch` API 的 Python 层抽象，专为前端交互场景设计（如按钮点击后调用后端接口、表单提交、数据查询）。它解决了原生`fetch`/`axios`需要写 JS 代码的痛点，让开发者完全在 Python 层面完成前端发起 HTTP 请求的逻辑，是 NiceGUI 连接前端交互与后端 API 的核心桥梁。

## 一、核心定位与价值

### 1. 核心作用

- 替代浏览器原生`fetch`/`axios`，在 Python 代码中直接发起前端 HTTP 请求（请求由用户浏览器执行，而非 NiceGUI 后端）；
- 支持 GET/POST/PUT/DELETE 等所有 HTTP 方法，适配 RESTful API 场景；
- 自动处理 JSON 序列化 / 反序列化，简化请求体 / 响应体处理；
- 与 NiceGUI 的异步生态（`async/await`）深度兼容，无 UI 阻塞风险。

### 2. 核心优势

| 对比维度    | `ui.request`（HTTP 请求 API） | 后端`httpx/requests` | 原生 JS `fetch`   |
| ----------- | ----------------------------- | -------------------- | ----------------- |
| 执行位置    | 前端浏览器                    | 后端服务器           | 前端浏览器        |
| 跨域处理    | 继承浏览器跨域规则            | 无跨域限制           | 需手动处理 CORS   |
| 代码范式    | Python 纯代码                 | Python 纯代码        | JS 代码           |
| UI 阻塞风险 | 异步无阻塞                    | 后端阻塞（易卡 UI）  | 异步无阻塞        |
| 集成性      | 与 NiceGUI 组件无缝联动       | 需手动同步 UI        | 需 JS-Python 桥接 |

## 二、完整函数签名与参数详解

### 1. 官方函数签名

```python
async def request(
    url: str,
    method: str = 'GET',
    headers: dict | None = None,
    data: Any | None = None,
    params: dict | None = None,
    timeout: float = 30.0,
    verify_ssl: bool = True,
    json: Any | None = None,  # 特殊参数：优先于data，强制JSON序列化
) -> Response:
    """前端发起异步HTTP请求，返回响应对象"""
```

### 2. 参数全解析

| 参数名       | 类型        | 默认值 | 核心说明                                                     |
| ------------ | ----------- | ------ | ------------------------------------------------------------ |
| `url`        | `str`       | 无     | 请求地址：- 相对 URL（推荐）：如`/api/users`（指向 NiceGUI 自身后端）；- 绝对 URL：如`https://api.example.com/data`（第三方接口） |
| `method`     | `str`       | `GET`  | HTTP 方法：`GET`/`POST`/`PUT`/`DELETE`/`PATCH`等（大小写不敏感） |
| `headers`    | `dict None` | `None` | 请求头字典：如`{'Content-Type': 'application/json', 'Authorization': 'Bearer token'}` |
| `data`       | `Any None`  | `None` | 请求体数据：- 非 JSON 场景（如表单`form-data`）：传字典 / 字符串；- JSON 场景：建议用`json`参数 |
| `params`     | `dict None` | `None` | URL 查询参数：自动拼接为`?key1=value1&key2=value2`（仅 GET/DELETE 等无请求体方法常用） |
| `timeout`    | `float`     | `30.0` | 请求超时时间（秒）：超过该时间未响应则抛出`TimeoutError`     |
| `verify_ssl` | `bool`      | `True` | 是否验证 SSL 证书：调试时可设为`False`跳过自签名证书验证（生产环境禁用） |
| `json`       | `Any None`  | `None` | 特殊参数：- 自动将数据序列化为 JSON；- 强制设置`Content-Type: application/json`；- 优先级高于`data`（同时传则`data`失效） |

### 3. 返回值：`Response`对象

`ui.request`返回的`Response`对象是浏览器`fetch`响应的 Python 封装，核心属性 / 方法如下：

| 属性 / 方法 | 类型             | 说明                                                         |
| ----------- | ---------------- | ------------------------------------------------------------ |
| `status`    | `int`            | HTTP 状态码（如 200/404/500）                                |
| `ok`        | `bool`           | 状态码在 200-299 之间则为`True`（快速判断请求是否成功）      |
| `headers`   | `dict`           | 响应头字典（如`{'Content-Type': 'application/json'}`）       |
| `json()`    | `async function` | 异步解析 JSON 响应体（需加`await`），返回 Python 字典 / 列表 |
| `text()`    | `async function` | 异步获取文本响应体（需加`await`），返回字符串（如 HTML / 纯文本） |
| `bytes()`   | `async function` | 异步获取二进制响应体（需加`await`），返回 bytes（如图片 / 文件） |
| `error`     | `str None`       | 请求错误信息（如网络错误、超时）                             |

## 三、典型使用场景与完整示例

### 场景 1：GET 请求（查询数据，最常用）

#### 核心逻辑

按钮点击 → 发起 GET 请求获取后端数据 → 解析响应并渲染到 UI 组件。

```python
from nicegui import ui, app

# 步骤1：定义后端API（NiceGUI基于FastAPI，直接写路由）
@app.get('/api/users')
async def get_users():
    """模拟后端用户列表接口"""
    # 模拟数据库查询耗时
    import asyncio
    await asyncio.sleep(0.5)
    return [
        {'id': 1, 'name': '张三', 'age': 25, 'email': 'zhangsan@example.com'},
        {'id': 2, 'name': '李四', 'age': 30, 'email': 'lisi@example.com'},
    ]

# 步骤2：前端UI布局
user_list_container = ui.column().classes('w-full max-w-md')
load_btn = ui.button('加载用户列表', on_click=load_users).classes('mt-2')

# 步骤3：前端发起GET请求的逻辑
async def load_users():
    """异步加载用户列表（调用HTTP请求API）"""
    # 清空旧数据 + 显示加载状态
    user_list_container.clear()
    loading_label = ui.label('加载中...').classes('text-blue-500')
    user_list_container.add(loading_label)

    try:
        # 核心：调用HTTP请求API发起GET请求
        response = await ui.request(
            url='/api/users',          # 相对URL（指向自身后端）
            method='GET',              # 默认GET，可省略
            params={'page': 1, 'size': 10},  # URL查询参数，拼接为/api/users?page=1&size=10
            timeout=10.0,              # 超时10秒
        )

        # 判断请求是否成功
        if response.ok:
            # 解析JSON响应体（必须加await）
            users = await response.json()
            # 渲染数据到UI
            loading_label.delete()  # 移除加载提示
            for user in users:
                with user_list_container:
                    ui.card(
                        ui.label(f'ID：{user["id"]}'),
                        ui.label(f'姓名：{user["name"]}'),
                        ui.label(f'年龄：{user["age"]}'),
                        ui.label(f'邮箱：{user["email"]}'),
                    ).classes('mb-2')
        else:
            # 接口返回错误状态码（如404/500）
            loading_label.set_text(f'接口错误：{response.status}').classes('text-red-500')
    except TimeoutError:
        # 超时异常处理
        loading_label.set_text('请求超时，请重试').classes('text-red-500')
    except Exception as e:
        # 其他异常（如网络错误、跨域）
        loading_label.set_text(f'请求失败：{str(e)}').classes('text-red-500')

# 启动应用
ui.run(port=8080, reload=True)
```

### 场景 2：POST 请求（提交表单数据）

#### 核心逻辑

表单输入 → 提交按钮点击 → 发起 POST 请求提交 JSON 数据 → 后端接收并返回结果 → 更新 UI 提示。

```python
from nicegui import ui, app
from fastapi import Request

# 步骤1：后端POST接口（接收表单数据）
@app.post('/api/submit-form')
async def submit_form(request: Request):
    """接收前端提交的表单数据"""
    # 解析JSON请求体
    form_data = await request.json()
    # 模拟业务逻辑（如数据校验、入库）
    if not form_data.get('name'):
        return {'code': 400, 'msg': '姓名不能为空'}
    return {'code': 200, 'msg': f'您好{form_data["name"]}，表单提交成功！'}

# 步骤2：前端表单UI
form_container = ui.column().classes('w-full max-w-sm')
with form_container:
    name_input = ui.input('姓名').placeholder('请输入您的姓名')
    phone_input = ui.input('手机号').placeholder('请输入您的手机号')
    submit_result = ui.label('').classes('mt-2')

# 步骤3：POST请求提交逻辑
async def submit_form_data():
    """提交表单（调用HTTP请求API）"""
    # 前置校验
    if not name_input.value.strip():
        submit_result.set_text('姓名不能为空！').classes('text-red-500')
        return

    # 构造请求数据
    post_data = {
        'name': name_input.value.strip(),
        'phone': phone_input.value.strip() or '',
    }

    try:
        # 核心：发起POST请求（用json参数自动序列化）
        response = await ui.request(
            url='/api/submit-form',
            method='POST',
            json=post_data,  # 自动转为JSON，设置Content-Type: application/json
            headers={'Authorization': 'Bearer my-token-123'},  # 自定义请求头
        )

        # 解析响应
        result = await response.json()
        if result['code'] == 200:
            submit_result.set_text(result['msg']).classes('text-green-500')
            # 清空表单
            name_input.value = ''
            phone_input.value = ''
        else:
            submit_result.set_text(result['msg']).classes('text-red-500')
    except Exception as e:
        submit_result.set_text(f'提交失败：{str(e)}').classes('text-red-500')

# 绑定提交按钮
ui.button('提交表单', on_click=submit_form_data).classes('mt-2')

ui.run(port=8080)
```

### 场景 3：PUT/DELETE 请求（RESTful API）

```python
from nicegui import ui, app

# 模拟用户数据存储
user_db = {'1': {'name': '张三', 'age': 25}}

# 后端PUT接口（更新用户）
@app.put('/api/users/{user_id}')
async def update_user(user_id: str, request: Request):
    data = await request.json()
    if user_id not in user_db:
        return {'code': 404, 'msg': '用户不存在'}
    user_db[user_id].update(data)
    return {'code': 200, 'msg': '更新成功', 'data': user_db[user_id]}

# 后端DELETE接口（删除用户）
@app.delete('/api/users/{user_id}')
async def delete_user(user_id: str):
    if user_id not in user_db:
        return {'code': 404, 'msg': '用户不存在'}
    del user_db[user_id]
    return {'code': 200, 'msg': '删除成功'}

# 前端更新用户逻辑
async def update_user():
    response = await ui.request(
        url='/api/users/1',
        method='PUT',
        json={'age': 26},  # 更新年龄
    )
    result = await response.json()
    ui.notify(result['msg'], type='success' if result['code'] == 200 else 'error')

# 前端删除用户逻辑
async def delete_user():
    response = await ui.request(
        url='/api/users/1',
        method='DELETE',
    )
    result = await response.json()
    ui.notify(result['msg'], type='success' if result['code'] == 200 else 'error')

# UI按钮
ui.button('更新用户年龄', on_click=update_user)
ui.button('删除用户', on_click=delete_user)

ui.run(port=8080)
```

### 场景 4：请求第三方接口（跨域处理）

```python
from nicegui import ui

async def get_weather():
    """请求第三方天气接口（需确保接口开启CORS）"""
    try:
        response = await ui.request(
            url='https://api.example.com/weather',  # 第三方绝对URL
            method='GET',
            params={'city': '北京'},
            timeout=5.0,
            verify_ssl=False,  # 调试时跳过SSL验证（生产禁用）
        )
        if response.ok:
            weather = await response.json()
            ui.label(f'北京天气：{weather["temp"]}℃').classes('text-green-500')
        else:
            ui.label(f'接口错误：{response.status}').classes('text-red-500')
    except Exception as e:
        ui.label(f'请求失败：{str(e)}').classes('text-red-500')

ui.button('查询北京天气', on_click=get_weather)
ui.run(port=8080)
```

## 四、关键注意事项与最佳实践

### 1. 跨域问题（核心坑点）

- `ui.request`由浏览器执行，遵循浏览器跨域规则：
  - 请求 NiceGUI 自身后端（相对 URL）：无跨域问题；
  - 请求第三方接口（绝对 URL）：需目标接口返回`Access-Control-Allow-Origin`等 CORS 响应头；
  - 调试解决方案：后端添加 CORS 中间件（如 FastAPI 的`CORSMiddleware`），或使用代理转发。

### 2. 参数优先级

- `json`参数优先级 > `data`参数：同时传时，`data`会被忽略，且`Content-Type`强制设为`application/json`；
- `params`仅作用于 URL 拼接，不影响请求体（POST/PUT 等方法中`params`仍有效）。

### 3. 异常处理（必须做）

`ui.request`可能抛出以下异常，需逐一捕获：

```python
try:
    response = await ui.request(...)
    # 业务逻辑
except TimeoutError:
    # 超时异常
    ui.notify('请求超时', type='error')
except RuntimeError as e:
    # 网络错误（如断网、跨域）
    ui.notify(f'网络错误：{str(e)}', type='error')
except Exception as e:
    # 其他未知异常
    ui.notify(f'请求失败：{str(e)}', type='error')
```

### 4. 避免 UI 阻塞

- 必须在`async def`函数中调用，且加`await`（同步函数中调用会报错）；

- 发起请求前可禁用按钮、显示加载状态，避免用户重复点击：

  ```python
  async def load_data():
      load_btn.disable()  # 禁用按钮
      try:
          # 请求逻辑
      finally:
          load_btn.enable()  # 无论成功失败，恢复按钮
  ```

### 5. 响应解析注意事项

- `response.json()`/`response.text()`/`response.bytes()`都是异步方法，**必须加`await`**；
- 非 JSON 响应（如 HTML）用`response.text()`，二进制数据（如图片）用`response.bytes()`。

### 6. 生产环境最佳实践

- 避免在`ui.request`中硬编码敏感信息（如 Token），可通过 NiceGUI 的`app.storage`或环境变量传递；
- 设置合理的超时时间（建议 5-10 秒），避免用户长时间等待；
- 验证 SSL 证书（`verify_ssl=True`），禁止生产环境跳过验证；
- 对大文件上传 / 下载，建议用专门的文件接口，避免`ui.request`处理超大请求体。

## 五、与后端`httpx`的对比选择

| 场景                            | 推荐使用`ui.request` | 推荐使用后端`httpx` |
| ------------------------------- | -------------------- | ------------------- |
| 前端交互触发的接口请求          | ✅                    | ❌（易阻塞 UI）      |
| 后端定时任务 / 后台线程调用接口 | ❌（前端无法执行）    | ✅                   |
| 请求第三方接口且无 CORS         | ❌（跨域失败）        | ✅（后端无跨域）     |
| 需要直接操作响应体渲染 UI       | ✅（前端直接处理）    | ❌（需同步 UI）      |

## 六、总结

`ui.request`（HTTP 请求 API）是 NiceGUI 前端侧 HTTP 请求的 “一站式工具”，核心价值是：

1. 纯 Python 代码完成前端请求，无需写 JS；
2. 异步无阻塞，适配 UI 交互场景；
3. 自动处理 JSON 序列化，简化接口调用逻辑。

掌握它的关键是：

- 区分 “前端请求” 与 “后端请求” 的执行位置；
- 重视跨域和异常处理；
- 遵循`async/await`异步范式；
- 结合 UI 状态管理（加载、禁用、提示）提升用户体验。

它是 NiceGUI 连接 “前端交互” 和 “后端 API” 的核心纽带，也是实现 RESTful API 交互的标准方式。

# NiceGUI 中 `ui.request`（UI 线程调度 API）深度解析

`ui.request`（UI 线程调度 API）是 NiceGUI 解决**多线程 / 异步场景下 UI 更新线程安全**的核心工具，也是保证 UI 操作稳定性的关键机制。本文从设计初衷、底层原理、完整语法、使用场景、避坑指南、性能优化等维度，全方位拆解这一 API 的细节与实践。

## 一、设计初衷：为什么需要 UI 线程调度 API？

NiceGUI 基于 FastAPI + Starlette 构建，其 UI 渲染、事件处理全部运行在**主线程（UI 线程）** 中，而 UI 组件（如`label`、`button`、`table`）本身是非线程安全的。

### 核心问题

若直接在**后台线程**（如`threading.Thread`）、**异步任务**（如`asyncio.create_task`）或**定时器**中修改 UI 组件，会导致：

1. 线程竞争：多个线程同时操作 UI 组件，引发数据错乱；
2. 渲染失效：UI 更新不生效，或界面 “假死”；
3. 程序崩溃：极端情况下触发 Python 解释器的线程安全断言，直接终止程序。

### 解决思路

`ui.request` 的核心是 **“调度” 而非 “直接执行”**：将非 UI 线程中要执行的 UI 操作，封装为函数并提交到 UI 主线程的事件循环中，由主线程串行执行，从根本上避免线程安全问题。

## 二、底层原理

### 1. 核心机制：事件循环调度

NiceGUI 内置一个**UI 事件队列**，`ui.request` 会将传入的 UI 操作函数加入队列，UI 主线程的事件循环会按优先级、入队顺序逐个执行队列中的函数：

```plaintext
后台线程/异步任务 → ui.request(UI操作函数) → 加入UI事件队列 → UI主线程事件循环执行 → 更新UI
```

### 2. 关键特性

| 特性       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 线程安全   | 无论调用方是哪个线程，UI 操作最终都在主线程执行              |
| 批量合并   | 短时间内的多个`ui.request`会合并为一次 UI 刷新，减少渲染开销 |
| 优先级调度 | 支持设置`priority`参数，高优先级操作优先执行（如紧急提示）   |
| 超时控制   | 可设置`timeout`，避免 UI 操作因队列阻塞无限等待              |
| 异步兼容   | 返回`asyncio.Future`，支持`await`等待执行完成，适配异步场景  |

## 三、完整语法与参数详解

### 1. 函数签名（官方标准）

```python
def request(
    func: Callable[[], None],  # 必选：待执行的UI操作函数
    *,
    priority: int = 0,         # 可选：执行优先级，默认0
    timeout: float | None = None,  # 可选：超时时间（秒），默认None（无超时）
) -> asyncio.Future:
    """将UI操作函数调度到UI主线程执行"""
```

### 2. 参数深度解析

#### （1）核心参数：`func`

- **类型**：无参数、无返回值的可调用对象（`Callable[[], None]`）；
- **作用**：封装所有需要执行的 UI 更新逻辑（如修改组件属性、添加 / 删除组件、更新样式）；
- **约束**：
  - 函数内部仅能包含 UI 操作，禁止耗时逻辑（如`time.sleep()`、网络请求、复杂计算）；
  - 函数不能有入参（若需传参，需通过闭包 / 默认参数实现）。

#### （2）优先级：`priority`

- **类型**：整数（正 / 负 / 0）；
- **规则**：数值越大，优先级越高，越先被执行；
- **典型场景**：
  - 紧急提示（如系统告警）：`priority=10`；
  - 普通更新（如进度条）：`priority=0`（默认）；
  - 低优先级刷新（如日志打印）：`priority=-5`。

#### （3）超时：`timeout`

- **类型**：浮点数（秒）或`None`；
- **作用**：设置函数在队列中的最大等待时间，超过时间未执行则抛出`asyncio.TimeoutError`；
- **使用场景**：避免因 UI 队列阻塞（如大量低优先级操作堆积）导致关键 UI 更新迟迟不执行。

### 3. 返回值：`asyncio.Future`

`ui.request` 返回一个`asyncio.Future`对象，代表 “待执行的 UI 操作”，支持两种使用方式：

| 使用方式           | 示例                        | 适用场景                                 |
| ------------------ | --------------------------- | ---------------------------------------- |
| 异步等待（推荐）   | `await ui.request(func)`    | 异步函数中，需等待 UI 更新完成后继续逻辑 |
| 同步阻塞（不推荐） | `ui.request(func).result()` | 同步线程中，强制等待执行结果（易卡 UI）  |

## 四、典型使用场景与实战示例

### 场景 1：后台线程更新 UI（最高频）

后台线程执行耗时任务（如数据计算、文件读写），需实时更新 UI 进度 / 状态：

```python
from nicegui import ui
import threading
import time

# 初始化UI组件
progress_bar = ui.progress(0).props('indeterminate=False')
status_label = ui.label('任务未开始').classes('text-gray-500')

def background_task():
    """后台线程执行耗时任务"""
    total_steps = 100
    for step in range(1, total_steps + 1):
        # 模拟耗时操作（如计算、IO）
        time.sleep(0.1)
        
        # 核心：用ui.request调度UI更新到主线程
        ui.request(
            # 闭包传参：通过默认参数绑定当前step值（避免循环变量陷阱）
            lambda current_step=step: (
                progress_bar.set_value(current_step / total_steps),
                status_label.set_text(f'任务进度：{current_step}%').classes('text-blue-500')
            )
        )
    
    # 任务完成后更新UI
    ui.request(lambda: status_label.set_text('任务完成！').classes('text-green-500'))

# 启动后台线程（daemon=True：主线程退出时自动终止）
threading.Thread(target=background_task, daemon=True).start()

ui.run()
```

### 场景 2：异步任务中安全更新 UI

在`async/await`异步函数中，确保 UI 操作在主线程执行（避免异步调度导致的时序问题）：

```python
from nicegui import ui
import asyncio

# 初始化UI组件
counter_label = ui.label('0').classes('text-2xl')

async def async_worker():
    """异步任务（模拟接口调用/异步IO）"""
    for count in range(1, 101):
        # 模拟异步耗时操作（如等待接口响应）
        await asyncio.sleep(0.05)
        
        # 异步场景下，await ui.request确保UI更新完成后再继续
        await ui.request(
            lambda current_count=count: counter_label.set_text(str(current_count))
        )

# 绑定按钮触发异步任务
ui.button('启动异步计数器', on_click=async_worker)

ui.run()
```

### 场景 3：高优先级 UI 更新

当需要优先执行紧急 UI 操作（如告警提示），覆盖普通更新：

```python
from nicegui import ui
import threading
import time

# 初始化UI组件
normal_label = ui.label('普通更新区')
alert_label = ui.label('').classes('text-red-500 text-xl font-bold')

def mixed_priority_task():
    """混合优先级的后台任务"""
    # 低优先级更新（默认0）
    ui.request(lambda: normal_label.set_text('普通更新中...'), priority=0)
    time.sleep(0.1)  # 模拟延迟
    
    # 高优先级更新（优先执行）
    ui.request(lambda: alert_label.set_text('⚠️ 系统紧急告警！'), priority=10)

# 启动后台线程
threading.Thread(target=mixed_priority_task, daemon=True).start()

ui.run()
```

### 场景 4：超时控制（避免 UI 操作阻塞）

若 UI 队列堆积导致操作迟迟不执行，可设置超时时间，触发异常并降级处理：

```python
from nicegui import ui
import asyncio

async def risky_ui_operation():
    """带超时的UI操作"""
    try:
        # 模拟一个可能阻塞的UI操作（实际中应避免）
        future = ui.request(
            lambda: time.sleep(3),  # 错误示例：func中包含耗时操作
            timeout=2  # 超时时间2秒
        )
        await future
    except asyncio.TimeoutError:
        # 超时后降级提示
        ui.notify('UI更新超时！', type='error')

ui.button('执行带超时的UI操作', on_click=risky_ui_operation)

ui.run()
```

## 五、关键避坑指南

### 1. 闭包变量陷阱（最常见）

**问题**：循环中直接引用循环变量，所有回调共享最后一个变量值：

```python
# 错误示例
for i in range(5):
    ui.request(lambda: ui.label(f'当前值：{i}'))  # 所有标签都显示4
```

**解决方案**：通过默认参数绑定当前变量值：

```python
# 正确示例
for i in range(5):
    ui.request(lambda i=i: ui.label(f'当前值：{i}'))  # 依次显示0-4
```

### 2. 禁止在`func`中执行耗时操作

`func`运行在 UI 主线程，耗时操作（如`time.sleep()`、`requests.get()`、复杂循环）会阻塞 UI 渲染，导致界面卡顿：

```python
# 错误示例
ui.request(lambda: (time.sleep(5), ui.label('完成')))  # UI阻塞5秒

# 正确示例：耗时操作放后台，仅UI更新入func
def worker():
    time.sleep(5)  # 耗时操作在后台线程
    ui.request(lambda: ui.label('完成'))  # 仅UI更新入func

threading.Thread(target=worker, daemon=True).start()
```

### 3. 无需嵌套调用`ui.request`

若`func`本身已在 UI 主线程执行（如按钮点击回调、页面加载函数），嵌套`ui.request`会增加不必要的调度开销：

```python
# 冗余示例
ui.button('点击', on_click=lambda: ui.request(lambda: ui.label('点击了')))

# 正确示例
ui.button('点击', on_click=lambda: ui.label('点击了'))
```

### 4. 避免频繁调用（性能优化）

短时间内大量调用`ui.request`会导致 UI 队列堆积，可通过**批量更新**减少调用次数：

```python
# 低效示例：每次循环都调用ui.request
for user in users:
    ui.request(lambda u=user: ui.label(u['name']))

# 高效示例：批量封装UI操作，一次调用ui.request
def render_users():
    for user in users:
        ui.label(user['name'])

ui.request(render_users)
```

### 5. 异步场景优先`await`

在异步函数中，`await ui.request(...)` 可确保 UI 操作完成后再执行后续逻辑，避免时序问题：

```python
async def async_logic():
    # 推荐：等待UI更新完成
    await ui.request(lambda: ui.label('第一步完成'))
    # 后续逻辑依赖UI更新结果
    await asyncio.sleep(1)
    await ui.request(lambda: ui.label('第二步完成'))
```

## 六、与其他 UI 更新方式的对比

| 方式                 | 适用场景                       | 线程安全 | 性能 | 备注                             |
| -------------------- | ------------------------------ | -------- | ---- | -------------------------------- |
| `ui.request`         | 非 UI 线程更新 UI              | ✅ 安全   | 中   | 通用方案，支持优先级 / 超时      |
| 直接修改组件属性     | UI 线程内更新 UI（如按钮回调） | ✅ 安全   | 高   | 无调度开销，推荐优先使用         |
| `component.update()` | 修改组件属性后手动刷新         | ✅ 安全   | 高   | 轻量刷新，仅适用于单个组件       |
| `ui.run_javascript`  | 调用前端 JS 更新 UI            | ✅ 安全   | 中   | 需配合`ui.request`确保主线程执行 |

## 七、性能优化技巧

### 1. 批量 UI 操作

将多个 UI 更新逻辑封装到一个`func`中，减少`ui.request`调用次数：

```python
# 优化前：3次调用
ui.request(lambda: label1.set_text('A'))
ui.request(lambda: label2.set_text('B'))
ui.request(lambda: progress.set_value(0.5))

# 优化后：1次调用
def batch_update():
    label1.set_text('A')
    label2.set_text('B')
    progress.set_value(0.5)

ui.request(batch_update)
```

### 2. 合理设置优先级

仅对紧急操作设置高优先级，避免高优先级操作堆积：

```python
# 紧急告警：高优先级
ui.request(lambda: alert.set_text('告警'), priority=10)
# 普通日志：低优先级
ui.request(lambda: log_label.set_text('日志'), priority=-5)
```

### 3. 复用 UI 组件（而非频繁创建）

频繁创建 / 删除组件会增加渲染开销，可复用现有组件：

```python
# 优化前：每次更新创建新label
ui.request(lambda: ui.label(f'进度：{i}%'))

# 优化后：复用现有label
progress_label = ui.label('进度：0%')
ui.request(lambda: progress_label.set_text(f'进度：{i}%'))
```

## 八、总结

`ui.request`（UI 线程调度 API）是 NiceGUI 处理**跨线程 UI 更新**的 “安全锁”，其核心价值可总结为：

1. **线程安全**：将 UI 操作调度到主线程，避免多线程竞争；
2. **灵活可控**：支持优先级、超时，适配不同场景需求；
3. **性能优化**：批量合并更新，减少渲染开销。

### 最佳实践口诀

- 后台线程更 UI，必用`ui.request`；
- func 只包 UI 逻辑，耗时操作放后台；
- 循环传参用默认，避免变量陷进坑；
- 异步场景加 await，同步直接调就行；
- 批量更新少调用，优先级只给紧急情。

掌握这一 API 的核心逻辑，即可解决 NiceGUI 中 90% 以上的 UI 更新稳定性问题，确保多线程 / 异步场景下界面流畅、无崩溃。