# Playwright 自动化测试与架构高频面试题

> 面向 Python Playwright 自动化测试工程师、SDET、测试开发工程师、质量平台工程师和测试架构师面试。内容覆盖浏览器自动化基础、稳定性设计、框架架构、并行执行、测试数据、网络模拟、CI/CD、可观测性与系统设计。
>
> 示例默认使用 Python 3.10+、`pytest` 和 `pytest-playwright`。面试时应先澄清系统类型、浏览器覆盖范围、发布频率、测试时限、环境约束及数据合规要求，再选择测试策略；不要把“写出能跑的 UI 脚本”等同于“构建可靠的质量体系”。

## 目录

1. [面试回答框架](#一面试回答框架)
2. [Playwright 基础与核心机制](#二playwright-基础与核心机制)
3. [定位、等待与稳定性](#三定位等待与稳定性)
4. [测试框架与代码组织](#四测试框架与代码组织)
5. [认证、数据与环境治理](#五认证数据与环境治理)
6. [网络控制与跨边界测试](#六网络控制与跨边界测试)
7. [并行、浏览器覆盖与性能](#七并行浏览器覆盖与性能)
8. [失败诊断与可观测性](#八失败诊断与可观测性)
9. [CIcd 与质量门禁](#九cicd-与质量门禁)
10. [架构与场景设计题](#十架构与场景设计题)
11. [现场 Coding 与追问清单](#十一现场-coding-与追问清单)
12. [面试前自检](#十二面试前自检)

---

## 一、面试回答框架

### 1. 面试官真正想判断什么？

Playwright 面试通常不是只考 API 记忆。面试官会判断候选人能否把不稳定、成本高、反馈慢的端到端测试，建设成可信、可维护、可诊断的工程系统。

|能力方向|面试官关注点|
|---|---|
|自动化基础|定位、断言、等待、页面和上下文生命周期|
|稳定性|消除竞态、隔离状态、控制外部依赖、处理重试|
|框架能力|Fixture、Page Object、测试分层、配置和报告|
|工程效率|并行、分片、标签筛选、浏览器矩阵和资源控制|
|质量策略|哪些场景做 E2E、哪些下沉到 API/组件/契约测试|
|平台能力|环境、测试数据、密钥、CI/CD、指标、故障归因|
|架构判断|在速度、覆盖、稳定性、成本和风险之间作取舍|

### 2. 开放题的推荐回答顺序

面对“如何设计自动化框架”或“如何治理 flaky test”一类问题，按以下顺序回答：

1. **明确目标与边界**：测试对象、关键用户旅程、SLO、发布频率、最大可接受反馈时间。
2. **分层策略**：说明 E2E、API、组件、契约、单元测试各自负责什么。
3. **核心设计**：隔离模型、定位约定、等待原则、数据与环境策略。
4. **执行设计**：并发模型、浏览器矩阵、分片、资源限制、失败重试规则。
5. **诊断与治理**：Trace、截图、视频、日志、分类指标、责任归属和退出准则。
6. **渐进落地**：先覆盖高风险路径，基线化指标，再逐步淘汰低价值或不稳定测试。

高分答案会量化目标，例如“PR 冒烟在 10 分钟内完成，主干回归在 30 分钟内完成；单用例首次失败可重试一次，但重试通过仍计为不稳定事件”。

---

## 二、Playwright 基础与核心机制

### Q1：Playwright 的核心对象有哪些？`Browser`、`BrowserContext`、`Page` 如何选择生命周期？

**回答：**

- `Browser` 是一个浏览器进程或浏览器连接，启动成本最高，通常在测试会话或 worker 范围复用。
- `BrowserContext` 是隔离的浏览器配置文件，拥有独立的 Cookie、localStorage、sessionStorage、权限、缓存和网络状态。它相当于一个无痕用户会话。
- `Page` 是 Context 内的标签页。一个 Context 可以创建多个 Page，用于新窗口、多标签或跨窗口业务流。

生产测试中一般按 **每个测试用例一个 Context** 隔离状态，再由该 Context 创建 Page。不要让多个测试共享同一个已登录 Page，否则 Cookie、页面状态和服务端副作用会互相污染。

```python
from playwright.sync_api import Browser


def test_new_context_isolates_session(browser: Browser) -> None:
    first_context = browser.new_context()
    second_context = browser.new_context()

    first_page = first_context.new_page()
    second_page = second_context.new_page()

    first_page.goto("https://example.test")
    first_page.context.add_cookies(
        [{"name": "tenant", "value": "first", "url": "https://example.test"}]
    )

    assert first_page.context.cookies()[0]["value"] == "first"
    assert second_page.context.cookies() == []

    first_context.close()
    second_context.close()
```

**追问：为什么不每个用例启动一个 Browser？** Browser 启动成本较高，会显著拉长总执行时间。Context 可以提供足够的会话隔离；但如果浏览器进程泄漏、崩溃或存在浏览器级副作用，则需要由 worker 或执行器重启 Browser 兜底。

### Q2：Playwright 与 Selenium 的主要差异是什么？

**回答：** Playwright 直接使用浏览器自动化协议并内置浏览器管理、自动等待、网络拦截、多 Context 隔离、Trace 和多语言 SDK。Selenium 基于 WebDriver 标准，生态和厂商兼容历史更长，在网格、遗留基础设施和特定企业浏览器环境中仍有价值。

不能简单回答“Playwright 更快”。更准确的比较维度如下：

|维度|Playwright|Selenium|
|---|---|---|
|等待模型|Locator 与 Web-first assertion 内置自动等待|通常需要显式等待策略|
|会话隔离|Browser Context 原生支持|常见做法是独立 WebDriver 会话|
|调试工件|Trace Viewer、截图、视频、网络控制集成较完整|依赖测试框架、Grid 或第三方能力组合|
|标准化|使用自身自动化接口|WebDriver 为行业标准|
|适用判断|现代 Web 应用、快速并行回归与可诊断 E2E|既有 Selenium 平台、特定网格和兼容性要求|

选型还应考虑团队语言、CI 运行环境、目标浏览器、已有资产迁移成本和安全要求。

### Q3：同步 API 和异步 API 如何选择？能否混用？

Python Playwright 提供同步 API（`playwright.sync_api`）与异步 API（`playwright.async_api`）。同步 API 对已有 `pytest` 用例更直观；异步 API 适合已有 `asyncio` 应用、需要异步客户端协作或希望控制高并发 I/O 的场景。

**同一个调用链中不能混用同步和异步 API。** 不要在 async 测试里调用同步 `Page`，也不要试图对同步方法使用 `await`。团队应选定一种风格，降低 fixture 和 helper 的认知成本。

```python
import pytest
from playwright.async_api import Page, expect


@pytest.mark.asyncio
async def test_async_login(page: Page) -> None:
    await page.goto("https://app.example.test/login")
    await page.get_by_label("Email").fill("user@example.test")
    await page.get_by_label("Password").fill("not-a-real-secret")
    await page.get_by_role("button", name="Sign in").click()
    await expect(page).to_have_url("https://app.example.test/dashboard")
```

**追问：异步 API 是否让 UI 操作天然并发？** 不是。一个 Page 上的操作仍须保持业务顺序。并发应发生在独立 Context、独立 Page 或独立测试 worker 之间，并要考虑被测环境容量和数据冲突。

### Q4：什么是 Auto-waiting？它解决了什么，又不能解决什么？

**回答：** Playwright 的 Locator 操作会等待元素满足 actionability 条件，例如元素已附着、可见、稳定、可接收事件和未被禁用。`expect` 断言也会轮询，直到条件成立或超时。因此多数场景不应写 `sleep`。

Auto-waiting 不能理解业务“数据最终一致”“异步任务完成”或“第三方回调到达”。例如点击“创建订单”后，后端可能异步生成订单号；此时应等待可观察的业务结果，如订单列表出现该订单、API 响应完成、状态字段变为 `READY`，而不是盲等 5 秒。

```python
from playwright.sync_api import Page, expect


def test_order_becomes_ready(page: Page) -> None:
    page.get_by_role("button", name="Create order").click()

    row = page.get_by_role("row").filter(has_text="ORDER-2026-001")
    expect(row.get_by_text("READY", exact=True)).to_be_visible()
```

### Q5：`Locator` 与 `ElementHandle` 有何区别？为什么优先使用 Locator？

**回答：** `ElementHandle` 是某一时刻 DOM 节点的句柄。前端重渲染后，该节点可能被替换而变成 stale。`Locator` 是延迟解析的查询计划；每次操作和断言时都会重新定位，并且拥有自动等待能力。

因此业务测试优先使用 Locator：

```python
# 推荐：在 click 时重新解析，且拥有自动等待。
page.get_by_role("button", name="Save").click()

# 仅在确实需要底层 DOM 句柄能力时使用。
handle = page.query_selector("#legacy-canvas")
```

`ElementHandle` 并非不能使用，但应当是例外，例如与低层 JS API 集成或处理无法通过 Locator 表达的遗留控件时。若频繁依赖它，往往意味着定位策略或页面可测试性需要改进。

### Q6：`Page`、`FrameLocator` 和 Popup 如何处理？

**回答：** iframe 内元素不属于主页面 DOM。使用 `frame_locator()` 在目标 frame 范围内定位；不要用全局 CSS 选择器猜 iframe 内部元素。对点击后弹出的新窗口，需要先注册等待，再触发动作，避免错过事件。

```python
from playwright.sync_api import Page, expect


def test_payment_in_iframe_and_receipt_popup(page: Page) -> None:
    payment_frame = page.frame_locator("iframe[title='Secure payment']")
    payment_frame.get_by_label("Card number").fill("4242 4242 4242 4242")

    with page.expect_popup() as popup_info:
        page.get_by_role("link", name="Open receipt").click()

    receipt_page = popup_info.value
    receipt_page.wait_for_load_state()
    expect(receipt_page.get_by_role("heading", name="Receipt")).to_be_visible()
```

**追问：何时用 `expect_navigation`？** 对传统整页导航可用，但现代 SPA 常通过 History API 更新 URL 或局部渲染。优先断言最终用户可见状态和 URL，而不是把“导航发生”当作唯一成功信号。

---

## 三、定位、等待与稳定性

### Q7：如何设计可靠的元素定位策略？

**回答：** 定位器应表达用户能感知的语义，并尽量与 DOM 结构、样式类名和多语言文案解耦。建议优先级：

1. `get_by_role()` 配合可访问名称，最贴近用户行为。
2. `get_by_label()`，适合表单控件。
3. `get_by_placeholder()`，仅当 placeholder 是稳定产品契约。
4. `get_by_text()`，适合稳定的用户可见文案。
5. `get_by_test_id()`，适合无障碍语义不充分或文案会频繁变化的元素。
6. CSS/XPath 仅用于无法改造的遗留界面；避免长层级、`nth-child`、自动生成 class 和绝对 XPath。

```python
# 优先：可访问语义。
page.get_by_role("button", name="Add to cart").click()

# 推荐：由前端明确提供的稳定测试契约。
page.get_by_test_id("checkout-submit").click()

# 脆弱：DOM 层级或 CSS 模块 hash 变化就会失效。
page.locator("div.container > div:nth-child(2) .button-9aX3").click()
```

前端可测试性是跨团队契约。测试团队应参与组件设计，要求关键控件具备正确 role、label 和可访问名称；`data-testid` 应是受治理的语义名称，不是随手添加的实现细节。

### Q8：为什么不应使用 `time.sleep()` 或 `wait_for_timeout()`？

固定等待有两个问题：页面快时浪费时间，页面慢时仍会失败。它既没有证明业务状态正确，也放大了执行时长。正确方法是等待**可验证的事件或状态**：

- 点击后等待按钮禁用、Toast 出现、目标行渲染。
- 提交请求后等待特定 API 响应或页面业务状态。
- 异步任务后轮询受控的业务 API，或在 UI 上断言最终状态。

```python
from playwright.sync_api import Page, expect


def test_export_started(page: Page) -> None:
    with page.expect_response(
        lambda response: "/api/exports" in response.url
        and response.request.method == "POST"
        and response.status == 202
    ):
        page.get_by_role("button", name="Export CSV").click()

    expect(page.get_by_role("status")).to_contain_text("Export is being prepared")
```

仅在调试复现、演示节奏或验证动画极短的特定场景下使用 `wait_for_timeout()`，并说明它不是业务同步机制。

### Q9：如何处理动态列表、重复元素和严格模式错误？

Locator 默认强调唯一性。若点击操作匹配多个元素，严格模式错误是在提醒测试没有表达清楚意图，不应立即用 `.first` 压制。

应先缩小范围到业务实体，再定位操作：

```python
from playwright.sync_api import Page


def delete_invoice(page: Page, invoice_number: str) -> None:
    invoice_row = page.get_by_role("row").filter(has_text=invoice_number)
    invoice_row.get_by_role("button", name="Delete").click()
    page.get_by_role("button", name="Confirm deletion").click()
```

`.first`、`.last`、`.nth()` 只适用于顺序本身就是业务契约的场景，例如按时间排序的最新通知。即使如此，也应在代码或测试名中说明该顺序约束。

### Q10：断言应该覆盖哪些层次？

**回答：** 端到端测试不应只断言“没有报错”或“URL 改变”。至少考虑：

- **交互反馈**：按钮状态、校验信息、Toast、加载状态。
- **业务结果**：订单金额、角色权限、状态迁移、列表记录。
- **网络契约**：关键请求的方法、状态码和必要字段。
- **持久化或副作用**：可通过受控 API 查询、事件消费记录或测试数据库验证。

不要把所有内部实现都断言一遍。E2E 的重点是用户可见结果与跨服务关键契约；内部算法分支应由单元或服务级测试负责。

```python
from playwright.sync_api import Page, expect


def test_discount_is_applied(page: Page) -> None:
    page.get_by_label("Coupon code").fill("SPRING20")
    page.get_by_role("button", name="Apply").click()

    expect(page.get_by_test_id("discount-amount")).to_have_text("-$20.00")
    expect(page.get_by_test_id("order-total")).to_have_text("$80.00")
```

### Q11：常见 flaky test 根因有哪些？如何系统治理？

常见根因与对应措施：

|根因|表现|治理方式|
|---|---|---|
|隐式竞态|偶发找不到元素、偶发点击无效|用 Locator 与 Web-first assertion 表达真实完成条件|
|共享状态|单跑通过、并行或全量失败|每例独立 Context、唯一测试数据、清理策略|
|弱定位|UI 微调后大量失败|语义化 Locator 与测试 ID 契约|
|外部依赖|第三方服务抖动导致失败|Mock、契约测试、沙箱或可控 stub|
|环境不稳定|延迟尖刺、部署窗口失败|健康检查、环境版本固定、容量隔离|
|时区/时间|跨日、夏令时、顺序相关失败|注入可控时钟、统一时区、测试边界日期|
|重试掩盖问题|首次失败、重试通过|保留首次失败工件并统计 flaky rate|

治理步骤不是“统一增加 timeout”。先保存 Trace、网络日志、截图、视频和环境版本；再对失败分类，修复最大的类别；为每个用例记录首次通过率、重试通过率、平均时长和失败归因。长期重复失败且业务价值低的用例应下线或重写，避免污染信任。

### Q12：Timeout 应该如何分层设置？

建议区分以下超时，不应只设置一个全局巨大值：

- **动作/断言超时**：等待元素可操作或结果出现，通常较短。
- **页面导航超时**：整页加载或跨域跳转，取决于应用性能目标。
- **单测试超时**：防止死循环或外部依赖挂死。
- **作业超时**：防止 CI worker 长期占用。

超时值应来自性能基线和 SLO，而不是猜测。若常态接口 $p95$ 为 800 ms，可以给 UI 最终状态更宽裕但有限的预算；若必须调大 timeout，先判断是测试等待条件错误，还是产品性能退化。两者都值得被观测。

---

## 四、测试框架与代码组织

### Q13：如何组织 Python Playwright 项目目录？

一个可演进的最小结构如下：

```text
tests/
  e2e/
    test_checkout.py
    test_account_settings.py
  api/
    test_orders_api.py
  pages/
    login_page.py
    checkout_page.py
  components/
    toast.py
    address_form.py
  fixtures/
    users.py
    orders.py
  conftest.py
  helpers/
    api_client.py
    polling.py
  .auth/
    user.json                 # 仅本地临时产物，必须被 gitignore
pyproject.toml
```

原则：测试文件表达业务场景；Page Object 表达页面或领域动作；组件对象表达可复用 UI 片段；fixture 负责资源构造和销毁；API client 用于测试准备和结果核验。不要把所有 helper 堆进一个 `utils.py`，也不要让 Page Object 承担数据库操作、断言和业务数据生成的所有职责。

### Q14：Page Object Model (POM) 的价值和常见误区是什么？

**价值：** POM 将页面结构变化集中在少量类中，使测试用例以业务语言表达意图，降低重复 Locator 的维护成本。

**误区：**

- 把每一个 DOM 节点都包装为 getter，形成没有业务意义的“选择器转发层”。
- 在 Page Object 内写大量断言，使失败信息丢失业务上下文。
- 建造跨十几个页面的“万能 BasePage”，继承层级复杂且难以理解。
- 把 API 调用、数据构造、清库和 UI 操作混在一个对象中。

推荐按领域动作封装：

```python
from playwright.sync_api import Locator, Page


class CheckoutPage:
    def __init__(self, page: Page) -> None:
        self.page = page
        self.submit_order_button: Locator = page.get_by_role(
            "button", name="Place order"
        )

    def open(self) -> None:
        self.page.goto("/checkout")

    def choose_shipping_method(self, method_name: str) -> None:
        self.page.get_by_role("radio", name=method_name).check()

    def place_order(self) -> None:
        self.submit_order_button.click()
```

对应测试应保留业务断言：

```python
from playwright.sync_api import Page, expect


def test_customer_can_place_order(page: Page) -> None:
    checkout = CheckoutPage(page)
    checkout.open()
    checkout.choose_shipping_method("Express")
    checkout.place_order()

    expect(page.get_by_role("heading", name="Order confirmed")).to_be_visible()
```

### Q15：pytest fixture 应如何设计作用域和清理逻辑？

Fixture 应明确资源所有权：谁创建、谁使用、何时销毁。通常 Browser 为 session 或 worker 级，Context 和 Page 为 function 级，测试用户和订单数据也尽量为 function 级或显式 namespace 隔离。

```python
import pytest
from playwright.sync_api import Browser, Page


@pytest.fixture
def authenticated_page(browser: Browser) -> Page:
    context = browser.new_context(storage_state=".auth/customer.json")
    page = context.new_page()

    yield page

    context.close()
```

使用 `yield` 而不是只返回对象，可以确保失败或异常后仍执行清理。若清理本身可能失败，要记录清晰错误，不要静默吞掉；服务端无法立即删除的资源应带唯一前缀，并由周期性 janitor 任务兜底回收。

### Q16：如何使用 Marker 管理测试层级？

建议按风险、耗时和依赖边界标记，而不是只按目录。常见标记：

- `smoke`：关键主路径，PR 必跑。
- `critical`：影响支付、权限、数据丢失等高风险流程。
- `e2e`：真实浏览器和多服务集成。
- `contract`：跨服务协议验证。
- `external`：依赖第三方或受限环境，通常不阻塞 PR。
- `serial`：明确不能并行的遗留测试，需持续治理而非无限增长。

`pyproject.toml` 示例：

```toml
[tool.pytest.ini_options]
addopts = "-ra --strict-markers"
markers = [
  "smoke: critical path tests for pull requests",
  "e2e: browser-based end-to-end tests",
  "external: tests that call third-party systems",
]
```

运行时可用 `pytest -m "smoke and not external"`。Marker 是调度契约，必须有明确准入规则和定期审计；不能让所有用例都贴 `smoke`。

### Q17：测试之间应该互相调用吗？

不应该。测试执行顺序不保证，任何测试都必须能独立运行和重复执行。共享的应是 fixture、领域 helper、Page Object 和数据工厂，不是前置测试的输出。

若“创建用户”耗时高，不应让后续测试依赖 `test_create_user`，而应通过 API fixture、数据库工厂或预置租户构造所需状态。这样既可并行，也能在单测失败时准确定位责任。

---

## 五、认证、数据与环境治理

### Q18：如何避免每个测试都通过 UI 登录？

UI 登录适合覆盖认证本身，不适合成为每个业务用例的共同前置条件。可以用一次 UI 登录生成 `storage_state`，然后在每个测试的新 Context 中加载该状态；或者使用受审计的测试 API 创建短期会话。

```python
from pathlib import Path
from playwright.sync_api import Page


AUTH_STATE = Path(".auth/customer.json")


def login_once_and_save_state(page: Page) -> None:
    page.goto("https://app.example.test/login")
    page.get_by_label("Email").fill("test-customer@example.test")
    page.get_by_label("Password").fill("provided-by-secret-store")
    page.get_by_role("button", name="Sign in").click()
    page.context.storage_state(path=AUTH_STATE)
```

注意事项：

- 状态文件可能含 session token，必须加入 `.gitignore`，CI 从安全的临时工作区生成。
- token 需要短生命周期、最小权限、可撤销，不能复用真实用户凭证。
- 含 MFA、验证码或硬件令牌的流程要分层测试：认证集成环境用受控测试绕过或专用 IdP，生产链路以合规的合成监控方式少量验证。
- 认证状态不应跨测试共享可变业务数据；会话可复用，领域实体仍应隔离。

### Q19：测试数据如何做到并行安全、可回收且符合隐私要求？

**回答：** 每个运行、worker 和测试用例都应拥有唯一标识，例如 `e2e-{run_id}-{worker_id}-{case_id}`。所有创建资源带上该标签，查询时也限定命名空间，避免误读其他测试或共享环境中的人工数据。

数据来源优先级通常是：

1. 合成数据工厂：最安全、最可控，适合常规功能。
2. 脱敏且最小化的样本：适合复杂边界数据，需要严格访问控制。
3. 契约化 seed 数据：适合固定参考状态和演示环境。

禁止把生产 PII、支付信息、身份证件或真实密钥直接复制到测试环境和测试报告。数据治理需要包含保留期限、删除策略、访问审计与泄漏扫描。

### Q20：如何处理测试环境不一致和配置漂移？

环境不一致常使“本地通过、CI 失败”变成长期问题。治理方式包括：

- 应用、依赖服务、浏览器和测试镜像使用可追溯版本。
- 将 URL、租户、特性开关和账户来源放入显式配置，不在测试里硬编码。
- 部署后先执行环境健康检查和契约 smoke test，再调度完整回归。
- 记录每次测试的 commit、环境版本、配置摘要和浏览器版本。
- 对临时环境使用基础设施即代码，销毁和重建而不是长期手工修改。

特性开关尤其需要治理：测试应声明需要的开关状态，并在 fixture 中设置和复原；不能依赖某个共享环境“刚好开着”的配置。

### Q21：前端、后端 API 和数据库应该如何配合来准备测试状态？

首选顺序通常是：**受支持的测试 API/领域 API > 消息或事件接口 > 受控数据库 fixture > 纯 UI 准备**。

UI 只用于验证用户旅程，使用 UI 大量构造前置数据会慢、脆弱且难以精确控制。直接写数据库则会绕过业务规则、缓存、事件和权限校验，必须谨慎，仅适合无法提供领域 API 的受控环境。

成熟团队应建设 test data API，例如创建租户、用户、商品、订单和特性开关，并确保幂等、审计、权限隔离和清理能力。它是测试平台的一部分，不是临时脚本集合。

---

## 六、网络控制与跨边界测试

### Q22：如何在 Playwright 中 Mock API？什么情况不应 Mock？

Playwright 可以用路由拦截请求，返回受控响应。它适合模拟第三方异常、罕见边界、前端加载/错误状态，以及消除不可控外部依赖。

```python
import json
from playwright.sync_api import Page, Route


def mock_product_api(route: Route) -> None:
    route.fulfill(
        status=200,
        content_type="application/json",
        body=json.dumps(
            {
                "items": [
                    {"id": "sku-1", "name": "Keyboard", "price": 99.0},
                ]
            }
        ),
    )


def test_catalog_renders_products(page: Page) -> None:
    page.route("**/api/products", mock_product_api)
    page.goto("/catalog")

    assert page.get_by_text("Keyboard").is_visible()
```

不应把所有 API 都 Mock。若关键流程始终 Mock，测试只能证明前端与模拟数据兼容，不能证明前后端集成可用。应保留一组真实依赖的少量关键 E2E，同时用契约测试和服务测试承担大部分组合覆盖。

### Q23：`route.fulfill()`、`route.continue_()` 和 `route.abort()` 分别何时使用？

- `route.fulfill()`：不访问真实服务，直接返回自定义响应。用于 mock、边界输入和错误状态。
- `route.continue_()`：放行请求，可修改 URL、headers、method 或 post data 后继续。用于注入关联 ID、验证请求改写或代理。
- `route.abort()`：主动取消请求。用于模拟网络故障、阻止广告/分析请求或验证离线降级。

路由规则必须在触发请求前注册，并在用例结束后由 Context 销毁或显式 `unroute` 清理。不要在全局 fixture 中无差别拦截 `**/*`，它会误伤静态资源、身份服务和应用更新请求，导致难以理解的失败。

### Q24：如何验证关键 API 请求，而不让 UI 测试过度耦合实现？

对真正关键的边界，例如支付提交、文件上传、权限变更，可以等待并检查请求/响应；只断言业务契约必需字段，不把每个 header 和内部字段都写死。

```python
from playwright.sync_api import Page


def test_submit_order_sends_expected_payload(page: Page) -> None:
    with page.expect_request(
        lambda request: request.url.endswith("/api/orders")
        and request.method == "POST"
    ) as request_info:
        page.get_by_role("button", name="Place order").click()

    payload = request_info.value.post_data_json
    assert payload["currency"] == "USD"
    assert payload["items"]
```

更完整的 API schema、错误码矩阵和版本兼容性，应通过 API 测试或消费者驱动契约测试验证。UI 测试保留少数高价值的跨边界检查即可。

### Q25：如何测试下载、上传和浏览器权限？

文件行为需要使用浏览器事件，而不是猜测下载完成时间。权限在 Context 创建时显式声明，避免依赖本机残留状态。

```python
from pathlib import Path
from playwright.sync_api import Page


def test_export_download(page: Page, tmp_path: Path) -> None:
    with page.expect_download() as download_info:
        page.get_by_role("button", name="Export CSV").click()

    download = download_info.value
    target_path = tmp_path / "orders.csv"
    download.save_as(target_path)

    assert target_path.read_text(encoding="utf-8").startswith("order_id,")
```

上传可用 `locator.set_input_files()`，不必通过操作系统文件选择器。地理位置、摄像头、麦克风等权限用 `browser.new_context(permissions=[...], geolocation=...)` 预先配置，并将权限场景放在专门测试中。

---

## 七、并行、浏览器覆盖与性能

### Q26：如何让测试并行执行，同时避免互相干扰？

并行的前提不是加 worker，而是隔离：

- 浏览器层：每例独立 Context；需要时独立 Page。
- 数据层：唯一 ID、独立租户/命名空间、无共享账户余额或库存。
- 时间层：可控时钟或明确的轮询结果，避免依赖执行顺序。
- 环境层：避免在同一环境中并发修改全局特性开关和系统级配置。
- 外部层：使用 sandbox、stub 或按容量限流。

使用 `pytest-xdist` 可以通过 `pytest -n auto` 分发用例，但不能在不了解环境容量时盲目提高并发。应先压测被测环境，观察数据库连接池、队列积压、CPU、第三方配额与失败率，给 CI 设置合理 worker 上限。

### Q27：如何估算总执行时间和并发收益？

若单测总时长为 $T$，并发 worker 数为 $W$，理想下界约为：

$$
T_{ideal} \approx \frac{T}{W}
$$

实际时间还受最长分片、浏览器启动、数据准备、CI 排队、共享资源争用和重试影响，可近似表示为：

$$
T_{actual} \approx \max(T_{shard}) + T_{setup} + T_{contention} + T_{retry}
$$

所以优化顺序通常是：先删除冗余 E2E、缩短数据准备、修复慢接口和不稳定用例，再增加并发。只增加 worker 往往将资源争用和 flaky 放大。

### Q28：浏览器矩阵如何设计？

不要在每个 PR 对每个浏览器跑完整回归。根据用户占比、业务风险、监管要求和浏览器差异设计矩阵，例如：

|触发场景|Chromium|Firefox|WebKit|移动视口|
|---|---:|---:|---:|---:|
|PR 冒烟|全量|核心路径|核心路径|核心路径|
|主干回归|全量|全量|全量|关键路径|
|夜间回归|全量|全量|全量|扩展设备矩阵|
|发布前|全量|高风险路径|高风险路径|高风险路径|

Playwright 的 WebKit 用于覆盖引擎差异，但不应宣称它等价于所有真实 Safari 版本。若业务对 iOS Safari、企业受管浏览器或真实设备行为有严格要求，应补充真实设备云或实体设备验证。

### Q29：无头和有头模式应如何使用？

CI 默认无头模式，资源消耗更低、运行更稳定。调试本地失败时使用有头模式或 `PWDEBUG=1`，配合 Inspector 和 Trace Viewer 检查动作、DOM 快照和网络。

不要因为“有头模式看起来更真实”而把 CI 长期改成有头。两者若存在差异，应记录浏览器版本、字体、GPU、窗口尺寸和系统依赖，构建可重现的容器镜像，再定位根因。

### Q30：如何做视觉回归测试？它的边界是什么？

视觉回归通过截图与基线图比较，适合捕捉 CSS、布局、主题和关键视觉状态的意外变化。但它对字体、反锯齿、时间、随机内容、动画和平台差异敏感。

稳定视觉测试需：固定浏览器与容器镜像、视口、字体、时区和主题；冻结时间和随机数；等待字体和数据加载完成；屏蔽动态区域；采用合理像素阈值；将基线更新视为需要审查的变更。

视觉测试不能替代语义、可访问性、业务规则和交互验证。它是质量信号的一层，而非 UI 正确性的唯一证明。

---

## 八、失败诊断与可观测性

### Q31：Trace、截图、视频和日志各自有什么作用？

- **Trace**：通常最有价值，包含操作时间线、DOM 快照、网络请求、控制台信息和截图，可在 Trace Viewer 中回放失败上下文。
- **截图**：快速展示最终页面状态，适合报告摘要。
- **视频**：适合观察动画、页面跳转和偶发时序问题，但存储成本较高。
- **应用日志/分布式追踪**：说明服务端为何返回错误、是否命中正确租户、下游是否超时。

推荐在 CI 中对失败保留 trace、截图和必要日志；成功测试只按抽样或短期保留，以控制成本。所有工件应以运行 ID、测试 ID、commit、浏览器和环境版本关联。

示例命令：

```bash
pytest tests/e2e \
  --browser chromium \
  --tracing retain-on-failure \
  --video retain-on-failure \
  --screenshot only-on-failure
```

### Q32：如何给一次失败做归因？

不要仅凭“测试失败”给测试团队背锅。可将失败初步分为：

1. **产品回归**：断言揭示真实功能、UI、接口或权限错误。
2. **测试缺陷**：定位错误、等待条件错误、数据假设过期。
3. **环境故障**：部署不完整、依赖服务不可用、配置漂移、容量不足。
4. **基础设施故障**：runner、网络、DNS、浏览器安装、工件上传失败。
5. **外部依赖故障**：第三方 sandbox、配额、网络波动。

归因需要证据：浏览器 Trace、请求 ID、服务端日志、部署版本、环境健康记录和重试结果。重试通过不是“没问题”，而是“不稳定证据”；应创建可跟踪的 flaky 事件。

### Q33：你会关注哪些自动化测试指标？

建议至少按套件、浏览器、环境和业务域拆分：

- 首次通过率（first-pass rate）。
- flaky rate：首次失败但重试通过的比例。
- 真缺陷发现数与漏测复盘。
- 平均、$p95$ 用例时长和总反馈时间。
- 重试次数、超时次数和失败类别。
- 被隔离、跳过、长期失败用例数。
- PR 质量门禁阻塞时长和资源成本。

指标应推动行动。例如 flaky rate 上升，按根因 Pareto 排序修复；总时长上升，检查慢用例、数据准备和 worker 争用。不要只追求“自动化用例数量”或“代码覆盖率”这类容易被刷高的数字。

### Q34：如何调试“本地通过、CI 失败”？

按可复现性逐层缩小：

1. 下载 CI 的 Trace、截图、视频、浏览器控制台和网络失败记录。
2. 确认同一个 commit、浏览器版本、依赖版本、环境配置和测试数据。
3. 在与 CI 相同的容器或镜像中单独重跑该用例。
4. 检查并发、时区、语言、字体、CPU 限制、网络策略与密钥权限。
5. 将错误归为产品、测试、环境或基础设施，再做最小修复。

“本地多跑几次”不是充分诊断。必须把 CI 环境的差异转化为可观测、可复现的输入。

---

## 九、CI/CD 与质量门禁

### Q35：如何设计 Playwright 的 CI 流水线？

一个分层流水线示例：

1. **快速反馈**：格式化、静态检查、单元测试、API contract 测试。
2. **PR 冒烟**：Chromium 上关键用户路径，使用隔离数据和短时限。
3. **合并后回归**：多浏览器、多 worker 的相关领域回归。
4. **夜间或发布前**：完整浏览器矩阵、外部 sandbox、视觉与可访问性检查。
5. **部署后验证**：在目标环境运行少量只读或可回收的 synthetic smoke。

每一步需要明确失败策略。PR 冒烟通常阻塞合并；不稳定的第三方检查可以报告但不立刻阻塞，同时必须有 owner 和治理 SLA。将所有检查都设为阻塞，会造成团队绕过或忽视信号。

### Q36：为什么要使用官方或可追溯的 Playwright 容器镜像？

浏览器自动化依赖系统库、浏览器二进制、字体和驱动。开发机与 CI 的差异会引入不可复现错误。固定且可追溯的镜像能使 Python 依赖、Playwright 浏览器版本和 OS 库版本一起版本化。

镜像治理包括：

- 固定依赖版本或使用 lock file。
- 定期更新浏览器与安全补丁，并先在非阻塞作业验证。
- 缓存 Python 依赖和浏览器下载，但不缓存含密钥的状态文件。
- 用非 root 用户执行浏览器，遵循组织容器安全策略。

不要为了临时解决 sandbox 或权限问题随意关闭所有安全限制；应理解 CI runner 的容器、内核和权限模型后作最小配置。

### Q37：测试失败时重试策略怎么设计？

重试是降低瞬时基础设施噪声的缓冲，不是稳定性的替代品。建议：

- 只对可判定为瞬态的失败进行有限重试，通常最多一次。
- 首次失败立即保存完整工件；重试通过也要在报告中标记为 flaky。
- 业务断言失败、权限失败、schema 不兼容等确定性问题不应盲目重试。
- 以时间窗口统计 flaky rate，超过阈值则创建治理任务、隔离或下线该用例。
- 任何隔离/跳过都要有原因、负责人和过期时间。

高分回答应强调：重试改变的是“CI 是否立即红”，不能改变“质量风险是否存在”。

### Q38：如何处理密钥、账户和测试工件安全？

- 密钥仅从 CI Secret Manager 或受控身份系统注入，不写入仓库、日志、Trace 或截图。
- 使用测试专用账户、最小权限、短期凭证和定期轮换。
- 工件上传前评估是否会含 PII、access token、订单信息或客户内容；必要时脱敏或限制访问与保留期。
- 日志采用字段级脱敏，禁止打印 Authorization、Cookie、完整请求体和密码。
- 对生产 smoke 使用专门、可审计的合成账户，禁止做破坏性操作。

安全要求不是上线后再补的运维项，应是测试框架设计的一部分。

---

## 十、架构与场景设计题

### Q39：请设计一个电商系统的自动化测试架构

**答题要点：**

先划分风险：登录、商品搜索、购物车、优惠、库存、下单、支付、订单状态和退款。支付与库存是高风险跨服务边界，不能全部依赖 UI 回归。

```text
                 Pull Request
                       |
         +-------------+-------------+
         |                           |
   Unit / Component              API / Contract
         |                           |
         +-------------+-------------+
                       |
               E2E Smoke (Chromium)
                       |
                 Merge / Release
                       |
       Multi-browser E2E + External Sandbox
                       |
        Trace / Logs / Metrics / Defect Triage
```

- **单元与组件层**：价格计算、优惠叠加、库存规则、前端组件状态。
- **API/契约层**：商品、订单、库存、支付适配器的输入输出和错误码。
- **E2E 冒烟层**：注册或登录、加购、结算、模拟支付、订单确认。
- **真实集成层**：少量支付 sandbox、税务或物流集成验证，按时段运行。
- **数据平台**：通过 Test Data API 创建独立租户、商品和库存；每例使用唯一订单号。
- **可观测性**：每个 UI 操作和服务请求传递 correlation ID，失败工件链接到服务端 trace。

取舍要讲清楚：支付失败、重复回调、库存不足等大多数分支用 API/契约测试覆盖；E2E 只保留能证明关键用户旅程正确的组合，避免组合爆炸。

### Q40：一个包含 2,000 个 E2E 用例的套件需要 3 小时，如何在不降低可信度的情况下将 PR 反馈降到 15 分钟？

不要承诺“让 2,000 个测试全部在 15 分钟跑完”。应分阶段处理：

1. **度量基线**：按时长、失败率、业务域、重复覆盖、环境依赖和数据准备时间分析。
2. **建立 PR 风险选择器**：以代码所有权、服务依赖和关键路径为依据，选择相关 smoke；不能只按文件名简单匹配。
3. **下沉覆盖**：把纯校验、异常矩阵和边界组合迁到单元/API/契约层。
4. **并行与分片**：将剩余 E2E 按历史时长平衡分片，控制并发以免压垮环境。
5. **复用认证与高效数据准备**：用短期 storage state 和 Test Data API，减少 UI 前置步骤。
6. **清理低价值测试**：合并重复旅程，删除长期不可信或不再覆盖真实风险的用例。
7. **分层门禁**：PR 运行 15 分钟高信号套件；完整多浏览器回归在主干、夜间或发布前运行。

结果要用首次通过率、逃逸缺陷、PR 等待时间和测试总成本验证，不能只优化流水线表面时长。

### Q41：如何测试微前端或多个独立团队交付的 Web 平台？

核心是把“整站 E2E”降为最后一道薄层验证：

- 每个微前端团队对本域组件、路由、可访问性和 API 客户端负责。
- Shell 与子应用之间通过版本化契约定义路由、鉴权 token 传递、事件、共享组件和错误边界。
- 用契约测试和集成 sandbox 验证接口兼容；E2E 只覆盖跨域高价值旅程。
- 提供统一的 `data-testid`、可访问性、日志 correlation ID 和错误上报规范。
- 发布时记录组件版本矩阵，兼容性检查阻断明显不匹配组合。

如果每个团队都直接修改一个共享的全站测试仓库，冲突、等待和责任不清会迅速放大。平台团队应提供公共 fixture、报告协议和运行能力，而不是集中拥有所有业务测试。

### Q42：如何为多租户 SaaS 设计 E2E 测试？

需要显式覆盖租户隔离而不仅仅是功能可用：

- 每个测试使用独立租户，或在共享租户中使用严格 namespace。
- 验证用户不能通过 URL、搜索、API 或缓存读到其他租户资源。
- 将租户配置、套餐、角色、特性开关作为可组合 fixture。
- 测试数据清理以租户为边界，避免误删其他执行或人工数据。
- 用 API 和数据库审计验证服务端隔离，UI 只验证用户可见的一面。

安全测试尤其要关注 IDOR（不安全直接对象引用）、缓存 key 是否包含租户、异步任务是否携带租户上下文，以及导出、通知、搜索索引和分析系统是否隔离。

### Q43：如何测试异步最终一致性流程？

例如用户提交申请后，事件经队列、风控服务和人工审核，几分钟后才显示最终状态。错误做法是 UI 测试 `sleep(60)`。

更好的设计：

1. 用领域 API 提交请求，并捕获 correlation ID。
2. 在受控环境中替换慢速外部服务，或触发可控的工作流模拟器。
3. 用带上限的轮询等待最终领域状态，记录每次状态变化。
4. 用 UI 验证用户看到的最终结果和中间可解释状态。
5. 超时后输出消息轨迹、下游响应、correlation ID 和当前状态。

端到端测试验证少量完整链路；大量状态转换和失败组合由工作流/服务级测试完成。

### Q44：你如何推动团队从“脚本自动化”升级为“质量工程平台”？

建议给出可执行的演进路线：

1. 建立现状基线：时长、首次通过率、缺陷逃逸、维护人力和环境失败比例。
2. 定义测试分层、Locator、数据、重试、标记和工件的工程标准。
3. 交付公共能力：fixture、认证、数据工厂、报告、Trace 聚合、运行模板。
4. 选择一个高风险业务域试点，验证速度和稳定性改善。
5. 为业务团队提供文档、示例、code review checklist 和可观测指标。
6. 将 flaky、跳过用例和环境债务纳入正常优先级管理。

关键是平台团队提供“自助能力和护栏”，业务团队仍对其领域风险和测试资产负责。集中团队代写所有脚本不可持续。

---

## 十一、现场 Coding 与追问清单

### Q45：现场实现一个登录测试，你会如何写？

先澄清成功定义：登录成功后显示什么？是否有 MFA？用户从何处来？是否允许用测试 API 创建用户？然后写最小、可读、稳定的版本。

```python
from playwright.sync_api import Page, expect


def test_registered_user_can_sign_in(page: Page) -> None:
    page.goto("/login")

    page.get_by_label("Email").fill("customer@example.test")
    page.get_by_label("Password").fill("test-password")
    page.get_by_role("button", name="Sign in").click()

    expect(page).to_have_url("https://app.example.test/dashboard")
    expect(page.get_by_role("heading", name="Dashboard")).to_be_visible()
```

**加分说明：**

- 测试账户不应硬编码真实密码，真实实现从 secret manager 或 fixture 注入。
- 页面 URL 和文本应根据实际 `base_url`、国际化策略和产品契约调整。
- 登录失败、锁定、MFA、会话过期和登出应由独立场景覆盖。
- 非认证测试不应重复走登录 UI，而应加载短期存储状态。

### Q46：现场如何写一个等待异步结果的 helper？

Helper 不应仅仅包一层 `sleep`。它必须有上限、错误信息和清晰的轮询间隔；在能使用 Playwright `expect` 时优先用 `expect`，在跨 UI/API 状态时才使用受控轮询。

```python
from collections.abc import Callable
from time import monotonic, sleep
from typing import TypeVar


Result = TypeVar("Result")


def wait_until(
    read_state: Callable[[], Result],
    is_complete: Callable[[Result], bool],
    timeout_seconds: float = 10.0,
    interval_seconds: float = 0.25,
) -> Result:
    deadline = monotonic() + timeout_seconds
    last_state: Result | None = None

    while monotonic() < deadline:
        last_state = read_state()
        if is_complete(last_state):
            return last_state
        sleep(interval_seconds)

    raise TimeoutError(
        f"condition was not satisfied within {timeout_seconds}s; "
        f"last state: {last_state!r}"
    )
```

**追问：** 真实项目中可将每次轮询的状态与 correlation ID 写入日志，并对 429/5xx 设置与业务语义一致的退避策略。不要对不可重试的 4xx 错误持续轮询。

### Q47：如何实现一个带失败工件的 pytest fixture？

`pytest-playwright` 已能生成常见工件。若需要补充领域截图，可在测试失败时生成；关键点是不会掩盖原始异常，且路径能被 CI 收集。

```python
from pathlib import Path

import pytest
from playwright.sync_api import Page


@pytest.fixture(autouse=True)
def capture_failure_screenshot(
    request: pytest.FixtureRequest,
    page: Page,
    tmp_path: Path,
):
    yield

    report = getattr(request.node, "rep_call", None)
    if report is not None and report.failed:
        page.screenshot(path=tmp_path / "failure.png", full_page=True)


@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item: pytest.Item, call: pytest.CallInfo[object]):
    outcome = yield
    report = outcome.get_result()
    if report.when == "call":
        setattr(item, "rep_call", report)
```

面试时可补充：fixture setup 失败时可能没有可用 Page，上传逻辑也不应因截图失败覆盖原异常；生产实现应配合统一工件目录、Trace 和报告链接。

### Q48：面对一段不稳定脚本，你会如何重构？

假设原始代码如下：

```python
page.click(".submit")
page.wait_for_timeout(3000)
assert page.locator(".message").inner_text() == "Success"
```

问题包括：定位依赖样式类、固定等待、`inner_text()` 立即读取导致竞态、断言不表达业务元素。重构为：

```python
from playwright.sync_api import Page, expect


def submit_profile(page: Page) -> None:
    page.get_by_role("button", name="Save profile").click()
    expect(page.get_by_role("status")).to_have_text("Profile saved")
```

若保存是异步工作流，则进一步等待明确的状态字段或受控 API 查询，而不是继续增大超时。重构后应在并行、多次重复和慢网络模拟下验证稳定性。

### Q49：现场面试中还应主动说明哪些边界？

- 是否允许修改被测应用，以增加 `data-testid` 或可访问性语义？
- 页面是 SSR、传统多页应用还是 SPA？是否有 iframe、Shadow DOM、新窗口？
- 后端是否存在异步队列、缓存、最终一致性和第三方回调？
- 测试数据是否可通过 API 创建和清理？是否有 PII/合规约束？
- 需要支持哪些真实浏览器、设备、语言、时区和无障碍标准？
- PR、主干和发布流水线各自的时长预算和阻塞策略是什么？
- 如何收集失败时的浏览器和服务端证据？

这些问题表明候选人理解自动化测试是完整系统工程，而不是孤立脚本。

---

## 十二、面试前自检

### 技术自检

- 能解释 Browser、Context、Page、Locator 的关系与生命周期。
- 能用 role、label、test id 写出稳定定位，并说明各自取舍。
- 能解释 auto-waiting 的边界，避免 `sleep` 和盲目增大 timeout。
- 能用 `expect` 编写 Web-first assertion，并区分 UI、API、契约测试职责。
- 能处理 iframe、popup、下载、上传、权限和网络 mock。
- 能使用 fixture 隔离 Context、会话、数据和清理逻辑。
- 能设计并行安全的测试数据、认证与特性开关方案。
- 能说明 Trace、截图、视频、日志如何支持失败归因。

### 架构自检

- 能为具体业务给出测试金字塔或测试奖杯的分层理由，而不是套图。
- 能量化 PR 反馈时限、浏览器矩阵、并发上限和 flaky 治理目标。
- 能说明哪些外部依赖应 mock，哪些必须在 sandbox 或真实集成环境验证。
- 能在可靠性、成本、运行时长与覆盖率之间给出清晰取舍。
- 能把环境、数据、密钥、工件与发布流程视为测试架构的一部分。
- 能用指标推动改进，而不是只汇报“自动化用例数”。

### 回答模板

> 我会先按风险把覆盖下沉到单元、API 和契约层，只保留少量关键旅程作为 Playwright E2E。每个用例使用独立 Browser Context 和唯一测试数据，通过语义化 Locator 与 Web-first assertion 消除大多数时序问题。CI 中按 PR 冒烟、主干回归和发布前矩阵分层执行；失败时保留 Trace、截图、视频和关联日志。对于首次失败后重试通过的用例，我会将其计为 flaky 并按根因持续治理，而不是把重试当作解决方案。

这段模板只能作为组织思路的骨架。正式面试必须结合具体业务、数据规模、合规约束和团队现状给出可执行的取舍与例子。
