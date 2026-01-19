# ui.dialog 全面详解

ui.dialog 是 NiceGUI 基于 Quasar QDialog 组件封装的模态对话框组件，核心用于在当前页面之上弹出交互窗口，阻断背景操作并聚焦用户注意力，支持自定义内容、按钮布局、尺寸控制、动画效果等高级功能。其本质是「模态交互容器」，适用于需要用户明确响应的场景（如确认操作、表单填写、信息展示等），是界面中实现复杂交互的核心组件之一。

## 一、核心概念与基础特性

### 1. 本质与用途

- 本质：模态弹出窗口，通过「触发元素」或「程序调用」显示，默认点击背景或关闭按钮时关闭，支持自定义关闭逻辑。
- 核心用途：适用于需要用户必须处理的交互场景，常见场景包括：
  - 确认操作：如 “删除数据？”“退出登录？” 等危险操作二次确认；
  - 表单填写：如 “新增用户”“编辑信息” 等需要分步输入的场景；
  - 信息展示：如 “系统公告”“详情弹窗” 等需要展示大量内容的场景；
  - 加载 / 处理中：如 “数据导出中”“文件上传中” 等需要用户等待的场景。
- 关键机制：
  - 模态特性：显示时阻断背景页面交互，用户必须关闭对话框才能继续操作；
  - 触发方式：支持绑定触发元素（如按钮）点击显示，或通过程序调用 `open()` 手动显示；
  - 布局支持：内部可嵌套任意 NiceGUI 组件（表单、表格、图片等），支持复杂布局；
  - 动画效果：默认带弹出 / 关闭动画，支持自定义动画类型和时长；
  - 位置适配：默认居中显示，支持自定义位置，页面边缘会自动调整以确保完全可见。

### 2. 基础结构

ui.dialog 有两种核心使用方式：**绑定触发元素**（自动关联显示 / 隐藏逻辑）和 **程序控制**（手动调用 `open()`/`close()`），基础示例如下：

```python
from nicegui import ui

# 1. 绑定触发元素（推荐，自动关联显示逻辑）
with ui.button('打开对话框', classes='mt-4') as trigger:
    with ui.dialog(trigger=trigger) as dialog:
        with ui.card():  # 用 card 优化内容布局（可选）
            ui.label('这是一个基础对话框').classes('text-lg font-medium mb-4')
            ui.label('用于展示简单信息或确认操作').classes('text-gray-600')
            with ui.row(justify='end').classes('mt-6'):
                ui.button('取消', on_click=dialog.close).classes('mr-2')
                ui.button('确认', on_click=lambda: (dialog.close(), ui.notify('确认操作'))).props('color=primary')

# 2. 程序控制（手动调用 open()/close()）
dialog2 = ui.dialog()
with dialog2, ui.card():
    ui.label('程序控制的对话框').classes('text-lg font-medium mb-4')
    ui.label('通过按钮手动触发显示/关闭').classes('text-gray-600')
    ui.button('关闭', on_click=dialog2.close).classes('mt-4')

ui.button('手动打开对话框', on_click=dialog2.open).classes('mt-2')

ui.run()
```

## 二、初始化配置项

实例化 `ui.dialog(**kwargs)` 时可通过参数配置触发方式、显示行为、样式等核心属性，参数说明如下（基于官方文档完整配置）：

| 参数名              | 类型            | 说明                                                         |
| ------------------- | --------------- | ------------------------------------------------------------ |
| trigger             | Element \| None | 绑定的触发元素（可选），点击该元素自动显示对话框；设为 `None` 时需手动调用 `open()` |
| value               | bool            | 初始显示状态（默认 `False`，即隐藏；设为 `True` 时初始显示） |
| on_value_change     | Callable        | 对话框显示 / 隐藏状态变化时触发的回调函数（接收 `ValueChangeEventArguments` 参数，`e.value` 为当前状态） |
| full_width          | bool            | 是否占满屏幕宽度（默认 `False`），适配移动端或宽屏场景       |
| full_height         | bool            | 是否占满屏幕高度（默认 `False`），适用于全屏表单或详情页     |
| max_width           | str \| int      | 最大宽度（默认 `'80%'`），支持像素值（如 `600`）、百分比（如 `'90%'`）或 CSS 单位（如 `'60rem'`） |
| max_height          | str \| int      | 最大高度（默认 `'80%'`），超出部分滚动显示                   |
| min_width           | str \| int      | 最小宽度（默认 `None`），支持像素值或 CSS 单位               |
| min_height          | str \| int      | 最小高度（默认 `None`），支持像素值或 CSS 单位               |
| position            | str             | 显示位置（默认 `'center'`），支持值：`'top'`/`'bottom'`/`'left'`/`'right'`/`'top-left'`/`'top-right'`/`'bottom-left'`/`'bottom-right'` |
| bordered            | bool            | 是否显示边框（默认 `False`），配合 `rounded` 可优化外观      |
| rounded             | bool            | 是否显示圆角（默认 `True`），设为 `False` 时为直角边框       |
| shadow              | bool            | 是否显示阴影（默认 `True`），设为 `False` 时无阴影效果       |
| modal               | bool            | 是否启用模态（默认 `True`），设为 `False` 时背景页面可交互   |
| persistent          | bool            | 是否点击背景不关闭（默认 `False`），设为 `True` 时仅可通过按钮关闭 |
| transition          | str             | 显示 / 关闭动画（默认 `'scale'`），支持值：`'fade'`/`'slide-up'`/`'slide-down'`/`'slide-left'`/`'slide-right'`/`'none'` |
| transition_duration | int             | 动画时长（单位：毫秒，默认 `300`）                           |
| on_close            | Callable        | 对话框关闭时触发的回调函数（无参数或接收 `UiEventArguments` 参数） |
| on_open             | Callable        | 对话框打开时触发的回调函数（无参数或接收 `UiEventArguments` 参数） |
| classes             | str             | CSS 类名（支持 Tailwind、Quasar 类，如 `'bg-gray-50'`）      |
| props               | str             | Quasar 组件属性（如 `'no-esc-dismiss'` 禁用 ESC 键关闭、`'no-backdrop-dismiss'` 禁用背景点击关闭） |
| style               | str             | 内联 CSS 样式（如 `'font-size: 14px;'`）                     |

### 配置示例（多参数组合）

```python
from nicegui import ui

# 复杂配置对话框：全屏宽度、自定义位置、禁用背景关闭、滑动动画
def on_dialog_open():
    ui.notify('对话框已打开')

def on_dialog_close():
    ui.notify('对话框已关闭')

with ui.button('复杂配置对话框', classes='mt-4') as trigger:
    dialog = ui.dialog(
        trigger=trigger,
        full_width=True,
        position='bottom',
        modal=True,
        persistent=True,  # 点击背景不关闭
        transition='slide-up',
        transition_duration=400,
        max_height='70%',
        bordered=True,
        on_open=on_dialog_open,
        on_close=on_dialog_close,
        props='no-esc-dismiss'  # 禁用 ESC 键关闭
    )
    with dialog, ui.card().classes('p-6'):
        ui.label('底部弹出的全屏宽度对话框').classes('text-lg font-medium mb-4')
        ui.label('仅可通过关闭按钮关闭，支持滑动动画').classes('text-gray-600 mb-6')
        with ui.row(justify='end'):
            ui.button('关闭', on_click=dialog.close).props('color=primary')

ui.run()
```

## 三、核心属性（可动态修改）

ui.dialog 实例的核心属性支持**动态赋值修改**，无需重新创建即可更新界面，关键属性如下：

| 属性名      | 类型                 | 说明                                                         |
| ----------- | -------------------- | ------------------------------------------------------------ |
| value       | bool（可设置）       | 对话框显示状态（`True` 显示、`False` 隐藏），支持双向绑定    |
| full_width  | bool（可设置）       | 动态切换是否占满宽度（如 `dialog.full_width = True`）        |
| full_height | bool（可设置）       | 动态切换是否占满高度                                         |
| max_width   | str \| int（可设置） | 动态修改最大宽度（如 `dialog.max_width = 800`）              |
| max_height  | str \| int（可设置） | 动态修改最大高度                                             |
| position    | str（可设置）        | 动态修改显示位置（如 `dialog.position = 'top'`）             |
| bordered    | bool（可设置）       | 动态切换是否显示边框                                         |
| rounded     | bool（可设置）       | 动态切换是否显示圆角                                         |
| shadow      | bool（可设置）       | 动态切换是否显示阴影                                         |
| modal       | bool（可设置）       | 动态切换是否启用模态                                         |
| persistent  | bool（可设置）       | 动态切换是否点击背景关闭                                     |
| visible     | BindableProperty     | 元素可见性（布尔值，支持双向绑定，与 `value` 区别：`visible` 控制元素是否存在，`value` 控制是否显示） |
| html_id     | str                  | HTML DOM 中的元素 ID（版本 2.16.0 新增，只读）               |
| is_deleted  | bool                 | 元素是否已删除（只读）                                       |

### 属性动态修改示例

```python
from nicegui import ui

# 动态修改对话框属性
dialog = ui.dialog()
with dialog, ui.card().classes('p-6'):
    ui.label('可动态修改属性的对话框').classes('text-lg font-medium mb-4')
    ui.label('点击按钮切换对话框样式和位置').classes('text-gray-600 mb-6')
    with ui.row(justify='end'):
        ui.button('关闭', on_click=dialog.close).props('color=primary')

# 控制按钮：修改位置和尺寸
def toggle_position():
    dialog.position = 'top' if dialog.position == 'center' else 'center'
    dialog.full_width = not dialog.full_width
    ui.notify(f'位置：{dialog.position}，全屏宽度：{dialog.full_width}')

# 控制按钮：修改样式
def toggle_style():
    dialog.bordered = not dialog.bordered
    dialog.shadow = not dialog.shadow
    ui.notify(f'边框：{dialog.bordered}，阴影：{dialog.shadow}')

ui.row(
    ui.button('打开对话框', on_click=dialog.open).classes('mt-4'),
    ui.button('切换位置/尺寸', on_click=toggle_position).classes('ml-2'),
    ui.button('切换样式', on_click=toggle_style).classes('ml-2')
)

ui.run()
```

## 四、核心方法

ui.dialog 实例提供丰富方法用于控制生命周期、状态和交互，常用方法如下：

| 方法名                        | 作用                                                | 示例                                           |
| ----------------------------- | --------------------------------------------------- | ---------------------------------------------- |
| open()                        | 手动打开对话框（触发 `on_open` 回调）               | `dialog.open()`                                |
| close()                       | 手动关闭对话框（触发 `on_close` 回调）              | `dialog.close()`                               |
| toggle()                      | 切换对话框显示 / 隐藏状态                           | `dialog.toggle()`                              |
| set_value(value: bool)        | 设置显示状态（`True` 打开，`False` 关闭）           | `dialog.set_value(True)`                       |
| set_visibility(visible: bool) | 控制元素可见性（`True` 显示元素，`False` 隐藏元素） | `dialog.set_visibility(False)`                 |
| update()                      | 强制更新客户端界面（修改属性后调用，确保同步）      | `dialog.update()`                              |
| delete()                      | 永久删除对话框元素（无法恢复）                      | `dialog.delete()`                              |
| on_open(callback)             | 动态绑定打开回调（覆盖初始化时的 `on_open`）        | `dialog.on_open(lambda: ui.notify('已打开'))`  |
| on_close(callback)            | 动态绑定关闭回调（覆盖初始化时的 `on_close`）       | `dialog.on_close(lambda: ui.notify('已关闭'))` |
| tooltip(text: str)            | 为触发元素添加悬浮提示（需绑定 `trigger`）          | `dialog.trigger.tooltip('打开对话框')`         |

### 方法使用示例

```python
from nicegui import ui

# 方法控制示例
dialog = ui.dialog(modal=True, persistent=True)
with dialog, ui.card().classes('p-6'):
    ui.label('方法控制的对话框').classes('text-lg font-medium mb-4')
    ui.label('通过外部按钮控制打开、关闭、切换').classes('text-gray-600 mb-6')

# 绑定动态回调
dialog.on_open(lambda: ui.notify('对话框打开（动态绑定）'))
dialog.on_close(lambda: ui.notify('对话框关闭（动态绑定）'))

# 控制按钮
ui.row(
    ui.button('打开', on_click=dialog.open).classes('mt-4'),
    ui.button('关闭', on_click=dialog.close).classes('ml-2'),
    ui.button('切换', on_click=dialog.toggle).classes('ml-2'),
    ui.button('隐藏元素', on_click=lambda: dialog.set_visibility(False)).classes('ml-2'),
    ui.button('显示元素', on_click=lambda: dialog.set_visibility(True)).classes('ml-2')
)

ui.run()
```

## 五、高级用法（实战场景）

### 1. 表单对话框（核心场景）

在对话框中嵌套表单，实现 “新增 / 编辑” 数据的交互流程，适用于图书管理系统的 “新增图书”“编辑图书信息” 等场景：

```python
from nicegui import ui
from pydantic import BaseModel, Field

# 定义数据模型（用于表单验证）
class Book(BaseModel):
    name: str = Field(title='书名', min_length=1, max_length=50)
    author: str = Field(title='作者', min_length=1, max_length=30)
    status: str = Field(title='状态', default='可借阅')

# 新增图书对话框
def show_add_book_dialog():
    # 创建表单
    form = ui.form(
        fields=[
            ui.input(label='书名').props('required'),
            ui.input(label='作者').props('required'),
            ui.select(label='状态', options=['可借阅', '已借出', '已下架']).props('required'),
        ],
        validation=Book,
        on_submit=lambda e: (
            dialog.close(),
            ui.notify(f'新增图书成功：{e.value["name"]}', type='success')
        )
    )

    # 创建对话框
    dialog = ui.dialog()
    with dialog, ui.card().classes('p-6 w-96'):
        ui.label('新增图书').classes('text-xl font-medium mb-4')
        form
        with ui.row(justify='end').classes('mt-4'):
            ui.button('取消', on_click=dialog.close).classes('mr-2')
            ui.button('提交', on_click=form.submit).props('color=primary')
    
    dialog.open()

# 触发按钮
ui.button('新增图书', on_click=show_add_book_dialog).classes('mt-4')

ui.run()
```

### 2. 确认对话框（危险操作二次确认）

用于删除、退出等危险操作的二次确认，支持自定义提示文本和按钮样式：

```python
from nicegui import ui

# 通用确认对话框函数（可复用）
def confirm_dialog(
    title: str = '确认操作',
    message: str = '是否执行该操作？',
    confirm_label: str = '确认',
    cancel_label: str = '取消',
    on_confirm: Callable = None
):
    dialog = ui.dialog()
    with dialog, ui.card().classes('p-6 w-80'):
        ui.label(title).classes('text-lg font-medium mb-2')
        ui.label(message).classes('text-gray-600 mb-6')
        with ui.row(justify='end'):
            ui.button(cancel_label, on_click=dialog.close).classes('mr-2')
            ui.button(
                confirm_label,
                on_click=lambda: (dialog.close(), on_confirm() if on_confirm else None)
            ).props('color=red')
    dialog.open()

# 触发危险操作
ui.button('删除选中图书', on_click=lambda: confirm_dialog(
    title='确认删除',
    message='删除后数据无法恢复，是否继续？',
    confirm_label='删除',
    on_confirm=lambda: ui.notify('图书已删除', type='success')
)).classes('mt-4 text-red')

ui.run()
```

### 3. 加载对话框（异步操作等待）

用于文件上传、数据导出等异步操作，显示加载动画并阻止用户交互：

```python
from nicegui import ui
import asyncio

# 加载对话框（异步操作等待）
async def export_books():
    # 打开加载对话框
    loading_dialog = ui.dialog(modal=True, persistent=True)
    with loading_dialog, ui.card().classes('p-6 w-64 text-center'):
        ui.spinner(size='3rem').classes('mb-4')
        ui.label('正在导出图书数据...').classes('text-lg')
    
    loading_dialog.open()

    # 模拟异步操作（3秒）
    await asyncio.sleep(3)

    # 关闭加载对话框，显示结果
    loading_dialog.close()
    ui.notify('图书数据导出成功', type='success')

# 触发按钮
ui.button('导出图书数据', on_click=export_books).classes('mt-4')

ui.run()
```

### 4. 嵌套对话框（特殊场景）

支持在对话框中嵌套另一个对话框（不推荐过度嵌套，避免交互复杂），适用于分步操作：

```python
from nicegui import ui

# 外层对话框
with ui.button('打开外层对话框', classes='mt-4') as trigger:
    outer_dialog = ui.dialog(trigger=trigger)
    with outer_dialog, ui.card().classes('p-6 w-96'):
        ui.label('外层对话框').classes('text-lg font-medium mb-4')
        ui.label('点击按钮打开内层对话框').classes('text-gray-600 mb-4')
        
        # 内层对话框
        inner_dialog = ui.dialog()
        with inner_dialog, ui.card().classes('p-4 w-72'):
            ui.label('内层对话框').classes('text-md font-medium mb-2')
            ui.button('关闭内层', on_click=inner_dialog.close).props('color=primary')
        
        ui.button('打开内层对话框', on_click=inner_dialog.open).classes('mr-2')
        ui.button('关闭外层', on_click=outer_dialog.close).props('color=primary')

ui.run()
```

### 5. 全屏对话框（详情展示）

用于展示大量内容（如图书详情、系统公告），占满屏幕宽度和高度，支持滚动：

```python
from nicegui import ui

# 全屏详情对话框
def show_book_detail(book: dict):
    dialog = ui.dialog(full_width=True, max_height='90%', transition='fade')
    with dialog, ui.card().classes('p-8 h-full overflow-auto'):
        ui.label(f'《{book["name"]}》详情').classes('text-2xl font-bold mb-6')
        with ui.grid(columns=2, gap=6).classes('mb-6'):
            ui.label('作者：').classes('font-medium'), ui.label(book['author'])
            ui.label('出版社：').classes('font-medium'), ui.label(book['publisher'])
            ui.label('出版日期：').classes('font-medium'), ui.label(book['publish_date'])
            ui.label('ISBN：').classes('font-medium'), ui.label(book['isbn'])
            ui.label('状态：').classes('font-medium'), ui.label(book['status'])
        ui.label('内容简介：').classes('text-xl font-medium mb-2')
        ui.label('-' * 500).classes('text-gray-600')  # 模拟长文本
        with ui.row(justify='end').classes('mt-8'):
            ui.button('关闭', on_click=dialog.close).props('color=primary')
    
    dialog.open()

# 模拟图书数据
book_data = {
    'name': 'Python 编程：从入门到实践',
    'author': '埃里克·马瑟斯',
    'publisher': '人民邮电出版社',
    'publish_date': '2020-07-01',
    'isbn': '9787115546081',
    'status': '可借阅'
}

# 触发按钮
ui.button('查看图书详情', on_click=lambda: show_book_detail(book_data)).classes('mt-4')

ui.run()
```

## 六、注意事项

1. 模态与持久化：`modal=True` 时阻断背景交互，`persistent=True` 时点击背景不关闭，两者结合适用于必须用户明确操作的场景（如表单提交、加载中）。
2. 键盘操作：默认支持 `ESC` 键关闭对话框，可通过 `props='no-esc-dismiss'` 禁用；支持 `Enter` 键提交表单（需配合 `ui.form` 组件）。
3. 布局优化：对话框内建议使用 `ui.card` 或 `ui.container` 包裹内容，配合 `p-*`（内边距）、`mt-*`（外边距）等 Tailwind 类优化布局，避免内容紧贴边框。
4. 性能注意：避免创建过多对话框实例，可通过 “复用实例”（动态修改内容）替代 “重复创建”；关闭对话框时无需删除实例，可保留用于后续再次打开。
5. 响应式适配：移动端建议使用 `full_width=True` 占满宽度，`max_height='80%'` 限制高度并允许滚动，避免内容溢出屏幕。
6. 版本兼容性：`full_width`/`full_height` 属性需 NiceGUI 1.2+ 版本；`transition_duration` 支持动态修改需 2.13.0+ 版本；`html_id` 属性需 2.16.0+ 版本，使用时需确认版本匹配。
7. 嵌套对话框限制：虽然支持嵌套，但建议最多嵌套 1 层，过度嵌套会导致用户交互复杂、视觉混乱。

通过以上配置与方法，ui.dialog 可灵活满足从简单确认到复杂表单的各类模态交互需求，是 NiceGUI 中构建用户友好界面的核心组件之一，尤其适用于图书管理系统等需要大量用户交互的项目。