# ui.expansion 全面详解

`ui.expansion` 是 NiceGUI 基于 Quasar 的 QExpansionItem 组件实现的可折叠容器组件，支持标题、副标题、图标、分组折叠（手风琴模式）、自定义头部等丰富功能，适用于需要隐藏 / 展示详细内容的场景（如菜单、详情面板、分类列表等）。

## 一、核心特性与基础概念

### 1. 核心作用

提供 “点击展开 / 折叠” 的交互容器，默认折叠状态，点击头部可显示 / 隐藏内部内容，兼顾界面简洁性与信息扩展性。

### 2. 依赖基础

基于 Quasar 框架的 QExpansionItem 组件封装，继承了 Quasar 组件的响应式、样式兼容性等特性，同时适配 NiceGUI 的 Python 语法风格与组件生态。

## 二、初始化参数（Initializer）

初始化 `ui.expansion` 时支持以下参数，用于定义组件基础属性：

| 参数名            | 类型 / 说明                                                  | 默认值 | 核心作用                                                     |
| ----------------- | ------------------------------------------------------------ | ------ | ------------------------------------------------------------ |
| `text`            | 字符串                                                       | -      | 组件头部的主标题文本（必填，除非使用自定义头部）             |
| `caption`         | 字符串（可选）                                               | None   | 标题下方的副标题 / 说明文本，用于补充标题信息                |
| `icon`            | 字符串（可选）                                               | None   | 标题左侧的图标（支持 Quasar 图标库名称，如 `'work'`、`'menu'`、`'arrow_down'`） |
| `group`           | 字符串（可选）                                               | None   | 分组名称，同组内的组件会开启 “手风琴模式”（同一时间仅一个可展开） |
| `value`           | 布尔值                                                       | False  | 组件初始化时是否展开（`True` 为默认展开，`False` 为默认折叠） |
| `on_value_change` | 回调函数（可选），参数为 `ValueChangeEventArguments` 或无参数 | None   | 当组件展开 / 折叠状态变化时触发的回调函数                    |

## 三、常用属性（Properties）

组件实例的可访问 / 修改属性，用于动态调整组件状态：

| 属性名               | 类型               | 说明                                                         |
| -------------------- | ------------------ | ------------------------------------------------------------ |
| `classes`            | `Classes[Self]`    | 组件的 CSS 类（支持 Tailwind/Quasar 类，如 `'w-full'` 设为全屏宽度） |
| `client`             | `Client`           | 组件所属的客户端实例（用于多客户端场景）                     |
| `enabled`            | `BindableProperty` | 组件是否启用（布尔值，禁用后无法点击展开 / 折叠）            |
| `html_id`            | `str`              | 组件在 HTML DOM 中的唯一 ID（2.16.0 版本新增）               |
| `icon`               | `BindableProperty` | 动态修改头部图标（与初始化 `icon` 参数一致）                 |
| `text`               | `BindableProperty` | 动态修改主标题文本                                           |
| `value`              | `BindableProperty` | 动态获取 / 设置展开状态（`True` 展开，`False` 折叠）         |
| `visible`            | `BindableProperty` | 组件是否可见（布尔值，隐藏后不占用界面空间）                 |
| `is_deleted`         | `bool`             | 组件是否已被删除（只读）                                     |
| `is_ignoring_events` | `bool`             | 组件是否忽略事件（只读，用于判断事件响应状态）               |

## 四、核心方法（Methods）

### 1. 基础交互方法

用于直接控制组件展开 / 折叠、状态修改的核心方法：

| 方法名                          | 参数说明                                 | 返回值 | 核心作用                                |
| ------------------------------- | ---------------------------------------- | ------ | --------------------------------------- |
| `open()`                        | 无                                       | None   | 强制展开组件                            |
| `close()`                       | 无                                       | None   | 强制折叠组件                            |
| `set_enabled(value: bool)`      | `value`: 是否启用（True/False）          | None   | 动态启用 / 禁用组件（禁用后点击无响应） |
| `set_icon(icon: Optional[str])` | `icon`: 图标名称（None 移除图标）        | None   | 动态修改头部图标                        |
| `set_text(text: str)`           | `text`: 新标题文本                       | None   | 动态修改主标题                          |
| `set_value(value: Any)`         | `value`: 布尔值（True 展开，False 折叠） | None   | 动态设置展开状态                        |
| `set_visibility(visible: bool)` | `visible`: 是否可见（True/False）        | None   | 动态显示 / 隐藏组件                     |
| `delete()`                      | 无                                       | None   | 删除组件及所有子元素（不可恢复）        |
| `clear()`                       | 无                                       | None   | 移除组件内所有子元素（保留组件本身）    |

### 2. 自定义头部 / 插槽方法

`ui.expansion` 支持通过插槽（Slot）自定义头部内容（如图片、按钮组合），核心方法：

| 方法名                             | 参数说明      | 返回值                                                       | 核心作用 |                                                          |
| ---------------------------------- | ------------- | ------------------------------------------------------------ | -------- | -------------------------------------------------------- |
| `add_slot(name: str, template: str | None = None)` | `name`: 插槽名称（头部用 `'header'`）；`template`: Vue 模板（可选） | `Slot`   | 添加自定义插槽，用于替换默认头部（如插入图片、复杂布局） |

### 3. 数据绑定方法

支持组件属性与 Python 对象属性双向绑定（响应式更新），常用方法：

| 方法名                                                   | 参数说明                                              | 返回值 | 核心作用                                                     |
| -------------------------------------------------------- | ----------------------------------------------------- | ------ | ------------------------------------------------------------ |
| `bind_value(target_object, target_name='value', ...)`    | 绑定目标对象及属性，支持正向 / 反向转换函数           | `Self` | 组件展开状态与目标对象属性双向绑定（一方变化，另一方自动更新） |
| `bind_text_from(target_object, target_name='text', ...)` | 从目标对象属性单向绑定标题文本                        | `Self` | 目标对象属性变化时，自动更新组件标题                         |
| `bind_icon_to(target_object, target_name='icon', ...)`   | 组件图标单向绑定到目标对象属性                        | `Self` | 组件图标变化时，自动更新目标对象属性                         |
| `bind_visibility(value: Any)`                            | 绑定可见性到目标值（仅当目标属性等于 `value` 时显示） | `Self` | 条件性显示组件（如仅当 `user.role == 'admin'` 时显示）       |

### 4. 事件监听方法

用于监听组件状态变化或用户交互：

| 方法名                        | 参数说明                                                     | 返回值 | 核心作用                                                     |
| ----------------------------- | ------------------------------------------------------------ | ------ | ------------------------------------------------------------ |
| `on_value_change(callback)`   | `callback`: 回调函数（支持无参或接收 `ValueChangeEventArguments`） | `Self` | 监听展开 / 折叠状态变化（替代初始化时的 `on_value_change` 参数） |
| `on(type: str, handler, ...)` | `type`: 事件类型（如 `'click'`）；`handler`: 事件处理函数    | `Self` | 监听通用事件（如点击、鼠标悬浮等，支持 Python/JS handler）   |
| `tooltip(text: str)`          | `text`: 提示文本                                             | `Self` | 为组件添加鼠标悬浮提示                                       |

## 五、实战示例（覆盖核心用法）

### 1. 基础用法（标题 + 图标 + 折叠内容）

```python
from nicegui import ui

# 基础折叠面板：标题+图标+全屏宽度
with ui.expansion('基础展开面板', icon='work').classes('w-full'):
    ui.label('这是折叠面板内的内容')
    ui.button('面板内按钮', on_click=lambda: ui.notify('点击了面板内按钮'))

ui.run()
```

### 2. 自定义头部（图片 + 标题组合）

```python
from nicegui import ui

with ui.expansion() as expansion:  # 不设置默认标题，用插槽自定义头部
    # 自定义头部：图片+文本组合
    with expansion.add_slot('header'):
        ui.image('https://nicegui.io/logo.png').classes('w-12 h-12 mr-2')  # 图标
        ui.label('带图片的自定义头部').classes('text-lg font-bold')  # 标题
    # 面板内容
    ui.label('自定义头部的折叠面板，支持任意UI元素组合')

ui.run()
```

### 3. 带副标题（caption）

```python
from nicegui import ui

# 标题+副标题组合
with ui.expansion('带副标题的面板', caption='这是副标题（补充说明）', icon='info').classes('w-full'):
    ui.label('副标题会显示在标题下方，字体更小、颜色更浅')

ui.run()
```

### 4. 分组手风琴模式（group）

同组内仅一个面板可展开，适合分类选择场景：

```python
from nicegui import ui

# 三个面板归为同一组，开启手风琴模式
with ui.expansion(text='分类1', group='category', value=True):  # 默认展开
    ui.label('分类1的详细内容')
with ui.expansion(text='分类2', group='category'):
    ui.label('分类2的详细内容')
with ui.expansion(text='分类3', group='category'):
    ui.label('分类3的详细内容')

ui.run()
```

### 5. 状态监听与动态控制

```python
from nicegui import ui

# 状态监听：展开/折叠时触发回调
def on_expand_change(e):
    ui.notify(f'面板状态：{"展开" if e.value else "折叠"}')

# 初始化时绑定状态变化回调
expansion = ui.expansion('可监听状态的面板', on_value_change=on_expand_change)
with expansion:
    ui.label('点击面板头部查看状态通知')

# 外部按钮控制面板展开/折叠
ui.row([
    ui.button('展开', on_click=expansion.open),
    ui.button('折叠', on_click=expansion.close),
    ui.button('切换状态', on_click=lambda: expansion.set_value(not expansion.value))
])

ui.run()
```

### 6. 数据绑定（响应式更新）

```python
from nicegui import ui

# 定义一个数据对象（支持属性绑定）
class Data:
    def __init__(self):
        self.panel_title = '绑定数据的面板'
        self.is_expanded = False

data = Data()

# 组件属性与数据对象绑定
with ui.expansion().bind_text_from(data, 'panel_title').bind_value(data, 'is_expanded'):
    ui.label('标题和展开状态与 data 对象绑定')

# 修改数据对象，组件自动更新
ui.button('修改标题', on_click=lambda: setattr(data, 'panel_title', '更新后的标题'))
ui.button('触发展开', on_click=lambda: setattr(data, 'is_expanded', True))

ui.run()
```

## 六、关键注意事项

1. **插槽使用限制**：自定义头部时需通过 `add_slot('header')` 实现，不能直接在 `ui.expansion` 初始化时混合 `text` 和自定义头部（会覆盖默认标题）。
2. **分组规则**：`group` 参数值相同的 `ui.expansion` 会自动归为一组，手风琴模式下切换展开时会自动折叠同组其他面板。
3. **样式兼容性**：`classes` 属性支持 Tailwind（如 `'w-full'`、`'p-4'`）和 Quasar 类（如 `'bg-primary'`），可混合使用调整布局和外观。
4. **版本差异**：`html_id` 属性需 NiceGUI 2.16.0+，`toggle` 参数在 `default_classes` 中需 2.7.0+，使用时注意版本兼容性。
5. **事件优先级**：`on_value_change` 仅监听展开 / 折叠状态变化，若需监听点击事件，可通过 `on('click', handler)` 额外绑定。

## 七、适用场景总结

- 分类菜单（如左侧导航栏，点击展开子菜单）；
- 详情面板（如列表项点击展开更多信息）；
- 表单分组（如折叠式表单，按模块展示输入项）；
- 可折叠面板组（如数据筛选条件，分组展示筛选项）。

通过 `ui.expansion` 的灵活配置和响应式能力，可快速实现界面的折叠交互，平衡信息密度与操作便捷性。