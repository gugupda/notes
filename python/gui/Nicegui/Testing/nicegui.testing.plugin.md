### NiceGUI `testing.plugin` 核心插件体系详解

`nicegui.testing.plugin` 是 NiceGUI 自动化测试框架的**顶层抽象插件**，是 `user_plugin`（用户交互）、`screen_plugin`（全局 UI 验证）的基础底座，同时提供测试环境初始化、应用生命周期管理、插件扩展等核心能力。它并非单一功能插件，而是一套 “可扩展的测试插件体系”，定义了测试插件的通用接口、应用启动 / 销毁规范，以及 Playwright 与 NiceGUI 应用的绑定逻辑，是所有 NiceGUI 测试插件的核心依赖。

#### 一、核心定位与设计初衷

NiceGUI 测试体系的核心诉求是 “低成本、高兼容” 地测试 Web 应用，而直接基于 Playwright 开发测试用例存在以下问题：

- 需手动管理 NiceGUI 应用的启动 / 停止（如确保测试前启动、测试后销毁）；
- 无统一的插件扩展机制，不同测试能力（交互、视觉、性能）难以整合；
- 测试环境与应用运行环境隔离性差（如端口冲突、残留进程）。

`testing.plugin` 针对这些问题设计了核心解决方案：

1. **标准化应用生命周期**：封装 `run_app()` 上下文管理器，自动处理应用启动、端口分配、进程销毁；
2. **插件化架构**：定义 `TestPlugin` 基类，所有测试插件（如 `user_plugin`/`screen_plugin`）均继承该类，保证接口统一；
3. **Playwright 深度绑定**：自动将 Playwright 页面与 NiceGUI 应用关联，无需手动配置浏览器连接；
4. **测试环境隔离**：每个测试用例 / 套件使用独立端口、独立浏览器上下文，避免环境污染。

#### 二、核心架构与关键组件

`testing.plugin` 的核心结构分为三层，自上而下为：

| 层级     | 组件                     | 作用                                                    |
| -------- | ------------------------ | ------------------------------------------------------- |
| 顶层接口 | `TestPlugin` 基类        | 定义所有测试插件的通用接口（如 `setup()`/`teardown()`） |
| 核心工具 | `run_app()` 上下文管理器 | 启动 / 停止 NiceGUI 应用，绑定 Playwright 页面          |
| 底层适配 | `PageExtension` 扩展类   | 为 Playwright Page 对象注入 NiceGUI 专属方法            |

##### 1. `TestPlugin` 基类（插件扩展的核心）

所有自定义 / 官方测试插件均需继承该类，定义了插件的生命周期方法：

```python
from nicegui.testing.plugin import TestPlugin

class CustomPlugin(TestPlugin):
    def __init__(self, page):
        super().__init__(page)  # 绑定 Playwright Page 对象
        self.page = page

    def setup(self):
        """插件初始化（测试用例执行前）"""
        pass

    def teardown(self):
        """插件清理（测试用例执行后）"""
        pass

    # 自定义测试方法
    def assert_component_count(self, component_type: str, expected: int):
        """断言指定类型的组件数量"""
        count = self.page.locator(f'data-testid^="{component_type}"').count()
        assert count == expected
```

##### 2. `run_app()` 上下文管理器（应用生命周期核心）

`run_app()` 是测试中最常用的工具，作用是启动 NiceGUI 应用并绑定 Playwright 页面，自动处理端口分配、应用销毁：

```python
from nicegui import testing
from app import main

# 基础用法：启动应用并获取 Playwright Page 对象
with testing.run_app(main) as page:
    # page 是 Playwright 原生 Page 对象，已绑定到运行中的 NiceGUI 应用
    page.goto("http://localhost:8080")  # 自动指向应用地址
```

**`run_app()` 关键参数**：

| 参数           | 作用                                            | 示例                                       |
| -------------- | ----------------------------------------------- | ------------------------------------------ |
| `app_function` | NiceGUI 应用入口函数（如 `main`）               | `testing.run_app(main)`                    |
| `port`         | 指定应用端口（默认随机分配，避免冲突）          | `testing.run_app(main, port=8080)`         |
| `browser`      | 指定测试浏览器（chromium/firefox/webkit）       | `testing.run_app(main, browser="firefox")` |
| `headless`     | 是否无头模式运行浏览器（默认 True）             | `testing.run_app(main, headless=False)`    |
| `native`       | 是否启用 NiceGUI 原生窗口（测试时需设为 False） | `testing.run_app(main, native=False)`      |

#### 三、核心使用流程（基础示例）

##### 1. 应用源码（`app.py`）

```python
# app.py
from nicegui import ui

def main():
    ui.label("Hello Testing!")
    ui.button("Click Me", on_click=lambda: ui.notify("Clicked!"))
    # 测试专用启动配置：禁用原生窗口、不自动打开浏览器
    ui.run(native=False, show=False, reload=False)

if __name__ == "__main__":
    main()
```

##### 2. 测试用例（`test_app.py`）

```python
# test_app.py
import pytest
from nicegui import testing
from nicegui.testing.plugin import TestPlugin
from app import main

# 步骤1：定义测试夹具，启动应用并初始化插件
@pytest.fixture(scope="function")
def test_page():
    # run_app() 自动启动应用，分配随机端口，创建 Playwright 页面
    with testing.run_app(main, browser="chromium") as page:
        yield page

# 步骤2：基础测试（直接使用 Playwright Page 对象）
def test_basic_interaction(test_page):
    # 验证页面标题
    assert test_page.title() == "NiceGUI"
    # 定位按钮并点击（Playwright 原生 API）
    button = test_page.get_by_text("Click Me")
    button.click()
    # 验证通知出现
    assert test_page.get_by_text("Clicked!").exists()

# 步骤3：自定义插件示例
class CountPlugin(TestPlugin):
    def count_buttons(self):
        """统计页面中按钮数量"""
        return self.page.get_by_role("button").count()

def test_custom_plugin(test_page):
    # 初始化自定义插件
    count_plugin = CountPlugin(test_page)
    # 调用插件方法断言
    assert count_plugin.count_buttons() == 1
```

##### 3. 运行测试

```bash
pytest test_app.py -v
```

#### 四、官方内置插件的整合逻辑

`user_plugin` 和 `screen_plugin` 均是基于 `testing.plugin` 扩展的具体实现，其核心整合逻辑如下：

1. **继承 `TestPlugin` 基类**：

   ```python
   # user_plugin 核心实现（简化版）
   from nicegui.testing.plugin import TestPlugin
   
   class Page(TestPlugin):
       def __init__(self, page):
           super().__init__(page)
       
       def click(self, element):
           """封装点击逻辑，适配 NiceGUI 组件"""
           element_locator = self._get_component_locator(element)
           element_locator.click()
   ```

2. **复用 `run_app()` 上下文**：

   官方插件无需重新实现应用启动逻辑，直接使用`run_app()`返回的 Playwright Page 对象初始化；

3. **扩展 Playwright Page 方法**：

   通过`PageExtension`为原生 Page 对象注入 NiceGUI 专属方法（如`get_by_component`）。

#### 五、进阶使用场景

##### 1. 多插件组合使用

`testing.plugin` 支持同时初始化多个插件，满足复杂测试场景（交互 + 视觉 + 自定义验证）：

```python
@pytest.fixture(scope="module")
def test_plugins():
    with testing.run_app(main) as page:
        # 初始化官方插件 + 自定义插件
        user = testing.user_plugin.Page(page)
        screen = testing.screen_plugin.Screen(page)
        count_plugin = CountPlugin(page)
        yield {
            "user": user,
            "screen": screen,
            "count": count_plugin
        }

def test_combined(test_plugins):
    # 1. user_plugin 模拟交互
    test_plugins["user"].click(test_plugins["user"].get_by_text("Click Me"))
    # 2. screen_plugin 全局验证
    test_plugins["screen"].assert_text("Clicked!")
    # 3. 自定义插件统计按钮
    assert test_plugins["count"].count_buttons() == 1
```

##### 2. 测试环境隔离（每个用例独立端口）

`run_app()` 默认随机分配端口，确保多个测试用例 / 套件并行运行时无端口冲突：

```python
def test_isolated_env_1():
    with testing.run_app(main) as page:
        assert page.url.startswith("http://localhost:")  # 随机端口

def test_isolated_env_2():
    with testing.run_app(main) as page:
        assert page.url.startswith("http://localhost:")  # 另一随机端口
```

##### 3. 自定义浏览器配置

通过 `run_app()` 的 `browser` 参数指定测试浏览器，或自定义 Playwright 启动参数：

```python
from playwright.sync_api import BrowserType

def test_firefox():
    # 指定 Firefox 浏览器
    with testing.run_app(main, browser="firefox") as page:
        assert page.browser.browser_type.name == "firefox"

# 自定义浏览器启动参数（如禁用图片加载）
def test_custom_browser_args():
    def browser_launcher(browser_type: BrowserType):
        return browser_type.launch(
            args=["--blink-settings=imagesEnabled=false"],
            headless=True
        )
    with testing.run_app(main, browser_launcher=browser_launcher) as page:
        # 验证图片未加载
        assert page.locator("img").count() == 0
```

#### 六、关键注意事项

1. **应用启动配置**：

   测试时需确保`ui.run()`配置为`native=False`、`show=False`、`reload=False`，否则会干扰 Playwright 控制的浏览器；

2. **插件生命周期**：

   自定义插件的`setup()`会在测试用例执行前调用，`teardown()`在测试用例执行后调用，可用于清理测试数据、重置 UI 状态；

3. **异步测试支持**：

   `testing.plugin`同时支持同步 / 异步测试，异步场景需使用`async_run_app()`上下文管理器：

   ```python
   import pytest
   import asyncio
   
   @pytest.mark.asyncio
   async def test_async():
       async with testing.async_run_app(main) as page:
           await page.get_by_text("Click Me").click()
           assert await page.get_by_text("Clicked!").is_visible()
   ```

4. **CI/CD 适配**：

   在 CI 环境中运行时，需确保 Playwright 依赖的系统库已安装（可通过`playwright install-deps`安装），并使用 `headless=True`模式。

#### 七、与纯 Playwright 测试的核心差异

| 特性         | `testing.plugin` + 官方插件         | 纯 Playwright                     |
| ------------ | ----------------------------------- | --------------------------------- |
| 应用生命周期 | 自动管理（启动 / 停止 / 端口分配）  | 需手动编写启动 / 停止逻辑         |
| 插件扩展     | 标准化接口，支持多插件组合          | 无统一扩展机制                    |
| 环境隔离     | 自动随机端口，无冲突                | 需手动管理端口 / 进程             |
| NiceGUI 适配 | 内置组件定位、异步等待适配          | 需手动适配 NiceGUI 前端逻辑       |
| 易用性       | 低学习成本（贴合 NiceGUI 开发习惯） | 高学习成本（需掌握 Web 测试细节） |

