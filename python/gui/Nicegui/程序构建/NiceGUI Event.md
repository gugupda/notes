# NiceGUI Event 全面详细解析

在 NiceGUI 框架中，Event（事件）是核心的通信机制，专为解决代码不同模块间的信息传递问题而设计，尤其适用于长生命周期对象（如数据模型）与短生命周期 UI 组件之间的交互。自 NiceGUI 3.0.0 版本起引入，支持同步 / 异步处理、带参传递等灵活特性，是实现代码解耦的关键工具。

## 一、核心定位与设计目标

Event 的核心价值在于**解耦**—— 避免不同功能模块（如 UI 交互、数据处理、业务逻辑）直接依赖，通过 “事件发布 - 订阅” 模式实现间接通信。其主要设计目标包括：

1. 打通长生命周期对象（如全局数据模型）与短生命周期对象（如页面 UI 组件）的信息传递通道；
2. 支持同步 / 异步回调逻辑，适配不同业务场景（如即时响应 UI、耗时数据操作）；
3. 提供灵活的事件触发与订阅机制，兼顾易用性与扩展性；
4. 内置内存泄漏防护（自动取消订阅），保障应用稳定性。

## 二、基本使用流程

Event 的使用遵循 “定义事件 → 订阅事件 → 触发事件” 的三步流程，以下通过官方示例拆解核心逻辑：

### 1. 定义事件

通过 `Event[类型注解]()` 定义事件，类型注解用于限制事件传递的参数类型（支持 Python 基础类型、自定义类型等），未指定则允许任意类型参数。

```python
from nicegui import Event

# 定义接收字符串类型参数的事件
tweet = Event[str]()
# 定义无参数事件（省略类型注解）
simple_event = Event()
```

### 2. 订阅事件

通过 `subscribe()` 方法为事件绑定回调函数，回调函数会在事件触发时执行。支持同步和异步回调，且回调参数需与事件定义的类型匹配。

```python
# 同步回调：接收事件参数并执行 UI 通知
tweet.subscribe(lambda m: ui.notify(f'Someone tweeted: "{m}"'))

# 异步回调：模拟耗时操作（如数据库写入）
import asyncio
@data_submitted.subscribe  # 装饰器语法，等价于 data_submitted.subscribe(backup)
async def backup(data: str):
    print(f'Saving "{data}"...')
    await asyncio.sleep(1)  # 模拟 IO 耗时
```

### 3. 触发事件

通过 `emit()` 或 `call()` 方法触发事件（二者核心区别在于是否等待回调完成），触发时可传递参数（需与事件定义的类型一致）。

```python
# 方式 1：使用 emit() 触发（不等待回调完成，非阻塞）
ui.button(icon='send', on_click=lambda: tweet.emit(message.value))

# 方式 2：使用 call() 触发（等待所有回调完成，阻塞）
await data_submitted.call(data.value)
```

## 三、核心方法详解

Event 提供 5 个核心方法，覆盖事件触发、订阅、取消订阅、等待事件等场景，具体说明如下：

| 方法名        | 签名                                                         | 功能描述                                         | 关键特性                                                     |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------ | ------------------------------------------------------------ |
| `call`        | `(*args: P.args, **kwargs: P.kwargs) -> None`                | 触发事件，**异步等待所有订阅的回调函数执行完成** | 阻塞当前流程，直到所有回调结束（成功 / 失败）；支持异常捕获（回调报错会向上传递） |
| `emit`        | `(*args: P.args, **kwargs: P.kwargs) -> None`                | 触发事件，**不等待回调函数执行**                 | 非阻塞，触发后立即返回；回调在后台执行，不影响当前流程       |
| `emitted`     | `(timeout: float                              | None = None) -> Any` | 等待事件被触发，返回事件传递的参数               | 用于 “被动等待事件发生” 场景（如等待 UI 交互完成）；`timeout` 为最大等待时间（None 表示无超时），超时未触发会抛出异常 |
| `subscribe`   | `(callback: Callable[P, Any]                  | Callable[[], Any], unsubscribe_on_delete: bool   | None = None) -> None` | 为事件订阅回调函数                               | 1. `callback`：事件触发时执行的函数，参数需与事件类型匹配；2. `unsubscribe_on_delete`：控制客户端断开连接时是否自动取消订阅（默认 None：UI 上下文内订阅时自动取消，避免内存泄漏；非 UI 上下文需手动取消） |
| `unsubscribe` | `(callback: Callable[P, Any]                  | Callable[[], Any]) -> None` | 取消回调函数的订阅                               | 需传入订阅时的原始回调函数（匿名函数无法取消，建议使用命名函数） |

### 方法使用场景对比

| 方法          | 适用场景                                                     | 示例场景                                           |
| ------------- | ------------------------------------------------------------ | -------------------------------------------------- |
| `call`        | 需依赖回调结果的流程（如提交数据后清空输入框、等待数据保存完成后更新 UI） | 表单提交后，等待数据写入数据库再重置按钮状态       |
| `emit`        | 无需等待结果的通知类场景（如日志上报、非关键信息推送）       | 用户操作后触发日志事件，无需等待日志写入完成       |
| `emitted`     | 需阻塞等待事件触发的场景（如等待子窗口关闭、等待异步操作回调） | 打开弹窗后，等待用户点击确认按钮再继续执行后续逻辑 |
| `subscribe`   | 初始化时绑定事件处理逻辑（如页面加载时订阅数据更新事件）     | 页面加载时订阅 “数据提交” 事件，实现数据备份逻辑   |
| `unsubscribe` | 动态解除事件绑定（如页面销毁时取消订阅、切换功能模块时解绑无用回调） | 页面关闭时，取消全局事件的订阅，避免无效回调执行   |

## 四、关键特性与注意事项

### 1. 同步与异步回调支持

Event 对同步和异步回调均原生支持，无需额外配置：

- 同步回调：直接执行，阻塞当前事件触发流程（仅 `call` 会等待，`emit` 不等待）；
- 异步回调（`async def` 定义）：`call` 会异步等待其执行完成，`emit` 会在后台异步执行（不阻塞）。

示例：混合同步与异步回调

```python
data_updated = Event[str]()

# 同步回调
def log_data(data: str):
    print(f'Data updated: {data}')

# 异步回调
async def sync_to_cloud(data: str):
    await asyncio.sleep(0.5)
    print(f'Data synced to cloud: {data}')

# 订阅两个回调
data_updated.subscribe(log_data)
data_updated.subscribe(sync_to_cloud)

# 触发事件：call 会等待两个回调都完成，emit 直接返回
await data_updated.call('test')  # 等待 0.5 秒（同步回调立即执行，异步回调耗时 0.5 秒）
data_updated.emit('test')  # 立即返回，回调在后台执行
```

### 2. 自动取消订阅与内存泄漏防护

`subscribe` 方法的 `unsubscribe_on_delete` 参数是保障内存安全的关键：

- 默认行为（None）：如果在 UI 上下文内订阅（如页面函数 `page()` 中），当客户端断开连接（如关闭页面）时，会自动取消该回调的订阅，避免因回调引用客户端对象导致内存泄漏；
- 非 UI 上下文订阅（如全局初始化时）：不会自动取消订阅，需手动调用 `unsubscribe` 解除绑定，否则回调会一直存在。

示例：手动取消订阅

```python
def handle_event():
    print('Event triggered')

# 订阅事件
event.subscribe(handle_event)

# 后续需要解除订阅时
event.unsubscribe(handle_event)
```

### 3. 事件参数传递规则

- 事件参数类型由定义时的类型注解限制（如 `Event[str]` 仅允许传递字符串），传递不匹配类型会抛出类型错误；
- 支持多参数传递：定义事件时可指定多类型注解（如 `Event[str, int]`），触发时需按顺序传递对应参数；
- 无参数事件：定义为 `Event()`，触发时无需传递参数，回调函数也不能有参数。

示例：多参数事件

```python
# 定义接收字符串（用户名）和整数（年龄）的事件
user_registered = Event[str, int]()

# 订阅回调（参数顺序与事件定义一致）
user_registered.subscribe(lambda name, age: print(f'New user: {name}, {age}'))

# 触发事件（传递对应参数）
user_registered.emit('Alice', 25)
```

## 五、典型应用场景示例

### 场景 1：UI 组件与数据模型解耦

通过事件实现 UI 操作触发数据更新，数据模型无需直接依赖 UI 组件：

```python
from nicegui import Event, ui

# 数据模型（长生命周期）
class DataModel:
    def __init__(self):
        self.value = ''
        self.updated = Event[str]()  # 数据更新事件

    def set_value(self, new_value: str):
        self.value = new_value
        self.updated.emit(new_value)  # 触发数据更新事件

# 初始化数据模型
model = DataModel()

# UI 页面（短生命周期）
@ui.page('/')
def page():
    # 订阅数据更新事件，更新 UI
    model.updated.subscribe(lambda val: ui.notify(f'Data updated to: {val}'))

    # UI 输入框触发数据更新
    with ui.row():
        input = ui.input('Enter value')
        ui.button('Update', on_click=lambda: model.set_value(input.value))

ui.run()
```

### 场景 2：使用 `call` 实现同步流程控制

表单提交后，等待数据保存完成再重置 UI 状态，确保流程一致性：

```python
import asyncio
from nicegui import Event, ui

# 定义数据提交事件
data_submitted = Event[str]()

# 异步回调：模拟数据库保存
@data_submitted.subscribe
async def save_to_db(data: str):
    print(f'Saving data: {data}')
    await asyncio.sleep(1)  # 模拟耗时操作
    if not data:
        raise ValueError('Data cannot be empty')  # 模拟异常

# UI 页面
@ui.page('/')
def page():
    async def submit():
        button.disable()  # 禁用按钮防止重复提交
        try:
            # 等待回调完成（包括异常）
            await data_submitted.call(input.value)
            input.value = ''  # 提交成功，清空输入框
            ui.notify('Submitted successfully!', color='positive')
        except Exception as e:
            ui.notify(f'Error: {str(e)}', color='negative')  # 捕获回调异常
        finally:
            button.enable()  # 无论成功失败，重新启用按钮

    with ui.row():
        input = ui.input('Enter data')
        button = ui.button('Submit', on_click=submit)

ui.run()
```

### 场景 3：使用 `emitted` 等待事件触发

等待弹窗确认事件，实现 “弹窗关闭后再执行后续逻辑”：

```python
from nicegui import Event, ui

# 定义弹窗确认事件
dialog_confirmed = Event[bool]()

@ui.page('/')
def page():
    async def open_dialog():
        # 打开弹窗
        with ui.dialog() as dialog:
            with ui.card():
                ui.label('Confirm action?')
                with ui.row():
                    ui.button('Yes', on_click=lambda: [dialog_confirmed.emit(True), dialog.close()])
                    ui.button('No', on_click=lambda: [dialog_confirmed.emit(False), dialog.close()])
        dialog.open()

        # 等待弹窗确认事件触发，获取结果
        confirmed = await dialog_confirmed.emitted(timeout=10)  # 10秒超时
        if confirmed:
            ui.notify('Action confirmed!')
        else:
            ui.notify('Action cancelled!')

    ui.button('Open Dialog', on_click=open_dialog)

ui.run()
```

## 六、总结

NiceGUI 的 Event 机制是一套灵活、高效的模块通信解决方案，核心优势包括：

1. 解耦：通过 “发布 - 订阅” 模式隔离 UI 与业务逻辑，提升代码可维护性；
2. 灵活：支持同步 / 异步回调、多参数传递、超时控制等特性，适配多样化场景；
3. 安全：内置自动取消订阅机制，有效避免内存泄漏；
4. 易用：API 简洁直观，支持装饰器与 lambda 表达式，降低使用成本。

使用关键：根据是否需要等待回调结果选择 `emit` 或 `call`，根据场景合理配置 `unsubscribe_on_delete` 参数，避免匿名函数订阅导致无法取消的问题。Event 是 NiceGUI 中连接不同模块的 “桥梁”，熟练运用可显著提升应用的架构合理性与扩展性。