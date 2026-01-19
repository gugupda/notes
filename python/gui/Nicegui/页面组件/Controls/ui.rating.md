# ui.rating 全面详细阐述

`ui.rating` 是 NiceGUI 框架中基于 Quasar 的 QRating 组件实现的评分组件，自版本 2.12.0 起新增，支持评分值设置、图标自定义、颜色配置、数值绑定等丰富功能，适用于表单评分、反馈收集等场景。以下从核心特性、使用方法、高级配置、API 详情等方面展开说明。

## 一、核心特性

1. **基础评分功能**：支持设置初始评分值、最大评分数量（默认 5 星），支持整数和半整数评分（如 3.5 分）。
2. **高度自定义**：可自定义未选中、选中、半选中状态的图标，支持设置图标大小、颜色（单个颜色或渐变颜色）。
3. **双向绑定**：支持与滑块（slider）等其他组件进行数值绑定，实现实时联动。
4. **事件响应**：提供值变化回调函数，可监听用户评分操作并触发对应逻辑。
5. **样式灵活配置**：支持 Quasar、Tailwind 或 CSS 颜色格式，支持标准尺寸（xs/sm/md/lg/xl）或自定义 CSS 单位（如 16px、2rem）。

## 二、基础使用方法

### 1. 最简示例

通过默认配置快速创建评分组件，默认 5 星、初始值为 4：

```python
from nicegui import ui

ui.rating(value=4)  # 初始评分 4 星，默认 5 星最大值
ui.run()
```

### 2. 自定义图标

可分别指定未选中、选中、半选中状态的图标，支持 NiceGUI 兼容的图标名称（如情绪类、星星类图标）：

```python
from nicegui import ui

# 示例 1：使用情绪图标
ui.rating(
    value=3.5,  # 支持半星评分
    size='lg',  # 大号图标
    icon='sentiment_dissatisfied',  # 未选中图标：不满意
    icon_selected='sentiment_satisfied',  # 选中图标：满意
    icon_half='sentiment_neutral'  # 半选中图标：中性
)

# 示例 2：使用星星图标（半星专用图标）
ui.rating(
    value=3.5,
    size='lg',
    icon='star',  # 未选中：空心星
    icon_selected='star',  # 选中：实心星
    icon_half='star_half'  # 半选中：半星
)

ui.run()
```

### 3. 自定义颜色

支持单个颜色或多色渐变（按评分星级依次应用颜色列表）：

```python
from nicegui import ui

# 单个颜色（Tailwind 颜色格式）
ui.rating(value=3, color='red-10')  # 所有星级均为红色系

# 渐变颜色（列表长度需与 max 一致，默认 max=5）
ui.rating(
    value=5,
    color=['green-2', 'green-4', 'green-6', 'green-8', 'green-10']  # 1-5 星依次加深绿色
)

ui.run()
```

### 4. 调整最大评分数量

通过 `max` 参数设置最大星级，支持与滑块组件绑定实现动态调整：

```python
from nicegui import ui

# 滑块控制最大评分数量（0-10 可调）
slider = ui.slider(value=5, min=0, max=10, label='最大评分数量')
# 评分组件与滑块双向绑定：滑块值变化时，评分最大星级同步变化
ui.rating(max=10, icon='circle').bind_value(slider)

ui.run()
```

### 5. 监听值变化事件

通过 `on_change` 参数设置回调函数，响应评分变化：

```python
from nicegui import ui

def on_rating_change(event):
    ui.notify(f'你给出的评分：{event.value} 星')  # 弹出通知

ui.rating(
    value=2,
    on_change=on_rating_change  # 绑定值变化事件
)

ui.run()
```

## 三、关键参数说明

- `value`：类型为 `float | None`，用于设置评分组件的初始评分值，既支持 1 - 5 这样的整数评分，也支持 1.5、2.5 这类半整数评分，默认值为 `None`。
- `max`：类型为 `int`，用于定义评分组件的最大评分数量，也就是星级总数，默认值为 5。
- `icon`：类型为 `str`，指定评分组件中未选中状态下显示的图标名称，默认值为 `'star'`（星星图标）。
- `icon_selected`：类型为 `str`，用于设置选中状态下显示的图标名称，默认情况下与 `icon` 参数的取值保持一致。
- `icon_half`：类型为 `str`，专门指定半选中状态时显示的图标名称，默认同样与 `icon` 参数的值相同。
- `color`：类型为 `str | list[str] | None`，用于配置图标的颜色。既可以传入单个颜色值，也可以传入颜色列表实现渐变效果，颜色格式支持 Quasar、Tailwind 或 CSS 规范，默认值为 `"primary"`（框架主题色）。
- `size`：类型为 `str`，用于设置图标的大小。支持 `xs`（超小）、`sm`（小）、`md`（中）、`lg`（大）、`xl`（超大）这类标准尺寸名称，也可以使用 `16px`、`2rem` 等自定义 CSS 单位，默认情况下为自适应大小。
- `on_change`：类型为 `Callable[[ValueChangeEvent], Any]`，是评分值发生变化时触发的回调函数，函数参数为事件对象，该对象包含当前评分的 `value` 属性，默认值为 `None`。

## 四、高级功能

### 1. 组件绑定

通过 `bind_value` 实现评分组件与其他组件（如滑块、输入框）的双向绑定，实时同步值：

```python
from nicegui import ui

# 示例：评分组件与输入框双向绑定
input_box = ui.number_input(label='评分', value=3, min=0, max=5, step=0.5)
rating = ui.rating(value=3)

# 双向绑定：输入框值变化 → 评分同步；评分变化 → 输入框同步
rating.bind_value(input_box)
input_box.bind_value(rating)

ui.run()
```

### 2. 样式与类名自定义

通过 `classes`、`style` 参数调整组件样式，支持 Tailwind 类或自定义 CSS：

```python
from nicegui import ui

ui.rating(
    value=4,
    size='2rem',  # 自定义图标大小（CSS 单位）
    color='blue-600',  # CSS 颜色
    classes='shadow-md',  # Tailwind 阴影类
    style='margin: 20px 0;'  # 自定义 CSS 样式
)

ui.run()
```

### 3. 禁用与隐藏

通过 `enabled`、`visible` 属性控制组件状态，支持动态绑定：

```python
from nicegui import ui

rating = ui.rating(value=3)

# 禁用组件（不可点击修改）
ui.button('禁用评分', on_click=lambda: rating.set_enabled(False))
# 启用组件
ui.button('启用评分', on_click=lambda: rating.set_enabled(True))
# 隐藏组件
ui.button('隐藏评分', on_click=lambda: rating.set_visibility(False))
# 显示组件
ui.button('显示评分', on_click=lambda: rating.set_visibility(True))

ui.run()
```

## 五、API 详情补充

### 1. 核心属性

| 属性名    | 类型               | 说明                                      |
| --------- | ------------------ | ----------------------------------------- |
| `classes` | `str`              | 组件的 HTML 类名，支持 Tailwind/Quasar 类 |
| `client`  | `Client`           | 组件所属的客户端实例                      |
| `enabled` | `BindableProperty` | 是否启用组件（可绑定）                    |
| `html_id` | `str`              | 组件的 HTML DOM ID（版本 2.16.0+）        |
| `value`   | `BindableProperty` | 当前评分值（可绑定，支持动态修改）        |
| `visible` | `BindableProperty` | 是否显示组件（可绑定）                    |

### 2. 常用方法

| 方法名                                    | 说明                                     | 参数                                                 |
| ----------------------------------------- | ---------------------------------------- | ---------------------------------------------------- |
| `bind_value(target, target_name='value')` | 双向绑定目标对象的属性（如输入框、滑块） | `target`：目标组件；`target_name`：目标属性名        |
| `bind_value_from(target)`                 | 单向绑定（目标对象 → 评分组件）          | 同 `bind_value`                                      |
| `bind_value_to(target)`                   | 单向绑定（评分组件 → 目标对象）          | 同 `bind_value`                                      |
| `set_value(value)`                        | 手动设置评分值                           | `value`：新评分值（整数 / 半整数）                   |
| `set_enabled(enabled)`                    | 启用 / 禁用组件                          | `enabled`：布尔值（True 启用，False 禁用）           |
| `set_visibility(visible)`                 | 显示 / 隐藏组件                          | `visible`：布尔值（True 显示，False 隐藏）           |
| `on_value_change(callback)`               | 绑定值变化回调（简化版 `on_change`）     | `callback`：无参或接收 `ValueChangeEvent` 参数的函数 |
| `tooltip(text)`                           | 为组件添加悬浮提示                       | `text`：提示文本                                     |

### 3. 事件说明

- **值变化事件**：通过 `on_change` 或 `on_value_change` 监听，事件对象包含 `value`（当前评分）、`sender`（组件实例）等属性。
- **通用事件**：支持通过 `on('click')`、`on('mousedown')` 等监听鼠标事件，可结合 `js_handler` 实现客户端侧逻辑。

## 六、注意事项

1. **图标兼容性**：图标名称需符合 NiceGUI/Quasar 的图标库规范（如 `star`、`sentiment_neutral` 等），不支持自定义图片图标。
2. **颜色列表长度**：当 `color` 为列表时，列表长度需与 `max` 参数一致，否则多余星级将沿用最后一个颜色。
3. **半星支持**：半星评分需指定 `icon_half`，否则半选中状态将使用 `icon` 图标，可能导致显示异常。
4. **版本兼容性**：`html_id` 属性需 NiceGUI 2.16.0+，`bind_*` 方法的 `strict` 参数需 3.0.0+。
5. **单位格式**：`size` 参数若使用自定义 CSS 单位，需包含单位名称（如 `16px` 而非 `16`）。

## 七、应用场景

1. 产品评价表单：收集用户对商品、服务的星级评分。
2. 反馈调研：让用户对内容、功能进行满意度评分（如情绪图标评分）。
3. 动态配置：结合滑块、输入框实现评分规则的实时调整（如最大星级、评分范围）。
4. 数据可视化辅助：通过颜色渐变直观展示评分等级（如低评分红色、高评分绿色）。

通过以上配置，`ui.rating` 可灵活适配各类评分场景，兼顾易用性和自定义需求，是 NiceGUI 中高效的交互组件之一。