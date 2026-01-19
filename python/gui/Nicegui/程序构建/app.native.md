# NiceGUI 中 app.native 的详细解析

`app.native`是 NiceGUI 暴露底层原生组件 / 实例的核心属性，其本质是**对 NiceGUI 封装的底层框架（FastAPI、Quasar、Uvicorn 等）的直接访问入口**。通过`app.native`，开发者可以突破 NiceGUI 的高层封装，直接操作底层框架的核心功能，实现定制化、高性能的扩展开发。以下从核心结构、关键属性、使用场景、进阶示例等维度全面解析：

------

## 一、核心定位与底层依赖

NiceGUI 的底层技术栈：

- 后端：基于 FastAPI（ASGI Web 框架）+ Uvicorn（ASGI 服务器）；
- 前端：基于 Quasar（Vue.js 组件库）+ WebSocket（前后端实时通信）；
- `app.native`的核心作用：将这些底层框架的实例 / API 暴露给开发者，实现「高层封装不够用，直接操作底层」的需求。

`app.native`的结构（简化版）：

```python
app.native = {
    'app': FastAPI实例,       # 核心：FastAPI应用实例
    'uvicorn_config': Uvicorn配置对象,  # Uvicorn服务器配置
    'websocket_manager': 内置WebSocket管理器,  # 前后端实时通信核心
    'quasar_config': Quasar前端配置,  # 前端Quasar框架配置
}
```

------

## 二、核心属性详解

### 1. `app.native.app`：FastAPI 实例（最常用）

`app.native.app`是 NiceGUI 底层的 FastAPI 应用实例，完全兼容 FastAPI 的所有 API，是`app.native`中最核心、最常用的属性。

#### 核心能力：

- 注册 FastAPI 原生路由（补充 NiceGUI 的`app.urls`）；
- 添加 FastAPI 中间件（如 CORS、认证、日志）；
- 配置 FastAPI 的依赖、响应模型、文档（Swagger/ReDoc）；
- 访问 FastAPI 的请求 / 响应生命周期、状态管理。

#### 示例 1：注册 FastAPI 原生路由

（补充 NiceGUI 路由的不足，如高性能 API、OpenAPI 文档自动生成）

```python
from nicegui import app, ui
from pydantic import BaseModel

# 定义FastAPI响应模型（自动生成OpenAPI文档）
class UserResponse(BaseModel):
    id: int
    name: str
    email: str

# 通过app.native.app直接注册FastAPI路由
@app.native.app.get('/api/fastapi/user/{user_id}', response_model=UserResponse)
def fastapi_user_api(user_id: int):
    # 原生FastAPI路由，支持自动参数校验、响应模型、Swagger文档
    return {'id': user_id, 'name': '张三', 'email': f'zhangsan{user_id}@example.com'}

ui.label('访问 http://localhost:8080/docs 查看FastAPI自动生成的API文档')
ui.run()
```

- 访问`http://localhost:8080/docs`可看到 FastAPI 自动生成的 Swagger 文档（NiceGUI 的`app.urls`注册的路由不会出现在该文档中）；
- 优势：FastAPI 原生路由支持参数校验、响应模型、依赖注入等高级特性，适合高性能 API 开发。

#### 示例 2：添加 FastAPI 中间件

（如全局日志、CORS 跨域、身份验证）

```python
from fastapi import Request
from fastapi.middleware.cors import CORSMiddleware
import time

# 1. 添加CORS中间件（解决跨域）
app.native.app.add_middleware(
    CORSMiddleware,
    allow_origins=['*'],  # 生产环境指定具体域名
    allow_credentials=True,
    allow_methods=['*'],
    allow_headers=['*'],
)

# 2. 添加自定义日志中间件
@app.native.app.middleware('http')
async def log_middleware(request: Request, call_next):
    start_time = time.time()
    # 执行请求处理
    response = await call_next(request)
    # 记录日志
    process_time = time.time() - start_time
    response.headers['X-Process-Time'] = str(process_time)
    print(f'请求路径：{request.url.path}，耗时：{process_time:.2f}秒')
    return response

ui.label('已添加FastAPI全局中间件')
ui.run()
```

### 2. `app.native.uvicorn_config`：Uvicorn 配置对象

`app.native.uvicorn_config`是 NiceGUI 启动时创建的 Uvicorn 服务器配置实例，包含服务器的所有运行参数（端口、主机、工作进程、日志级别等）。

#### 核心能力：

- 动态修改 Uvicorn 配置（如运行时调整日志级别）；
- 查看当前服务器的运行状态（如绑定的地址、工作进程数）。

#### 示例：查看 / 修改 Uvicorn 配置

```python
from nicegui import app, ui

# 启动前查看默认Uvicorn配置
print('默认Uvicorn配置：', app.native.uvicorn_config)

# 自定义Uvicorn配置并启动
ui.run(
    host='0.0.0.0',
    port=9000,
    uvicorn_logging_level='debug',
    workers=2
)

# 启动后查看修改后的配置
print('启动后Uvicorn配置：', app.native.uvicorn_config)
# 输出示例：UvicornConfig(app=<fastapi.applications.FastAPI object>, host='0.0.0.0', port=9000, ...)
```

### 3. `app.native.websocket_manager`：WebSocket 管理器

NiceGUI 的前后端实时通信（如组件状态同步、事件触发）依赖内置的 WebSocket 管理器，`app.native.websocket_manager`是该管理器的实例，负责管理所有客户端的 WebSocket 连接。

#### 核心能力：

- 主动向指定客户端推送消息；
- 广播消息到所有连接的客户端；
- 查看当前连接的客户端数量 / ID。

#### 示例：主动推送 WebSocket 消息

```python
from nicegui import app, ui
import asyncio

# 向所有客户端广播消息
async def broadcast_message():
    while True:
        # 通过websocket_manager主动推送消息
        await app.native.websocket_manager.broadcast({
            'type': 'custom_message',
            'content': f'当前时间：{asyncio.get_event_loop().time():.2f}'
        })
        await asyncio.sleep(1)

# 前端接收WebSocket消息并显示
ui.add_head_html('''
<script>
    // 监听NiceGUI的WebSocket消息
    window.socket.addEventListener('message', (event) => {
        const data = JSON.parse(event.data);
        if (data.type === 'custom_message') {
            document.getElementById('message').innerText = data.content;
        }
    });
</script>
''')
ui.label('实时消息：').id('message')

# 启动后台任务推送消息
ui.run(on_startup=lambda: asyncio.create_task(broadcast_message()))
```

- 原理：NiceGUI 的前端通过 WebSocket 与后端保持连接，`websocket_manager`负责管理这些连接并推送消息；
- 场景：实时监控、消息通知、多人协作等功能。

### 4. `app.native.quasar_config`：Quasar 前端配置

`app.native.quasar_config`用于配置前端 Quasar 框架的全局属性（如主题、图标、语言、全局组件）。

#### 示例：修改 Quasar 全局主题

```python
from nicegui import app, ui

# 配置Quasar主题为暗色模式
app.native.quasar_config['dark'] = True
# 配置Quasar默认语言为中文
app.native.quasar_config['lang'] = 'zh-CN'

ui.label('Quasar暗色模式 + 中文语言')
ui.button('切换主题', on_click=lambda: ui.dark_mode.toggle())
ui.run()
```

- 支持的配置项：参考 Quasar 官方文档（如`theme`、`iconSet`、`rtl`等）；
- 注意：需在`ui.run()`前配置，否则可能不生效。

------

## 三、关键使用场景

### 1. 扩展 NiceGUI 的 Web 能力

- 当 NiceGUI 的`app.urls`满足不了复杂 API 需求时（如 OpenAPI 文档、参数校验），使用`app.native.app`注册 FastAPI 原生路由；
- 需全局中间件（如 CORS、日志、认证）时，通过`app.native.app`添加 FastAPI 中间件。

### 2. 实时通信与消息推送

- 基于`app.native.websocket_manager`实现主动推送（如实时数据更新、告警通知），替代 NiceGUI 的`ui.notify`（仅前端提示）。

### 3. 定制化服务器配置

- 通过`app.native.uvicorn_config`动态调整 Uvicorn 参数（如运行时修改日志级别、工作进程数）。

### 4. 集成第三方 FastAPI 扩展

- FastAPI 生态的所有扩展（如`fastapi-jwt-auth`、`fastapi-cache2`、`fastapi-sqlalchemy`）都可通过`app.native.app`集成到 NiceGUI 中：

  ```python
  from fastapi_jwt_auth import AuthJWT
  from pydantic import BaseModel
  
  class Settings(BaseModel):
      authjwt_secret_key: str = "secret"
  
  @AuthJWT.load_config
  def get_config():
      return Settings()
  
  # 将JWT扩展绑定到NiceGUI的FastAPI实例
  AuthJWT(app.native.app)
  
  @app.native.app.post('/api/login')
  def login(Authorize: AuthJWT = Depends()):
      access_token = Authorize.create_access_token(subject="user123")
      return {"access_token": access_token}
  ```

------

## 四、注意事项

### 1. 版本兼容性

- `app.native`的属性依赖 NiceGUI 的底层实现，不同版本的 NiceGUI 可能调整`app.native`的结构（如早期版本`app.native`仅暴露`app`和`uvicorn_config`）；
- 建议查看 NiceGUI 官方源码（`nicegui/app.py`）确认`app.native`的具体属性。

### 2. 避免与 NiceGUI 高层 API 冲突

- 若同时使用`app.urls`和`app.native.app`注册路由，需确保路径不冲突（路由匹配优先级：FastAPI 原生路由 > NiceGUI 路由）；
- 修改`app.native.websocket_manager`时，避免破坏 NiceGUI 的前后端通信（如修改 WebSocket 消息格式）。

### 3. 异步 / 同步注意事项

- FastAPI 原生路由支持异步（`async def`）和同步（`def`），而 NiceGUI 的路由处理函数默认同步；
- 操作`app.native.websocket_manager`时需使用异步（`await`），需在异步上下文（如`async def`函数、`asyncio.create_task`）中执行。

# NiceGUI 中 app.native.window_args 的详细解析

`app.native.window_args`是 NiceGUI 针对**桌面端应用场景**暴露的核心配置属性，用于定制化通过`ui.run()`启动的桌面窗口（基于 PyWebView 封装）的行为与外观。它仅在「桌面模式」（即`ui.run()`的`native`参数为`True`时）生效，是突破 NiceGUI 默认窗口样式、实现桌面应用级定制的关键入口。以下从核心定位、配置项、使用场景、示例等维度全面解析：

------

## 一、核心定位与底层依赖

### 1. 适用场景

NiceGUI 支持两种运行模式：

- **Web 模式（默认）**：以网页形式运行（浏览器访问），`app.native.window_args`无作用；
- **桌面模式**：通过`ui.run(native=True)`启动独立桌面窗口（基于 PyWebView 库，底层调用 OS 原生窗口 API），`app.native.window_args`用于配置该窗口的所有属性。

### 2. 底层原理

`app.native.window_args`本质是传递给 PyWebView`create_window()`函数的参数字典，NiceGUI 将其封装为`app.native`的属性，开发者可直接修改该字典，实现对桌面窗口的精细化控制（如尺寸、标题、是否可缩放、置顶等）。

------

## 二、核心配置项详解

`app.native.window_args`是一个字典，支持 PyWebView`create_window()`的所有参数（不同操作系统支持的参数略有差异），核心配置项如下：

| 配置项          | 类型  | 默认值（NiceGUI） | 作用                                  | 跨平台支持          |
| --------------- | ----- | ----------------- | ------------------------------------- | ------------------- |
| `title`         | str   | `'NiceGUI'`       | 窗口标题栏文本                        | Windows/macOS/Linux |
| `width`         | int   | `800`             | 窗口宽度（像素）                      | 全平台              |
| `height`        | int   | `600`             | 窗口高度（像素）                      | 全平台              |
| `x`             | int   | `None`            | 窗口左上角横坐标（None = 居中）       | 全平台              |
| `y`             | int   | `None`            | 窗口左上角纵坐标（None = 居中）       | 全平台              |
| `resizable`     | bool  | `True`            | 窗口是否可缩放                        | 全平台              |
| `min_size`      | tuple | `(400, 300)`      | 窗口最小尺寸（宽，高）                | 全平台              |
| `max_size`      | tuple | `None`            | 窗口最大尺寸（宽，高）                | 全平台              |
| `fullscreen`    | bool  | `False`           | 是否全屏启动                          | 全平台              |
| `hidden`        | bool  | `False`           | 启动时是否隐藏窗口                    | 全平台              |
| `frameless`     | bool  | `False`           | 是否无边框窗口（无标题栏 / 关闭按钮） | Windows/macOS       |
| `always_on_top` | bool  | `False`           | 窗口是否置顶（总在其他窗口上层）      | 全平台              |
| `transparent`   | bool  | `False`           | 窗口是否透明（仅无边框时生效）        | Windows/macOS       |
| `icon`          | str   | `None`            | 窗口图标路径（支持.ico/.png）         | Windows/macOS/Linux |
| `zoom`          | float | `1.0`             | 窗口内容缩放比例                      | 全平台              |

### 系统专属配置项

| 配置项         | 系统    | 作用                                 |
| -------------- | ------- | ------------------------------------ |
| `taskbar_icon` | Windows | 任务栏图标路径（区别于窗口图标）     |
| `shadow`       | macOS   | 是否显示窗口阴影（frameless 时生效） |
| `decorations`  | Linux   | 是否显示窗口装饰（标题栏 / 边框）    |

------

## 三、使用方式与示例

### 1. 基础用法：修改窗口尺寸与标题

```python
from nicegui import app, ui

# 配置桌面窗口属性（需在ui.run()前修改）
app.native.window_args.update({
    'title': '我的NiceGUI桌面应用',  # 自定义标题
    'width': 1000,                  # 宽度1000px
    'height': 700,                 # 高度700px
    'resizable': False,            # 禁止缩放窗口
    'always_on_top': True          # 窗口置顶
})

ui.label('自定义桌面窗口示例')
# 启动桌面模式（native=True）
ui.run(native=True)
```

### 2. 进阶示例：无边框透明窗口

```python
from nicegui import app, ui

# 配置无边框+透明窗口
app.native.window_args.update({
    'frameless': True,       # 无边框（无标题栏/关闭按钮）
    'transparent': True,     # 透明背景
    'width': 500,
    'height': 300,
    'always_on_top': True,
    'shadow': False          # macOS：关闭阴影（可选）
})

# 自定义UI（透明背景下的内容）
ui.label('无边框透明窗口').classes('text-2xl text-white')
ui.button('关闭窗口', on_click=lambda: app.native.window.destroy()).classes('mt-4')

# 启动桌面模式
ui.run(native=True, dark_mode=True)
```

- 注意：透明窗口仅在`frameless=True`时生效，且不同操作系统的透明效果略有差异（Windows 需启用硬件加速）；
- 无边框窗口需自定义关闭 / 最小化按钮（通过`app.native.window.destroy()`关闭窗口）。

### 3. 固定窗口位置 + 自定义图标

```python
from nicegui import app, ui

# 配置窗口位置（屏幕左上角x=200, y=100）+ 自定义图标
app.native.window_args.update({
    'x': 200,
    'y': 100,
    'icon': './my_app_icon.ico',  # 本地图标文件（Windows推荐.ico，macOS推荐.icns）
    'min_size': (600, 400),       # 最小尺寸限制
    'max_size': (1200, 800)       # 最大尺寸限制
})

ui.label('固定位置+自定义图标窗口')
ui.run(native=True)
```

### 4. 全屏启动 + 动态调整窗口

```python
from nicegui import app, ui

# 初始全屏启动
app.native.window_args['fullscreen'] = True

# 动态切换全屏/窗口模式
def toggle_fullscreen():
    current_state = app.native.window.fullscreen
    app.native.window.fullscreen = not current_state

ui.button('切换全屏', on_click=toggle_fullscreen)
ui.run(native=True)
```

- `app.native.window`是 PyWebView 的窗口实例，启动后可通过该对象动态修改窗口属性（如`fullscreen`、`size`、`position`）。

------

## 四、关键注意事项

### 1. 生效时机

`app.native.window_args`的修改必须在`ui.run(native=True)`之前完成，启动后修改该字典不会生效；若需运行时调整窗口属性，需直接操作`app.native.window`（PyWebView 实例）。

### 2. 跨平台兼容性

- Windows：支持所有核心配置项，透明 / 无边框效果最佳；
- macOS：`transparent`仅在 macOS 10.14 + 生效，`frameless`窗口的拖动需自定义实现；
- Linux：依赖系统桌面环境（如 GNOME/KDE），部分配置项（如`transparent`）可能无效。

### 3. 依赖安装

桌面模式依赖 PyWebView 的底层库，需额外安装对应系统的依赖：

```bash
# Windows（默认已包含）
pip install nicegui[pywebview]

# macOS
brew install pyobjc-framework-WebKit
pip install nicegui[pywebview]

# Linux（Debian/Ubuntu）
sudo apt install python3-gi python3-gi-cairo gir1.2-webkit2-4.0
pip install nicegui[pywebview]
```

### 4. 与 Web 模式的冲突

- 若`ui.run()`未设置`native=True`，`app.native.window_args`和`app.native.window`均为`None`，操作会报错；

- 建议通过`if app.native.window:`判断是否为桌面模式后再操作：

  ```python
  if app.native.window:
      app.native.window.always_on_top = True  # 仅桌面模式执行
  ```

### 5. 性能与资源占用

- 透明 / 无边框窗口会增加少量性能开销，低配置设备可能出现卡顿；
- 避免同时启用`transparent`和`fullscreen`，部分系统会出现渲染异常。

# NiceGUI 中 app.native.start_args 的详细解析

`app.native.start_args`是 NiceGUI 针对**PyWebView 桌面窗口启动流程**暴露的配置属性，用于定制化 PyWebView`start()`函数的入参，是桌面模式下控制窗口运行时行为（如事件循环、多线程、调试模式）的核心入口。它与`app.native.window_args`（窗口外观 / 尺寸配置）互补，前者聚焦「启动逻辑」，后者聚焦「窗口属性」，仅在`ui.run(native=True)`（桌面模式）下生效。

------

## 一、核心定位与底层原理

### 1. 适用场景

NiceGUI 的桌面模式基于 PyWebView 实现：

- `app.native.window_args` → 传递给 PyWebView `create_window()`（创建窗口时的静态属性）；

- `app.native.start_args` → 传递给 PyWebView `start()`（启动窗口事件循环时的运行时参数）。

  两者共同决定桌面窗口的最终行为，`start_args`主要控制「窗口如何运行」（而非「窗口长什么样」）。

### 2. 底层依赖

PyWebView 的`start()`函数负责启动窗口的事件循环、绑定系统原生消息处理，`app.native.start_args`本质是该函数的参数字典，NiceGUI 将其封装为`app.native`的属性，开发者可通过修改该字典覆盖默认启动行为。

------

## 二、核心配置项详解

`app.native.start_args`是一个字典，支持 PyWebView `start()`函数的所有参数（不同操作系统 / PyWebView 版本支持度略有差异），核心配置项如下：

| 配置项         | 类型                      | 默认值（NiceGUI）  | 作用                                                         | 跨平台支持    |
| -------------- | ------------------------- | ------------------ | ------------------------------------------------------------ | ------------- |
| `debug`        | bool                      | `False`            | 启用 PyWebView 调试模式：- 输出详细日志（含前端 / 后端通信）；- 前端可打开开发者工具（F12）；- 便于排查桌面窗口渲染 / 通信问题 | 全平台        |
| `http_server`  | bool                      | `True`             | 是否启动内置 HTTP 服务器：- `True`：NiceGUI 通过本地 HTTP 服务加载前端资源（与 Web 模式一致）；- `False`：以本地文件模式加载（`file://`协议），需处理跨域 / 资源路径问题 | 全平台        |
| `gui`          | str                       | `None`（自动选择） | 指定 PyWebView 使用的底层 GUI 引擎：- `'qt'`：Qt WebEngine（跨平台，兼容性好）；- `'gtk'`：GTK WebKit（Linux）；- `'webkit'`：macOS 原生 WebKit；- `'edgechromium'`：Windows Edge 内核 | 按系统区分    |
| `loop`         | asyncio.AbstractEventLoop | `None`             | 指定自定义的 asyncio 事件循环：- 用于集成第三方异步框架（如 Trio、Curio）；- 默认为 NiceGUI 创建的默认事件循环 | 全平台        |
| `private_mode` | bool                      | `False`            | 启用私有模式：- 禁用缓存、Cookie、本地存储；- 适合隐私敏感的桌面应用 | Windows/macOS |

### 扩展配置项（PyWebView 高级参数）

| 配置项            | 适用系统 | 作用                                             |
| ----------------- | -------- | ------------------------------------------------ |
| `backend`         | 全平台   | 别名`gui`，优先级低于`gui`（兼容旧版 PyWebView） |
| `ssl`             | 全平台   | 为内置 HTTP 服务器启用 SSL（需传入证书路径）     |
| `service_workers` | bool     | 启用 / 禁用 Service Worker（前端离线缓存）       |

------

## 三、使用方式与示例

### 1. 基础用法：启用调试模式 + 指定 GUI 引擎

```python
from nicegui import app, ui

# 配置窗口启动参数（启动逻辑）
app.native.start_args.update({
    'debug': True,  # 启用调试模式（F12打开开发者工具）
    'gui': 'qt'     # 强制使用Qt WebEngine引擎（跨平台兼容）
})
# 配置窗口属性（外观）
app.native.window_args.update({
    'title': '调试模式示例',
    'width': 900,
    'height': 600
})

ui.label('桌面模式调试示例（按F12打开开发者工具）')
ui.run(native=True)
```

- 调试模式下，控制台会输出 PyWebView 的详细日志（如 WebSocket 连接、资源加载）；
- 前端可通过 F12 调试组件、查看网络请求，与浏览器调试体验一致。

### 2. 进阶示例：禁用 HTTP 服务器（本地文件模式）

```python
from nicegui import app, ui
import os

# 配置启动参数：禁用内置HTTP服务器，以本地文件模式运行
app.native.start_args['http_server'] = False
# 需手动指定静态资源路径（本地文件模式下NiceGUI无法自动映射）
app.add_static_files('/static', os.path.join(os.getcwd(), 'static'))

app.native.window_args.update({
    'title': '本地文件模式示例',
    'width': 800,
    'height': 500
})

ui.label('本地文件模式运行（无HTTP服务器）')
# 注意：本地文件模式下部分功能（如WebSocket）可能受限
ui.run(native=True)
```

- 适用场景：离线运行的桌面应用（无需启动 HTTP 服务）；
- 限制：WebSocket、跨域请求可能失效，需额外处理资源路径。

### 3. 自定义事件循环（集成异步框架）

```python
from nicegui import app, ui
import asyncio

# 创建自定义asyncio事件循环
custom_loop = asyncio.new_event_loop()
asyncio.set_event_loop(custom_loop)

# 配置启动参数：使用自定义事件循环
app.native.start_args['loop'] = custom_loop

# 后台异步任务（基于自定义循环）
async def background_task():
    while True:
        print('自定义循环的后台任务')
        await asyncio.sleep(2)

# 启动任务
custom_loop.create_task(background_task())

ui.label('自定义事件循环示例')
ui.run(native=True)
```

- 适用场景：需集成第三方异步框架（如 Trio）或精细控制事件循环的场景；
- 注意：自定义循环需确保线程安全，避免与 NiceGUI 内置循环冲突。

### 4. 启用私有模式 + SSL（安全场景）

```python
from nicegui import app, ui

# 配置启动参数：私有模式+SSL
app.native.start_args.update({
    'private_mode': True,  # 禁用缓存/Cookie
    'ssl': ('cert.pem', 'key.pem')  # 传入SSL证书（需自行生成）
})

app.native.window_args['title'] = '安全模式示例'
ui.label('私有模式+SSL加密运行')
ui.run(native=True, host='0.0.0.0', port=8443)  # SSL需指定端口
```

- 适用场景：处理敏感数据的桌面应用（如密码管理、金融工具）；
- 注意：SSL 证书需自行生成（如通过 OpenSSL），且仅在`http_server=True`时生效。

------

## 四、关键注意事项

### 1. 生效时机

- `app.native.start_args`的修改必须在`ui.run(native=True)`之前完成，启动后修改无效；
- 若需运行时调整事件循环，需直接操作`app.native.start_args['loop']`对应的循环对象。

### 2. 与`window_args`的分工

| 维度     | `app.native.window_args`         | `app.native.start_args`          |
| -------- | -------------------------------- | -------------------------------- |
| 作用     | 窗口静态属性（尺寸、标题、边框） | 窗口运行时行为（调试、事件循环） |
| 底层函数 | PyWebView `create_window()`      | PyWebView `start()`              |
| 生效阶段 | 窗口创建时                       | 窗口启动（事件循环）时           |

### 3. 跨平台兼容性

- `gui`参数：Windows 优先`edgechromium`，macOS 优先`webkit`，Linux 优先`gtk`；强制指定`qt`可获得跨平台一致体验，但需安装 PyQt5/PySide2；
- `private_mode`：Linux 桌面环境（如 GNOME）可能不支持，需测试验证；
- `ssl`：本地文件模式（`http_server=False`）下无效。

### 4. 依赖与版本问题

- PyWebView 版本差异：`start()`参数在 PyWebView 4.x 与 3.x 中略有不同（如`backend`→`gui`），NiceGUI 默认适配最新版；
- 调试模式限制：macOS 下`debug=True`可能导致窗口卡顿，建议仅开发环境启用。

### 5. 错误处理

- 若`start_args`配置错误（如指定不存在的`gui`引擎），PyWebView 会抛出`NotImplementedError`，需捕获并调整配置：

  ```python
  try:
      ui.run(native=True)
  except NotImplementedError as e:
      print(f'启动失败：{e}')
      app.native.start_args['gui'] = None  # 恢复自动选择
      ui.run(native=True)
  ```

# NiceGUI 中 app.native.settings 的详细解析

`app.native.settings`是 NiceGUI 在**桌面模式**（`ui.run(native=True)`）下暴露的 PyWebView 全局设置入口，用于配置 PyWebView 底层运行时的全局行为（如资源加载策略、JS 交互权限、缓存规则等），是比`app.native.window_args`（窗口属性）、`app.native.start_args`（启动参数）更底层的全局配置项。它面向 PyWebView 的核心运行时规则，而非单个窗口或启动流程，仅在桌面模式下有效。

------

## 一、核心定位与底层原理

### 1. 适用场景

NiceGUI 桌面模式基于 PyWebView 实现，PyWebView 提供`webview.settings`全局配置对象，用于统一管控所有窗口的底层行为（如是否允许 JS 调用 Python、是否启用缓存、资源加载限制等）。`app.native.settings`正是对该全局对象的直接封装，开发者可通过修改该属性实现跨窗口的全局规则配置。

### 2. 核心价值

- 区别于`window_args`（单窗口属性）和`start_args`（启动流程），`settings`聚焦**全局运行时规则**；
- 所有通过 PyWebView 创建的窗口（包括 NiceGUI 的主窗口）都会继承`settings`的配置；
- 支持运行时动态修改（部分配置需重启窗口生效）。

------

## 二、核心配置项详解

`app.native.settings`是一个类字典对象（PyWebView 的`Settings`实例），支持以下核心配置项（跨平台通用，部分项受系统限制）：

| 配置项                                  | 类型 | 默认值   | 作用                                          | 关键说明                                        |
| --------------------------------------- | ---- | -------- | --------------------------------------------- | ----------------------------------------------- |
| `ALLOW_FILE_ACCESS_FROM_FILE_URLS`      | bool | `False`  | 允许本地文件 URL（`file://`）访问其他本地文件 | 本地文件模式下加载静态资源需开启，存在安全风险  |
| `ALLOW_UNIVERSAL_ACCESS_FROM_FILE_URLS` | bool | `False`  | 允许本地文件 URL 访问任意资源（包括网络）     | 比上一项权限更大，仅开发环境启用                |
| `CACHE_ENABLED`                         | bool | `True`   | 启用 HTTP / 本地文件缓存                      | 禁用后可强制每次加载最新资源，适合调试          |
| `DEVELOPER_TOOLS_ENABLED`               | bool | `False`  | 启用开发者工具（F12）                         | 等价于`start_args['debug']`，优先级更高         |
| `JAVASCRIPT_ENABLED`                    | bool | `True`   | 启用 JavaScript 执行                          | 禁用后 NiceGUI 前端交互会失效（核心依赖 JS）    |
| `JAVASCRIPT_CAN_OPEN_WINDOW`            | bool | `False`  | 允许 JS 通过`window.open()`创建新窗口         | NiceGUI 的`ui.navigate`不依赖此配置             |
| `JAVASCRIPT_CAN_ACCESS_CLIPBOARD`       | bool | `False`  | 允许 JS 读写剪贴板                            | 启用后前端可通过`navigator.clipboard`操作剪贴板 |
| `LOAD_IMAGES`                           | bool | `True`   | 加载图片资源                                  | 禁用后可提升低性能设备运行速度                  |
| `LOCAL_STORAGE_ENABLED`                 | bool | `True`   | 启用前端 LocalStorage                         | NiceGUI 的`app.storage.local`依赖此配置         |
| `REMOTE_DEBUGGING_PORT`                 | int  | `None`   | 远程调试端口（如 9222）                       | 可通过 Chrome DevTools 远程调试窗口             |
| `USER_AGENT`                            | str  | 系统默认 | 自定义浏览器 User-Agent                       | 用于模拟不同浏览器 / 设备的请求                 |

------

## 三、使用方式与示例

### 1. 基础用法：启用开发者工具 + 禁用缓存

```python
from nicegui import app, ui

# 仅桌面模式下配置settings
if app.native:
    # 启用开发者工具（F12）
    app.native.settings.DEVELOPER_TOOLS_ENABLED = True
    # 禁用缓存（强制加载最新资源）
    app.native.settings.CACHE_ENABLED = False
    # 允许JS访问剪贴板
    app.native.settings.JAVASCRIPT_CAN_ACCESS_CLIPBOARD = True

# 配置窗口属性
app.native.window_args.update({
    'title': '全局设置示例',
    'width': 900,
    'height': 600
})

# 测试剪贴板功能
def copy_to_clipboard():
    ui.run_javascript('navigator.clipboard.writeText("测试剪贴板内容")')

ui.button('复制文本到剪贴板', on_click=copy_to_clipboard)
ui.run(native=True)
```

- 效果：启动后按 F12 可打开开发者工具，修改前端代码无需清理缓存，点击按钮可直接写入剪贴板。

### 2. 进阶示例：本地文件模式下允许资源访问

```python
from nicegui import app, ui
import os

# 桌面模式配置
if app.native:
    # 禁用HTTP服务器（本地文件模式）
    app.native.start_args['http_server'] = False
    # 允许本地文件URL访问其他文件（关键配置）
    app.native.settings.ALLOW_FILE_ACCESS_FROM_FILE_URLS = True
    # 允许本地文件URL访问网络资源（可选）
    app.native.settings.ALLOW_UNIVERSAL_ACCESS_FROM_FILE_URLS = True

# 挂载静态资源（本地文件模式需手动指定）
static_dir = os.path.join(os.getcwd(), 'static')
app.add_static_files('/static', static_dir)

# 加载本地图片（依赖ALLOW_FILE_ACCESS_FROM_FILE_URLS=True）
ui.image('/static/test.png').classes('w-64')
ui.run(native=True)
```

- 核心：本地文件模式（`http_server=False`）下，默认禁止本地文件访问其他资源，需开启`ALLOW_FILE_ACCESS_FROM_FILE_URLS`才能加载本地图片 / JS/CSS。

### 3. 自定义 User-Agent + 启用远程调试

```python
from nicegui import app, ui

if app.native:
    # 自定义User-Agent（模拟Chrome浏览器）
    app.native.settings.USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
    # 启用远程调试（端口9222）
    app.native.settings.REMOTE_DEBUGGING_PORT = 9222
    # 禁用LocalStorage（隐私保护）
    app.native.settings.LOCAL_STORAGE_ENABLED = False

app.native.window_args['title'] = '远程调试+自定义UA示例'
ui.label('访问 chrome://inspect 可远程调试该窗口')
ui.run(native=True)
```

- 远程调试：启动后打开 Chrome 浏览器，访问`chrome://inspect`，可识别到该桌面窗口并进行调试；
- 自定义 UA：适用于需要模拟特定浏览器的场景（如兼容特定网页）。

### 4. 运行时动态修改设置

```python
from nicegui import app, ui

# 初始禁用图片加载
if app.native:
    app.native.settings.LOAD_IMAGES = False

# 动态切换图片加载状态
def toggle_image_load():
    if app.native:
        app.native.settings.LOAD_IMAGES = not app.native.settings.LOAD_IMAGES
        # 刷新页面使配置生效
        ui.run_javascript('window.location.reload()')

ui.button('切换图片加载状态', on_click=toggle_image_load)
ui.image('https://picsum.photos/200').classes('mt-4')  # 测试图片加载
ui.run(native=True)
```

- 注意：部分设置（如`LOAD_IMAGES`）修改后需刷新页面才能生效；
- 动态修改适合根据用户操作调整运行时规则（如低性能模式下禁用图片加载）。

------

## 四、关键注意事项

### 1. 生效范围与优先级

- `app.native.settings`是**全局配置**，对所有 PyWebView 窗口生效（包括 NiceGUI 主窗口、通过`webview.create_window`创建的子窗口）；
- 优先级：`settings`全局配置 > `start_args`启动参数 > `window_args`窗口属性；例如`settings.DEVELOPER_TOOLS_ENABLED=True`会覆盖`start_args['debug']=False`。

### 2. 安全风险提示

- `ALLOW_UNIVERSAL_ACCESS_FROM_FILE_URLS=True`：允许本地文件访问网络资源，可能被恶意 JS 利用，生产环境禁止启用；
- `JAVASCRIPT_CAN_ACCESS_CLIPBOARD=True`：需确保前端代码可信，避免剪贴板数据泄露；
- 建议：开发环境可放宽配置，生产环境仅启用必要权限。

### 3. 跨平台兼容性

- Windows/macOS：支持所有核心配置项，`REMOTE_DEBUGGING_PORT`、`USER_AGENT`效果稳定；
- Linux：`ALLOW_FILE_ACCESS_FROM_FILE_URLS`可能受桌面环境限制（如 GNOME 需额外配置），`REMOTE_DEBUGGING_PORT`仅部分浏览器支持；
- 禁用`JAVASCRIPT_ENABLED`会导致 NiceGUI 完全失效（NiceGUI 前端基于 Vue/JS）。

### 4. 生效时机

- 大部分配置（如`CACHE_ENABLED`、`LOAD_IMAGES`）修改后需刷新页面（`window.location.reload()`）才能生效；
- 少数配置（如`REMOTE_DEBUGGING_PORT`）需重启应用才能生效；
- 建议：核心配置在`ui.run()`前完成，运行时动态修改仅用于临时调整。

### 5. 与 Web 模式的区别

- Web 模式下`app.native`为`None`，`app.native.settings`不存在，操作会报错；

- 建议通过`if app.native:`判断模式后再配置：

  ```python
  if app.native:
      app.native.settings.DEVELOPER_TOOLS_ENABLED = True
  else:
      print('当前为Web模式，settings配置无效')
  ```

# NiceGUI 中 app.native.main_window 对象的详细解析

`app.native.main_window`是 NiceGUI 桌面模式（`ui.run(native=True)`）下暴露的**主窗口实例**，本质是 PyWebView 库的`Window`对象，是操作桌面窗口运行时行为的核心入口。它区别于静态配置（`window_args`/`start_args`/`settings`），聚焦「运行时动态控制」—— 可在应用启动后实时调整窗口状态（如尺寸、位置、全屏）、触发窗口行为（如最小化、关闭）、交互前端页面（如执行 JS、获取 DOM），是桌面应用定制化的最终落地载体。

------

## 一、核心定位与底层原理

### 1. 适用场景

- 仅在`ui.run(native=True)`启动的桌面模式下存在，Web 模式下`app.native.main_window`为`None`；
- 是 PyWebView `Window`类的实例，继承了 PyWebView 所有窗口操作 API，NiceGUI 仅做透传封装，无额外限制。

### 2. 与其他 native 属性的关系

| 组件          | 作用                     | 与 main_window 的关系                     |
| ------------- | ------------------------ | ----------------------------------------- |
| `window_args` | 窗口创建时的静态属性配置 | 初始化 main_window 的基础属性             |
| `start_args`  | 窗口启动时的运行时参数   | 决定 main_window 的事件循环 / 引擎配置    |
| `settings`    | 全局运行时规则           | 约束 main_window 的底层行为（如 JS 权限） |
| `main_window` | 运行时窗口实例           | 可动态覆盖上述配置的最终操作对象          |

------

## 二、核心属性与方法详解

`app.native.main_window`继承 PyWebView `Window`对象的所有属性和方法，核心可分为「状态查询 / 修改」「窗口行为控制」「前后端交互」「事件监听」四大类，以下是高频使用的核心能力：

### （一）状态查询 / 修改（实时获取 / 调整窗口属性）

| 属性 / 方法           | 类型 / 参数       | 作用                                                         |
| --------------------- | ----------------- | ------------------------------------------------------------ |
| `width`/`height`      | int（可读可写）   | 获取 / 设置窗口宽度 / 高度（像素），修改后立即生效           |
| `x`/`y`               | int（可读可写）   | 获取 / 设置窗口左上角坐标（相对于屏幕），修改后窗口立即移动  |
| `title`               | str（可读可写）   | 获取 / 修改窗口标题栏文本                                    |
| `fullscreen`          | bool（可读可写）  | 获取 / 设置窗口是否全屏（True = 全屏，False = 窗口模式）     |
| `min_size`/`max_size` | tuple（可读可写） | 获取 / 设置窗口最小 / 最大尺寸（如`(400, 300)`）             |
| `resizable`           | bool（可读可写）  | 获取 / 设置窗口是否可缩放                                    |
| `always_on_top`       | bool（可读可写）  | 获取 / 设置窗口是否置顶（总在其他窗口上层）                  |
| `visible`             | bool（可读可写）  | 获取 / 设置窗口是否可见（False = 隐藏，True = 显示）         |
| `transparent`         | bool（只读）      | 查询窗口是否透明（需初始化时通过`window_args`设置，运行时不可改） |
| `frameless`           | bool（只读）      | 查询窗口是否无边框（需初始化时设置，运行时不可改）           |

#### 示例：实时调整窗口尺寸与位置

```python
from nicegui import app, ui

# 初始化窗口属性
app.native.window_args.update({
    'title': '动态调整窗口示例',
    'width': 800,
    'height': 600,
    'x': 100,
    'y': 100
})

# 放大窗口
def enlarge_window():
    if app.native.main_window:
        app.native.main_window.width += 100
        app.native.main_window.height += 50

# 移动窗口到屏幕中央
def center_window():
    if app.native.main_window:
        # 获取屏幕尺寸（需通过JS获取）
        ui.run_javascript('''
            return {
                screen_width: window.screen.width,
                screen_height: window.screen.height
            }
        ''', callback=lambda data: adjust_position(data))

def adjust_position(data):
    w = app.native.main_window.width
    h = app.native.main_window.height
    app.native.main_window.x = (data['screen_width'] - w) // 2
    app.native.main_window.y = (data['screen_height'] - h) // 2

ui.button('放大窗口', on_click=enlarge_window)
ui.button('居中窗口', on_click=center_window)
ui.run(native=True)
```

### （二）窗口行为控制（触发窗口动作）

| 方法                  | 参数           | 作用                                                        |
| --------------------- | -------------- | ----------------------------------------------------------- |
| `minimize()`          | 无             | 最小化窗口                                                  |
| `maximize()`          | 无             | 最大化窗口                                                  |
| `restore()`           | 无             | 还原窗口（从最大化 / 最小化恢复）                           |
| `close()`             | 无             | 关闭窗口（等价于点击标题栏关闭按钮，会触发`on_closed`事件） |
| `destroy()`           | 无             | 强制销毁窗口（不触发关闭事件，慎用）                        |
| `set_icon(icon_path)` | icon_path: str | 动态修改窗口图标（支持.ico/.png，跨平台）                   |
| `toggle_fullscreen()` | 无             | 切换全屏 / 窗口模式（等价于修改`fullscreen`属性）           |

#### 示例：窗口行为控制组合

```python
from nicegui import app, ui

def control_window(action):
    w = app.native.main_window
    if not w:
        return
    if action == 'min':
        w.minimize()
    elif action == 'max':
        w.maximize()
    elif action == 'restore':
        w.restore()
    elif action == 'close':
        w.close()
    elif action == 'icon':
        w.set_icon('./custom_icon.ico')  # 动态修改图标

ui.button('最小化', on_click=lambda: control_window('min'))
ui.button('最大化', on_click=lambda: control_window('max'))
ui.button('还原', on_click=lambda: control_window('restore'))
ui.button('修改图标', on_click=lambda: control_window('icon'))
ui.button('关闭', on_click=lambda: control_window('close'))
ui.run(native=True, title='窗口行为控制示例')
```

### （三）前后端交互（窗口级页面操作）

| 方法                          | 参数         | 作用                                                         |
| ----------------------------- | ------------ | ------------------------------------------------------------ |
| `evaluate_js(js_code)`        | js_code: str | 同步执行前端 JS 代码，返回执行结果（如获取 DOM 值、调用前端函数） |
| `async evaluate_js_async(js)` | js: str      | 异步执行 JS 代码（适合耗时操作，如请求数据）                 |
| `load_url(url)`               | url: str     | 让窗口加载指定 URL（可替换当前 NiceGUI 页面，如加载外部网页） |
| `load_html(html)`             | html: str    | 让窗口加载自定义 HTML 内容（替代当前页面）                   |
| `get_current_url()`           | 无           | 获取窗口当前加载的 URL                                       |

#### 示例：执行 JS 获取 DOM 值 + 加载外部网页

```python
from nicegui import app, ui

# 执行JS获取输入框值
def get_input_value():
    if app.native.main_window:
        # 同步执行JS，获取id为"input1"的输入框值
        value = app.native.main_window.evaluate_js('document.getElementById("input1").value')
        ui.notify(f'输入框值：{value}')

# 加载外部网页
def load_baidu():
    if app.native.main_window:
        app.native.main_window.load_url('https://www.baidu.com')

ui.input('测试输入框').id('input1')
ui.button('获取输入框值', on_click=get_input_value)
ui.button('加载百度', on_click=load_baidu)
ui.run(native=True)
```

### （四）事件监听（监听窗口生命周期事件）

`app.native.main_window`支持监听窗口的原生事件，通过`events`属性绑定回调函数，核心事件如下：

| 事件名         | 触发时机                     | 回调参数        |
| -------------- | ---------------------------- | --------------- |
| `on_closed`    | 窗口关闭时                   | 无              |
| `on_closing`   | 窗口即将关闭时（可阻止关闭） | 无              |
| `on_resized`   | 窗口尺寸变化时               | (width, height) |
| `on_moved`     | 窗口位置变化时               | (x, y)          |
| `on_minimized` | 窗口最小化时                 | 无              |
| `on_maximized` | 窗口最大化时                 | 无              |
| `on_restored`  | 窗口还原时                   | 无              |

#### 示例：监听窗口关闭 / 尺寸变化事件

```python
from nicegui import app, ui

def setup_window_events():
    w = app.native.main_window
    if not w:
        return
    
    # 窗口即将关闭时提示
    def on_closing():
        ui.notify('窗口即将关闭！')
        # 可通过返回False阻止关闭（部分系统支持）
        # return False
    
    # 窗口尺寸变化时打印新尺寸
    def on_resized(width, height):
        print(f'窗口尺寸变为：{width}x{height}')
    
    # 绑定事件
    w.events.on_closing += on_closing
    w.events.on_resized += on_resized
    # 解绑事件（如需）：w.events.on_resized -= on_resized

# 启动后绑定事件
ui.run(native=True, on_startup=setup_window_events)
```

------

## 三、高级使用场景

### 1. 无边框窗口自定义拖动

无边框窗口（`frameless=True`）默认无法拖动，需通过`main_window`结合 JS 实现自定义拖动区域：

```python
from nicegui import app, ui

# 配置无边框窗口
app.native.window_args['frameless'] = True

# 自定义拖动区域的JS
drag_js = '''
// 给id为"drag-area"的元素绑定拖动事件
let dragArea = document.getElementById('drag-area');
let isDragging = false;
let startX, startY, windowX, windowY;

dragArea.addEventListener('mousedown', (e) => {
    isDragging = true;
    startX = e.clientX;
    startY = e.clientY;
    // 获取当前窗口位置（通过Python调用返回）
    windowX = await window.pywebview.api.get_window_x();
    windowY = await window.pywebview.api.get_window_y();
});

document.addEventListener('mousemove', (e) => {
    if (!isDragging) return;
    const dx = e.clientX - startX;
    const dy = e.clientY - startY;
    // 调用Python方法移动窗口
    window.pywebview.api.move_window(windowX + dx, windowY + dy);
});

document.addEventListener('mouseup', () => {
    isDragging = false;
});
'''

# 暴露Python方法给JS调用
def get_window_x():
    return app.native.main_window.x

def get_window_y():
    return app.native.main_window.y

def move_window(x, y):
    app.native.main_window.x = x
    app.native.main_window.y = y

ui.expose_api('get_window_x', get_window_x)
ui.expose_api('get_window_y', get_window_y)
ui.expose_api('move_window', move_window)

# 自定义拖动区域
ui.row().id('drag-area').classes('w-full h-10 bg-gray-800').content('拖动此区域移动窗口')
ui.run_javascript(drag_js)
ui.run(native=True)
```

### 2. 多窗口管理（主窗口 + 子窗口）

`app.native.main_window`是主窗口，可通过 PyWebView 原生 API 创建子窗口，并通过主窗口协调管理：

```python
from nicegui import app, ui
import webview  # 直接使用PyWebView库

def open_sub_window():
    # 创建子窗口
    sub_window = webview.create_window(
        title='子窗口',
        url='https://nicegui.io',
        width=600,
        height=400
    )
    # 主窗口居中，子窗口定位在主窗口右侧
    main_x = app.native.main_window.x
    main_y = app.native.main_window.y
    sub_window.x = main_x + app.native.main_window.width + 20
    sub_window.y = main_y

ui.button('打开子窗口', on_click=open_sub_window)
ui.run(native=True, title='主窗口')
```

### 3. 窗口透明 + 动态调整透明度

（仅 Windows/macOS 支持，需初始化时设置`transparent=True`）

```python
from nicegui import app, ui

# 初始化透明无边框窗口
app.native.window_args.update({
    'frameless': True,
    'transparent': True,
    'width': 500,
    'height': 300
})

# Windows下通过API调整透明度（需调用系统原生API）
import ctypes
def set_opacity(opacity):
    if app.native.main_window and os.name == 'nt':
        # 获取窗口句柄
        hwnd = app.native.main_window.get_handle()
        # 设置透明度（0-255，0=全透，255=不透明）
        ctypes.windll.user32.SetLayeredWindowAttributes(hwnd, 0, int(opacity), 2)

ui.slider(min=0, max=255, step=1, value=200).on_change(lambda e: set_opacity(e.value))
ui.run(native=True)
```

------

## 四、关键注意事项

### 1. 访问时机

- `app.native.main_window`仅在`ui.run(native=True)`启动后才初始化完成，**启动前访问会报错**；

- 若需在启动后立即操作，需通过`on_startup`回调：

  ```python
  def on_app_start():
      # 启动后立即置顶窗口
      app.native.main_window.always_on_top = True
  
  ui.run(native=True, on_startup=on_app_start)
  ```

### 2. 跨平台兼容性

| 功能                   | Windows | macOS | Linux    | 备注                 |
| ---------------------- | ------- | ----- | -------- | -------------------- |
| 尺寸 / 位置修改        | ✅       | ✅     | ✅        | 全平台支持           |
| 置顶 / 最小化 / 最大化 | ✅       | ✅     | ✅        | Linux 需桌面环境支持 |
| 透明窗口 + 透明度调整  | ✅       | ✅     | ❌        | Linux 无统一透明 API |
| 自定义图标             | ✅       | ✅     | ✅        | Linux 需.svg 格式    |
| 事件监听（on_resized） | ✅       | ✅     | 部分支持 | KDE/GNOME 差异       |

### 3. 线程安全

- `app.native.main_window`的方法**非线程安全**，若在后台线程中操作，需通过`asyncio`或`ui.call_from_thread`包装：

  ```python
  def background_task():
      # 后台线程中修改窗口标题
      ui.call_from_thread(lambda: app.native.main_window.title = '新标题')
  
  ui.timer(5, background_task, once=True)
  ```

### 4. 与 NiceGUI 前端的冲突

- 调用`load_url()`/`load_html()`会替换当前 NiceGUI 页面，导致原有 UI 组件失效；
- 若需保留 NiceGUI 页面，建议通过`ui.run_javascript()`修改 DOM，而非直接加载新 URL。

### 5. 资源释放

- 调用`close()`会触发正常的窗口关闭流程（执行`on_shutdown`回调）；
- 调用`destroy()`会强制销毁窗口，可能导致资源泄漏（如数据库连接未关闭），仅应急使用。

# NiceGUI 中 app.native 深度解析

在 NiceGUI 框架中，`app.native` 是**直接访问底层 FastAPI/Starlette 应用实例的核心属性**，其本质是对 NiceGUI 封装的原生 Web 框架实例的暴露。通过 `app.native`，开发者可以突破 NiceGUI 上层封装的限制，直接调用 FastAPI/Starlette 的所有原生 API，实现高级路由配置、中间件定制、事件注册、ASGI 应用组合等底层操作，是连接 NiceGUI 上层组件与底层 Web 框架的关键桥梁。本文将从**核心原理、基础特性、典型应用场景、与 NiceGUI 上层 API 的协同、注意事项**五个维度，全方位解析 `app.native` 的机制与使用方式。

## 一、app.native 的核心原理

### 1.1 NiceGUI 的架构分层

NiceGUI 是一款**基于 FastAPI/Starlette 构建的前端框架**，其架构可分为三层：

1. **上层**：NiceGUI 封装的组件系统（如 `ui.button`、`ui.page`）、状态管理（`ui.session`、`app.state`）、生命周期钩子（`app.on_startup`）等易用性 API；
2. **中层**：NiceGUI 对底层 Web 框架的适配层，负责将上层组件的交互逻辑映射为 Web 请求 / 响应、WebSocket 通信；
3. **底层**：FastAPI/Starlette 原生应用实例，处理 HTTP 路由、WebSocket 连接、ASGI 协议交互等核心 Web 能力。

`app.native` 正是对**底层 FastAPI/Starlette 原生应用实例**的直接引用，开发者通过它可以绕开 NiceGUI 的中层适配，直接操作底层框架的所有功能。

### 1.2 app.native 的本质与类型

- **默认类型**：`app.native` 本质上是一个 **FastAPI 应用实例（`fastapi.FastAPI` 对象）**，而 FastAPI 本身又是对 Starlette 应用的扩展（继承自 `starlette.applications.Starlette`）；
- **核心继承关系**：`fastapi.FastAPI` → `starlette.applications.Starlette` → `starlette.routing.Router`，因此 `app.native` 同时具备 FastAPI 的 API 文档、依赖注入、数据验证能力，以及 Starlette 的轻量路由、WebSocket、ASGI 适配能力；
- **不可替代性**：NiceGUI 的上层 API 仅封装了底层框架的常用功能，而 `app.native` 则暴露了所有原生能力，是实现高级定制的唯一入口。

### 1.3 app.native 与 app 的关系

NiceGUI 中的 `app` 是一个**封装后的应用对象**，提供了简化的上层 API（如 `app.middleware`、`app.on_startup`），而这些 API 最终都会通过 `app.native` 转发到底层 FastAPI 实例。二者的核心关系：

- **封装与被封装**：`app` 是对 `app.native` 的上层封装，`app.native` 是 `app` 的底层实现载体；
- **API 转发**：`app` 的多数方法（如 `app.add_middleware`）本质是调用 `app.native.add_middleware`；
- **能力互补**：`app` 提供易用的简化 API，`app.native` 提供底层的完整能力。

## 二、app.native 的基础特性与原生 API 访问

`app.native` 直接暴露了 FastAPI/Starlette 的所有原生 API，开发者可以像使用纯 FastAPI 应用一样操作它，以下是最核心的原生能力访问方式。

### 2.1 直接配置 FastAPI 原生属性

FastAPI 应用的核心配置（如文档地址、跨域默认配置、标题）均可通过 `app.native` 直接修改，覆盖 NiceGUI 的默认设置。

```python
from nicegui import ui, app

# 通过 app.native 配置 FastAPI 原生属性
app.native.title = "NiceGUI Native 示例"  # 应用标题（显示在 OpenAPI 文档中）
app.native.description = "通过 app.native 操作底层 FastAPI 实例"  # 应用描述
app.native.version = "1.0.0"  # 应用版本
app.native.docs_url = "/api-docs"  # 自定义 Swagger 文档地址（默认 /docs）
app.native.redoc_url = "/redoc-docs"  # 自定义 ReDoc 文档地址（默认 /redoc）
app.native.openapi_url = "/openapi.json"  # 自定义 OpenAPI 规范地址

@ui.page('/')
def index():
    ui.label('通过 app.native 配置 FastAPI 原生属性').classes('text-3xl')
    ui.link('查看 Swagger 文档', '/api-docs').classes('mt-4')

if __name__ in {'__main__'}:
    ui.run()
```

### 2.2 注册 FastAPI 原生 API 路由

通过 `app.native` 可以直接使用 FastAPI 的装饰器注册原生 API 路由，支持 FastAPI 特有的**数据验证、依赖注入、响应模型**等高级特性，与 NiceGUI 的页面路由无缝共存。

```python
from nicegui import ui, app
from pydantic import BaseModel
from fastapi import Depends, HTTPException

# 定义 Pydantic 数据模型（FastAPI 原生特性）
class User(BaseModel):
    name: str
    age: int
    email: str | None = None

# 定义依赖注入函数（FastAPI 原生特性）
def get_token(token: str = "default_token"):
    if token != "valid_token":
        raise HTTPException(status_code=401, detail="无效的令牌")
    return token

# 通过 app.native 注册 FastAPI 原生 API 路由
@app.native.get("/api/users/{user_id}")
async def get_user(user_id: int, token: str = Depends(get_token)):
    """获取用户信息（支持路径参数、依赖注入）"""
    return {
        "user_id": user_id,
        "name": "张三",
        "age": 20,
        "token": token
    }

@app.native.post("/api/users")
async def create_user(user: User):
    """创建用户（支持请求体数据验证）"""
    return {
        "message": "用户创建成功",
        "user": user
    }

@ui.page('/')
def index():
    ui.label('FastAPI 原生 API 示例').classes('text-3xl')
    ui.link('查看 Swagger 文档', '/api-docs').classes('mt-4')

if __name__ in {'__main__'}:
    ui.run()
```

### 2.3 注册 Starlette 原生 WebSocket 路由

Starlette 提供了轻量的 WebSocket 路由支持，通过 `app.native` 可以注册原生 WebSocket 路由，与 NiceGUI 内置的 WebSocket 通信互补，实现更灵活的实时交互。

```python
from nicegui import ui, app
from starlette.websockets import WebSocket

# 通过 app.native 注册 Starlette 原生 WebSocket 路由
@app.native.websocket("/ws/echo")
async def echo_websocket(websocket: WebSocket):
    """原生 WebSocket 回声服务"""
    await websocket.accept()
    while True:
        # 接收客户端消息
        data = await websocket.receive_text()
        # 回声发送
        await websocket.send_text(f"服务器收到（原生 WebSocket）：{data}")

@ui.page('/')
def index():
    ui.label('Starlette 原生 WebSocket 示例').classes('text-3xl')
    # 前端连接原生 WebSocket 路由
    ui.run_javascript('''
        const ws = new WebSocket(`ws://${window.location.host}/ws/echo`);
        ws.onopen = () => ws.send('Hello NiceGUI Native!');
        ws.onmessage = (e) => {
            console.log('原生 WebSocket 消息：', e.data);
            alert(e.data);
        };
    ''')

if __name__ in {'__main__'}:
    ui.run()
```

### 2.4 添加 Starlette/FastAPI 原生中间件

通过 `app.native.add_middleware` 可以添加 Starlette/FastAPI 的所有原生中间件，覆盖 NiceGUI 上层 `app.middleware` 的简化封装，实现更精细的请求处理。

```python
from nicegui import ui, app
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware
from starlette.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.cors import CORSMiddleware

# 通过 app.native 添加 FastAPI/Starlette 原生中间件
# 1. 强制 HTTPS 重定向（生产环境使用）
# app.native.add_middleware(HTTPSRedirectMiddleware)
# 2. 信任主机中间件，限制允许的域名
app.native.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "127.0.0.1"]
)
# 3. 跨域中间件（FastAPI 原生实现，功能更丰富）
app.native.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.native.middleware("http")
async def custom_native_middleware(request, call_next):
    """FastAPI 原生 HTTP 中间件"""
    print(f"原生中间件：收到请求 {request.method} {request.url}")
    response = await call_next(request)
    response.headers["X-Native-Middleware"] = "FastAPI"
    return response

@ui.page('/')
def index():
    ui.label('原生中间件示例').classes('text-3xl')

if __name__ in {'__main__'}:
    ui.run()
```

## 三、app.native 的典型应用场景

`app.native` 的核心价值在于突破 NiceGUI 上层封装的限制，实现底层 Web 框架的高级定制，以下是开发中最常见的典型应用场景。

### 3.1 实现 FastAPI 原生的依赖注入与接口文档

FastAPI 的核心优势之一是**自动生成 OpenAPI 文档**和**强大的依赖注入系统**，通过 `app.native` 可以在 NiceGUI 中充分利用这些特性，构建标准化的 RESTful API，同时保留 NiceGUI 的前端组件能力。

```python
from nicegui import ui, app
from fastapi import Depends, HTTPException, Query
from pydantic import BaseModel
from typing import List, Optional

# 定义数据模型
class Item(BaseModel):
    name: str
    price: float
    is_offer: Optional[bool] = None

# 定义依赖：分页参数
def get_pagination(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100)
):
    return {"skip": skip, "limit": limit}

# 模拟数据库
fake_items_db = [{"name": "Foo", "price": 42}, {"name": "Bar", "price": 56}]

# 注册 FastAPI 原生 API，支持依赖注入和自动文档
@app.native.get("/api/items/", response_model=List[Item])
async def read_items(pagination: dict = Depends(get_pagination)):
    """获取物品列表（支持分页）"""
    skip = pagination["skip"]
    limit = pagination["limit"]
    return fake_items_db[skip : skip + limit]

@app.native.get("/api/items/{item_id}", response_model=Item)
async def read_item(item_id: int, q: Optional[str] = None):
    """获取单个物品"""
    if item_id < 1 or item_id > len(fake_items_db):
        raise HTTPException(status_code=404, detail="Item not found")
    item = fake_items_db[item_id - 1]
    if q:
        item["q"] = q
    return item

@ui.page('/')
def index():
    ui.label('FastAPI 原生 API 与文档示例').classes('text-3xl')
    ui.link('查看 Swagger 文档（自动生成）', '/api-docs').classes('mt-4')
    ui.link('查看 ReDoc 文档（自动生成）', '/redoc-docs').classes('mt-4')

if __name__ in {'__main__'}:
    ui.run()
```

### 3.2 组合多个 ASGI 应用

ASGI 是 Python 异步 Web 框架的标准协议，通过 `app.native.mount` 可以将其他 ASGI 应用（如 FastAPI 子应用、Starlette 子应用、静态文件服务）挂载到 NiceGUI 应用中，实现应用的模块化拆分与组合。

```python
from nicegui import ui, app
from fastapi import FastAPI
from starlette.staticfiles import StaticFiles

# 1. 创建 FastAPI 子应用
api_app = FastAPI(title="子应用 API")

@api_app.get("/sub/hello")
async def sub_hello():
    return {"message": "Hello from Sub App"}

# 2. 通过 app.native 挂载子应用
app.native.mount("/sub-api", api_app)

# 3. 挂载静态文件服务（Starlette 原生）
# 确保 static 目录存在，存放静态文件（如图片、JS、CSS）
app.native.mount("/static", StaticFiles(directory="static"), name="static")

@ui.page('/')
def index():
    ui.label('ASGI 应用组合示例').classes('text-3xl')
    ui.link('访问子应用 API', '/sub-api/sub/hello').classes('mt-2')
    ui.link('访问静态文件（示例：/static/test.jpg）', '/static').classes('mt-2')

if __name__ in {'__main__'}:
    ui.run()
```

### 3.3 自定义 FastAPI 的异常处理器

FastAPI 允许自定义全局异常处理器，通过 `app.native` 可以注册原生的异常处理函数，替代 NiceGUI 的默认错误页面，实现更灵活的错误处理。

```python
from nicegui import ui, app
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse, HTMLResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

# 1. 自定义 FastAPI 原生异常处理器：处理 HTTP 异常
@app.native.exception_handler(StarletteHTTPException)
async def http_exception_handler(request, exc):
    """自定义 HTTP 异常处理，返回 JSON 响应"""
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail, "path": str(request.url.path)},
    )

# 2. 自定义全局异常处理器：处理所有未捕获的异常
@app.native.exception_handler(Exception)
async def general_exception_handler(request, exc):
    """自定义全局异常处理，返回 HTML 页面"""
    return HTMLResponse(
        status_code=500,
        content=f"<h1>服务器内部错误</h1><p>错误信息：{str(exc)}</p>",
    )

# 测试异常的 API
@app.native.get("/api/error/{code}")
async def trigger_error(code: int):
    if code == 404:
        raise HTTPException(status_code=404, detail="资源未找到")
    elif code == 500:
        raise Exception("模拟服务器内部错误")
    return {"message": "正常响应"}

@ui.page('/')
def index():
    ui.label('自定义异常处理器示例').classes('text-3xl')
    ui.link('触发 404 异常', '/api/error/404').classes('mt-2')
    ui.link('触发 500 异常', '/api/error/500').classes('mt-2')

if __name__ in {'__main__'}:
    ui.run()
```

### 3.4 集成 FastAPI 第三方插件

FastAPI 拥有丰富的第三方生态，通过 `app.native` 可以将 FastAPI 插件（如身份认证、缓存、限流）无缝集成到 NiceGUI 应用中，扩展应用的底层能力。

以下示例集成 **FastAPI Limiter**（请求限流插件），实现 API 接口的频率限制：

```bash
# 先安装依赖
pip install fastapi-limiter redis
```

```python
from nicegui import ui, app
from fastapi import FastAPI
from fastapi_limiter import FastAPILimiter
from fastapi_limiter.depends import RateLimiter
import redis.asyncio as redis

# 初始化 Redis 连接（用于限流的计数器存储）
@app.on_startup
async def init_limiter():
    redis_client = redis.from_url("redis://localhost", encoding="utf-8", decode_responses=True)
    await FastAPILimiter.init(redis_client)

# 通过 app.native 注册带限流的 API
@app.native.get(
    "/api/limited",
    dependencies=[Depends(RateLimiter(times=5, seconds=10))]  # 10 秒内最多 5 次请求
)
async def limited_api():
    return {"message": "成功访问限流接口"}

@ui.page('/')
def index():
    ui.label('FastAPI 第三方插件集成（限流）').classes('text-3xl')
    ui.link('访问限流接口（10 秒内最多 5 次）', '/api/limited').classes('mt-4')

if __name__ in {'__main__'}:
    ui.run()
```

### 3.5 直接操作 Starlette 的请求 / 响应对象

通过 `app.native` 可以在路由中直接使用 Starlette 的原生 `Request` 和 `Response` 对象，实现对请求 / 响应的精细控制（如自定义响应头、Cookie、状态码）。

```python
from nicegui import ui, app
from starlette.requests import Request
from starlette.responses import Response, RedirectResponse

# 原生路由：直接操作 Request 和 Response
@app.native.get("/api/raw")
async def raw_request_response(request: Request):
    """获取原生请求信息，返回自定义响应"""
    # 获取请求头、客户端 IP 等原生信息
    client_ip = request.client.host
    user_agent = request.headers.get("user-agent", "Unknown")
    # 构建自定义响应
    response = Response(
        content=f"客户端 IP：{client_ip}\nUser-Agent：{user_agent}",
        media_type="text/plain",
        status_code=200,
    )
    # 设置自定义响应头和 Cookie
    response.headers["X-Client-IP"] = client_ip
    response.set_cookie("nicegui_native", "test_value", max_age=3600)
    return response

# 原生路由：重定向
@app.native.get("/api/redirect")
async def redirect():
    return RedirectResponse(url="/")

@ui.page('/')
def index():
    ui.label('原生 Request/Response 操作示例').classes('text-3xl')
    ui.link('访问原生 API', '/api/raw').classes('mt-2')
    ui.link('测试重定向', '/api/redirect').classes('mt-2')

if __name__ in {'__main__'}:
    ui.run()
```

## 四、app.native 与 NiceGUI 上层 API 的协同使用

`app.native` 并非替代 NiceGUI 的上层 API，而是与其**协同工作**，实现 “前端用 NiceGUI 快速开发，后端用 FastAPI 原生能力构建标准化 API” 的全栈开发模式。以下是关键的协同原则与示例。

### 4.1 路由协同：NiceGUI 页面与 FastAPI API 共存

NiceGUI 的 `ui.page` 注册的前端页面路由，与 `app.native` 注册的 FastAPI 原生 API 路由可以无缝共存，共享同一个 Web 服务端口，实现前后端一体化开发。

```python
from nicegui import ui, app
from pydantic import BaseModel

# FastAPI 原生 API 路由（后端）
class Message(BaseModel):
    content: str

@app.native.post("/api/send-message")
async def send_message(message: Message):
    return {"status": "success", "received": message.content}

# NiceGUI 页面路由（前端）
@ui.page('/')
def index():
    ui.label('前后端协同示例').classes('text-3xl')
    # 输入框和发送按钮
    msg_input = ui.input(label='请输入消息').classes('w-full mt-4')
    send_btn = ui.button('发送到 API')
    # 发送请求到 FastAPI 原生 API
    send_btn.on_click(lambda: ui.run_javascript(f'''
        fetch('/api/send-message', {{
            method: 'POST',
            headers: {{'Content-Type': 'application/json'}},
            body: JSON.stringify({{content: '{msg_input.value}'}})
        }}).then(res => res.json()).then(data => {{
            alert(`API 响应：${{JSON.stringify(data)}}`);
        }});
    '''))

if __name__ in {'__main__'}:
    ui.run()
```

### 4.2 状态协同：app.state 与 FastAPI 原生状态共享

NiceGUI 的 `app.state` 本质是存储在 `app.native.state` 中的全局状态，因此通过 `app.native.state` 也可以访问和修改全局状态，实现上层 API 与原生 API 的状态共享。

```python
from nicegui import ui, app

# 初始化全局状态（NiceGUI 上层 API）
app.state.counter = 0

# FastAPI 原生 API：修改全局状态
@app.native.get("/api/counter/increment")
async def increment_counter():
    app.state.counter += 1  # 直接修改 NiceGUI 的全局状态
    return {"counter": app.state.counter}

# NiceGUI 页面：展示并修改全局状态
@ui.page('/')
def index():
    counter_label = ui.label(f'全局计数器：{app.state.counter}').classes('text-3xl')
    # 页面内修改计数器
    ui.button('页面内加 1', on_click=lambda: (setattr(app.state, 'counter', app.state.counter + 1), counter_label.set_text(f'全局计数器：{app.state.counter}'))).classes('mt-2')
    # 通过 API 修改计数器
    ui.button('通过 API 加 1', on_click=lambda: ui.run_javascript('''
        fetch('/api/counter/increment').then(res => res.json()).then(data => {
            document.querySelector('p').textContent = `全局计数器：${data.counter}`;
        });
    ''')).classes('mt-2')

if __name__ in {'__main__'}:
    ui.run()
```

### 4.3 生命周期协同：NiceGUI 钩子与 FastAPI 原生钩子共享

NiceGUI 的 `app.on_startup`/`app.on_shutdown` 钩子，本质是注册到 `app.native` 的 FastAPI 原生生命周期钩子，因此二者可以混合使用，实现初始化逻辑的拆分。

```python
from nicegui import ui, app

# NiceGUI 上层启动钩子
@app.on_startup
async def nicegui_startup():
    print("NiceGUI 启动钩子执行")
    app.state.nicegui_init = True

# FastAPI 原生启动钩子
@app.native.on_event("startup")
async def fastapi_startup():
    print("FastAPI 原生启动钩子执行")
    app.state.fastapi_init = True

# NiceGUI 上层关闭钩子
@app.on_shutdown
async def nicegui_shutdown():
    print("NiceGUI 关闭钩子执行")

# FastAPI 原生关闭钩子
@app.native.on_event("shutdown")
async def fastapi_shutdown():
    print("FastAPI 原生关闭钩子执行")

@ui.page('/')
def index():
    ui.label('生命周期钩子协同示例').classes('text-3xl')
    ui.label(f'NiceGUI 初始化：{app.state.nicegui_init}').classes('mt-2')
    ui.label(f'FastAPI 初始化：{app.state.fastapi_init}').classes('mt-2')

if __name__ in {'__main__'}:
    ui.run()
```

## 五、使用 app.native 的注意事项

`app.native` 提供了强大的底层定制能力，但使用不当可能导致 NiceGUI 上层功能异常，需注意以下关键问题：

### 5.1 避免覆盖 NiceGUI 的核心配置

- NiceGUI 会自动配置部分底层参数（如 WebSocket 路由、静态文件服务、会话管理），通过 `app.native` 修改这些配置时需谨慎，避免覆盖 NiceGUI 的核心逻辑；
- 例如：不要随意修改 `app.native.router` 的核心路由规则，以免导致 NiceGUI 页面无法访问。

### 5.2 异步与同步的兼容性

- FastAPI/Starlette 是**异步优先**的框架，`app.native` 的所有原生 API 均支持异步，需避免在原生路由 / 中间件中执行同步阻塞操作（如耗时的数据库查询），建议使用 `asyncio.to_thread` 封装；
- NiceGUI 的上层 API 部分为同步封装，与原生异步 API 混合使用时，需确保异步上下文的正确性。

### 5.3 依赖版本的一致性

- NiceGUI 对 FastAPI/Starlette 的版本有特定要求，通过 `app.native` 集成第三方 FastAPI 插件时，需确保插件与 NiceGUI 依赖的 FastAPI/Starlette 版本兼容，避免版本冲突；
- 可通过 `pip show nicegui` 查看 NiceGUI 的依赖版本。

### 5.4 避免重复注册中间件 / 钩子

- NiceGUI 的上层 API（如 `app.middleware`、`app.on_startup`）与 `app.native` 的原生 API 会注册到同一底层实例，避免重复注册相同的中间件 / 钩子，导致逻辑执行多次；
- 例如：不要同时通过 `app.middleware` 和 `app.native.add_middleware` 注册同一个跨域中间件。

### 5.5 生产环境的部署兼容性

- 通过 `app.native` 定制的底层功能，在生产环境部署时（如使用 Uvicorn/Gunicorn）需确保部署方式与 FastAPI 原生应用一致；
- 例如：使用 Gunicorn 部署时，需指定 `worker_class=uvicorn.workers.UvicornWorker`，以支持 ASGI 异步特性。

## 六、总结

`app.native` 是 NiceGUI 框架中**连接上层组件与底层 FastAPI/Starlette 框架的关键桥梁**，其本质是对原生 Web 应用实例的直接暴露。通过 `app.native`，开发者可以突破 NiceGUI 上层封装的限制，充分利用 FastAPI/Starlette 的原生能力，实现高级 API 设计、依赖注入、接口文档、ASGI 应用组合、第三方插件集成等底层定制，同时与 NiceGUI 的前端组件系统无缝协同，构建 “前端快速开发，后端标准化设计” 的全栈 Web 应用。

开发中的最佳实践总结：

1. **分层开发**：前端页面使用 NiceGUI 的上层组件 API，后端接口使用 `app.native` 注册 FastAPI 原生 API；
2. **状态共享**：通过 `app.state` 实现 NiceGUI 上层与 FastAPI 原生代码的全局状态共享；
3. **生态集成**：利用 `app.native` 集成 FastAPI 丰富的第三方插件，扩展应用的底层能力；
4. **谨慎定制**：修改底层配置时避免覆盖 NiceGUI 的核心逻辑，确保上层功能的稳定性；
5. **异步优先**：原生代码中优先使用异步 API，避免同步阻塞操作影响应用性能。

通过合理使用 `app.native`，可以充分发挥 NiceGUI 与 FastAPI/Starlette 的协同优势，兼顾开发效率与底层能力，构建出功能强大、体验优秀的现代 Web 应用。

