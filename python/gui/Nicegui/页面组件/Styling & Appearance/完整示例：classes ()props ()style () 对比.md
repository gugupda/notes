# 完整示例：classes ()/props ()/style () 对比

以下示例通过 `ui.button` 展示三种样式方式的用法、优先级及适用场景，包含详细注释和对比说明：

```python
from nicegui import ui

# 1. 定义自定义CSS（用于classes()的自定义类演示）
ui.add_css('''
.custom-btn-class {
    border: 2px solid #f97316; /* 橙色边框 */
    padding: 12px 24px; /* 内边距 */
    font-weight: 600; /* 字体加粗 */
}
''')

# 标题说明
ui.label('classes()/props()/style() 样式对比').classes('text-xl font-bold mb-6')

# ========== 示例1：仅用 props() （Quasar属性） ==========
ui.label('1. 仅 props()：控制Quasar组件属性（部分影响样式）').classes('text-sm text-gray-600 mb-2')
btn1 = ui.button('仅Props')
# props()：绑定Quasar按钮的核心属性（非纯样式，是组件功能/基础样式属性）
# 适用场景：控制组件的内置行为/基础样式（如是否扁平、图标、禁用等）
btn1.props('''
    flat  /* Quasar属性：扁平按钮（无背景） */
    icon=heart  /* 图标 */
    color=primary  /* Quasar主题色：主色 */
    rounded  /* 圆角 */
''')

ui.separator().classes('my-4')  # 分隔线

# ========== 示例2：仅用 classes() （Tailwind/Quasar/自定义类） ==========
ui.label('2. 仅 classes()：绑定CSS类（Tailwind/Quasar/自定义）').classes('text-sm text-gray-600 mb-2')
btn2 = ui.button('仅Classes')
# classes()：混合使用Tailwind、Quasar、自定义类
# 适用场景：批量复用样式、全局统一风格、混合多套CSS生态
btn2.classes('''
    custom-btn-class  /* 自定义类 */
    bg-primary !bg-orange-500  /* Quasar类 + Tailwind !important覆盖 */
    text-white  /* Tailwind类：文字白色 */
    shadow-lg  /* Tailwind类：阴影 */
''')

ui.separator().classes('my-4')

# ========== 示例3：仅用 style() （内联CSS） ==========
ui.label('3. 仅 style()：内联CSS（优先级最高）').classes('text-sm text-gray-600 mb-2')
btn3 = ui.button('仅Style')
# style()：直接写内联CSS，优先级高于classes()
# 适用场景：临时/唯一样式、快速调试、动态计算样式值
btn3.style('''
    background-color: #ef4444 !important; /* 红色背景（!important强化） */
    color: white;
    padding: 14px 28px;
    border-radius: 8px;
    border: none;
    box-shadow: 0 4px 12px rgba(239, 68, 68, 0.4);
''')

ui.separator().classes('my-4')

# ========== 示例4：三者混合（验证优先级） ==========
ui.label('4. 混合使用：style() > classes(!important) > classes(普通) > props()').classes('text-sm text-gray-600 mb-2')
btn4 = ui.button('混合样式（看优先级）')
# Step1: props() 定义基础样式（优先级最低）
btn4.props('color=secondary rounded-lg')
# Step2: classes() 定义普通类 + !important类（优先级中等）
btn4.classes('bg-secondary !bg-blue-500 text-white px-6 py-3')
# Step3: style() 定义内联样式（优先级最高，覆盖所有）
btn4.style('background-color: #8b5cf6 !important; /* 紫色背景，最终生效 */')

# 优先级总结说明
ui.label('''
优先级排序（从高到低）：
1. style() 内联CSS（带!important）
2. classes() 中带!important的Tailwind类
3. classes() 中普通Tailwind/Quasar/自定义类
4. props() 中Quasar组件属性对应的样式
''').classes('mt-6 text-sm text-gray-700')

ui.run()
```

### 核心说明

| 方法        | 优先级 | 适用场景                                                     |
| ----------- | ------ | ------------------------------------------------------------ |
| `style()`   | 最高   | 1. 临时调试样式；2. 动态计算的唯一样式（如根据变量改颜色）；3. 覆盖所有其他样式 |
| `classes()` | 中等   | 1. 复用全局样式（自定义类 / Tailwind 预设）；2. 混合 Tailwind/Quasar 生态；3. 批量调整样式 |
| `props()`   | 最低   | 1. 控制 Quasar 组件的核心行为（如是否禁用、显示图标）；2. 基础样式兜底；3. 利用 Quasar 主题体系 |

### 关键细节

1. `props()` 本质不是 “样式方法”，而是绑定 Quasar 组件的原生属性，部分属性（如`color`、`rounded`）会间接影响样式，但优先级最低；
2. `classes()` 中的`!important` Tailwind 类可覆盖 Quasar 原生样式（NiceGUI 调整了 CSS 层优先级），但无法覆盖`style()`的内联`!important`；
3. 实际开发中，优先用`classes()`（易维护、可复用），仅在需要 “强制覆盖” 或 “动态唯一样式” 时用`style()`，`props()`仅用于组件属性控制。