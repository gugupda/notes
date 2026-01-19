# element.style

### 一、`element.style` 核心定义与作用

在 NiceGUI 中，`element.style()` 是用于为单个 UI 元素（如示例中的 `ui.button`）直接设置**内联 CSS 样式**的方法。内联样式优先级极高（仅次于 `!important` 修饰的样式），可精准覆盖元素默认样式（包括 Quasar 组件的基础样式），是自定义元素视觉表现的核心方式之一。

### 二、语法与使用规则

#### 1. 基础语法

```python
element.style('CSS属性名: 属性值; [可选：更多CSS属性]')
```

- 传入的字符串需符合标准 CSS 语法：**属性名：值** 的键值对形式，多个属性用分号分隔；
- 支持所有标准 CSS 属性（如 `background-color`、`font-size`、`padding` 等）；
- 可使用 `!important` 强制提升样式优先级，突破 Quasar 内置样式的限制（NiceGUI 已将 Quasar 的 `!important` 样式降级到低优先级 CSS 层，因此自定义 `!important` 可覆盖）。

#### 2. 示例演示

以 `ui.button` 为例，常见的 `style` 配置场景：

```python
from nicegui import ui

# 基础样式：背景色、文字色、字体大小
btn = ui.button('Custom Style Button')
btn.style('''
    background-color: #4CAF50;  /* 绿色背景 */
    color: white;               /* 白色文字 */
    font-size: 16px;            /* 字体大小 */
    padding: 12px 24px;         /* 内边距：上下12px，左右24px */
    border-radius: 8px;         /* 圆角 */
    border: none;               /* 去掉边框 */
    cursor: pointer;            /* 鼠标悬浮为手型 */
''')

# 覆盖 Quasar 内置样式（带 !important）
btn2 = ui.button('Override Quasar')
btn2.style('background-color: red !important; color: white !important;')

ui.run()
```

### 三、关键特性

1. **优先级**：

   - 内联样式（`style`）优先级高于 Tailwind 类（`classes`）和 Quasar 原生样式；
   - 带 `!important` 的 `style` 可覆盖所有其他样式（包括 NiceGUI 降级后的 Quasar `!important` 样式）。

2. **与 `classes`/`props` 的区别**：

   | 方法        | 作用                     | 适用场景                              |
   | ----------- | ------------------------ | ------------------------------------- |
   | `style()`   | 内联 CSS，精准控制       | 单个元素的个性化样式                  |
   | `classes()` | 绑定 Tailwind / 自定义类 | 批量复用的样式（如通用按钮）          |
   | `props()`   | 传递 Quasar 组件属性     | 控制组件行为 / 基础样式（如按钮大小） |

3. **支持的 CSS 范围**：

   涵盖所有标准 CSS 属性，包括布局（`display`/`position`）、视觉（`color`/`box-shadow`）、交互（`hover` 需特殊处理）等。

   - 注意：伪类（如 `hover`）无法直接通过内联 `style` 设置，需结合 `classes` 或自定义 CSS 类：

     ```python
     # 实现 hover 效果：先定义类，再通过 classes 绑定
     ui.add_css('.btn-hover:hover { background-color: #3e8e41 !important; }')
     btn.classes('btn-hover')
     btn.style('background-color: #4CAF50;')
     ```

### 四、常见使用场景

1. **快速调试样式**：直接修改 `style` 字符串，无需定义类，适合临时调整；

2. **覆盖组件默认样式**：突破 Quasar 组件的样式限制（如修改按钮默认背景色）；

3. **动态样式绑定**：结合变量动态修改样式（NiceGUI 支持响应式更新）：

   ```python
   from nicegui import ui, reactive
   
   state = reactive({'bg_color': 'blue'})
   
   btn = ui.button('Dynamic Style')
   # 响应式更新样式
   ui.button('Change Color').on('click', lambda: setattr(state, 'bg_color', 'orange'))
   btn.bind_style('background-color', state, 'bg_color')
   
   ui.run()
   ```

### 五、注意事项

1. 避免过度使用：内联样式会增加代码冗余，通用样式建议优先用 `classes`（Tailwind）；
2. 语法规范：CSS 属性名需用中划线（如 `background-color`），而非 Python 驼峰式（`backgroundColor`）；
3. 兼容性：所有 CSS 属性需符合浏览器标准，NiceGUI 基于 Web 渲染，无额外兼容性限制。

# NiceGUI element 的 `style` 方法详解

NiceGUI 是一款基于 Python 的轻量级 Web 界面框架，其核心元素（Element）的 `style` 方法是用于**自定义元素 CSS 样式**的核心接口，能够灵活修改元素的外观、布局、交互效果等，完全兼容标准 CSS 语法。

#### 一、基本定义与作用

`style` 方法是所有 NiceGUI 元素（如 `ui.label`、`ui.button`、`ui.card` 等）的通用方法，作用是为元素添加 / 修改内联 CSS 样式（对应 HTML 元素的 `style` 属性），优先级高于外部样式表，可快速实现元素的个性化样式定制。

#### 二、语法与调用方式

##### 1. 基础调用

```python
from nicegui import ui

# 创建元素并调用 style 方法
label = ui.label("自定义样式文本")
# 单个样式：参数为 CSS 键值对（驼峰式/短横线式均支持）
label.style('color: red; font-size: 20px;')

# 链式调用（推荐）
ui.button("提交") \
    .style('background-color: #4CAF50; color: white; padding: 10px 20px; border-radius: 5px;') \
    .style('cursor: pointer; border: none;')  # 多次调用会合并样式

ui.run()
```

##### 2. 支持的参数格式

- **字符串格式**（最常用）：直接传入标准 CSS 样式字符串，多个样式用分号分隔；

- **关键字参数**（进阶）：通过关键字传递样式（需将 CSS 短横线命名转为驼峰命名，如 `background-color` → `backgroundColor`）：

  ```python
  ui.input("输入框").style(
      width='300px',
      border='1px solid #ccc',
      borderRadius='4px',
      padding='8px 12px'
  )
  ```

#### 三、核心特性

##### 1. 样式合并与覆盖

- 多次调用 `style` 方法时，样式会**合并**；若重复定义同一属性，**后调用的会覆盖先调用的**：

  ```python
  btn = ui.button("测试")
  btn.style('color: red; font-size: 16px;')
  btn.style('color: blue;')  # 最终文字颜色为 blue，font-size 仍为 16px
  ```

##### 2. 动态修改样式

可结合 NiceGUI 的响应式机制（如 `ui.timer`、事件回调）动态修改样式，实现交互效果：

```python
from nicegui import ui

btn = ui.button("点击变色")

def change_style():
    # 动态切换背景色
    current_bg = btn.style.get('background-color', 'white')
    btn.style('background-color: green;' if current_bg == 'white' else 'background-color: white;')

btn.on('click', change_style)
ui.run()
```

##### 3. 兼容 CSS 所有属性

`style` 方法支持所有标准 CSS 属性，包括：

- 布局类：`width`、`height`、`margin`、`padding`、`display`、`position`；
- 视觉类：`color`、`background-color`、`border`、`border-radius`、`box-shadow`；
- 文本类：`font-size`、`font-weight`、`text-align`、`text-decoration`；
- 交互类：`cursor`、`opacity`、`transition`（过渡动画）。

示例（带过渡动画的按钮）：

```python
ui.button("hover 动效") \
    .style('padding: 10px 20px; background-color: #2196F3; color: white; border: none; border-radius: 4px;') \
    .style('cursor: pointer; transition: background-color 0.3s ease;') \
    .style(':hover { background-color: #0b7dda; }')  # 伪类样式
```

##### 4. 伪类 / 伪元素支持

可直接在 `style` 中使用 CSS 伪类（如 `:hover`、`:active`、`:focus`）和伪元素（如 `::before`、`::after`）：

```python
ui.label("带前缀的文本") \
    .style('position: relative; padding-left: 20px;') \
    .style('::before { content: "●"; color: red; position: absolute; left: 0; top: 0; }')
```

#### 四、常见注意事项

1. **优先级问题**：内联样式（`style` 方法）优先级高于 NiceGUI 内置样式和外部 CSS，若需覆盖需确保属性书写正确；
2. **单位问题**：CSS 数值属性需带单位（如 `10px`、`50%`），仅 `0` 可省略单位；
3. **驼峰 vs 短横线**：关键字参数需用驼峰（`backgroundColor`），字符串参数可用短横线（`background-color`），两种方式等价；
4. **特殊字符转义**：若样式中包含引号，需用单 / 双引号嵌套（如 `style("content: '★';")`）；
5. **动态样式重置**：若需清空样式，可调用 `element.style('')`（传入空字符串）。

#### 五、进阶用法：结合 `classes` 方法

`style` 方法适合单个元素的样式定制，若需批量复用样式，可结合 `classes` 方法（绑定 CSS 类）：

```python
# 定义全局样式类
ui.add_head_html('''
    <style>
        .custom-btn {
            padding: 10px 20px;
            background-color: purple;
            color: white;
            border-radius: 8px;
        }
        .custom-btn:hover {
            background-color: #800080;
        }
    </style>
''')

# 批量应用类 + 局部样式调整
ui.button("按钮1").classes('custom-btn').style('margin-right: 10px;')
ui.button("按钮2").classes('custom-btn').style('margin-left: 10px;')
```

