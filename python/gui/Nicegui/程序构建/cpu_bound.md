# NiceGUI的run模块

NiceGUI 的`run`模块是其核心启动入口，负责初始化 Web 服务器、管理应用生命周期、处理配置参数，并协调前端与后端的交互。以下从**核心功能、参数详解、运行机制、高级用法、常见问题** 五个维度详细拆解`run`模块。

### 一、核心定位与基础用法

`run()`是 NiceGUI 应用的启动函数，所有 UI 组件的渲染、事件响应、WebSocket 通信都依赖该函数初始化的运行时环境。

#### 基础示例

```python
from nicegui import ui

ui.label('Hello NiceGUI!')

# 最简启动（默认配置）
ui.run()
```

### 二、`run`模块的核心参数详解

`ui.run()`的参数可分为**服务器配置、网络配置、开发调试、性能优化、生命周期** 五大类，以下是高频参数的详细说明：

| 类别       | 参数名               | 类型     | 默认值    | 核心作用                                                     |
| ---------- | -------------------- | -------- | --------- | ------------------------------------------------------------ |
| 服务器配置 | `title`              | str      | 'NiceGUI' | 浏览器标签页标题                                             |
|            | `favicon`            | str/Path | None      | 浏览器标签页图标（支持本地路径或 URL）                       |
|            | `reload`             | bool     | False     | 开发模式：代码修改后自动重启服务器（仅本地开发用）           |
|            | `uvicorn_log_level`  | str/int  | 'warning' | Uvicorn 日志级别（debug/info/warning/error/critical）        |
| 网络配置   | `host`               | str      | '0.0.0.0' | 绑定的 IP 地址（'0.0.0.0' 允许外网访问，'127.0.0.1' 仅本地访问） |
|            | `port`               | int      | 8080      | 监听端口（冲突时自动递增）                                   |
|            | `ssl_keyfile`        | str/Path | None      | SSL 私钥文件路径（启用 HTTPS）                               |
|            | `ssl_certfile`       | str/Path | None      | SSL 证书文件路径（启用 HTTPS）                               |
| 开发调试   | `show`               | bool     | True      | 启动后自动打开浏览器窗口                                     |
|            | `on_startup`         | Callable | None      | 服务器启动后执行的回调函数                                   |
|            | `on_shutdown`        | Callable | None      | 服务器关闭前执行的回调函数                                   |
| 性能优化   | `storage_secret`     | str      | None      | 本地存储加密密钥（用于`ui.storage.local`）                   |
|            | `max_upload_size`    | int      | 100       | 最大上传文件大小（MB）                                       |
|            | `mount`              | list     | []        | 挂载额外的静态文件目录（如`mount=['/static', './static_files']`） |
| 高级配置   | `api_base_url`       | str      | '/api'    | 后端 API 接口的基础路径                                      |
|            | `websocket_base_url` | str      | '/ws'     | WebSocket 通信的基础路径（前端与后端实时交互）               |
|            | `plugins`            | list     | []        | 加载自定义 NiceGUI 插件                                      |

#### 参数使用示例

```python
from nicegui import ui
from pathlib import Path

def startup():
    print('服务器启动完成！')

def shutdown():
    print('服务器即将关闭！')

ui.label('HTTPS + 自动重载 + 自定义端口')

ui.run(
    host='127.0.0.1',       # 仅本地访问
    port=9000,              # 自定义端口
    reload=True,            # 开发模式自动重载
    show=False,             # 不自动打开浏览器
    title='My App',         # 自定义标题
    favicon=Path('icon.ico'),# 自定义图标
    ssl_keyfile='key.pem',  # HTTPS私钥
    ssl_certfile='cert.pem',# HTTPS证书
    on_startup=startup,     # 启动回调
    on_shutdown=shutdown,   # 关闭回调
    max_upload_size=200,    # 最大上传200MB
)
```

### 三、`run`模块的运行机制

NiceGUI 的`run`模块底层基于**Uvicorn（ASGI 服务器）** + **FastAPI（后端框架）** + **WebSocket（实时通信）**，核心流程如下：

#### 1. 初始化阶段

- 解析`run()`参数，生成 ASGI 服务器配置；
- 创建 FastAPI 应用实例，注册内置 API 路由（如静态文件、WebSocket、UI 渲染接口）；
- 初始化前端资源（Vue.js 组件、CSS 样式），将 Python 定义的 UI 组件转换为前端可渲染的 JSON 结构；
- 若开启`reload`，启动文件监控器（基于`watchfiles`），监听代码文件变化并触发重启。

#### 2. 启动阶段

- 启动 Uvicorn 服务器，绑定指定`host`和`port`；
- 若`show=True`，调用系统默认浏览器打开`http://host:port`；
- 执行`on_startup`回调函数（若有）；
- 建立 WebSocket 连接，实现前后端实时通信（如按钮点击、输入框变化的双向绑定）。

#### 3. 运行阶段

- 接收前端请求：静态文件请求由 FastAPI 直接返回，动态 UI 请求通过 WebSocket 实时同步；
- 处理 Python 事件：前端触发的事件（如`ui.button(on_click=...)`）通过 WebSocket 传递到后端，执行对应的 Python 函数；
- 响应前端更新：Python 中修改 UI 组件（如`ui.label.set_text('new')`）会自动推送到前端，更新页面内容。

#### 4. 关闭阶段

- 捕获终止信号（如`Ctrl+C`），关闭 WebSocket 连接；
- 执行`on_shutdown`回调函数；
- 优雅关闭 Uvicorn 服务器，释放端口和资源。

### 四、高级用法

#### 1. 后台运行（非阻塞）

默认`ui.run()`是阻塞的，若需在后台运行（如结合其他线程 / 进程），可通过`uvicorn`直接启动：

```python
from nicegui import ui
import uvicorn
from threading import Thread

ui.label('后台运行的NiceGUI')

# 创建FastAPI应用实例
app = ui.run_with(
    title='Background App',
    reload=False,
    show=False,
)

# 后台线程启动服务器
def start_server():
    uvicorn.run(app, host='0.0.0.0', port=8080)

Thread(target=start_server, daemon=True).start()

# 主线程继续执行其他逻辑
print('NiceGUI已在后台启动，可访问http://localhost:8080')
while True:
    pass
```

#### 2. 多实例 / 多端口运行

通过`run_with()`创建独立的应用实例，实现多端口启动：

```python
from nicegui import ui
import uvicorn

# 实例1：端口8080
app1 = ui.run_with(title='App 1', show=False)
ui.label('App 1 on 8080')

# 实例2：端口8081
app2 = ui.run_with(title='App 2', show=False)
with ui.page('/'):  # 重新定义根页面
    ui.label('App 2 on 8081')

# 启动两个实例
uvicorn.run(app1, host='0.0.0.0', port=8080)
# 若需同时运行，需在不同线程/进程中启动
```

#### 3. 自定义 ASGI 中间件

通过`run()`的`middleware`参数添加 FastAPI 中间件（如日志、认证）：

```python
from nicegui import ui
from fastapi.middleware.cors import CORSMiddleware

ui.label('带CORS中间件的应用')

ui.run(
    host='0.0.0.0',
    port=8080,
    middleware=[
        {
            'middleware_class': CORSMiddleware,
            'allow_origins': ['*'],  # 允许所有跨域请求（仅测试用）
            'allow_credentials': True,
            'allow_methods': ['*'],
            'allow_headers': ['*'],
        }
    ]
)
```

#### 4. 集成到现有 FastAPI 应用

将 NiceGUI 作为子应用挂载到现有 FastAPI 项目：

```python
from fastapi import FastAPI
from nicegui import ui, APIRouter

# 现有FastAPI应用
main_app = FastAPI()

# 定义NiceGUI路由
nicegui_router = APIRouter(prefix='/nicegui')
ui.run_with(router=nicegui_router, show=False)

# 添加NiceGUI页面
@ui.page('/')
def home():
    ui.label('集成到FastAPI的NiceGUI')

# 挂载到主应用
main_app.include_router(nicegui_router)

# 启动主应用（替代ui.run()）
if __name__ == '__main__':
    import uvicorn
    uvicorn.run(main_app, host='0.0.0.0', port=8000)
```

### 五、常见问题与解决方案

#### 1. 端口被占用

- 现象：启动时报`OSError: [Errno 98] Address already in use`；
- 解决：
  - 指定未被占用的端口：`ui.run(port=8081)`；
  - 启用自动端口递增：`ui.run(port=0)`（系统随机分配可用端口）。

#### 2. `reload`模式不生效

- 原因：`reload`仅监听当前工作目录下的文件，或依赖`watchfiles`未安装；
- 解决：
  - 安装依赖：`pip install watchfiles`；
  - 指定监听目录：`ui.run(reload=True, reload_dirs=['./my_app'])`。

#### 3. 外网无法访问

- 原因：`host`设为`127.0.0.1`（仅本地），或防火墙 / 端口未开放；
- 解决：
  - 设`host='0.0.0.0'`：`ui.run(host='0.0.0.0')`；
  - 开放服务器端口（如阿里云 / 腾讯云安全组配置）。

#### 4. HTTPS 启动失败

- 原因：SSL 证书 / 私钥路径错误，或证书格式不合法；

- 解决：

  - 检查证书路径：确保`ssl_certfile`和`ssl_keyfile`指向有效文件；

  - 生成自签名证书（测试用）：

    ```bash
    openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
    ```

### 六、总结

NiceGUI 的`run`模块是连接 Python 后端与前端 UI 的核心桥梁，通过封装 Uvicorn/FastAPI 简化了 Web 应用的启动流程，同时提供丰富的参数和扩展能力：

- 基础场景：直接使用`ui.run()`快速启动；
- 开发调试：开启`reload=True`提升效率；
- 生产环境：配置`host='0.0.0.0'`、HTTPS、`reload=False`保证稳定性；
- 高级集成：通过`run_with()`或直接使用 Uvicorn 将 NiceGUI 嵌入现有项目。

掌握`run`模块的参数和运行机制，能灵活适配本地开发、内网部署、外网发布等不同场景的需求。

# ui.run()与run模块对比

要清晰对比 `ui.run()` 和 NiceGUI 的 `run` 模块，首先需明确**二者的本质关系**：`ui.run()` 是 `run` 模块对外暴露的核心入口函数，而 `run` 模块是支撑该函数运行的完整功能集合（包含底层逻辑、辅助函数、配置解析、生命周期管理等）。以下从**定义与定位、功能范围、使用场景、底层逻辑、扩展能力** 五个维度展开详细对比，并补充核心差异表。

### 一、核心定义与定位

| 维度     | `ui.run()`                                                | `run` 模块                                                   |
| -------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| 本质     | 单入口函数（API），是 `run` 模块的 “门面”                 | 完整的功能模块（Python 包 / 模块），包含 `run()` 及底层支撑逻辑 |
| 定位     | 面向开发者的极简启动接口，封装了复杂逻辑                  | 支撑 `ui.run()` 运行的核心引擎，处理配置、服务器启动、生命周期等 |
| 访问方式 | 从 `nicegui` 直接导入：`from nicegui import ui; ui.run()` | 需导入模块：`from nicegui import run`（或内部调用）          |
| 核心目标 | 让开发者一行代码启动应用                                  | 实现应用启动的全流程逻辑（初始化、运行、关闭）               |

### 二、功能范围对比

`ui.run()` 是 `run` 模块的 “子集”——`run` 模块包含 `ui.run()` 的所有功能，且提供更多底层扩展能力。

#### 1. `ui.run()` 的功能边界

`ui.run()` 仅聚焦**应用启动的 “最终执行”**，核心能力是：

- 接收开发者传入的启动参数（如 `host`、`port`、`reload` 等）；
- 调用 `run` 模块的底层函数完成服务器启动；
- 对外暴露极简的调用方式（无需关注模块内部逻辑）。

其功能是 “单点式” 的：仅负责**启动应用**，不提供额外的底层操作接口。

#### 2. `run` 模块的功能边界

`run` 模块是 NiceGUI 应用启动的 “全流程引擎”，功能覆盖**配置解析、服务器初始化、生命周期管理、扩展能力** 四大类，核心组件 / 函数包括：

| 模块内核心组件     | 功能说明                                                     |
| ------------------ | ------------------------------------------------------------ |
| `run()` 函数       | `ui.run()` 的底层实现（`ui.run()` 本质是 `run.run()` 的别名 / 封装） |
| `run_with()` 函数  | 创建独立的应用实例（用于多实例、集成现有 FastAPI 应用）      |
| `Config` 类        | 解析 `ui.run()` 参数，生成标准化的服务器配置（如 ASGI 配置、日志级别） |
| `startup/shutdown` | 生命周期钩子函数的注册与执行逻辑                             |
| `reload` 相关逻辑  | 文件监控、热重载的实现（基于 `watchfiles`）                  |
| `server` 子模块    | Uvicorn 服务器的启动、关闭逻辑                               |
| `websocket` 子模块 | 前后端实时通信的初始化与管理                                 |

简单来说：`ui.run()` 是 “用户层接口”，`run` 模块是 “引擎层实现”—— 开发者调用 `ui.run()` 时，实际是触发了 `run` 模块的一整套流程。

### 三、使用场景对比

| 场景类型            | 推荐使用 `ui.run()`                      | 推荐使用 `run` 模块                  |
| ------------------- | ---------------------------------------- | ------------------------------------ |
| 快速原型开发        | ✅ 一行代码启动，无需关注底层             | ❌ 过度复杂，无必要                   |
| 标准单机部署        | ✅ 传入 `host`/`port`/`reload` 等参数即可 | ❌ 直接用 `ui.run()` 已足够           |
| 多实例 / 多端口运行 | ❌ 无法直接实现                           | ✅ 调用 `run.run_with()` 创建独立实例 |
| 集成现有 FastAPI    | ❌ 仅能启动独立应用                       | ✅ 用 `run.run_with()` 挂载到主应用   |
| 后台非阻塞运行      | ❌ 默认阻塞，需额外封装                   | ✅ 调用 `run` 模块的 `server` 子模块  |
| 自定义服务器配置    | ❌ 仅能通过参数传递有限配置               | ✅ 直接修改 `Config` 类或服务器参数   |
| 扩展生命周期逻辑    | ❌ 仅能传入 `on_startup`/`on_shutdown`    | ✅ 注册自定义钩子、修改启动流程       |

#### 场景示例对比

##### 场景 1：快速启动（优先 `ui.run()`）

```python
from nicegui import ui
ui.label('Hello World')
ui.run()  # 极简，无需接触run模块
```

##### 场景 2：多实例运行（必须用 `run` 模块）

```python
from nicegui import run, ui
import uvicorn
from threading import Thread

# 实例1：通过run模块创建独立应用
app1 = run.run_with(title='App 1', show=False)
with ui.page('/'):
    ui.label('App 1 (8080)')

# 实例2：另一个独立应用
app2 = run.run_with(title='App 2', show=False)
with ui.page('/'):  # 重新定义根页面
    ui.label('App 2 (8081)')

# 后台启动两个实例
Thread(target=uvicorn.run, args=(app1,), kwargs={'host': '0.0.0.0', 'port': 8080}, daemon=True).start()
Thread(target=uvicorn.run, args=(app2,), kwargs={'host': '0.0.0.0', 'port': 8081}, daemon=True).start()

print('两个实例已启动：8080/8081')
while True:
    pass
```

### 四、底层逻辑对比

#### 1. `ui.run()` 的执行逻辑

`ui.run()` 是一个**封装函数**，核心逻辑仅 3 步：

```python
# 伪代码：ui.run() 的简化实现
def run(**kwargs):
    from nicegui import run as run_module
    # 1. 转发参数到run模块的核心run函数
    return run_module.run(** kwargs)
```

可见，`ui.run()` 本质是 `run` 模块中 `run()` 函数的 “别名”，目的是降低开发者的使用成本（无需导入 `run` 模块，直接通过 `ui` 调用）。

#### 2. `run` 模块的执行逻辑

`run` 模块的 `run()` 函数是**全流程引擎**，执行逻辑分为 6 步：

```python
# 伪代码：run模块的核心run()函数
def run(** kwargs):
    # 1. 解析参数，生成标准化配置（Config类）
    config = Config(**kwargs)
    
    # 2. 初始化FastAPI应用实例
    app = create_fastapi_app(config)
    
    # 3. 注册静态文件、WebSocket、UI路由
    register_routes(app, config)
    
    # 4. 初始化热重载（若开启）
    if config.reload:
        setup_reload(app, config)
    
    # 5. 启动Uvicorn服务器（阻塞执行）
    start_uvicorn_server(app, config)
    
    # 6. 注册并执行生命周期钩子
    execute_hooks(config.on_startup)
    try:
        server.wait()
    finally:
        execute_hooks(config.on_shutdown)
```

简言之：`ui.run()` 是 “调用入口”，`run` 模块是 “执行引擎”—— 前者负责接收用户参数，后者负责把参数转化为可运行的服务器实例。

### 五、扩展能力对比

| 扩展需求              | `ui.run()` 的支持程度           | `run` 模块的支持程度 | 实现方式                                                     |
| --------------------- | ------------------------------- | -------------------- | ------------------------------------------------------------ |
| 修改服务器日志级别    | 支持（通过参数）                | 完全支持             | `ui.run(uvicorn_log_level='debug')` / `run.Config(uvicorn_log_level='debug')` |
| 自定义 ASGI 中间件    | 有限支持（参数传递）            | 完全支持             | `ui.run(middleware=[...])` / 直接修改 FastAPI 实例的 middleware |
| 自定义 WebSocket 路径 | 支持（通过参数）                | 完全支持             | `ui.run(websocket_base_url='/custom_ws')` / 修改 Config 类参数 |
| 多实例 / 非阻塞启动   | 不支持                          | 完全支持             | `run.run_with()` 创建独立实例 + 线程启动                     |
| 嵌入现有 Web 框架     | 不支持                          | 完全支持             | `run.run_with(router=APIRouter())` 挂载到 FastAPI/Starlette  |
| 自定义服务器启动逻辑  | 不支持                          | 完全支持             | 直接调用 `run.server.start()`，自定义启动流程                |
| 扩展生命周期钩子      | 有限支持（仅 startup/shutdown） | 完全支持             | 注册自定义钩子到 `run` 模块的生命周期管理器                  |

### 六、核心差异总结表

| 对比维度   | `ui.run()`                            | `run` 模块                                 |
| ---------- | ------------------------------------- | ------------------------------------------ |
| 本质       | 封装后的入口函数                      | 完整的启动引擎模块                         |
| 功能       | 仅负责启动应用（参数传递 + 触发执行） | 处理启动全流程（配置、初始化、运行、关闭） |
| 使用复杂度 | 极低（一行代码）                      | 较高（需理解底层逻辑）                     |
| 适用场景   | 快速开发、标准部署                    | 高级扩展、集成、定制化启动                 |
| 扩展能力   | 有限（仅通过参数）                    | 完全开放（可修改任意底层逻辑）             |
| 依赖关系   | 依赖 `run` 模块实现                   | 无依赖（核心底层模块）                     |
| 阻塞性     | 默认阻塞（无法直接后台运行）          | 可灵活控制（支持阻塞 / 非阻塞）            |

### 七、实践建议

1. **90% 的场景用 `ui.run()`**：

   快速开发、单机部署、原型验证等场景，`ui.run()` 已足够，无需接触`run`模块，降低心智负担。

2. **高级场景用 `run` 模块**：

   当需要多实例运行、集成现有 FastAPI 应用、自定义服务器启动逻辑、非阻塞运行时，直接使用`run`模块的`run_with()`、`Config`类等底层接口。

3. **不要重复造轮子**：

   `ui.run()`是`run`模块的最佳实践封装，除非有定制化需求，否则无需绕过`ui.run()`直接调用`run`模块的核心函数。

### 补充：源码层面的关联

NiceGUI 的源码中，`ui.run()` 的定义如下（简化版）：

```python
# nicegui/__init__.py
from .run import run as _run

class UI:
    def run(self, **kwargs):
        return _run(** kwargs)

ui = UI()
```

可见，`ui.run()` 本质是直接调用 `run` 模块的 `run()` 函数 —— 二者是 “接口” 与 “实现” 的关系，而非并列关系。

# run模块中的函数

以下将按**核心启动函数、应用实例创建函数、辅助工具函数、内部支撑函数（关键暴露接口）** 分类展开，每个函数包含「功能概述」「语法格式」「参数说明」「使用示例」「适用场景」，确保覆盖实际开发中常用及关键的接口。

## 一、核心启动函数（对外核心入口）

### 1. `run()` 函数

#### 功能概述

`run()` 是 `run` 模块的核心启动函数，也是 `ui.run()` 的底层实现，负责解析配置、初始化 ASGI 应用、启动 Uvicorn 服务器，并管理应用全生命周期（启动、运行、关闭），默认以阻塞方式运行。

#### 语法格式

```python
from nicegui import run

def run(
    host: str = '0.0.0.0',
    port: int = 8080,
    title: str = 'NiceGUI',
    favicon: Optional[Union[str, Path]] = None,
    reload: bool = False,
    reload_dirs: Optional[List[str]] = None,
    reload_includes: Optional[List[str]] = None,
    reload_excludes: Optional[List[str]] = None,
    show: bool = True,
    on_startup: Optional[Callable] = None,
    on_shutdown: Optional[Callable] = None,
    uvicorn_log_level: Union[str, int] = 'warning',
    ssl_keyfile: Optional[Union[str, Path]] = None,
    ssl_certfile: Optional[Union[str, Path]] = None,
    ssl_keyfile_password: Optional[str] = None,
    storage_secret: Optional[str] = None,
    max_upload_size: int = 100,
    mount: Optional[List[Union[str, Path]]] = None,
    api_base_url: str = '/api',
    websocket_base_url: str = '/ws',
    plugins: Optional[List[Any]] = None,
    middleware: Optional[List[Dict[str, Any]]] = None,
    **kwargs
) -> None:
```

#### 核心参数

| 参数名                       | 核心作用                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| `host`/`port`                | 绑定服务器 IP 和端口，`0.0.0.0`允许外网访问，`port=0`自动分配可用端口 |
| `reload`                     | 是否开启开发热重载（基于 watchfiles），代码修改后自动重启服务器 |
| `show`                       | 是否启动后自动打开系统默认浏览器                             |
| `on_startup`/`on_shutdown`   | 应用启动后 / 关闭前执行的回调函数                            |
| `ssl_keyfile`/`ssl_certfile` | 指定 SSL 私钥和证书路径，启用 HTTPS 协议                     |
| `max_upload_size`            | 设置最大文件上传大小（单位：MB）                             |
| `middleware`                 | 传入 FastAPI 中间件配置，扩展应用功能（如 CORS 跨域、日志记录） |

#### 使用示例

```python
from nicegui import run

def startup_callback():
    print("应用已成功启动！")

# 启动NiceGUI应用（底层直接调用run模块）
run(
    host='127.0.0.1',
    port=9000,
    title='My NiceGUI App',
    reload=True,
    show=False,
    on_startup=startup_callback,
    max_upload_size=200
)
```

#### 适用场景

- 直接启动独立的 NiceGUI 应用（替代`ui.run()`，功能完全一致）；
- 需直接操作`run`模块，不依赖`ui`实例的场景。

## 二、应用实例创建函数（高级扩展核心）

### 1. `run_with()` 函数

#### 功能概述

`run_with()` 是`run`模块的高级核心函数，用于**创建独立的、可复用的 NiceGUI 应用实例（FastAPI/ASGI 实例）**，而非直接启动服务器。支持多实例创建、挂载到现有 Web 框架（FastAPI/Starlette）、后台非阻塞运行等高级场景。

#### 语法格式

```python
from nicegui import run
from fastapi import APIRouter

def run_with(
    router: Optional[APIRouter] = None,
    title: str = 'NiceGUI',
    favicon: Optional[Union[str, Path]] = None,
    storage_secret: Optional[str] = None,
    max_upload_size: int = 100,
    api_base_url: str = '/api',
    websocket_base_url: str = '/ws',
    plugins: Optional[List[Any]] = None,
    middleware: Optional[List[Dict[str, Any]]] = None,
    **kwargs
) -> FastAPI:
```

#### 核心参数

| 参数名            | 核心作用                                                     |
| ----------------- | ------------------------------------------------------------ |
| `router`          | 自定义 FastAPI APIRouter 实例，用于将 NiceGUI 挂载到指定路由前缀（如`/app`） |
| `title`/`favicon` | 应用标题和浏览器图标，与`run()`函数一致                      |
| `storage_secret`  | 本地存储加密密钥，用于`ui.storage.local`的安全存储           |
| 其他参数          | 与`run()`函数对应参数功能一致，用于配置应用实例属性          |

#### 返回值

返回一个**FastAPI 应用实例**，可直接通过 Uvicorn 启动，或挂载到其他 Web 应用中。

#### 使用示例

##### 示例 1：创建独立应用实例并后台启动

```python
from nicegui import run, ui
import uvicorn
from threading import Thread

# 1. 创建NiceGUI应用实例（不直接启动服务器）
app = run.run_with(
    title='Background App',
    storage_secret='my_secure_secret_123'
)

# 2. 定义UI页面
ui.label('这是通过run_with()创建的后台应用')

# 3. 后台线程启动应用
def start_server():
    uvicorn.run(app, host='0.0.0.0', port=8080)

Thread(target=start_server, daemon=True).start()

# 主线程继续执行其他逻辑
print("NiceGUI应用已在后台启动，访问地址：http://localhost:8080")
while True:
    pass
```

##### 示例 2：挂载到现有 FastAPI 应用

```python
from fastapi import FastAPI, APIRouter
from nicegui import run, ui

# 1. 现有主FastAPI应用
main_app = FastAPI(title='主应用')

# 2. 创建NiceGUI路由（前缀：/nicegui）
ng_router = APIRouter(prefix='/nicegui')
# 3. 将NiceGUI绑定到自定义路由
ng_app = run.run_with(router=ng_router)

# 4. 定义NiceGUI页面
@ui.page('/')
def ng_home():
    ui.label('这是挂载到主应用的NiceGUI子应用')

# 5. 挂载NiceGUI路由到主应用
main_app.include_router(ng_router)

# 6. 启动主应用
if __name__ == '__main__':
    import uvicorn
    uvicorn.run(main_app, host='0.0.0.0', port=8000)
```

#### 适用场景

- 多实例 / 多端口运行 NiceGUI 应用；
- 将 NiceGUI 集成到现有 FastAPI/Starlette 项目；
- 非阻塞启动 NiceGUI（后台运行，不阻塞主线程）；
- 自定义路由前缀，实现多应用隔离部署。

## 三、辅助工具函数（实用支撑接口）

### 1. `stop()` 函数

#### 功能概述

`stop()` 函数用于**优雅关闭正在运行的 NiceGUI 应用**，主动触发应用关闭流程：关闭 WebSocket 连接、执行`on_shutdown`回调、释放端口资源、停止 Uvicorn 服务器，适用于程序化关闭应用的场景。

#### 语法格式

```python
from nicegui import run

def stop() -> None:
```

#### 使用示例

```python
from nicegui import run, ui
import time
import threading

# 定义关闭回调
def shutdown_callback():
    print("应用即将关闭，正在清理资源...")

# 1. 启动应用
def start_app():
    run.run(
        host='127.0.0.1',
        port=8080,
        on_shutdown=shutdown_callback
    )

# 2. 3秒后自动关闭应用
def auto_stop():
    time.sleep(3)
    print("触发自动关闭...")
    run.stop()  # 程序化关闭应用

# 后台启动应用和自动关闭线程
threading.Thread(target=start_app, daemon=True).start()
threading.Thread(target=auto_stop, daemon=True).start()

while True:
    pass
```

#### 适用场景

- 定时关闭 NiceGUI 应用；
- 基于条件判断关闭应用（如用户触发、任务完成后）；
- 多线程 / 多进程场景下，主动终止应用实例。

### 2. `reload()` 函数

#### 功能概述

`reload()` 函数用于**手动触发应用热重载**（仅在`reload=True`模式下生效），无需等待文件监控器检测到代码变化，可程序化触发应用重启，适用于开发调试场景。

#### 语法格式

```python
from nicegui import run

def reload() -> None:
```

#### 使用示例

```python
from nicegui import run, ui

# 启动应用并开启热重载
ui.label('手动触发热重载示例')

# 按钮点击时手动触发重载
ui.button('重启应用', on_click=lambda: run.reload())

run.run(
    host='127.0.0.1',
    port=8080,
    reload=True  # 必须开启热重载模式才生效
)
```

#### 适用场景

- 开发调试时，手动触发应用重启（无需修改文件）；
- 程序化触发重载（如配置文件更新后，自动调用`reload()`）；
- 补充文件监控器的不足（如监控不到的动态文件变更）。

## 四、内部支撑函数（关键暴露接口，进阶使用）

### 1. `create_app()` 函数

#### 功能概述

`create_app()` 是`run`模块的内部核心支撑函数，用于**创建原始的 FastAPI 应用实例**，并完成基础配置（注册静态文件路由、WebSocket 接口、API 基础路径等），是`run()`和`run_with()`的底层依赖函数，适用于高度定制化应用实例的场景。

#### 语法格式

```python
from nicegui import run
from fastapi import APIRouter

def create_app(
    router: Optional[APIRouter] = None,
    title: str = 'NiceGUI',
    favicon: Optional[Union[str, Path]] = None,
    storage_secret: Optional[str] = None,
    max_upload_size: int = 100,
    api_base_url: str = '/api',
    websocket_base_url: str = '/ws',
    plugins: Optional[List[Any]] = None,
    middleware: Optional[List[Dict[str, Any]]] = None,
) -> FastAPI:
```

#### 使用示例

```python
from nicegui import run
import uvicorn

# 高度定制化创建FastAPI应用实例
app = run.create_app(
    title='Custom App',
    max_upload_size=300,
    api_base_url='/custom_api',
    websocket_base_url='/custom_ws'
)

# 手动启动服务器
uvicorn.run(app, host='0.0.0.0', port=8080)
```

#### 适用场景

- 高度定制化 NiceGUI 应用实例（如自定义 API 路径、WebSocket 路径）；
- 绕过`run()`和`run_with()`的封装，直接操作原始 FastAPI 实例；
- 扩展 NiceGUI 的底层应用配置（如添加自定义路由、全局依赖）。

### 2. `setup_reload()` 函数

#### 功能概述

`setup_reload()` 函数用于**手动配置应用热重载机制**，指定监控的目录、包含 / 排除的文件类型，是`run()`函数中`reload=True`的底层实现，适用于自定义热重载规则的场景。

#### 语法格式

```python
from nicegui import run
from fastapi import FastAPI

def setup_reload(
    app: FastAPI,
    reload_dirs: Optional[List[str]] = None,
    reload_includes: Optional[List[str]] = None,
    reload_excludes: Optional[List[str]] = None,
    **kwargs
) -> None:
```

#### 适用场景

- 自定义热重载监控目录（如仅监控`./src`目录，不监控其他目录）；
- 指定热重载包含 / 排除的文件类型（如仅监控`.py`和`.html`文件）；
- 手动启用热重载，不依赖`run()`函数的`reload`参数。

## 五、run 模块函数汇总表

| 函数分类         | 函数名           | 核心功能                                                    | 对外暴露   | 适用场景                                  |
| ---------------- | ---------------- | ----------------------------------------------------------- | ---------- | ----------------------------------------- |
| 核心启动函数     | `run()`          | 解析配置、启动 Uvicorn 服务器、管理应用生命周期（阻塞运行） | 是         | 独立应用启动、替代`ui.run()`              |
| 应用实例创建函数 | `run_with()`     | 创建独立的 FastAPI 应用实例（不启动服务器）                 | 是         | 多实例运行、集成现有 Web 框架、非阻塞启动 |
| 辅助工具函数     | `stop()`         | 优雅关闭正在运行的 NiceGUI 应用                             | 是         | 程序化关闭、定时关闭、条件触发关闭        |
| 辅助工具函数     | `reload()`       | 手动触发应用热重载（仅重载模式下生效）                      | 是         | 开发调试、程序化重启应用                  |
| 内部支撑函数     | `create_app()`   | 创建原始 FastAPI 应用实例，完成基础配置                     | 是（进阶） | 高度定制化应用实例、扩展底层配置          |
| 内部支撑函数     | `setup_reload()` | 手动配置热重载规则，指定监控目录 / 文件类型                 | 是（进阶） | 自定义热重载策略、精细化文件监控          |

## 六、关键说明

1. **对外核心函数优先级**：`run()`（快速启动）> `run_with()`（高级扩展）> `stop()`/`reload()`（辅助操作），90% 的场景只需使用前 3 个对外暴露函数；
2. **内部函数使用场景**：`create_app()`和`setup_reload()`为进阶接口，仅在需要高度定制化 NiceGUI 底层配置时使用，普通开发无需直接调用；
3. **与`ui`模块的关联**：`ui.run()`是`run.run()`的封装别名，`ui`模块未提供额外的启动 / 实例创建能力，所有核心逻辑均来自`run`模块的函数。

### 总结

1. NiceGUI `run` 模块的函数分为「核心启动、实例创建、辅助工具、内部支撑」四大类，覆盖从简单启动到高级定制的全场景；
2. 核心常用函数：`run()`（启动应用）、`run_with()`（创建实例）、`stop()`（关闭应用）、`reload()`（手动重载）；
3. 进阶函数：`create_app()`（自定义应用实例）、`setup_reload()`（自定义热重载），适用于高度定制化需求；
4. 所有函数均围绕「简化 Web 应用启动」和「提升扩展灵活性」设计，底层依赖 FastAPI 和 Uvicorn，保持了 Python Web 开发的一致性。

# `cpu_bound` 函数全面解析

`cpu_bound` 是 NiceGUI 框架中 `run` 模块下的核心函数，专门用于**异步执行 CPU 密集型任务**，解决 UI 主线程阻塞、界面无响应的问题。以下从核心特性、工作原理、使用规则、示例拆解等维度详细说明：

#### 一、核心定位与解决的问题

CPU 密集型任务（如复杂计算、大数据处理、循环运算）会长时间占用 CPU，若直接在 NiceGUI 的事件循环（UI 主线程）中执行，会导致：

- UI 界面卡死（按钮点击、输入等操作无响应）；
- 事件循环阻塞，其他异步任务（如网络请求、定时器）无法执行。

`cpu_bound` 的核心作用：**将 CPU 密集型任务分发到独立的子进程中执行**，主线程（UI）可继续响应交互，任务完成后通过异步 Future 返回结果。

#### 二、工作原理

1. **进程隔离**：

   `cpu_bound`基于 Python 的多进程（而非多线程）实现 —— 因为 Python 的 GIL（全局解释器锁）会导致多线程无法真正并行执行 CPU 密集型任务，而多进程可绕过 GIL，充分利用多核 CPU。

2. **序列化传输**：

   子进程与主进程之间的参数 / 结果传递依赖`pickle` 序列化：

   - 主进程将待执行的函数、传入的参数序列化后，发送给子进程；
   - 子进程执行函数，将结果序列化后返回主进程；
   - 主进程反序列化结果，通过 Future 对象让异步函数（如示例中的 `handle_click`）获取结果。

3. **异步封装**：

   函数返回一个可`await`的 Future 对象，符合 NiceGUI 的异步事件处理逻辑，无需手动管理进程的启动 / 等待。

#### 三、使用规则与最佳实践

官方明确的约束和建议是使用该函数的关键：

1. **函数类型要求**：
   - 优先使用「无状态的自由函数（顶层函数）」或「静态方法」，避免使用类实例方法、闭包；
   - 原因：类实例 / 闭包包含上下文引用，`pickle` 序列化时易出错（如无法序列化 UI 组件、类实例）。
2. **参数 / 返回值要求**：
   - 参数和返回值必须是 `pickle` 可序列化的类型（如 int/float/str/list/dict 等基础类型）；
   - 禁止传递 UI 组件（如 `ui.button`、`ui.label`）、文件句柄、网络连接等不可序列化对象。
3. **数据传递方式**：
   - 所有需要处理的数据必须通过参数显式传入，禁止依赖全局变量、类属性；
   - 结果必须通过函数返回值传递，禁止直接修改全局 / 类属性（子进程无法修改主进程的变量）。

#### 四、示例代码逐行拆解

以官方示例为例，理解每一步的设计逻辑：

```python
import time
from nicegui import run, ui

# 1. 定义CPU密集型任务：自由函数+纯基础类型参数/返回值
def compute_sum(a: float, b: float) -> float:
    time.sleep(1)  # 模拟耗时计算（替代真实的CPU密集逻辑）
    return a + b   # 结果通过返回值传递，不依赖任何外部状态

# 2. 异步事件处理函数：UI交互的入口
async def handle_click():
    # 3. 调用cpu_bound：传入函数+参数，await获取结果
    result = await run.cpu_bound(compute_sum, 1, 2)
    # 4. 结果回显到UI：主进程处理UI更新
    ui.notify(f'Sum is {result}')

# 5. UI组件绑定异步处理函数
ui.button('Compute', on_click=handle_click)

ui.run()
```

- `compute_sum` 是纯函数（无外部依赖），符合序列化要求；
- `handle_click` 是异步函数，保证 UI 线程不阻塞；
- `run.cpu_bound(compute_sum, 1, 2)` 中，`1`和`2`是基础类型，可被 pickle 序列化；
- 结果通过返回值获取后，在主进程中更新 UI（子进程无法直接操作 UI）。

#### 五、常见错误与避坑

1. 错误示例 1：依赖全局变量（子进程无法读取主进程全局变量的最新值）

```python
# 错误写法
total = 0
def compute_sum(a, b):
    global total
    total = a + b  # 子进程修改的是自己的全局变量，主进程无法获取
    return None

async def handle_click():
    await run.cpu_bound(compute_sum, 1, 2)
    ui.notify(f'Sum is {total}')  # 输出仍为0
```

2. 错误示例 2：传递 UI 组件（pickle 序列化失败）

```python
# 错误写法
def compute_sum(a, b, label):
    result = a + b
    label.set_text(result)  # 试图修改UI组件，直接报错
    return result

async def handle_click():
    label = ui.label()
    await run.cpu_bound(compute_sum, 1, 2, label)  # 序列化label失败
```

