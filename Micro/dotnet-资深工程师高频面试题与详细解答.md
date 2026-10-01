# .NET 资深工程师高频面试题与详细解答

> 适用岗位：5 年以上 .NET 详细设计与开发经验，具备 C#/.NET、自动化测试、SQL、REST API/WebService、Vite、MVC、WinForms、WPF 和日文设计书协作能力。
>
> 使用方式：不要背诵整段答案。每题先用一句结论说明判断，再解释原理、工程做法和自己做过的案例；当面试官追问时，再展开风险、取舍和代码细节。

## 目录

1. [面试作答框架与能力映射](#一面试作答框架与能力映射)
2. [C# 与 .NET](#二c-与-net)
3. [测试与质量保障](#三测试与质量保障)
4. [数据库与 SQL](#四数据库与-sql)
5. [REST API 与 WebService](#五rest-api-与-webservice)
6. [ASP.NET MVC、前端与 Vite](#六aspnet-mvc前端与-vite)
7. [WinForms 与 WPF](#七winforms-与-wpf)
8. [详细设计书与日语协作](#八详细设计书与日语协作)
9. [综合场景题](#九综合场景题)
10. [面试前自检清单](#十面试前自检清单)

---

## 一、面试作答框架与能力映射

### 1. 如何在自我介绍中匹配这份 JD？

**参考回答：**

我有 $N$ 年 C#/.NET 的详细设计和开发经验，主要负责过业务功能拆分、数据库设计、API/桌面端交付和测试。我会把业务规则放在可测试的服务或领域层，UI 层只处理展示与交互；后端通过参数化 SQL 或 ORM 访问数据，接口按 REST 约定设计并覆盖鉴权、异常、日志和集成测试。桌面端方面，我做过 WinForms/WPF，其中 WPF 采用 MVVM、异步命令和数据绑定来控制复杂界面的可维护性。与日方协作时，我可以按模板编写或修订基本设计书、详细设计书、测试规格书，并在评审中确认异常处理、边界条件和验收标准。

**面试要点：**

- 不要只罗列技术名词。每一个技术至少配一个“做了什么、为什么这样做、结果怎样”的案例。
- “熟悉测试”应能说出单元测试、集成测试、接口测试分别测什么，而不是只说“写过 NUnit”。
- 日语要求通常更关注工作沟通与文档准确性。不要夸大口语水平；可说明能处理的会议、邮件和文档范围。

### 2. 面试官问“你负责过详细设计吗”，应该回答哪些内容？

**参考回答：**

详细设计不是把画面和字段列出来，而是把实现前仍存在歧义的部分收敛掉。我通常会写清楚：画面或 API 的输入输出、字段约束、处理流程、业务校验、状态迁移、权限、异常消息、事务边界、表和索引变更、外部接口、日志点以及测试观点评审。实现前会和业务方、测试人员确认正常、异常、边界三类场景；实现后可由设计书映射到测试用例和代码评审项。

**高质量补充：**

当需求是“允许修改订单”时，设计书必须继续回答：什么状态可改？谁可改？并发修改如何处理？修改后是否重算金额？是否保留审计记录？失败时是否回滚？这些问题没有写清，开发人员各自理解就会造成返工。

---

## 二、C# 与 .NET

### 3. `async`/`await` 的作用是什么？它是否一定会创建新线程？

**结论：**`async`/`await` 的核心是以非阻塞方式组织异步操作；`await` 不等于创建线程。

当执行到尚未完成的 `await` 时，当前方法把后续逻辑注册为 continuation 并将控制权返回给调用方，因此 ASP.NET 可释放请求线程，WPF/WinForms 可继续响应 UI。文件、网络和数据库驱动支持的异步 I/O 通常不需要为等待阶段占用一个线程。CPU 密集型计算才可能使用 `Task.Run` 转移到线程池，但不能把它当作所有异步问题的通用解法。

```csharp
public sealed class CustomerService(HttpClient httpClient)
{
    public async Task<CustomerDto> GetAsync(Guid id, CancellationToken cancellationToken)
    {
        using var response = await httpClient.GetAsync(
            $"api/customers/{id}",
            cancellationToken);

        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<CustomerDto>(cancellationToken: cancellationToken)
            ?? throw new InvalidOperationException("Response body was empty.");
    }
}
```

**常见追问：**

- 不要在 ASP.NET Core 代码中使用 `.Result` 或 `.Wait()`；它会阻塞线程，降低吞吐量，在有同步上下文的 UI/旧 ASP.NET 中还可能死锁。
- 异步调用应一路向上传播 `CancellationToken`，让取消、超时和应用关闭能及时终止工作。
- 库代码通常可使用 `ConfigureAwait(false)`，但 ASP.NET Core 默认没有请求同步上下文，不能把它说成 ASP.NET Core 的性能必需项。

### 4. `Task`、线程池线程和 `Thread` 有什么区别？

`Task` 是一个异步操作的抽象，表示结果、异常和取消状态，不等同于线程。线程池负责复用线程来执行短时间的 CPU 工作或异步完成后的 continuation；`Thread` 是操作系统线程，创建成本高，通常只用于需要专属、长生命周期线程的特殊场景。

**工程选择：**

| 场景 | 推荐做法 | 原因 |
| --- | --- | --- |
| HTTP、数据库、文件 I/O | 使用库提供的异步 API | 等待期间不占用工作线程 |
| 有限的 CPU 密集计算 | `Task.Run`，设置并发上限 | 避免阻塞 UI 或请求线程 |
| 后台循环消费任务 | `BackgroundService` / 受控队列 | 有生命周期、日志和取消管理 |
| 需要专属线程的 SDK | 显式 `Thread`，明确关闭策略 | 仅在框架无法满足时使用 |

### 5. 解释依赖注入（DI）及 `Singleton`、`Scoped`、`Transient` 的差异。

DI 的目的不是“少写 `new`”，而是让业务类依赖抽象、由组合根决定具体实现，从而可替换、可测试并统一管理生命周期。

- `Singleton`：应用进程内一个实例。适合无状态、线程安全的服务或缓存；不能保存单个用户或请求状态。
- `Scoped`：一个 HTTP 请求或显式作用域一个实例。典型例子是 EF Core `DbContext` 和事务相关服务。
- `Transient`：每次解析都新建。适合轻量、无状态对象；频繁创建重资源对象会造成额外开销。

**高频陷阱：** 单例不能直接依赖 scoped 服务，否则会把短生命周期对象错误地延长到全局。需要时应使用 `IServiceScopeFactory` 在操作范围内创建作用域，或调整服务职责和生命周期。

### 6. `IEnumerable<T>`、`IQueryable<T>`、`ICollection<T>` 如何选择？

- `IEnumerable<T>`：内存中的可枚举序列，LINQ 由 .NET 执行；适合已经取回的数据。
- `IQueryable<T>`：表达式树可被数据库提供方翻译为 SQL；适合在查询尚未执行前继续组合筛选、排序和投影。
- `ICollection<T>`：支持计数和增删的集合接口，常用于实体导航属性或需要写操作的集合。

不要把 `IQueryable` 从数据访问层暴露到 Controller 或 UI。这样调用方可以无意中拼出低效或不兼容的查询，也让数据访问边界失控。数据访问层应接收明确的筛选条件，返回 DTO、分页结果或受控的读取模型。

```csharp
public async Task<PagedResult<OrderSummary>> SearchAsync(
    OrderSearchCriteria criteria,
    CancellationToken cancellationToken)
{
    var query = dbContext.Orders.AsNoTracking()
        .Where(order => order.CreatedAt >= criteria.From && order.CreatedAt < criteria.To);

    if (!string.IsNullOrWhiteSpace(criteria.CustomerName))
    {
        query = query.Where(order => order.Customer.Name.Contains(criteria.CustomerName));
    }

    var totalCount = await query.CountAsync(cancellationToken);
    var items = await query.OrderByDescending(order => order.CreatedAt)
        .Skip(criteria.Offset)
        .Take(criteria.Limit)
        .Select(order => new OrderSummary(order.Id, order.OrderNo, order.TotalAmount))
        .ToListAsync(cancellationToken);

    return new PagedResult<OrderSummary>(items, totalCount);
}
```

### 7. .NET 的 GC 和 `IDisposable` 各解决什么问题？

GC 管理托管堆中对象的内存回收，但不保证对象何时被回收。数据库连接、文件句柄、Socket、GDI 资源等属于稀缺的非托管或外部资源，必须确定性释放，因此应实现或使用 `IDisposable`/`IAsyncDisposable`。

```csharp
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync(cancellationToken);
// 使用 connection；作用域结束时归还连接池。
```

不要手动调用 `GC.Collect()` 试图“优化”内存，除非经过性能诊断并明确存在极特殊场景。更常见的问题是事件订阅、缓存、静态集合或未释放资源造成对象仍被引用。对于拥有资源的类型，调用方应遵循所有权约定，使用 `using` 或由 DI 容器在作用域结束时释放。

### 8. 如何设计异常处理与日志？

**参考回答：**

异常只应在能够恢复、补充上下文或转换为边界契约的位置捕获。业务层可以抛出有语义的领域/应用异常；API 边界通过全局异常处理中间件统一转换为问题详情（Problem Details），避免每个 Controller 重复 `try/catch`。日志应记录事件、关联 ID、用户或对象标识、异常堆栈和耗时，但绝不能记录密码、令牌、完整身份证号等敏感数据。

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        logger.LogError(exception, "Unhandled exception. TraceId: {TraceId}", context.TraceIdentifier);
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "An unexpected error occurred.",
            Extensions = { ["traceId"] = context.TraceIdentifier }
        });
    });
});
```

客户端看到的信息应可理解但不泄露内部 SQL、堆栈或基础设施细节；运维人员则根据 `traceId` 在日志系统中定位完整原因。

### 9. 多线程下如何保护共享状态？

先问能否避免共享可变状态。例如按请求作用域保存状态、使用不可变对象、通过队列串行化写入，往往比加锁可靠。确实共享时，再根据语义选择：短临界区可用 `lock`；异步路径用 `SemaphoreSlim.WaitAsync`；简单计数用 `Interlocked`；跨进程一致性需要数据库约束、乐观并发或分布式锁，而不是进程内 `lock`。

**常见错误：** 在 `lock` 中执行网络/数据库调用；在异步方法中跨 `await` 持有普通锁；以为 `ConcurrentDictionary` 能让“先查再写”的复合业务操作自动原子化。

---

## 三、测试与质量保障

### 10. 单元测试、集成测试、端到端测试各测什么？

| 类型 | 被测范围 | 典型目标 | 特点 |
| --- | --- | --- | --- |
| 单元测试 | 单个业务类或纯函数 | 规则、边界、异常分支 | 快、稳定、数量最多 |
| 集成测试 | API + DB/消息/文件等真实或近真实依赖 | 映射、事务、中间件、SQL | 较慢，验证关键集成 |
| 端到端测试 | 浏览器/桌面 UI 到后端 | 用户关键流程 | 最贴近真实，数量应少 |

**参考回答：** 我不会用 UI 测试替代全部测试。金额计算、状态迁移、权限判断这类规则优先单元测试；Repository、迁移和接口契约用集成测试；登录、创建订单、审批等关键用户路径保留少量端到端测试。这样既能快速定位问题，也能覆盖真实集成风险。

### 11. Mock、Stub、Fake 有何区别？什么时候不该 Mock？

- **Stub**：返回预设数据，让测试走到目标分支。
- **Mock**：验证与依赖的交互，例如“成功保存后只发送一次通知”。
- **Fake**：可运行的轻量实现，例如内存实现的文件存储或测试消息总线。

不要为了测试而 Mock 自己的所有内部类。过度验证实现细节会使重构后测试大量失败、却没有行为回归。优先测试可观察结果：返回值、状态变更、发出的领域事件。对于 EF Core 查询、SQL 方言、序列化和认证中间件，Mock 往往不能代表真实行为，应使用测试数据库或 `WebApplicationFactory` 做集成测试。

```csharp
[Fact]
public async Task ApproveAsync_rejects_an_expired_request()
{
    var clock = new FakeClock(new DateTimeOffset(2026, 9, 30, 0, 0, 0, TimeSpan.Zero));
    var service = new ApprovalService(clock, repository, notifier);
    var request = ApprovalRequest.Create("APR-001", clock.UtcNow.AddDays(-1));

    var exception = await Assert.ThrowsAsync<BusinessRuleException>(
        () => service.ApproveAsync(request.Id, CancellationToken.None));

    Assert.Equal("Approval request has expired.", exception.Message);
    notifier.DidNotReceive().SendApproved(Arg.Any<ApprovalRequest>());
}
```

示例中的测试同时验证了业务结果和关键副作用未发生。实际项目应统一测试框架和 Mock 库，不必拘泥于示例中的具体库。

### 12. 如何为 REST API 编写集成测试？

**参考回答：** 集成测试启动接近生产配置的 Web Host，通过 HTTP 调用真实路由、模型绑定、过滤器和异常处理中间件；外部系统替换为可控测试实现，数据库使用隔离实例或容器。每个测试自己准备数据并清理，不能依赖执行顺序或共用脏数据。

重点覆盖：认证与授权、成功和失败状态码、输入校验、分页排序、并发更新、异常契约、数据库事务回滚，以及 OpenAPI/JSON 序列化的字段兼容性。

```csharp
[Fact]
public async Task Post_orders_returns_201_and_location()
{
    using var response = await client.PostAsJsonAsync("/api/orders", new
    {
        customerId = seededCustomerId,
        lines = new[] { new { productId = seededProductId, quantity = 2 } }
    });

    Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    Assert.NotNull(response.Headers.Location);
}
```

### 13. 如何处理测试不稳定（flaky test）？

先将失败分类，而不是简单重跑掩盖问题：时间/时区、并发竞争、共享数据、网络依赖、随机数、异步未等待、动画或 UI 定位器变化，都是常见根因。测试中应注入时钟和随机数源，显式等待业务状态而非 `Thread.Sleep`，隔离数据并为异步操作设置合理超时。确认真实产品缺陷时，先修产品；确认测试环境不稳定时，修环境和测试本身，并记录原因。

质量指标不只看覆盖率。覆盖率只能说明代码被执行过，不能证明断言有效。更应关注关键规则是否有断言、失败是否易定位、主干构建是否稳定、缺陷是否能通过回归测试复现。

---

## 四、数据库与 SQL

### 14. 请写出“每个客户最近一笔已完成订单”的 SQL，并说明注意点。

```sql
WITH RankedOrders AS (
    SELECT
        o.customer_id,
        o.order_no,
        o.completed_at,
        o.total_amount,
        ROW_NUMBER() OVER (
            PARTITION BY o.customer_id
            ORDER BY o.completed_at DESC, o.id DESC
        ) AS row_number
    FROM orders AS o
    WHERE o.status = 'Completed'
)
SELECT
    customer_id,
    order_no,
    completed_at,
    total_amount
FROM RankedOrders
WHERE row_number = 1;
```

**说明：** `ROW_NUMBER()` 能保证每个客户只返回一条记录；当完成时间相同，使用 `id DESC` 作为稳定的第二排序条件。为此类查询评估复合索引，例如以筛选条件和分组、排序字段为基础建立 `(status, customer_id, completed_at DESC)`，并用实际执行计划验证，而不是机械添加索引。

### 15. `INNER JOIN`、`LEFT JOIN`、`WHERE` 与 `HAVING` 的常见区别是什么？

- `INNER JOIN` 只返回两边匹配的数据。
- `LEFT JOIN` 保留左表全部数据，右表未匹配字段为 `NULL`。
- `WHERE` 在分组前过滤行。
- `HAVING` 在 `GROUP BY` 聚合后过滤组。

**高频陷阱：**

```sql
-- 这会把 LEFT JOIN 实际变成 INNER JOIN，因为 WHERE 排除了 NULL。
SELECT c.id, o.id
FROM customers AS c
LEFT JOIN orders AS o ON o.customer_id = c.id
WHERE o.status = 'Completed';

-- 若需要保留没有完成订单的客户，把右表过滤条件放到 JOIN 条件中。
SELECT c.id, o.id
FROM customers AS c
LEFT JOIN orders AS o
    ON o.customer_id = c.id
   AND o.status = 'Completed';
```

### 16. 如何防止 SQL 注入？仅使用 ORM 是否足够？

所有外部输入都必须作为参数传递，不能用字符串拼接 SQL。ORM 的正常查询 API 通常会参数化，但 `FromSqlRaw`、动态排序列、原生 SQL、报表查询和存储过程仍可能引入注入风险。

```csharp
const string sql = """
    SELECT id, order_no, total_amount
    FROM orders
    WHERE customer_id = @customerId AND created_at >= @from;
    """;

await using var command = new NpgsqlCommand(sql, connection);
command.Parameters.AddWithValue("customerId", customerId);
command.Parameters.AddWithValue("from", from);
```

参数不能替代对象名（列名、表名、排序方向）。对于允许排序的接口，应把客户端字段映射为服务端白名单表达式，例如 `createdAt -> order.CreatedAt`，而不是把 `sort` 原样拼进 SQL。

### 17. 事务隔离级别如何选择？如何解释死锁？

事务要保证一组业务操作要么全部成功、要么全部失败；隔离级别是在一致性与并发之间选择。常用默认级别是 `Read Committed`，可避免脏读，但不天然避免不可重复读或幻读。需要更严格一致性时，可考虑乐观并发版本列、显式锁或更高隔离级别，但要评估锁竞争和吞吐。

死锁是多个事务互相等待对方持有的锁，数据库通常中止其中一个事务作为受害者。处理方法：

1. 所有业务路径按一致顺序访问表/行。
2. 缩短事务，避免在事务内调用 HTTP、等待用户输入或执行耗时计算。
3. 为查询建立合适索引，减少扫描和锁范围。
4. 对可安全重试的短暂数据库错误实施有限、带退避的重试；重试前确认操作幂等。

### 18. 遇到慢 SQL 时，你的排查步骤是什么？

先用监控或慢查询日志确定实际 SQL、参数、频率和耗时分位数，再看执行计划而不是凭直觉加索引。重点检查：全表扫描、预估行数与实际行数偏差、隐式转换、`SELECT *`、在索引列上套函数、缺失或冗余索引、大偏移量分页、N+1 查询和锁等待。

典型优化顺序是：修正查询条件/投影 -> 补充或调整索引 -> 更新统计信息和检查数据倾斜 -> 调整分页或预计算方案。修改后必须在代表性数据量上对比执行计划与耗时，并关注写入成本，因为每个索引都会增加写入和存储负担。

### 19. 如何实现安全的分页查询？

小数据量下可使用 `OFFSET ... FETCH`；页数很深时，数据库仍需跳过大量行，性能可能退化。高并发大表应考虑基于稳定排序键的 Keyset/Cursor 分页。

```sql
SELECT id, order_no, created_at, total_amount
FROM orders
WHERE created_at < @lastCreatedAt
   OR (created_at = @lastCreatedAt AND id < @lastId)
ORDER BY created_at DESC, id DESC
FETCH FIRST @pageSize ROWS ONLY;
```

接口需要限制 `pageSize` 上限，排序应稳定并由白名单控制。若产品必须展示准确总数，应意识到 `COUNT(*)` 对超大表可能昂贵；可按业务允许的精度采用缓存、估算或延迟加载策略。

---

## 五、REST API 与 WebService

### 20. REST API 应如何设计资源、HTTP 方法和状态码？

资源使用名词和复数路径，例如 `/api/orders`、`/api/orders/{id}`；HTTP 方法表达动作语义：`GET` 查询、`POST` 创建、`PUT` 全量替换、`PATCH` 局部更新、`DELETE` 删除或逻辑删除。对于状态迁移等有业务语义的命令，可设计为 `/api/orders/{id}/approval` 等明确子资源，避免伪装成不透明动词。

| 场景 | 常见状态码 |
| --- | --- |
| 查询成功 | `200 OK` |
| 创建成功 | `201 Created`，带 `Location` |
| 异步受理 | `202 Accepted` |
| 无返回体的成功 | `204 No Content` |
| 输入格式/字段无效 | `400 Bad Request` 或 `422 Unprocessable Content`，团队需统一 |
| 未认证/无权限 | `401 Unauthorized` / `403 Forbidden` |
| 资源不存在 | `404 Not Found` |
| 乐观并发冲突 | `409 Conflict` 或 `412 Precondition Failed` |

状态码不能代替错误契约。错误体应有稳定的错误码、可显示信息、字段错误和关联 ID，便于前端展示和运维排查。

### 21. 什么是幂等性？为什么创建接口可能也需要幂等？

同一请求执行一次或多次，资源最终状态相同，则该操作是幂等的。`GET`、`PUT`、`DELETE` 在语义上应当幂等，`POST` 默认不保证幂等。但支付、下单、消息发送等请求可能因网络超时被客户端重试，若服务端重复创建数据会造成严重问题。

解决方式是在客户端提供 `Idempotency-Key`，服务端在唯一约束保护下保存请求键、请求摘要和第一次处理结果：相同键和相同摘要返回原结果；相同键但不同内容返回冲突。不要只在内存中记录键，因为多实例部署和进程重启会使保护失效。

### 22. 如何处理 API 版本、兼容性和弃用？

优先做可向后兼容的演进：新增可选字段、保留旧字段、消费者忽略未知字段。对于破坏性变更，如字段含义改变、必填规则改变、响应结构重构，应新建版本并有弃用计划。版本可放在路径、请求头或媒体类型，关键是全团队一致且文档、网关、测试同步。

每个公开接口应有 OpenAPI 定义、示例、错误码和兼容性说明。变更前检查消费者；变更后在监控中观察旧版本流量，再按公告期限下线。不要因为“内部 API”就跳过契约管理，内部调用一样会形成依赖。

### 23. SOAP WebService 与 REST 的差异和适用场景是什么？

SOAP 是基于 XML 信封和契约（WSDL）的消息协议，支持标准化的 WS-* 能力，在传统企业集成、严格契约和某些安全/事务要求中仍常见。REST 是围绕资源和 HTTP 语义的架构风格，常用 JSON，浏览器和移动端集成更轻量。

对接 WebService 时，重点不只是“生成代理类”：还要确认 WSDL 版本、编码、超时、认证、证书、重试是否会产生重复操作、故障（SOAP Fault）映射、日志脱敏和契约测试。新建面向现代前端的业务接口通常优先 REST；已有外部合作方要求 SOAP 时，建立适配层，不让 SOAP DTO 和异常模型渗透进核心业务层。

### 24. `HttpClient` 为什么不建议每次请求都 `new`？

频繁创建并销毁 `HttpClient` 可能导致连接无法有效复用、端口耗尽，并使 DNS 更新策略不可控。ASP.NET Core 中通常使用 `IHttpClientFactory` 配置命名或类型化客户端，统一配置基地址、超时、认证处理器、日志和韧性策略。

重试只适合短暂错误，并且要限制次数、使用指数退避和抖动。对非幂等写操作，未具备幂等键或去重能力前不能盲目自动重试。还应分别设置连接超时、整体请求超时和下游服务的超时预算，避免层层等待造成级联雪崩。

### 25. API 的身份认证和授权如何区分？

认证（Authentication）回答“你是谁”，例如 Cookie、JWT、OIDC 登录。授权（Authorization）回答“你能否对这个资源执行这个动作”，例如角色、权限、资源归属和业务状态。

不要只在前端隐藏按钮来做授权。后端必须基于当前身份、权限和资源关系进行校验。例如“订单审批员”角色不代表可以审批所有订单，还可能需要所属部门、金额阈值或当前状态满足条件。JWT 也不是永久可信：需要验证签名、签发者、受众、过期时间，并处理撤销/权限变更策略。

---

## 六、ASP.NET MVC、前端与 Vite

### 26. ASP.NET MVC 的一次请求如何流转？

请求会经过 Web 服务器和 ASP.NET Core 中间件管道，路由选择 Controller/Action，模型绑定把路由、查询字符串、表单或 JSON 转换为参数，模型验证产生验证结果，过滤器可处理授权、日志、异常等横切逻辑，Action 调用应用服务后返回 View、JSON、重定向或状态码。

**面试要点：**

- Controller 应薄：负责 HTTP 层参数和响应，不承载复杂业务规则。
- 使用 ViewModel/Request DTO，不要直接把数据库实体绑定到外部输入，避免 over-posting。
- 服务端始终校验 `ModelState` 和业务规则；客户端校验只是提升体验。
- 横切逻辑优先选择合适位置：全局异常在中间件，授权在认证/授权管道或过滤器，业务事务在应用服务边界。

### 27. MVC 中如何避免 over-posting（批量赋值）漏洞？

不要让 Action 直接接收可持久化实体，也不要把实体原样更新到数据库。定义只包含允许编辑字段的请求模型，再在服务层读取当前实体、显式赋值并执行权限与业务校验。

```csharp
public sealed record UpdateProfileRequest(string DisplayName, string PhoneNumber);

[HttpPut("me/profile")]
public async Task<IActionResult> UpdateProfile(
    UpdateProfileRequest request,
    CancellationToken cancellationToken)
{
    await profileService.UpdateAsync(User, request, cancellationToken);
    return NoContent();
}
```

例如 `IsAdmin`、`Balance`、`OwnerId`、审批状态等服务端控制字段不应出现在客户端可提交模型中。前端禁用字段并不能阻止攻击者手工构造请求。

### 28. Vite 的开发和生产流程是怎样的？

Vite 在开发期利用原生 ES Module 和按需转换提供快速 HMR；生产构建使用 Rollup 打包、代码分割、压缩和静态资源指纹。其价值不只是“启动快”，还在于明确区分开发服务器、构建产物和运行时配置。

**与 .NET API 协作的推荐做法：**

1. 开发期配置 Vite `server.proxy` 转发 `/api` 到后端，避免浏览器跨域复杂度。
2. 生产期由反向代理统一托管前端静态文件和 API，或将构建产物发布到 CDN/静态站点。
3. 将公开配置放在 `VITE_` 前缀环境变量中；它们会进入浏览器包，绝不能放数据库密码、API 私钥或客户端密钥。
4. API 基地址、认证回调地址等环境差异应在构建/部署流程中管理，不要散落在源代码中。

### 29. 浏览器前端调用 REST API 时，如何处理 CORS、认证与错误？

同源部署优先，复杂度最低。跨域时由 API 明确允许可信来源、方法和请求头，不能使用宽泛的 `AllowAnyOrigin` 配合凭据。Cookie 认证需要考虑 `SameSite`、`Secure`、CSRF 防护；Bearer Token 要防 XSS 泄露，避免把长期敏感令牌随意放入本地存储。

前端请求层应统一处理：超时/取消、登录过期、稳定错误码映射、网络异常、关联 ID 和用户友好提示。组件里不应各自硬编码 `fetch` 错误分支。对于表单，字段错误应映射到具体控件，未知错误显示通用消息并保留追踪 ID。

### 30. MVC 服务端渲染与 Vite SPA 如何选型？

不是技术先进程度之争，而是产品约束选择：

| 条件 | 更适合 MVC/Razor | 更适合 Vite SPA |
| --- | --- | --- |
| 页面流程简单、表单为主、SEO 有要求 | 是 | 视情况 |
| 交互复杂、长会话、多面板实时状态 | 需要更多前端脚本 | 是 |
| 团队以 .NET 为主、前端能力有限 | 是 | 需评估维护成本 |
| 前后端独立发布、多端复用 API | 可行 | 是 |

也可以采用渐进方案：基础页面使用 Razor，在局部复杂交互区域挂载 Vite 构建的组件。无论选择哪一种，领域规则和 API 契约都应由后端可靠保护。

---

## 七、WinForms 与 WPF

### 31. WinForms 和 WPF 的主要差异及选型依据是什么？

WinForms 基于传统控件模型，简单表单和已有系统改造的开发效率高，第三方控件和 Windows 生态成熟。WPF 基于 XAML、依赖属性、数据绑定、样式模板和更丰富的渲染能力，更适合复杂界面、可维护的 MVVM 架构与高度定制。

选型还要考虑团队现有资产、硬件/COM/ActiveX 控件兼容性、部署环境、长期维护人员和改造风险。不要为了“新”而将稳定 WinForms 系统整体重写为 WPF；可以先把业务层、API 和测试从 UI 中抽离，再评估局部迁移。

### 32. WinForms/WPF 为什么会出现“跨线程访问控件”异常？如何正确处理？

桌面 UI 控件通常只能由创建它的 UI 线程访问。耗时操作若在 UI 线程同步执行会导致窗口无响应；但后台线程直接修改控件又会违反线程亲和性。

正确方式是异步执行 I/O 或后台计算，完成后回到 UI 上下文更新绑定状态。WinForms 可使用 `Invoke`/`BeginInvoke`；WPF 可使用 `Dispatcher`。更推荐让 ViewModel 暴露状态和集合，通过绑定更新 UI，并避免后台线程直接操作控件。

```csharp
private async void RefreshButton_Click(object? sender, EventArgs e)
{
    refreshButton.Enabled = false;
    try
    {
        var items = await customerService.GetCustomersAsync(CancellationToken.None);
        customerBindingSource.DataSource = items;
    }
    catch (Exception exception)
    {
        logger.LogError(exception, "Failed to load customers.");
        MessageBox.Show("客户加载失败，请稍后重试。");
    }
    finally
    {
        refreshButton.Enabled = true;
    }
}
```

事件处理器可以是 `async void`，但真正业务逻辑应放在返回 `Task` 的服务方法中，才能被等待和测试。

### 33. 请解释 WPF 的 MVVM 及其收益。

MVVM 将 View、ViewModel、Model 分离：View 用 XAML 声明界面；ViewModel 持有可绑定状态、命令和校验；Model/服务负责业务数据和用例。ViewModel 不应依赖 `Window`、`MessageBox`、`DataGrid` 等具体 UI 类型。

收益是业务逻辑可以脱离 UI 进行单元测试，界面模板替换不影响规则，复杂页面的状态和交互有清晰归属。实际项目还应处理导航、对话框、通知等 UI 抽象，例如让 ViewModel 依赖 `IDialogService`，由 WPF 层提供实现。

```xml
<Button Content="保存"
        Command="{Binding SaveCommand}"
        IsEnabled="{Binding CanSave}" />
```

```csharp
public sealed class CustomerEditorViewModel : INotifyPropertyChanged
{
    public string Name
    {
        get => name;
        set
        {
            if (name == value) return;
            name = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Name)));
            SaveCommand.RaiseCanExecuteChanged();
        }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
}
```

### 34. `INotifyPropertyChanged`、`ObservableCollection<T>` 和依赖属性分别何时使用？

- ViewModel 普通属性变化：实现 `INotifyPropertyChanged`，让绑定目标刷新。
- 列表的新增、删除、移动：用 `ObservableCollection<T>`，让 ItemsControl 感知集合变更。
- 自定义 WPF 控件需要支持绑定、样式、模板、动画或属性系统：定义依赖属性。

`ObservableCollection<T>` 不会自动通知其中对象的属性变化；列表项自身仍需实现 `INotifyPropertyChanged`。也不要把依赖属性当作所有业务对象的默认属性模型，它属于 WPF UI 层。

### 35. WPF 绑定不更新时如何排查？

按以下顺序排查：

1. 查看输出窗口的 binding error，确认路径、`DataContext` 和类型。
2. 确认绑定模式：编辑输入通常需要 `Mode=TwoWay`。
3. 确认更新时机：`TextBox.Text` 默认失焦回写，需要实时校验时设置 `UpdateSourceTrigger=PropertyChanged`。
4. 确认源属性是否发出 `PropertyChanged`，集合是否发出集合变更通知。
5. 检查是否被更高优先级的本地值、样式或触发器覆盖。

对于复杂绑定，可在开发期使用 `PresentationTraceSources.TraceLevel=High`。不要用定时刷新或重新设置整个 `DataContext` 来“修复”通知问题，这会掩盖模型设计错误并影响性能。

### 36. 如何提高 WPF 大数据列表的性能？

首先分页或按需加载，避免一次把数十万条数据交给 UI。`DataGrid`/`ListView` 需要启用并保留 UI 虚拟化和回收，不能在外层再包一层导致虚拟化失效的 `ScrollViewer`。数据加载、筛选和导出使用异步操作；批量更新集合时避免逐项触发昂贵重绘；单元格模板保持轻量，避免复杂嵌套和每行创建大量转换器。

性能问题要通过工具定位：观察 UI 线程帧率、GC、布局/渲染耗时、网络/数据库耗时。不能仅凭“换成异步”就宣称完成优化，因为 UI 最终仍可能被大量数据绑定和布局计算拖慢。

### 37. 桌面应用如何实现输入校验、异常提示和可恢复操作？

校验分层：UI/VM 做必填和格式校验；应用服务做权限、重复性和跨字段校验；数据库约束保证最终完整性。WPF 可通过 `INotifyDataErrorInfo` 将字段错误绑定给控件；WinForms 可使用 `ErrorProvider`。保存失败时保留用户已输入的数据，明确展示可理解的信息；不可恢复的系统错误记录日志和关联 ID，不将堆栈直接显示给用户。

涉及外部设备、文件或网络时，命令应防止重复点击，支持取消或超时，并将部分成功与完全失败分别呈现。任何可能产生外部副作用的“重试”都必须先考虑幂等性。

---

## 八、详细设计书与日语协作

### 38. 一份可交付的详细设计书最少应包含哪些章节？

| 章节 | 应回答的问题 |
| --- | --- |
| 概要与范围 | 做什么、不做什么、关联需求是什么 |
| 前提与术语 | 角色、业务术语、外部依赖是否一致 |
| 画面/API/批处理规格 | 输入、输出、字段、权限、调用条件 |
| 处理流程 | 正常、异常、边界和状态变化如何流转 |
| 数据设计 | 表、字段、约束、索引、迁移与保留策略 |
| 外部接口 | 契约、认证、超时、重试、错误映射、版本 |
| 非功能 | 性能、安全、日志、监控、可用性、部署 |
| 测试观点评审 | 正常、异常、边界、权限、并发、回归场景 |

高质量设计书应能让另一位开发人员独立实现，并让测试人员据此设计用例。流程图、时序图和状态图用于消除歧义，但不能替代异常处理和字段规则的文字说明。

### 39. 如何把需求转换成可测试的设计与测试用例？

以“用户可提交报销单”为例，不能只写“点击提交后保存”。应拆为可验证规则：

- 金额必须大于零，且币种符合公司配置。
- 提交人只能提交自己的草稿。
- 草稿状态才能提交；已提交、已撤回状态不可重复提交。
- 提交后写入审计记录并发送通知；通知失败是否回滚由业务规则决定。
- 两人同时提交同一单据时，只有一人成功，另一人收到并发冲突。

随后每条规则至少有正常、异常或边界测试。设计评审时请业务方确认“应该怎样”，而不只是技术人员猜测“现在怎样”。

### 40. 与日方协作时，设计书和缺陷报告如何保证准确？

先统一术语、主语和状态名，避免同一对象多种叫法。文档使用短句和可验证表达：条件、动作、结果、错误处理分开写；数值、日期格式、时区、空值、全半角等明确化。评审后记录决定事项（決定事項）、未决事项（確認事項）和负责人/期限，而不是仅写“已讨论”。

**常用表达示例：**

| 目的 | 日语表达 | 使用提示 |
| --- | --- | --- |
| 请求确认 | `認識合わせのため、ご確認をお願いいたします。` | 说明是为了对齐理解 |
| 描述前提 | `本処理は、対象データが有効であることを前提とします。` | 前提必须可验证 |
| 说明异常 | `入力値が不正な場合、エラーメッセージを表示し、登録処理を中止します。` | 明确结果和后续动作 |
| 标注待确认 | `こちらは確認事項として管理します。` | 不把未确认事项伪装成结论 |
| 说明影响 | `本変更により、既存APIのレスポンス項目に影響があります。` | 同时说明影响范围和应对方案 |

N3 相当通常不足以支撑复杂商务谈判，因此真实且专业的回答是：可独立阅读常规设计书、按模板编写规格和邮件；涉及合同、复杂业务规则或语义有歧义的场合，会使用术语表、示例和评审记录确认，不凭猜测落地。

### 41. 设计评审中发现需求歧义，你会怎么处理？

先将歧义改写为可选方案和可验证问题，例如“取消后能否再次提交”改为：取消后状态是 `Draft` 还是 `Cancelled`？是否允许再次编辑？审计记录保留多久？对外通知是否撤回？然后说明每种选择对状态机、数据库、接口和测试的影响，请产品负责人作业务决定。

技术人员可以提出推荐方案，但不能以代码实现方便为由替代业务决策。最终决定要回写需求/设计书、接口契约和测试用例，避免会议口头结论丢失。

---

## 九、综合场景题

### 42. 设计一个“订单管理”功能：Vite 前端、.NET API、SQL 数据库和 WPF 内部客户端，你会如何分层？

**参考答案：**

1. **领域/应用层**：定义订单状态、金额计算、提交/审批/取消规则和用例接口；不依赖 MVC、WPF 或 EF Core。
2. **基础设施层**：实现数据库 Repository、外部支付/通知客户端、审计日志；数据库使用唯一约束和并发版本保护关键不变量。
3. **API 层**：使用请求/响应 DTO，负责认证授权、模型验证、错误契约和 OpenAPI；Controller 只调用应用服务。
4. **Vite 前端**：封装 API Client、认证、错误提示和路由；表单负责交互校验，不能替代后端规则。
5. **WPF 客户端**：采用 MVVM，通过同一 API 或共享应用服务访问能力；长耗时加载使用异步命令，列表分页/虚拟化。
6. **测试**：领域规则单元测试，API + DB 集成测试，关键 Web/WPF 流程端到端或验收测试；发布流水线运行静态检查和迁移验证。

**关键取舍：** Web 和 WPF 不应直接共享 UI ViewModel。可共享 DTO 或稳定的应用契约，但各自维护适合自身交互的状态模型。

### 43. 用户反馈“有时重复创建订单”，你如何排查和修复？

先收集订单号、用户、时间、请求关联 ID、客户端版本和请求日志，判断重复来自双击、前端重试、网关重试、消息重复投递还是并发竞态。然后检查创建 API 是否有幂等键、数据库是否有业务唯一约束、重试策略是否作用于非幂等请求、事务是否覆盖订单和审计/消息外盒记录。

修复不能只禁用前端按钮：前端可以防止常见双击，但服务端仍需通过幂等键和数据库唯一约束保证最终一致性。对于已经重复的数据，需要制定数据修复和审计方案，不能直接删除而破坏关联记录。

### 44. 将旧 WinForms 系统逐步现代化，你会怎么做？

先建立测试和可观测性，梳理高风险业务流程及现有数据库/接口依赖。第一阶段将业务规则和数据访问从窗体事件中提取为可测试服务，保留 WinForms UI；第二阶段为服务提供 REST API 或统一应用层；第三阶段按业务价值逐页迁移到 WPF 或 Web，不要求一次性重写。

每阶段都应可部署、可回滚、可验证。数据库 Schema 变更采用向后兼容迁移，旧新客户端并存期间不要立即删除旧字段/API。重写的最大风险通常不是 UI 技术，而是隐含在旧代码中的业务规则没有被发现和回归测试保护。

---

## 十、面试前自检清单

### 技术准备

- 能讲清一个从需求、详细设计、开发、测试到上线的完整案例，并量化自己的责任和结果。
- 能现场写出 `JOIN`、`GROUP BY`、窗口函数、分页、参数化查询，并解释索引和执行计划。
- 能解释 `async/await`、DI 生命周期、`IDisposable`、异常处理和并发控制的工程边界。
- 能说明单元/集成/端到端测试各自覆盖什么，并展示一个真实测试案例。
- 能给出 REST 的资源设计、状态码、错误格式、幂等、鉴权和 API 版本策略。
- 能解释 MVC 请求流、DTO 防 over-posting、Vite 开发/构建/环境变量与 CORS 安全边界。
- 能比较 WinForms/WPF，解释 MVVM、数据绑定、UI 线程和性能优化。

### 项目与沟通准备

- 准备一份脱敏后的设计书目录或用一个功能口述字段、流程、异常、事务和测试观点评审。
- 准备两到三个你曾发现并解决的问题：性能、并发、接口故障、质量缺陷或需求歧义均可。
- 用日语准备简短自我介绍、项目说明、确认问题和风险说明；不确定的表达以书面确认和术语表为准。
- 面试中主动说明假设条件。架构题没有唯一答案，清晰的约束、取舍和验证方式比堆砌框架名更有说服力。

---

## 附：高频追问速答

| 问题 | 一句话回答 |
| --- | --- |
| 为什么不把业务逻辑放 Controller？ | Controller 属于 HTTP 边界；规则放应用/领域层才能复用、测试并被 WPF/批处理共享。 |
| 为什么不直接在 UI 访问数据库？ | 会把连接、事务、安全和业务规则分散在客户端，难以升级、审计和测试。 |
| 为什么有单元测试仍需集成测试？ | Mock 无法代表真实 SQL、序列化、认证和中间件行为。 |
| 为什么前端校验不够？ | 请求可被绕过或篡改，服务端和数据库必须保护规则。 |
| 为什么不能到处重试？ | 重试会放大流量，且非幂等操作可能重复产生副作用。 |
| 为什么不只靠 ORM？ | ORM 提升开发效率，但索引、事务、执行计划和数据库约束仍决定正确性与性能。 |
| WPF 何时不适合？ | 需要跨平台、轻量页面或团队无法长期维护 XAML/MVVM 时，应评估 Web、Avalonia 或其他方案。 |
