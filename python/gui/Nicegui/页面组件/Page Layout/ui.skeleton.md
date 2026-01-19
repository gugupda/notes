# ui.skeleton 全面详解

## 一、核心概述

`ui.skeleton` 是 NiceGUI 框架中的加载占位组件，基于 Quasar 的 `QSkeleton` 组件实现，核心作用是在内容加载完成前，显示结构化的占位布局，提升用户体验（避免页面空白或布局抖动）。其本质是模拟目标内容的形状、尺寸和结构，通过预设样式和动画效果，让用户直观感知 “内容正在加载中”，广泛适用于卡片、菜单、列表、表单等各类需要异步加载数据的场景。

与传统的 “加载中” 文字或图标相比，`ui.skeleton` 更贴近最终内容的布局结构，视觉过渡更自然，且支持高度定制化（形状、尺寸、动画、样式等），是现代界面开发中提升加载体验的关键组件。

## 二、基础用法

### 1. 最简示例（默认矩形占位）

直接调用 `ui.skeleton()` 即可创建默认样式的占位组件，默认类型为矩形（`rect`），带波浪动画（`wave`）：

```python
from nicegui import ui

# 默认矩形占位，设置全宽适配父容器
ui.skeleton().classes('w-full')

ui.run()
```

运行后效果：显示一个占满父容器宽度的矩形占位，带有从左到右的波浪动画，暗示内容正在加载。

### 2. 核心特性

- 零配置上手：默认样式适用于大多数基础场景，无需额外参数；
- 结构化模拟：支持模拟文本、工具栏、头像等多种内容类型（通过 `type` 参数配置）；
- 动画增强：内置多种加载动画，提升视觉反馈；
- 样式灵活：支持自定义尺寸、边框、圆角、颜色等样式；
- 布局兼容：可无缝嵌入 `ui.card`、`ui.column`、`ui.row` 等各类容器组件。

## 三、核心参数（Properties）

`ui.skeleton` 提供了丰富的参数用于定制占位效果，覆盖类型、尺寸、动画、样式等核心维度，参数详情如下：

| 参数名            | 类型    | 默认值   | 说明                                                         |
| ----------------- | ------- | -------- | ------------------------------------------------------------ |
| `type`            | `str`   | `"rect"` | 占位组件的类型，对应模拟的内容形态，支持 Quasar 所有原生类型（如 `"rect"` 矩形、`"text"` 文本、`"QToolbar"` 工具栏、`"QCard"` 卡片等，完整类型列表可参考 Quasar 文档） |
| `tag`             | `str`   | `"div"`  | 渲染时使用的 HTML 标签（如 `"div"`、`"span"`、`"section"` 等），用于适配特定 DOM 结构需求 |
| `animation`       | `str`   | `"wave"` | 加载动画效果，可选值：`"pulse"`（脉冲）、`"wave"`（波浪）、`"pulse-x"`（水平脉冲）、`"pulse-y"`（垂直脉冲）、`"fade"`（渐变）、`"blink"`（闪烁）、`"none"`（无动画） |
| `animation_speed` | `float` | `1.5`    | 动画速度（单位：秒），数值越小动画越快                       |
| `square`          | `bool`  | `False`  | 是否取消圆角，设置为 `True` 时显示直角边框（默认带圆角）     |
| `bordered`        | `bool`  | `False`  | 是否显示默认边框，设置为 `True` 时添加浅灰色边框             |
| `size`            | `str`   | `None`   | 统一设置宽高（CSS 单位，如 `"100px"`、`"2rem"`），优先级高于 `width` 和 `height`（若设置则覆盖后两者） |
| `width`           | `str`   | `None`   | 宽度（CSS 单位，如 `"200px"`、`"50%"`），未设置时自适应父容器 |
| `height`          | `str`   | `None`   | 高度（CSS 单位，如 `"80px"`、`"3rem"`），未设置时使用默认高度 |

### 常用参数示例

#### （1）定制文本类型占位

模拟文本加载效果，使用 `type="text"` 并配合样式类调整尺寸：

```python
from nicegui import ui

# 模拟大标题文本占位
ui.skeleton(type='text').classes('text-2xl w-3/4 mb-2')
# 模拟正文文本占位（较短）
ui.skeleton(type='text').classes('text-base w-1/2 mb-2')
# 模拟小文本占位
ui.skeleton(type='text').classes('text-sm w-1/3')

ui.run()
```

效果：显示三条不同长度、不同字号的文本占位，模拟标题 + 正文 + 辅助文本的结构。

#### （2）修改动画与样式

设置垂直脉冲动画、直角边框、显示边框，并自定义尺寸：

```python
from nicegui import ui

# 垂直脉冲动画、直角、带边框、自定义宽高
ui.skeleton(
    type='rect',
    animation='pulse-y',
    square=True,
    bordered=True,
    width='300px',
    height='150px'
).classes('mx-auto')  # 水平居中

ui.run()
```

#### （3）使用 `size` 参数统一设置宽高

适用于正方形占位（如头像、图标）：

```python
from nicegui import ui

# 50px × 50px 的正方形占位（模拟头像）
ui.skeleton(type='rect', size='50px', square=True).classes('rounded-full')  # 圆形头像占位

ui.run()
```

## 四、样式定制（Classes & Style）

`ui.skeleton` 支持通过 NiceGUI 的 `classes` 和 `style` 属性进一步定制样式，兼容 Tailwind CSS、Quasar 类和自定义 CSS，常见场景如下：

### 1. 自定义背景色与边框色

```python
from nicegui import ui

# 浅灰色背景、红色边框、圆角
ui.skeleton(
    bordered=True,
    width='200px',
    height='100px'
).classes('bg-gray-100 border-red-300 rounded-xl')

ui.run()
```

### 2. 适配布局容器

嵌入卡片组件，模拟卡片内容加载：

```python
from nicegui import ui

with ui.card().classes('w-64 mx-auto'):
    # 卡片头部图片占位
    ui.skeleton(type='rect', height='120px', square=True).classes('w-full')
    with ui.card_section():
        # 卡片标题占位
        ui.skeleton(type='text').classes('text-lg font-semibold mb-2')
        # 卡片描述文本占位（两行）
        ui.skeleton(type='text').classes('w-full mb-1')
        ui.skeleton(type='text').classes('w-4/5')
    with ui.card_actions():
        # 按钮占位
        ui.skeleton(type='rect', width='80px', height='30px').classes('rounded-md')

ui.run()
```

效果：模拟带图片、标题、描述、按钮的完整卡片加载状态。

## 五、高级应用场景

### 1. 模拟 YouTube 视频卡片占位

官方示例：模拟 YouTube 视频卡片的加载状态（缩略图 + 标题 + 作者 + 观看量）：

```python
from nicegui import ui

with ui.card().tight().classes('w-full max-w-md mx-auto'):
    # 视频缩略图占位（16:9 比例）
    ui.skeleton(square=True, animation='fade', height='150px', width='100%')
    with ui.card_section().classes('w-full p-3'):
        # 视频标题占位（大文本）
        ui.skeleton('text').classes('text-subtitle1 font-medium mb-2')
        # 作者名称占位（中等长度文本）
        ui.skeleton('text').classes('text-subtitle1 w-1/2 mb-1')
        # 观看量+时间占位（小文本）
        ui.skeleton('text').classes('text-caption text-gray-500')

ui.run()
```

效果：高度还原 YouTube 视频卡片的结构，加载动画为渐变效果，视觉更柔和。

### 2. 表单加载占位

模拟表单输入框、按钮的加载状态：

```python
from nicegui import ui

with ui.column().classes('w-80 mx-auto gap-4'):
    ui.label('登录表单（加载中）').classes('text-xl font-bold text-center')
    # 用户名输入框占位
    ui.skeleton(type='rect', height='40px', bordered=True).classes('w-full rounded-md')
    # 密码输入框占位
    ui.skeleton(type='rect', height='40px', bordered=True).classes('w-full rounded-md')
    # 登录按钮占位
    ui.skeleton(type='rect', height='40px', square=True).classes('w-full bg-blue-100 rounded-md')

ui.run()
```

### 3. 列表批量占位

模拟多列表项的加载状态（如商品列表、消息列表）：

```python
from nicegui import ui

with ui.column().classes('w-96 mx-auto gap-3'):
    ui.label('商品列表（加载中）').classes('text-lg font-bold')
    # 3 个列表项占位
    for _ in range(3):
        with ui.row().classes('items-center gap-3 p-3 border rounded-lg'):
            # 商品图片占位（正方形）
            ui.skeleton(type='rect', size='60px', square=True).classes('rounded-md')
            with ui.column().classes('flex-1 gap-1'):
                # 商品名称占位
                ui.skeleton(type='text').classes('font-medium')
                # 商品价格占位
                ui.skeleton(type='text').classes('text-green-600 w-1/4')

ui.run()
```

效果：显示 3 个商品列表项占位，每个项包含图片、名称、价格结构，模拟批量数据加载。

## 六、注意事项

1. 类型兼容性：`type` 参数支持的具体值需参考 Quasar 的 `QSkeleton` 文档（如 `"QToolbar"`、`"QCard"` 等需匹配 Quasar 组件类型），自定义类型可能导致显示异常；
2. 尺寸单位规范：`size`、`width`、`height` 需传入合法的 CSS 单位（如 `px`、`rem`、`%`），未指定单位可能导致布局错乱；
3. 动画性能：若页面存在大量 `ui.skeleton` 组件，建议避免使用过于复杂的动画（如 `wave`），可选择 `pulse` 或 `none` 以优化性能；
4. 样式优先级：`bordered` 参数启用的默认边框样式可通过 `classes` 覆盖（如 `border-red-500` 替换默认边框色）；
5. 与内容的一致性：占位组件的尺寸、结构应尽量与最终加载的内容一致，避免加载完成后布局大幅变动（如文本占位长度匹配实际文本、图片占位比例匹配实际图片）。

## 总结

`ui.skeleton` 是 NiceGUI 中功能强大的加载占位组件，核心价值在于通过结构化占位和动画效果，提升异步加载场景的用户体验。其优势在于：

- 高度定制化：支持类型、尺寸、动画、样式的全方位定制，适配各类内容形态；
- 布局兼容性强：可无缝嵌入各类容器组件，无需额外调整布局逻辑；
- 易用性高：零配置即可上手，复杂场景通过参数和样式类快速实现。

适用于几乎所有需要异步加载数据的界面（如列表、卡片、表单、详情页等），是现代前端开发中不可或缺的 “体验优化工具”。通过合理设计占位结构，可有效降低用户的等待焦虑，让加载过程更自然、更直观。