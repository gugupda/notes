# ui.page_sticky 全面详细解析

`ui.page_sticky` 是 NiceGUI 框架中用于创建**固定定位元素**的核心布局组件，基于 CSS `position: sticky` 特性与 Quasar 底层封装，支持元素在页面滚动时 “粘性吸附” 到指定位置（如顶部、底部、侧边），且不脱离文档流，仅在父容器可视范围内生效。该组件继承自 `Element`、`Visibility` 等基类，具备样式定制、状态控制、事件绑定等通用能力，适用于导航栏、悬浮按钮、侧边工具栏等场景，是实现复杂滚动交互的关键组件。以下从核心特性、初始化参数、属性与方法、使用示例、注意事项等方面进行全面解析。

## 一、核心特性

1. **粘性定位机制**：结合 `position: sticky` 特性，元素在滚动过程中会 “吸附” 到 `top`/`bottom`/`left`/`right` 指定的位置，直到父容器滚动出可视范围后才跟随滚动。
2. **方位灵活配置**：支持通过参数指定吸附方位（顶部、底部、左侧、右侧），可组合使用（如 `top=20, right=10` 实现右上角悬浮）。
3. **视觉层级可控**：支持通过 `z_index` 参数设置层级优先级，避免被其他固定定位元素遮挡。
4. **样式高度定制**：可通过 `style`、`classes` 调整背景色、内边距、阴影、圆角等样式，适配各类设计需求。
5. **状态与事件支持**：继承基类的显示 / 隐藏、删除、事件绑定等能力，可动态控制组件状态，响应点击、滚动等事件。
6. **父容器约束**：粘性效果仅在父容器可视范围内生效（父容器需有滚动空间或高度限制），超出父容器后元素将跟随父容器滚动。

## 二、初始化参数（Initializer）

`ui.page_sticky` 的初始化参数聚焦定位逻辑与基础样式，所有参数均为可选，默认值适配通用场景，详细说明如下：

| 参数名    | 类型            | 说明                                                         | 默认值 |
| --------- | --------------- | ------------------------------------------------------------ | ------ |
| top       | int \| None     | 元素吸附到顶部的距离（单位：像素），`None` 表示不吸附顶部    | None   |
| bottom    | int \| None     | 元素吸附到底部的距离（单位：像素），`None` 表示不吸附底部    | None   |
| left      | int \| None     | 元素吸附到左侧的距离（单位：像素），`None` 表示不吸附左侧    | None   |
| right     | int \| None     | 元素吸附到右侧的距离（单位：像素），`None` 表示不吸附右侧    | None   |
| z_index   | int             | 元素的 CSS 层级优先级（`z-index`），数值越大越靠上层，避免被其他元素遮挡 | 1000   |
| container | Element \| None | 粘性定位的父容器，元素仅在该容器可视范围内生效：- `None`：默认以 `<body>` 为父容器（全局粘性）- 传入其他组件实例（如 `ui.column()`）：仅在该组件内生效 | None   |

## 三、核心属性（Properties）

继承自 `Element`、`Visibility` 等基类，支持动态获取和修改组件状态与配置，核心属性如下：

| 属性名             | 类型             | 说明                                                         |
| ------------------ | ---------------- | ------------------------------------------------------------ |
| classes            | Classes[Self]    | 组件的 HTML 类名，支持通过 Tailwind/Quasar 类调整布局（如 `p-3 rounded-full`）和样式（如 `shadow-lg`），支持链式调用修改 |
| client             | Client           | 组件所属的客户端实例，关联 websocket 连接、布局上下文等      |
| html_id            | str              | 组件在 HTML DOM 中的唯一 ID（2.16.0 版本新增），可用于直接通过 DOM 操作组件 |
| is_deleted         | bool             | 组件是否已被删除（只读属性），用于判断组件生命周期状态       |
| is_ignoring_events | bool             | 组件是否正在忽略事件（只读属性），用于事件控制逻辑           |
| parent             | Element \| None  | 组件的父元素（即 `container` 参数指定的容器），可动态修改    |
| props              | Props[Self]      | 组件的 Quasar 原生属性，支持链式调用添加 / 删除（如 `props('flat')`），扩展组件功能 |
| style              | Style[Self]      | 组件的内联 CSS 样式，支持链式调用（如 `style('background-color: #fff').style('padding: 10px')`），精准控制外观 |
| visible            | BindableProperty | 组件的可见性（`True` 为显示，`False` 为隐藏），可绑定数据动态控制 |
| z_index            | int              | 组件的层级优先级，可动态修改（如 `sticky.z_index = 2000`）   |

## 四、关键方法（Methods）

`ui.page_sticky` 继承了基类的丰富方法，支持组件操作、样式修改、事件绑定等，按功能分类解析核心方法：

### 1. 基础状态操作

| 方法名   | 参数 | 说明                                               |
| -------- | ---- | -------------------------------------------------- |
| show()   | -    | 显示粘性组件                                       |
| hide()   | -    | 隐藏粘性组件                                       |
| toggle() | -    | 切换组件的显示 / 隐藏状态                          |
| delete() | -    | 删除组件及所有子元素，释放资源                     |
| clear()  | -    | 清除组件内的所有子元素（保留组件本身）             |
| update() | -    | 同步组件状态到客户端，修改样式、属性后需调用以生效 |

### 2. 样式与属性修改

| 方法名         | 核心参数                                     | 说明                                                         |
| -------------- | -------------------------------------------- | ------------------------------------------------------------ |
| style()        | css_str                                      | 设置内联样式，支持多轮链式调用（如 `style('position: sticky').style('top: 30px')`），优先级高于 `classes` |
| classes()      | class_str                                    | 设置 HTML 类名，可结合 Tailwind 实现快速样式（如 `classes('bg-blue-500 text-white p-2 rounded')`） |
| props()        | props_str                                    | 设置 Quasar 原生属性，扩展组件功能（如 `props('shadow-md')` 添加中等阴影） |
| set_position() | top=None, bottom=None, left=None, right=None | 动态修改吸附位置（如 `sticky.set_position(top=50, right=20)`），无需重新实例化 |
| set_z_index()  | z_index: int                                 | 动态修改层级优先级（如 `sticky.set_z_index(1500)`），解决遮挡问题 |

### 3. 数据绑定方法

支持将组件状态（如 `visible`）与 Python 对象属性动态绑定，实现单向 / 双向同步：

| 方法名                 | 核心参数                   | 说明                                                         |
| ---------------------- | -------------------------- | ------------------------------------------------------------ |
| bind_visibility()      | target_object, target_name | 双向绑定：组件可见性与目标对象属性同步（一方修改，另一方自动更新） |
| bind_visibility_from() | target_object, target_name | 单向绑定（从目标到组件）：目标对象属性变化时，同步更新组件可见性 |
| bind_visibility_to()   | target_object, target_name | 单向绑定（从组件到目标）：组件可见性变化时，同步更新目标对象属性 |

### 4. 事件绑定方法

| 方法名     | 核心参数                  | 说明                                                         |
| ---------- | ------------------------- | ------------------------------------------------------------ |
| on()       | type, handler, js_handler | 绑定组件事件（如 `click`、`mouseenter`、`scroll` 等），支持 Python 处理器（服务端处理）或 JavaScript 处理器（客户端处理），2.18.0+ 版本可同时指定两者 |
| on_click() | callback                  | 快捷绑定点击事件（如按钮点击、悬浮卡片点击）                 |

### 5. 其他实用方法

| 方法名         | 核心参数                       | 说明                                                         |
| -------------- | ------------------------------ | ------------------------------------------------------------ |
| tooltip()      | text                           | 为组件添加悬浮提示文本（鼠标悬浮时显示）                     |
| mark()         | *markers                       | 为组件添加标记（如 `mark('sticky-button back-to-top')`），用于测试查询或依赖管理 |
| move()         | target_container, target_index | 将组件移动到其他父容器中，修改粘性生效范围                   |
| add_resource() | path                           | 为组件添加静态资源（如 CSS/JS 文件），扩展样式或功能         |

## 五、使用示例

### 1. 基础用法：全局顶部粘性导航

```python
from nicegui import ui

@ui.page('/basic_sticky')
def basic_sticky_demo():
    # 创建全局顶部粘性导航（距离顶部 0px，层级 1000）
    with ui.page_sticky(top=0, z_index=1000).style('background-color: #3874c8; color: white; width: 100%'):
        with ui.row().classes('justify-between items-center p-3'):
            ui.label('全局粘性导航').classes('font-bold text-lg')
            ui.button('首页', on_click=lambda: ui.notify('点击首页')).props('flat color=white')
            ui.button('关于', on_click=lambda: ui.notify('点击关于')).props('flat color=white')
    
    # 页面主体内容（长文本用于测试滚动）
    with ui.column().classes('p-6'):
        ui.label('页面主体内容').classes('text-2xl font-bold mb-4')
        [ui.label(f'内容行 {i}：滚动页面查看粘性导航效果') for i in range(100)]

ui.run()
```

### 2. 高级用法：局部粘性侧边栏（父容器约束）

```python
from nicegui import ui

@ui.page('/local_sticky')
def local_sticky_demo():
    # 创建父容器（带滚动条，高度限制 500px）
    with ui.column().style('height: 500px; overflow-y: auto; border: 1px solid #eee; p-4') as parent_container:
        ui.label('局部粘性区域（仅在该容器内生效）').classes('text-xl font-bold mb-4')
        
        # 创建局部粘性侧边栏（吸附到左侧 0px，父容器为 parent_container）
        with ui.page_sticky(left=0, container=parent_container).style('width: 200px; background-color: #f8f9fa; border-right: 1px solid #eee; height: 100%'):
            ui.label('局部粘性侧边栏').classes('font-bold p-3 border-b')
            ui.link('菜单1', '/menu1').classes('block p-3 hover:bg-gray-100')
            ui.link('菜单2', '/menu2').classes('block p-3 hover:bg-gray-100')
            ui.link('菜单3', '/menu3').classes('block p-3 hover:bg-gray-100')
        
        # 父容器内的主体内容（长文本）
        with ui.column().classes('ml-24'):  # 避开侧边栏
            [ui.label(f'局部内容行 {i}：仅在父容器滚动时粘性生效') for i in range(50)]
    
    # 父容器外的内容（不影响粘性组件）
    ui.label('父容器外的内容：滚动时不会触发局部粘性').classes('mt-4 p-4')

ui.run()
```

### 3. 实用场景：右下角悬浮回到顶部按钮

```python
from nicegui import ui

@ui.page('/back_to_top_sticky')
def back_to_top_demo():
    # 创建右下角悬浮按钮（吸附到底部 20px、右侧 20px，圆形样式）
    with ui.page_sticky(bottom=20, right=20, z_index=1500).classes('rounded-full shadow-lg'):
        def scroll_to_top():
            # 执行 JavaScript 实现平滑滚动到顶部
            ui.run_javascript('window.scrollTo({top: 0, behavior: "smooth"})')
        
        ui.button(icon='arrow_upward', on_click=scroll_to_top).props('flat color=white bg-blue-500 hover:bg-blue-600')
    
    # 页面主体内容（超长文本）
    with ui.column().classes('p-6'):
        ui.label('回到顶部按钮演示').classes('text-2xl font-bold mb-4')
        [ui.label(f'内容行 {i}：滚动到底部后点击悬浮按钮回到顶部') for i in range(200)]

ui.run()
```

### 4. 动态控制：绑定状态切换粘性组件可见性

```python
from nicegui import ui

class AppState:
    def __init__(self):
        self.show_sticky = True  # 控制粘性组件可见性

@ui.page('/dynamic_sticky')
def dynamic_sticky_demo():
    state = AppState()
    
    # 创建粘性组件并绑定可见性
    sticky = ui.page_sticky(top=10, right=10, z_index=1000).bind_visibility(state, 'show_sticky')
    with sticky.style('background-color: #fff; border: 1px solid #ddd; p-3 rounded'):
        ui.label('动态粘性组件').classes('font-medium')
        ui.checkbox('显示粘性组件', value=state.show_sticky).bind_value(state, 'show_sticky')
    
    # 页面主体内容
    with ui.column().classes('p-6'):
        ui.label('动态控制粘性组件可见性').classes('text-2xl font-bold mb-4')
        ui.button('隐藏', on_click=lambda: sticky.hide()).props('flat')
        ui.button('显示', on_click=lambda: sticky.show()).props('flat')
        ui.button('切换', on_click=lambda: sticky.toggle()).props('flat')
        [ui.label(f'内容行 {i}') for i in range(100)]

ui.run()
```

## 六、注意事项

1. **父容器约束规则**：
   - 若 `container` 为 `None`（默认），父容器为 `<body>`，需确保 `<body>` 无 `overflow: hidden` 样式，否则粘性效果失效。
   - 若指定自定义父容器，该容器必须具备滚动能力（如 `overflow-y: auto`）或固定高度，否则粘性效果无法触发。
2. **吸附方位冲突**：
   - 不可同时设置 `top` 和 `bottom`（垂直方向冲突），或 `left` 和 `right`（水平方向冲突），否则浏览器将优先采用 `top`/`left`。
   - 建议仅设置单一方向或相邻方向（如 `top=20, right=10`），避免布局异常。
3. **层级与遮挡问题**：
   - 若粘性组件被其他固定定位元素（如 `ui.header`）遮挡，需提高 `z_index` 值（默认 1000，可调整为 2000+）。
   - 注意 `z_index` 不要过度设置（如超过 10000），避免与浏览器原生组件（如滚动条）冲突。
4. **样式与布局适配**：
   - 粘性组件默认宽度由内容决定，若需全屏宽度（如顶部导航），需手动设置 `style('width: 100%')`。
   - 小屏幕下建议结合响应式类（如 `md:hidden`）隐藏非必要粘性组件，避免布局拥挤。
5. **滚动性能优化**：
   - 避免在粘性组件中添加过于复杂的子元素（如大量图片、动画），否则可能影响页面滚动流畅度。
   - 高频事件（如 `scroll`）绑定需添加节流控制（通过 `on()` 方法的 `throttle` 参数），减少服务器交互。
6. **版本兼容性**：
   - `html_id` 属性需 NiceGUI 2.16.0+ 版本，`js_handler` 支持同时指定 Python 与 JS 处理器需 2.18.0+ 版本。
   - 自定义父容器 `container` 参数在 2.10.0+ 版本支持，低版本需升级后使用。

## 七、扩展参考

- **CSS 粘性定位原理**：`ui.page_sticky` 底层依赖 `position: sticky`，其行为介于 `relative` 和 `fixed` 之间，仅在父容器滚动到指定位置时触发固定定位，详细可参考 [MDN 粘性定位文档](https://developer.mozilla.org/zh-CN/docs/Web/CSS/position#sticky)。
- **Quasar 原生属性**：可通过 `props()` 方法传递 Quasar 组件属性（如 `shadow-xl` 加强阴影、`rounded-xl` 圆角），详细支持的属性可参考 [Quasar 通用组件属性](https://quasar.dev/components/overview#common-props)。
- **响应式适配**：结合 Tailwind 响应式类（如 `sm:hidden md:block`）实现不同屏幕尺寸下的粘性组件显示 / 隐藏，或调整吸附位置（如 `md:top=20 sm:top=10`）。

# NiceGUI 中`ui.page_sticky`的全维度解析

`ui.page_sticky`是 NiceGUI 提供的**页面粘性布局组件**，用于将指定 UI 元素固定在页面的某个位置（如顶部、底部、侧边），且元素会随页面滚动保持固定（类似 CSS 的`position: sticky/fixed`）。它是实现 “固定导航栏、悬浮操作按钮、侧边工具栏” 等常见 UI 布局的核心组件，兼具灵活性和易用性。

------

## 一、核心作用与适用场景

### 1. 核心作用

- 让目标元素脱离普通文档流，固定在页面的指定方位（上 / 下 / 左 / 右）；
- 支持自定义偏移量（如距离顶部 10px）、层级（z-index）、响应式隐藏等特性；
- 不影响其他元素的布局，且可灵活控制元素的显示 / 隐藏。

### 2. 典型适用场景

| 场景           | 示例                               |
| -------------- | ---------------------------------- |
| 固定导航栏     | 页面顶部的菜单栏，滚动时始终可见   |
| 悬浮操作按钮   | 页面右下角的 “返回顶部”“新建” 按钮 |
| 侧边固定工具栏 | 左侧 / 右侧的筛选栏、操作面板      |
| 底部固定提示栏 | 页面底部的版权信息、通知条         |

------

## 二、基本语法与使用方式

### 1. 基础语法

```python
from nicegui import ui

# 基础用法：包裹需要固定的元素，指定位置
with ui.page_sticky(position='bottom-right', offset=20):
    # 被包裹的元素会固定在页面右下角，偏移20px
    ui.button('返回顶部', on_click=lambda: ui.scroll_to(top=True))

# 完整参数示例
with ui.page_sticky(
    position: str = 'bottom-right',  # 固定位置
    offset: int | tuple = 20,        # 偏移量（像素）
    z_index: int = 1000,             # 层级（避免被其他元素遮挡）
    visible: bool = True,            # 是否可见
    breakpoints: dict | None = None  # 响应式断点（不同屏幕尺寸隐藏/显示）
):
    # 要固定的UI元素
    ui.label('固定元素')
```

### 2. 核心参数详解

| 参数名        | 类型        | 取值说明                                                     | 默认值         |
| ------------- | ----------- | ------------------------------------------------------------ | -------------- |
| `position`    | str         | 固定位置：- 上下：`top`/`bottom`- 左右：`left`/`right`- 组合：`top-left`/`top-right`/`bottom-left`/`bottom-right` | `bottom-right` |
| `offset`      | int / tuple | 偏移量：- 单个 int：所有方向偏移相同（如`20`= 上下左右均 20px）- 元组：(水平偏移，垂直偏移)（如`(10, 20)`= 水平 10px，垂直 20px） | `20`           |
| `z_index`     | int         | CSS 层级，数值越大越靠前（避免被弹窗、其他组件遮挡）         | `1000`         |
| `visible`     | bool        | 初始是否可见，可后续通过`.set_visible()`动态修改             | `True`         |
| `breakpoints` | dict        | 响应式配置：键为屏幕尺寸（`xs`/`sm`/`md`/`lg`/`xl`），值为是否可见（如`{'xs': False}`= 小屏隐藏） | `None`         |

### 3. 基础示例（固定导航栏）

```python
from nicegui import ui

# 顶部固定导航栏
with ui.page_sticky(position='top', offset=0, z_index=999):
    with ui.row().classes('w-full bg-blue-500 text-white p-2'):
        ui.label('NiceGUI 导航栏').classes('text-xl font-bold')
        ui.space()  # 自动填充空白
        ui.button('首页', on_click=lambda: ui.notify('首页'))
        ui.button('文档', on_click=lambda: ui.notify('文档'))
        ui.button('关于', on_click=lambda: ui.notify('关于'))

# 页面主体内容（用于测试滚动）
for i in range(50):
    ui.card(f'内容卡片 {i+1}').classes('w-full my-2')

ui.run()
```

### 4. 进阶示例（悬浮操作按钮 + 响应式）

```python
from nicegui import ui

# 右下角悬浮按钮组（响应式：小屏隐藏）
with ui.page_sticky(
    position='bottom-right',
    offset=(20, 20),
    z_index=1000,
    breakpoints={'xs': False}  # xs（超小屏，<600px）隐藏
):
    with ui.column():
        ui.button('🔄', on_click=lambda: ui.notify('刷新')).props('fab round')
        ui.button('⬆️', on_click=lambda: ui.scroll_to(top=True)).props('fab round').classes('mt-2')
        ui.button('❌', on_click=lambda: sticky.set_visible(False)).props('fab round').classes('mt-2')

# 保存sticky引用，用于动态控制
sticky = ui.page_sticky.current

# 页面主体
ui.label('滚动页面查看悬浮按钮').classes('text-2xl text-center my-10')
for i in range(30):
    ui.div().classes('h-20 bg-gray-100 my-2')

ui.run()
```

------

## 三、关键特性与注意事项

### 1. 布局特性

- **脱离文档流**：`ui.page_sticky`包裹的元素不会占据普通文档流空间，因此不会导致页面布局偏移；
- **层级控制**：通过`z_index`参数避免被其他组件（如`ui.dialog`、`ui.card`）遮挡，建议根据场景调整（如弹窗层级通常为 2000，可设置`z_index=1500`）；
- **偏移量精度**：元组形式的`offset`可分别控制水平 / 垂直偏移，例如`offset=(10, 30)`表示 “距离右侧 10px，距离底部 30px”。

### 2. 动态控制

- 可通过`ui.page_sticky.current`获取当前 sticky 组件的引用，或直接赋值给变量，实现动态修改：

  ```python
  # 动态修改可见性
  sticky = ui.page_sticky(position='top')
  sticky.set_visible(False)  # 隐藏
  sticky.set_visible(True)   # 显示
  
  # 动态修改位置/偏移（底层通过更新CSS实现）
  sticky.style.update({
      'bottom': '50px',
      'right': '10px'
  })
  ```

### 3. 响应式断点

NiceGUI 的断点遵循 Material Design 规范，对应屏幕尺寸如下：

| 断点 | 屏幕宽度    | 说明               |
| ---- | ----------- | ------------------ |
| xs   | <600px      | 超小屏（手机竖屏） |
| sm   | 600-960px   | 小屏（手机横屏）   |
| md   | 960-1264px  | 中屏（平板）       |
| lg   | 1264-1904px | 大屏（桌面）       |
| xl   | >1904px     | 超大屏             |

示例：大屏显示、中屏及以下隐藏

```python
with ui.page_sticky(breakpoints={'md': False, 'sm': False, 'xs': False}):
    ui.button('仅大屏可见').props('fab')
```

### 4. 与 CSS 类的配合

- 可通过`.classes()`为`ui.page_sticky`或内部元素添加自定义样式，例如：

  ```python
  with ui.page_sticky(position='top', offset=0).classes('bg-white shadow-md'):
      ui.label('带阴影的顶部导航').classes('p-3 w-full')
  ```

- 避免修改`position`相关的 CSS（如`position: fixed`），以免覆盖组件原生逻辑。

### 5. 常见陷阱

- **偏移量单位**：`offset`仅支持像素（px），不支持百分比等其他单位；
- **层级冲突**：若元素被遮挡，优先检查`z_index`（建议设置为 1000+）；
- **响应式失效**：`breakpoints`的键必须是`xs/sm/md/lg/xl`，值必须是布尔值；
- **多 sticky 重叠**：多个`ui.page_sticky`组件可通过不同`z_index`控制显示顺序。

------

## 四、实战场景示例

### 1. 左侧固定工具栏

```python
from nicegui import ui

# 左侧固定工具栏
with ui.page_sticky(position='left', offset=(20, 50), z_index=999):
    with ui.column().classes('bg-gray-100 p-3 rounded-lg shadow'):
        ui.button('🗂️', on_click=lambda: ui.notify('文件')).props('icon round')
        ui.button('🔍', on_click=lambda: ui.notify('搜索')).props('icon round').classes('mt-2')
        ui.button('⚙️', on_click=lambda: ui.notify('设置')).props('icon round').classes('mt-2')

# 主体内容
ui.label('左侧固定工具栏示例').classes('text-2xl text-center my-5')
for i in range(20):
    ui.card(f'内容项 {i+1}').classes('w-4/5 mx-auto my-2')

ui.run()
```

### 2. 底部固定通知条（可关闭）

```python
from nicegui import ui

# 底部固定通知条
notification_bar = ui.page_sticky(position='bottom', offset=0, z_index=999)
with notification_bar:
    with ui.row().classes('w-full bg-amber-50 p-3 border-t border-amber-200'):
        ui.icon('info').classes('text-amber-500 mr-2')
        ui.label('这是一条底部固定通知，点击关闭可隐藏').classes('flex-1')
        ui.button('×', on_click=lambda: notification_bar.set_visible(False)).props('flat round')

# 主体内容
for i in range(15):
    ui.div().classes('h-20 bg-gray-50 my-1')

ui.run()
```

