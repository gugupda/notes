# element.props

### 一、`element.props` 核心概念

在 NiceGUI 中，`element.props()` 是用于配置**基于 Quasar 框架**的 UI 元素属性的核心方法。NiceGUI 底层大量复用 Quasar 组件（如 `ui.button` 对应 Quasar 的 `QBtn` 组件），`props()` 本质是将 Quasar 组件的原生属性（props）传递给底层元素，从而实现对元素行为、外观、交互的基础配置。

### 二、`element.props` 语法与使用规则

#### 1. 基本语法

```python
from nicegui import ui

# 单个属性：直接传入 "属性名=值" 字符串
btn = ui.button("普通按钮")
btn.props("color=primary")

# 多个属性：空格分隔 "属性1=值1 属性2=值2"
btn.props("color=red size=lg rounded=true")

# 布尔属性（无需赋值）：直接写属性名
btn.props("disabled loading")

# 覆盖/重置：多次调用会覆盖之前的同属性，或传入空字符串清空所有props
btn.props("")  # 清空所有已设置的props
```

#### 2. 取值规则

- **字符串值**：直接写（如 `color=blue`）；含特殊字符需用引号包裹（如 `label='Hello World'`）。
- **布尔值**：`true`/`false` 或省略值（仅写属性名等价于 `true`，如 `disabled` = `disabled=true`）。
- **数值 / 对象**：Quasar 部分属性支持数值 / 对象，需按 Quasar 文档格式传入（如 `ripple={early: true}`）。

### 三、核心用途（以 `ui.button` 为例）

#### 1. 外观基础配置

```python
btn = ui.button("操作按钮")
# 颜色（Quasar 预设色：primary/secondary/red/green/amber 等）
btn.props("color=green")
# 尺寸（xs/sm/md/lg/xl）
btn.props("size=lg")
# 形状（圆角/圆形/无圆角）
btn.props("rounded=true")  # 圆角
btn.props("circle=true")   # 圆形（需配合图标使用）
# 边框
btn.props("bordered=true")
```

#### 2. 状态控制

```python
btn = ui.button("状态按钮")
# 禁用
btn.props("disabled=true")
# 加载中（显示加载动画）
btn.props("loading=true")
# 激活态
btn.props("active=true")
```

#### 3. 交互行为

```python
btn = ui.button("交互按钮")
# 点击后自动禁用（防止重复点击）
btn.props("disable-after-click=true")
# 波纹效果（开启/关闭）
btn.props("ripple=false")  # 关闭点击波纹
# 链接模式（按钮表现为超链接）
btn.props("to=/home")  # 跳转至 NiceGUI 路由 /home
btn.props("href=https://nicegui.io target=_blank")  # 外部链接
```

#### 4. 图标与文本组合

```python
btn = ui.button("带图标按钮")
# 左侧图标（Quasar 内置图标，或自定义图标）
btn.props("icon=menu")
# 右侧图标
btn.props("icon-right=arrow_forward")
# 仅图标（无文本）
btn = ui.button("")
btn.props("icon=settings circle=true")  # 圆形图标按钮
```

### 四、与 Tailwind / 原生 CSS 的优先级关系

1. **Quasar props 基础样式**：NiceGUI 已将 Quasar 标记为 `!important` 的样式降级到低优先级 CSS 层，因此：
   - Tailwind 类（如 `!bg-red-500`）可覆盖 `props` 设置的颜色（如 `color=primary`）；
   - 原生 CSS `style()` 中带 `!important` 的样式（如 `background-color: red !important`）也可覆盖 `props` 样式。
2. **非 `!important` 样式**：`props` 设置的基础布局（如尺寸、形状）优先级高于无 `!important` 的 Tailwind / 原生 CSS。

示例：

```python
btn = ui.button("优先级测试")
# props 设置颜色（Quasar 预设）
btn.props("color=primary")
# Tailwind 覆盖颜色（! 标记提升优先级）
btn.classes("!bg-red-500")
# 原生 CSS 覆盖（最终生效）
btn.style("background-color: blue !important; color: white !important;")
```

### 五、关键注意事项

1. **属性兼容性**：`props` 的取值完全依赖底层 Quasar 组件，需对照 [Quasar 组件文档](https://quasar.dev/components/button)（如 `QBtn` 文档）确认属性是否支持；

2. **多属性拼接**：多个属性必须用**空格分隔**，而非逗号或分号；

3. **动态修改**：可通过变量 / 响应式数据动态修改 props，例如：

   ```python
   from nicegui import ui, reactive
   
   state = reactive({"loading": False})
   btn = ui.button("动态按钮")
   btn.props(f"loading={state.loading}")
   
   def toggle_loading():
       state.loading = not state.loading
       btn.props(f"loading={state.loading}")
   
   ui.button("切换加载态", on_click=toggle_loading)
   ```

4. **与 NiceGUI 自身方法的配合**：`props()` 仅配置 Quasar 原生属性，事件绑定（如 `on_click`）、布局（如 `classes()` 中的 flex 类）仍需用 NiceGUI 自身方法。

# NiceGUI element 的 props 方法详解

NiceGUI 是一款基于 Python 的轻量级 Web 框架，核心是通过封装 Vue 3 + Quasar 组件实现前端交互，`props` 方法是 NiceGUI 中**配置底层 Quasar 组件属性**的核心方式，用于自定义元素的外观、行为和功能。

#### 一、props 方法的核心作用

`props` 方法的本质是将 Python 层面的配置传递给底层 Quasar 组件的 Vue 属性（Props），从而覆盖组件的默认行为。几乎所有 NiceGUI 元素（如 `ui.button`、`ui.input`、`ui.select` 等）都继承了 `props` 方法，是自定义组件的核心入口。

#### 二、基本语法

```python
element.props(prop_string: str | None = None, **kwargs) -> Self
```

- **参数 1（prop_string）**：字符串形式的属性配置，支持 Quasar 组件的原生 Props，多个属性用空格分隔，属性值用 `=` 连接（值为布尔型时可省略 `=true`）。
- **参数 2（kwargs）**：关键字参数形式的属性配置，与字符串形式等价，更符合 Python 语法习惯。
- **返回值**：返回元素自身（链式调用友好）。

#### 三、使用方式

##### 方式 1：字符串形式（贴近 Quasar 原生）

适合快速配置多个简单属性，直接复用 Quasar 文档中的属性写法。

```python
from nicegui import ui

# 示例1：配置按钮的 Quasar 属性（圆形、无边框、图标）
ui.button('点击').props('round borderless icon=home')

# 示例2：配置输入框为禁用、占位符、清除按钮
ui.input().props('disabled placeholder="请输入内容" clearable')

# 示例3：布尔型属性（省略 =true）
ui.checkbox('同意协议').props('checked')  # 等价于 checked=true
```

##### 方式 2：关键字参数形式（Python 风格）

适合复杂属性或动态配置，支持变量代入，可读性更强。

```python
from nicegui import ui

# 示例1：关键字参数配置按钮
btn = ui.button('提交')
btn.props(round=True, borderless=True, icon='send')

# 示例2：动态配置输入框属性
placeholder_text = '请输入手机号'
ui.input().props(clearable=True, placeholder=placeholder_text, maxlength=11)

# 示例3：混合字符串和关键字参数（后者覆盖前者重复属性）
ui.select(['A', 'B', 'C']).props('label=选项 disabled', label='自定义标签')  # label 最终为「自定义标签」
```

#### 四、关键注意事项

1. **属性来源**：`props` 支持的属性完全对应底层 Quasar 组件的 Props，需参考 [Quasar 组件文档](https://quasar.dev/components)（如按钮对应 `QBtn`，输入框对应 `QInput`）。

2. **命名转换**：Quasar 中驼峰命名的属性（如 `hideBottomSpace`），在 Python 中可写为蛇形（`hide_bottom_space`）或保持驼峰。

3. **优先级**：`props` 配置会覆盖 NiceGUI 元素自身的参数（如 `ui.button(icon='home')` 等价于 `ui.button().props('icon=home')`）。

4. **重复调用**：多次调用 `props` 会合并属性，后调用的重复属性会覆盖前者。

   ```python
   btn = ui.button('测试')
   btn.props('icon=home round')
   btn.props('icon=settings')  # 最终 icon 为 settings，round 保留
   ```

5. **特殊值处理**：

   - 布尔值：`True` 可省略值（如 `props('disabled')`），`False` 需显式写 `disabled=false`。
   - 数值 / 字符串：直接传递（如 `props('maxlength=10')`、`props('label="用户名"')`）。

#### 五、实战示例

```python
from nicegui import ui

# 1. 配置下拉选择框（Quasar QSelect 组件）
ui.select(
    options=['Python', 'JavaScript', 'Java'],
    label='编程语言'
).props(
    'bordered clearable multiple dense',  # 多行属性
    emit_value=True,  # 关键字参数
    hide_selected=True  # 隐藏已选选项
)

# 2. 配置卡片（Quasar QCard 组件）
ui.card().props('bordered shadow="2" radius="10"').content(
    lambda: ui.label('自定义卡片样式')
)

# 3. 动态修改 props
btn = ui.button('动态修改')
btn.props('color=primary')
ui.button('切换颜色').on('click', lambda: btn.props('color=secondary'))

ui.run()
```

# NiceGUI 常用元素对应 Quasar 核心 props 速查表（优化版）

说明：按元素类型拆分表格，聚焦核心常用 props，兼顾实用性与易读性。属性支持「字符串形式」（如 `props('round')`）和「关键字参数形式」（如 `props(round=True)`）。

## 一、按钮（对应 Quasar 组件：QBtn）

| 属性名     | 作用                               | 使用示例                                | 备注                                       |
| :--------- | :--------------------------------- | :-------------------------------------- | :----------------------------------------- |
| round      | 设置为圆形按钮                     | ui.button('圆形').props('round')        | 布尔型，可省略 =true                       |
| borderless | 移除按钮边框                       | ui.button('无边框').props('borderless') | -                                          |
| icon       | 设置内置图标                       | ui.button().props('icon=home')          | 等价于 ui.button(icon='home')              |
| color      | 设置主色（预设色：primary/red 等） | ui.button().props('color=primary')      | 支持动态修改：btn.props('color=secondary') |
| disabled   | 禁用按钮                           | ui.button().props('disabled')           | 取消禁用：props('disabled=false')          |
| dense      | 紧凑模式（减小内边距）             | ui.button('紧凑').props('dense')        | -                                          |

## 二、输入框（对应 Quasar 组件：QInput）

| 属性名      | 作用                             | 使用示例                                 | 备注                                  |
| :---------- | :------------------------------- | :--------------------------------------- | :------------------------------------ |
| clearable   | 显示清空内容按钮                 | ui.input().props('clearable')            | -                                     |
| disabled    | 禁用输入框                       | ui.input().props('disabled')             | -                                     |
| placeholder | 设置占位提示文本                 | ui.input().props('placeholder="请输入"') | 等价于 ui.input(placeholder="请输入") |
| maxlength   | 限制最大输入字符数               | ui.input().props('maxlength=11')         | 数值型直接写值                        |
| bordered    | 显示边框（替代默认下划线）       | ui.input().props('bordered')             | -                                     |
| type        | 设置输入类型（text/password 等） | ui.input().props('type=password')        | 等价于 ui.input(type='password')      |

## 三、选择框（对应 Quasar 组件：QSelect）

| 属性名        | 作用                       | 使用示例                                    | 备注                 |
| :------------ | :------------------------- | :------------------------------------------ | :------------------- |
| clearable     | 显示清空已选内容按钮       | ui.select(['A','B']).props('clearable')     | -                    |
| multiple      | 开启多选模式               | ui.select(['A','B']).props('multiple')      | 多选时返回列表类型值 |
| bordered      | 显示边框样式               | ui.select(['A','B']).props('bordered')      | -                    |
| emit_value    | 直接返回选项值（简化取值） | ui.select(['A','B']).props('emit_value')    | NiceGUI 中建议开启   |
| hide_selected | 多选时隐藏已选选项         | ui.select(['A','B']).props('hide_selected') | -                    |
| dense         | 紧凑模式（减小组件高度）   | ui.select(['A','B']).props('dense')         | -                    |

## 四、卡片（对应 Quasar 组件：QCard）

| 属性名            | 作用                         | 使用示例                             | 备注                             |
| :---------------- | :--------------------------- | :----------------------------------- | :------------------------------- |
| bordered          | 显示卡片边框                 | ui.card().props('bordered')          | -                                |
| shadow            | 设置阴影强度（0-24）         | ui.card().props('shadow=2')          | 数值越大阴影越明显               |
| radius            | 设置圆角半径                 | ui.card().props('radius=10')         | 数值越大圆角越明显               |
| flat              | 仅保留边框（移除背景和阴影） | ui.card().props('flat')              | -                                |
| hide_bottom_space | 移除底部默认内边距           | ui.card().props('hide_bottom_space') | 驼峰名也可写为 hide_bottom_space |

补充：更多属性可参考 Quasar 官方文档
- 按钮：https://quasar.dev/components/button
- 输入框：https://quasar.dev/components/input
- 选择框：https://quasar.dev/components/select
- 卡片：https://quasar.dev/components/card