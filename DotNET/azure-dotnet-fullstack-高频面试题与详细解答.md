# .NET + Azure 全栈工程师高频面试题与详细解答

> **适用岗位**：C# / .NET Core 后端开发、Azure 云开发、全栈工程师、技术顾问  
> **JD 覆盖范围**：C#、ASP.NET Core、Azure Functions、Blob Storage、App Service、Azure Data Factory（ADF）、React/Vue/Angular、前后端分离，以及独立问题解决能力。  
> **使用方式**：每题先用“核心回答”在 1 至 2 分钟内作答；面试官继续追问时，再展开“深入说明”和“实战要点”。示例中的资源名称、指标和项目背景须替换为个人真实经历，切勿把示例当作个人履历。

---

## 目录

1. [C# 与 .NET 基础](#一c-与-net-基础)
2. [ASP.NET Core 与 Web API](#二aspnet-core-与-web-api)
3. [Azure Functions](#三azure-functions)
4. [Azure Blob Storage](#四azure-blob-storage)
5. [Azure App Service](#五azure-app-service)
6. [Azure Data Factory（ADF）](#六azure-data-factoryadf)
7. [前后端分离与 React/Vue/Angular](#七前后端分离与-reactvueangular)
8. [综合架构与独立问题解决](#八综合架构与独立问题解决)
9. [面试收尾：反问与准备清单](#九面试收尾反问与准备清单)

---

## 一、C# 与 .NET 基础

### Q1：`async` / `await` 的本质是什么？它会创建新线程吗？

**核心回答：**  
`async` / `await` 是编译器提供的异步控制流语法。编译器会把 `async` 方法转换成状态机；当代码遇到尚未完成的可等待对象时，当前方法注册后续回调并把线程还给线程池或调用方，完成后从挂起点继续执行。它**不会天然创建新线程**。对于 HTTP、数据库、Blob 等 I/O 操作，通常不占用线程等待；CPU 密集型计算才需要谨慎使用 `Task.Run` 或独立的计算资源。

```csharp
public async Task<OrderDto> GetOrderAsync(Guid id, CancellationToken cancellationToken)
{
    using var response = await _httpClient.GetAsync(
        $"orders/{id}", cancellationToken);

    response.EnsureSuccessStatusCode();
    return (await response.Content.ReadFromJsonAsync<OrderDto>(
        cancellationToken: cancellationToken))!;
}
```

**深入说明：**

- `Task` 表示“未来可能完成的操作”，不等同于线程；一个 `Task` 可以由 I/O 完成端口、定时器或线程池任务完成。
- ASP.NET Core 没有传统 ASP.NET 的请求同步上下文，因此通常不需要也不应在业务代码中写 `ConfigureAwait(false)`；更不应在请求路径上使用 `.Result` 或 `.Wait()`，否则会阻塞线程池并降低吞吐量。
- 异步应“向上传播”：控制器、服务、仓储层都返回 `Task` / `Task<T>`。把异步调用包成同步调用，会抵消其价值。
- `Task.Run` 适合把短时间 CPU 任务转移到线程池，不适合包装本来就支持异步的 I/O，也不适合在 Web 请求中执行长时间任务。

**实战追问：如何处理取消？**  
在 API 中接收 `CancellationToken`，并将它传到 EF Core、`HttpClient`、Azure SDK 等下游操作。当客户端断开或请求超时时，系统可尽快停止无意义工作。取消是正常控制流，不应被记录为应用错误。

---

### Q2：值类型、引用类型、装箱和拆箱会如何影响性能？

**核心回答：**  
值类型直接保存数据，赋值通常复制值；引用类型变量保存对象引用，赋值复制引用。把值类型转换为 `object` 或接口会发生装箱，在托管堆上分配对象；从 `object` 转回值类型为拆箱，必须精确匹配原始类型。高频装箱会增加分配量和 GC 压力，应优先使用泛型集合和泛型 API。

```csharp
var numbers = new List<int>();       // 不装箱
numbers.Add(42);

object boxed = 42;                   // 装箱
var value = (int)boxed;              // 拆箱
```

**深入说明：**

- `struct` 适合小型、不可变、具有值语义的数据，例如 `DateTime`、`Guid`、`decimal`。大型或可变 struct 的复制成本高，不应仅为了“放到栈上”而滥用。
- `string` 是引用类型，但不可变；修改字符串会得到新对象。频繁拼接使用 `StringBuilder`，但普通少量插值不需要过度优化。
- 使用 `IEnumerable<T>`、`List<T>`、泛型委托等可以避免非泛型 API 中的装箱。
- 性能分析不能只靠猜测。应通过 Application Insights、`dotnet-counters`、`dotnet-trace` 或基准测试确认热点，再优化。

---

### Q3：依赖注入的 `Transient`、`Scoped` 和 `Singleton` 如何选择？最常见的生命周期错误是什么？

**核心回答：**

| 生命周期 | 创建时机 | 典型用途 | 注意事项 |
|---|---|---|---|
| `Transient` | 每次解析创建 | 无状态、轻量服务 | 频繁创建成本不可忽略 |
| `Scoped` | 每个 HTTP 请求一个实例 | `DbContext`、请求上下文、业务服务 | 不可被单例直接持有 |
| `Singleton` | 进程生命周期内共享 | 无状态客户端、缓存、配置读取器 | 必须线程安全，不能持有请求状态 |

最常见错误是把 `Scoped` 服务注入 `Singleton`，形成 captive dependency：单例长期持有一次请求的对象，导致释放时机、线程安全和数据隔离错误。

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)); // 默认 Scoped
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddSingleton<TimeProvider>(TimeProvider.System);
```

**深入说明：**

- `HttpClient` 不应每次 `new`。使用 `AddHttpClient`，由 `IHttpClientFactory` 管理连接生命周期，避免端口耗尽和 DNS 变更失效。
- 如果单例后台服务确实需要使用 scoped 服务，应注入 `IServiceScopeFactory`，在每次独立工作单元中创建并释放 scope。
- `DbContext` 不是线程安全对象，不可跨请求共享，也不应并发执行多个查询。

---

### Q4：如何解释 `IEnumerable<T>`、`IQueryable<T>`、`IAsyncEnumerable<T>` 的区别？

**核心回答：**

- `IEnumerable<T>`：在内存中枚举；LINQ 操作通常由 .NET 执行。
- `IQueryable<T>`：保存表达式树，ORM 或远程提供程序可翻译为 SQL 等查询；应尽量在它还没有物化前完成筛选、排序和分页。
- `IAsyncEnumerable<T>`：异步逐项产出数据，适合流式读取或大量数据处理。

```csharp
var page = await _dbContext.Orders
    .Where(x => x.CustomerId == customerId)
    .OrderByDescending(x => x.CreatedAt)
    .Select(x => new OrderSummaryDto(x.Id, x.Status, x.Total))
    .Skip(offset)
    .Take(pageSize)
    .ToListAsync(cancellationToken);
```

**深入说明：**

- 不要过早调用 `ToList()`，否则后续 `Where` 可能在应用内存中执行，并把大量数据从数据库拉回。
- `IQueryable<T>` 不应从仓储层泄漏到无限制的上层，避免调用方任意拼接查询、引入难以控制的性能与安全问题。可在应用服务层收敛查询能力，返回 DTO 或分页结果。
- 使用 `IAsyncEnumerable<T>` 时要确保数据库连接和枚举生命周期正确，并谨慎处理慢消费者造成的连接长时间占用。

---

### Q5：异常处理、日志和可观测性应如何设计？

**核心回答：**  
异常应在最了解如何恢复的位置处理；无法恢复的异常交给全局异常处理中间件，统一转换为标准错误响应，并记录结构化日志、Trace ID 和必要上下文。不要吞掉异常，也不要把数据库连接字符串、令牌、个人信息写入日志。

```csharp
app.UseExceptionHandler(exceptionApp =>
{
    exceptionApp.Run(async context =>
    {
        var feature = context.Features.Get<IExceptionHandlerFeature>();
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogError(feature?.Error,
            "Unhandled exception. TraceId: {TraceId}", context.TraceIdentifier);

        await Results.Problem(
            statusCode: StatusCodes.Status500InternalServerError,
            title: "An unexpected error occurred")
            .ExecuteAsync(context);
    });
});
```

**深入说明：**

- 对可预期的业务错误（库存不足、状态不允许变更）返回明确的 4xx 和业务错误码，而不是用异常驱动正常流程。
- 结构化日志用占位符而非字符串拼接，例如 `LogInformation("Order {OrderId} created", orderId)`，便于在 Application Insights / Log Analytics 查询。
- 关键指标至少包括：请求量、P50/P95/P99 延迟、错误率、依赖调用耗时、队列长度、Function 失败与重试次数。
- 使用 W3C Trace Context 或 OpenTelemetry 让 API、Function、消息处理、数据库依赖使用同一关联 ID，排障时能沿一条调用链定位。

---

## 二、ASP.NET Core 与 Web API

### Q6：ASP.NET Core 中间件管道为何顺序敏感？

**核心回答：**  
中间件按注册顺序组成嵌套调用链。每个中间件可在调用下一个组件前处理请求、在其返回后处理响应，或直接短路请求。认证、授权、路由、异常处理等都有依赖顺序，顺序错误会造成未认证访问、404、CORS 失败或异常未被统一处理。

```csharp
app.UseExceptionHandler("/error");
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors("WebClient");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

**深入说明：**

- `UseAuthentication()` 应在 `UseAuthorization()` 之前，否则授权阶段无法得到已验证的用户。
- CORS 必须在终结点实际执行前运行；对于控制器 API，通常放在路由与认证授权附近，并使用命名策略。
- 需要捕获下游异常的中间件应尽可能靠前，例如异常处理与请求关联 ID。
- `MapControllers()` 等终结点映射不是传统意义上可在其后继续处理的中间件；应避免把依赖所有终结点的全局逻辑注册在映射之后。

---

### Q7：如何设计一个可靠、可演进的 REST API？

**核心回答：**  
以资源和业务能力为中心设计清晰 URI，正确使用 HTTP 动词、状态码和幂等语义；请求与响应使用 DTO，不直接暴露 EF 实体；验证输入、分页、筛选、排序、版本和一致的错误格式都要在首版考虑。

| 场景 | 建议 |
|---|---|
| 创建资源 | `POST /api/orders`，成功返回 `201 Created` 和 `Location` |
| 查询单项 | `GET /api/orders/{id}`，不存在返回 `404` |
| 全量替换 | `PUT /api/orders/{id}`，应具备幂等性 |
| 局部修改 | `PATCH /api/orders/{id}`，明确允许更新的字段 |
| 删除 | `DELETE /api/orders/{id}`，成功可返回 `204 No Content` |
| 参数错误 | `400 Bad Request`，返回字段级校验信息 |
| 未认证 / 无权限 | `401` / `403`，语义必须区分 |

**深入说明：**

- 列表接口必须限制最大 `pageSize`，并定义稳定排序；大表场景优先游标分页（例如按创建时间和 ID），避免深度 `OFFSET` 的性能退化。
- 破坏性变更可采用 URL、请求头或媒体类型版本策略。版本不是替代兼容设计的借口，优先新增字段、保留旧行为，设置清晰弃用窗口。
- 对创建订单、支付等可重试写操作，支持 `Idempotency-Key`：保存键、请求摘要和最终结果；相同键的重试返回第一次的结果，键冲突但请求不同则拒绝。
- 使用 OpenAPI/Swagger 生成契约，前端基于契约生成客户端或类型，减少联调歧义。

---

### Q8：JWT 身份验证有哪些关键安全点？为什么“只验证签名”不够？

**核心回答：**  
JWT 验证至少包括签名、签发者（issuer）、受众（audience）、过期时间、允许算法及必要声明。只验证签名仍可能接受签给其他应用的令牌、已过期令牌或不符合授权范围的令牌。认证回答“是谁”，授权回答“能做什么”。

```csharp
builder.Services.AddAuthentication().AddJwtBearer(options =>
{
    options.Authority = builder.Configuration["Authentication:Authority"];
    options.Audience = "orders-api";
    options.TokenValidationParameters.ValidateLifetime = true;
});

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanApproveOrder", policy =>
        policy.RequireClaim("scope", "orders.approve"));
});
```

**深入说明：**

- 生产环境使用 Microsoft Entra ID 或其他专业身份提供方的 OpenID Connect / OAuth 2.0 流程，不自行实现密码存储和令牌签发。
- 前端 SPA 使用授权码流加 PKCE。不要把客户端密钥放在浏览器代码、`localStorage` 配置文件或公开仓库。
- 访问令牌应短生命周期；撤销、会话失效和高风险操作要有相应策略，不能假设 JWT 一经签发就永远有效。
- CORS 不是认证授权措施；它只约束浏览器是否允许某页面读取响应，不能阻止直接 HTTP 调用。

---

### Q9：EF Core 如何避免 N+1 查询、跟踪开销和并发更新丢失？

**核心回答：**  
对读取 DTO 使用投影和 `AsNoTracking()`，避免逐条访问导航属性造成 N+1；写入时按业务边界加载必要实体；对于并发可修改的数据，使用并发令牌（如 `rowversion`）检测冲突，并返回可理解的冲突结果。

```csharp
var orders = await _dbContext.Orders
    .AsNoTracking()
    .Where(x => x.Status == OrderStatus.Pending)
    .Select(x => new PendingOrderDto(
        x.Id,
        x.Customer.Name,
        x.Lines.Sum(line => line.Quantity * line.UnitPrice)))
    .ToListAsync(cancellationToken);
```

**深入说明：**

- `Include` 适用于需要实体图的情况；纯展示接口更推荐 `Select` 投影到 DTO，只取需要字段。
- 开发和压测阶段开启 SQL 日志或使用 Application Insights 观察依赖调用，确认是否有重复 SQL。不要凭感觉添加所有 `Include`，否则也可能产生巨大联表结果。
- 为并发字段配置 `[Timestamp]` / `IsRowVersion()`。更新时 EF Core 检测受影响行数为 0 会抛出 `DbUpdateConcurrencyException`，应用可让用户刷新、合并或重试。
- 事务只包裹必须原子化的数据库操作；跨数据库、消息系统的最终一致性问题通常用 Outbox 等模式解决，而非把网络调用塞进长事务。

---

### Q10：如何实现健康检查、优雅关闭和后台任务？

**核心回答：**  
健康检查应区分“进程存活”（liveness）和“可接流量”（readiness）；应用停止时通过 `CancellationToken` 停止接收新工作并完成或安全地放弃当前工作。后台轮询任务实现为 `BackgroundService`，但耗时、可扩展或需要独立失败域的任务应改用队列和独立 Worker/Function。

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString, name: "sql", tags: ["ready"]);

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false
});
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

**深入说明：**

- 不要让 liveness 检查依赖外部服务，否则数据库短暂故障会导致平台反复重启健康的应用实例。
- 定时任务需要防重入、可取消、可观测。多实例部署时还要避免所有实例执行同一任务，可使用分布式锁、队列或专门调度服务。
- 长任务不要在 HTTP 请求内等待完成。API 返回 `202 Accepted` 和任务 ID，由后台异步处理，前端再轮询、订阅通知或查询结果。

---

## 三、Azure Functions

### Q11：Azure Functions 的常见触发器如何选择？HTTP Trigger 和 Blob Trigger 能互相替代吗？

**核心回答：**  
触发器由事件来源和交互语义决定，不能简单互相替代。HTTP Trigger 用于同步请求/响应；Blob Trigger 用于文件到达后的异步处理；Service Bus Trigger 用于可靠异步命令或工作队列；Timer Trigger 用于周期性任务；Event Grid Trigger 用于高扇出、低延迟的事件通知。

| 需求 | 合适触发器 | 原因 |
|---|---|---|
| 用户提交表单并立即获得结果 | HTTP | 需要同步响应 |
| 上传文件后的解析、病毒扫描 | Blob 或 Event Grid + HTTP/队列 | 文件事件驱动、可异步 |
| 订单创建后发送邮件、扣积分 | Service Bus | 解耦、削峰、重试与死信 |
| 每日生成报表 | Timer | 明确调度 |
| 存储账户事件通知多个下游 | Event Grid | 发布订阅、低延迟扇出 |

**深入说明：**

- Blob Trigger 更适合“处理 Blob 内容”的消费者；事件量高或需要多个独立订阅者时，可让 Event Grid 把 Blob 创建事件路由到队列、Function 或 Webhook。
- HTTP API 不应同步处理大型文件转换或长时间批处理。应先把文件放 Blob，创建异步任务，快速返回追踪 ID。
- 选择时还要考虑事件投递保证、重试、顺序、最大执行时间、冷启动、私网访问和成本，而非只看开发便利。

---

### Q12：Azure Functions 的至少一次投递意味着什么？怎样实现幂等？

**核心回答：**  
许多事件源或消息系统以至少一次方式投递：函数可能因超时、主机重启、确认丢失等原因收到重复事件。因此消费者必须允许同一业务事件被安全重复执行。不能仅依赖“平台通常只投一次”。

**推荐实现：**

1. 为每个业务事件定义稳定的唯一键，例如 `orderId + eventType + version`，不要只用随机运行 ID。
2. 在持久化存储中用唯一约束或条件写入记录处理结果。
3. 只有成功完成关键写入后才标记已处理；失败则让平台重试或明确转入补偿流程。
4. 依赖外部 API 时也传递幂等键，并记录请求与响应。

```csharp
public async Task ProcessAsync(OrderCreated message, CancellationToken cancellationToken)
{
    var receipt = new EventReceipt(message.EventId, DateTimeOffset.UtcNow);
    _dbContext.EventReceipts.Add(receipt);

    try
    {
        await _dbContext.SaveChangesAsync(cancellationToken); // EventId 唯一索引
    }
    catch (DbUpdateException ex) when (IsUniqueConstraintViolation(ex))
    {
        return; // 同一事件已经成功占位或处理
    }

    await _orderService.ApplyCreatedOrderAsync(message, cancellationToken);
}
```

**深入说明：**

- 上例展示了去重入口，但生产设计还要考虑“占位成功后业务执行失败”的恢复策略。更稳妥的是将事件状态机、业务更新和 Outbox 写入放在同一数据库事务里，由可重试的 Worker 推进状态。
- “幂等”不等于“忽略所有重复”。如果重复消息内容不同，应记录冲突并告警，而不是静默覆盖。
- 对有顺序要求的业务，应使用分区键、会话（如 Service Bus Sessions）或版本号，拒绝旧版本覆盖新状态。

---

### Q13：Consumption、Premium 与 Dedicated（App Service Plan）计划如何选？

**核心回答：**

| 计划 | 适合场景 | 优点 | 主要取舍 |
|---|---|---|---|
| Consumption | 波动明显、事件驱动、短任务 | 按执行量付费、自动扩缩 | 可能冷启动，网络与运行能力有约束 |
| Premium | 低延迟、VNet、持续高负载或更强实例能力 | 预热实例、较强扩缩与网络能力 | 有基础预留成本 |
| Dedicated / App Service Plan | 已有计划、稳定负载、需与 Web App 共用计算 | 容量可控、可统一管理 | 即使空闲也需为实例付费，扩缩需自行规划 |

**深入说明：**

- 先从流量曲线、SLA、首请求延迟容忍度、单次执行时长、私网依赖、并发模型与总拥有成本判断，而非认为“Serverless 永远最便宜”。
- 生产环境设置扩缩上限、预算告警与并发/批量参数，避免突发事件导致成本失控或压垮下游数据库。
- 运行时版本、隔离进程模型、部署槽、区域容量会影响具体可选项。方案设计与上线前应以当前 Azure 官方产品能力及组织政策复核。

---

### Q14：如何降低 Function 冷启动并防止扩缩时压垮下游？

**核心回答：**  
冷启动通过选择适合的托管计划、减少启动路径工作、复用客户端和依赖预热来改善；下游保护通过队列削峰、限制并发、批量处理、重试退避和熔断来实现。只追求函数扩容速度，可能使 SQL、第三方 API 或 Blob 帐户先成为瓶颈。

**具体做法：**

- 在依赖注入容器中注册复用型客户端，如 `HttpClient`、Azure SDK client；不要在每次调用中创建。
- 删除启动时不必要的远程调用、大型配置加载和反射扫描。
- 对队列触发器根据下游承载能力设置合理批量与并发，不把最大吞吐量当作唯一目标。
- 对临时错误采用有限重试和指数退避；对持续故障启用死信/隔离队列、告警和人工/自动补偿。
- 用 Application Insights 观察冷启动、执行时长、依赖失败率、队列积压和实例数，再基于数据调参。

---

### Q15：Function 的配置与密钥应如何管理？

**核心回答：**  
配置按环境放在 Function App Application Settings 或集中配置服务中；机密放 Azure Key Vault；应用通过系统分配或用户分配的 Managed Identity 读取机密与访问其他 Azure 资源。生产环境不在代码、`local.settings.json`、前端包或日志中保存密钥。

```csharp
var credential = new DefaultAzureCredential();
var blobServiceClient = new BlobServiceClient(
    new Uri("https://contoso.blob.core.windows.net"),
    credential);
```

**深入说明：**

- `DefaultAzureCredential` 在本地可使用开发者登录身份，在 Azure 中优先使用托管标识，减少环境差异；但必须显式授予最小 RBAC 权限。
- Key Vault 引用或 SDK 读取均可，选择取决于轮换、缓存、网络和运维模式。无论哪种方式，密钥轮换都需要演练。
- 配置变更应可审计并按环境隔离。区分非敏感配置、连接端点、机密和功能开关，避免所有设置混在一个文件。

---

## 四、Azure Blob Storage

### Q16：Blob Storage 有哪些 Blob 类型？文件上传应该如何选型和设计？

**核心回答：**  
Block Blob 适用于文档、图片、日志、数据文件等绝大多数对象；Append Blob 适合持续追加日志；Page Blob 面向随机读写和虚拟磁盘场景。常规文件上传选择 Block Blob，并配合分块上传、内容类型、元数据、校验、访问控制和生命周期规则。

**深入说明：**

- 浏览器大文件上传建议由 API 进行身份和业务授权后，签发时间短、权限最小的用户委派 SAS 或受控上传会话；浏览器直接上传 Blob，避免应用服务器成为带宽瓶颈。
- SAS 是持有者令牌，应限制容器/对象、读写权限、IP（适用时）和有效期；不要生成账户级、长期、万能权限 SAS。
- 客户端提交文件后不能只凭“上传成功”认定业务完成。应记录待扫描/待解析状态，异步校验大小、MIME 类型、内容安全和业务格式，成功后再移动或标记为可用。
- Blob 名称可表达分区，如 `orders/year=2026/month=10/day=26/<id>.json`，但不要在路径中使用未经清洗的用户输入或敏感信息。

---

### Q17：Blob 的访问层、冗余和生命周期管理如何选择？

**核心回答：**  
访问层根据数据访问频率选择 Hot、Cool、Cold、Archive；冗余根据可用性和灾难恢复目标选择 LRS、ZRS、GRS/RA-GRS 等。生命周期规则自动把符合条件的数据降层、删除或管理版本，达到成本与合规平衡。

| 决策维度 | 需要澄清的问题 |
|---|---|
| Hot / Cool / Cold / Archive | 读取频率、最短存储期限、恢复时延、读取费用 |
| LRS / ZRS | 是否需要可用区级容灾、RPO/RTO 是什么 |
| GRS 系列 | 是否要求跨区域灾难恢复、是否需要读访问次要区域 |
| 版本与软删除 | 是否需要抵御误删、勒索和覆盖错误，保留多久 |
| 不可变策略 | 是否有 WORM、审计或监管留存要求 |

**深入说明：**

- Archive 数据在线读取前需要再水化，且有延迟和费用，不能用于需要即时恢复的生产热数据。
- 生命周期策略应在代表性数据上验证，尤其确认前缀、版本、快照和法律保留是否符合预期；不要把“删除规则”直接当作低风险成本优化。
- 生产环境启用软删除、版本控制与适当的恢复演练；对关键容器限制公共匿名访问，并记录诊断日志。

---

### Q18：如何保证 Blob 处理流程的数据一致性和安全性？

**核心回答：**  
Blob 本身不是关系事务数据库。要通过不可变输入、处理状态、条件操作、幂等消费者、事件可重放和可审计元数据建立可靠流程；安全方面采用 Microsoft Entra ID + RBAC、私有终结点、最小化 SAS、加密和网络限制。

**一个可靠的文件处理流程：**

1. 上传到隔离的 `incoming` 容器，标记为 `PendingScan`。
2. 安全扫描和格式校验通过后，写入内容哈希、业务 ID、版本和状态。
3. 处理服务以幂等键执行转换，输出写入新对象或新版本，不覆盖原始输入。
4. 成功后原子性地更新元数据/业务数据库状态，失败进入可追踪的 `failed` 区域。
5. 通过重试、死信、告警和人工重放处理异常，不依赖手工猜测。

**实战追问：为什么不能只监听 Blob 创建事件后直接处理？**  
因为上传可能是分块、覆盖或重试操作；事件可能重复或延迟。处理器需要确认对象已完整可读，并以业务状态和幂等性控制最终结果。

---

## 五、Azure App Service

### Q19：如何把 ASP.NET Core 应用部署到 App Service，并做到安全的持续交付？

**核心回答：**  
通过 CI 构建、测试、生成不可变制品，再用 CD 部署到 App Service；应用配置与机密独立于制品；生产变更通过部署槽预热、验证与交换，失败可快速回退。部署身份使用最小权限的联邦凭据或托管身份，避免在流水线保存长期发布密码。

**推荐流水线阶段：**

1. 还原依赖，执行单元测试、静态分析和构建。
2. 发布版本化制品，生成依赖/安全扫描结果。
3. 部署至开发或预发布槽，运行冒烟测试和 API 契约测试。
4. 预热预发布槽，确认健康检查、关键依赖、错误率和延迟。
5. 交换到生产槽；持续监控，异常时交换回退或部署前一已验证版本。

**深入说明：**

- 槽设置（slot settings）确保生产密钥、数据库端点等不会因交换而错误互换。
- 流水线应把基础设施和应用发布分离但可追溯；用 Bicep/Terraform 等 IaC 声明 App Service Plan、Web App、诊断、身份、私网与 RBAC。
- 成功部署不等于业务可用。必须验证关键用户路径、依赖健康和可观测性是否正常。

---

### Q20：Deployment Slot 的交换机制是什么？哪些配置容易出错？

**核心回答：**  
部署槽让新版本先在非生产槽运行和预热，再将槽内容交换到生产，减少冷启动和停机。交换时遵循槽配置规则：标记为 slot setting 的配置留在原槽；未标记的配置随应用内容交换。最常见错误是把生产数据库、回调地址、队列消费者开关等关键配置错误标记或遗漏标记。

**上线检查：**

- 预发布槽是否用了隔离的数据库、队列、第三方回调和数据容器。
- 生产专属密钥、连接串、域名、监控标签是否设为 slot setting。
- 预发布环境是否会误消费生产消息或发送真实邮件；需要时禁用消费者或使用独立资源。
- 交换前是否已经访问预发布槽并完成预热。
- 是否定义回滚条件，例如 5xx 错误率或 P95 延迟在观察窗口内超过阈值。

---

### Q21：App Service 的扩缩容和性能排障通常怎么做？

**核心回答：**  
先用监控数据定位瓶颈类别：CPU、内存、线程池、GC、依赖、数据库、网络或应用代码，再决定扩实例、升规格、优化代码或保护下游。自动扩缩规则应有最小/最大实例、冷却期和业务指标，不能只盯 CPU。

**排障步骤：**

1. 在 Application Insights 查看失败率、请求 P95/P99、依赖调用、异常和实例分布。
2. 使用 Trace ID 找到单次慢请求，比较应用处理时间和 SQL/HTTP/Blob 等依赖耗时。
3. 检查 CPU、内存、GC、线程池队列和连接池；确认没有同步阻塞、连接泄漏或无限重试。
4. 复现后用 Profiler、诊断工具或受控负载测试验证假设。
5. 采取针对性修复并设置仪表盘、告警和容量阈值，验证指标回归。

**实战要点：**  
若瓶颈在 SQL，盲目将 Web App 从 2 个扩到 10 个实例只会放大连接和查询压力。应先优化查询、索引或缓存，并对数据库连接池和 API 并发设置上限。

---

### Q22：App Service 如何安全访问 Azure SQL、Storage 或 Key Vault？

**核心回答：**  
为 App Service 启用 Managed Identity，在目标资源上授予满足业务需要的最小 RBAC 角色，应用使用 `DefaultAzureCredential` 获取令牌。结合 VNet Integration、Private Endpoint、私有 DNS 和禁用公共网络访问（在条件允许时），构建无密钥、最小暴露的访问路径。

**深入说明：**

- Storage 数据面权限使用如 `Storage Blob Data Reader` / `Storage Blob Data Contributor`，而不是仅授予资源管理平面 `Contributor`。
- Key Vault 使用密钥读取类权限；应用不需要 Vault 管理员权限。
- 对 Azure SQL，可使用 Microsoft Entra 身份认证与最小数据库权限，避免共享 SQL 用户名密码。
- 私有终结点后必须验证 DNS 解析到私有 IP，否则应用仍可能访问公共终结点或发生连接失败。

---

## 六、Azure Data Factory（ADF）

### Q23：ADF 的 Pipeline、Activity、Dataset、Linked Service、Integration Runtime 分别是什么？

**核心回答：**

| 概念 | 含义 | 类比 |
|---|---|---|
| Pipeline | 一次数据工作流的编排容器 | 流程定义 |
| Activity | 具体步骤，例如 Copy、Lookup、Data Flow、ForEach | 流程中的任务 |
| Dataset | 数据结构或数据位置的逻辑描述 | 表/文件的引用模板 |
| Linked Service | 到数据源或计算资源的连接信息 | 连接配置 |
| Integration Runtime（IR） | 执行数据移动、活动或 Data Flow 的计算基础设施 | 执行引擎/代理 |

**深入说明：**

- Azure IR 适用于云端数据源和托管执行；Self-hosted IR 用于访问本地网络或受限网络中的数据源；Azure-SSIS IR 用于迁移/运行 SSIS 包。
- Linked Service 不应硬编码凭据，优先使用 Managed Identity 或 Key Vault 引用。
- Dataset 和 Pipeline 参数化可以减少复制粘贴。日期、环境、表名、路径等作为参数传入，使同一模板服务多张表或多个环境。

---

### Q24：如何设计一个可复用、可监控的 ADF 增量数据管道？

**核心回答：**  
以元数据驱动和水位线（watermark）实现增量：控制表保存源表、目标、增量列、上次成功水位和状态；Pipeline 读取配置并参数化执行；成功写入目标后再原子更新水位。每次运行记录批次 ID、读取量、写入量、耗时、失败原因和数据质量结果。

**流程示例：**

1. `Lookup` 读取启用的源表配置及最后成功水位。
2. `ForEach` 在受控并发下执行每张表的 Copy Activity 或数据转换。
3. 源端查询使用范围条件，如 `UpdatedAt > @lastWatermark AND UpdatedAt <= @currentUpperBound`。
4. 数据落到 Raw 层，执行格式、空值、重复、主键和行数校验。
5. 成功后写入 Curated 层，并更新控制表；失败时保留水位不前进。
6. 向 Log Analytics / Monitor 发送指标与告警。

**深入说明：**

- 上界 `currentUpperBound` 要在运行开始时固定，防止运行中持续出现的新数据造成永不结束或遗漏边界。
- 时间戳精度、时区、相同更新时间和源系统回补都会造成边界问题。可用复合水位（时间 + 主键）或 Change Tracking / CDC 提升可靠性。
- 不能只因 Copy Activity 成功就认定数据正确；应校验行数、重复率、字段约束和业务规则。

---

### Q25：ADF Copy Activity 与 Mapping Data Flow 如何选择？

**核心回答：**  
Copy Activity 面向高效的数据搬运和轻量字段映射；Mapping Data Flow 适合可视化的大规模转换、Join、Aggregate、派生列、数据质量分流等 Spark 型处理。简单搬运不用 Data Flow，复杂转换也不应硬塞在 Copy 的有限映射能力中。

| 情况 | 优先选择 |
|---|---|
| 数据库表复制到 Data Lake / Blob | Copy Activity |
| 文件格式转换和轻量列映射 | Copy Activity |
| 多源 Join、聚合、去重、复杂规则 | Mapping Data Flow |
| 高度定制、已有 Spark 生态或复杂代码逻辑 | Databricks / Synapse 等计算服务，由 ADF 编排 |

**实战追问：Data Flow 出现脏数据怎么办？**  
在转换中将格式不合法、关键字段为空、违反规则的数据路由到隔离区，保留原因和原始记录；对可修复问题按规则清洗，对不可修复问题产生告警和可回放任务。不能悄悄丢弃，也不能让一条坏记录无条件阻断所有批次。

---

### Q26：ADF 如何实现安全、可运维的生产部署？

**核心回答：**  
ADF 的开发、测试、生产环境隔离；代码和 ARM/Bicep/Terraform 定义纳入版本控制；通过 CI/CD 发布并在环境阶段注入参数；凭据由 Key Vault 和 Managed Identity 管理；用 Azure Monitor 对失败、时延和数据质量建立告警。

**关键实践：**

- Git 分支策略、代码审查和发布审批与团队标准一致，避免直接在生产工作区手工修改。
- 通过全局参数、参数文件或变量组管理环境差异，避免复制整套 Pipeline。
- 为活动设置合理的 timeout 与 retry；重试只用于临时错误，数据格式错误、权限错误等确定性错误要快速失败并告警。
- 运行失败时保留输入位置、参数、错误详情、批次 ID 与可重放入口；不要仅依靠截图或口头排查。

---

## 七、前后端分离与 React/Vue/Angular

### Q27：什么是前后端分离？它解决什么问题，又引入什么成本？

**核心回答：**  
前后端分离指前端 SPA/SSR 与后端 API 独立开发、部署和扩缩，二者通过明确 API 契约协作。它让 UI 演进、后端能力复用和团队并行更高效，但也引入契约治理、认证、CORS、版本兼容、跨团队联调和端到端可观测性成本。

**深入说明：**

- 分离不是“前端直接访问数据库”或“后端只返回任意 JSON”。后端仍应负责授权、业务规则、数据一致性与服务端安全边界。
- DTO 是对外契约，不能直接输出 EF 实体，否则会泄漏内部字段、引发循环序列化或使数据库模型绑死 API。
- 建议维护 OpenAPI 文档、Mock 策略、契约测试与弃用规则。前端升级与后端发布可以独立，但必须兼容。

---

### Q28：React、Vue、Angular 如何选择？面试中怎样避免“框架站队”？

**核心回答：**  
三者都能构建成熟企业应用，选型应基于团队经验、项目规模、现有生态、类型安全、状态管理、招聘与维护成本，而非绝对性能或个人偏好。React 灵活、生态丰富；Vue 上手快、渐进式；Angular 提供更强约束与完整框架能力，常适合大型标准化团队。

| 维度 | React | Vue | Angular |
|---|---|---|---|
| 定位 | UI 库，组合选择多 | 渐进式框架 | 完整框架 |
| 常见特点 | Hooks、JSX、生态丰富 | 模板直观、组合式 API | TypeScript、DI、RxJS、规范完整 |
| 选型关注点 | 需明确路由/状态/数据方案 | 团队学习与工程规范 | 学习曲线与框架约束 |

**高质量回答示例：**  
“若团队已有 React + TypeScript + 查询缓存的成熟模板，我会优先复用，降低交付风险；若新建后台系统且组织偏好统一规范，Angular 的 DI、表单和模块化也很合适。无论框架，组件边界、可访问性、测试、错误处理、性能监控与 API 契约比框架名称更影响结果。”

---

### Q29：SPA 应如何调用 .NET API 并安全处理认证、CORS 与错误？

**核心回答：**  
前端通过统一 API client 调用后端；使用 OAuth 2.0 授权码流 + PKCE 获取访问令牌；后端验证令牌并按 scope/role/policy 授权；CORS 只允许受信任的精确来源、方法和请求头；前端将错误分为验证错误、认证过期、权限拒绝、可重试网络错误与不可恢复错误，提供相应 UX。

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("WebClient", policy => policy
        .WithOrigins("https://app.contoso.example")
        .AllowAnyHeader()
        .AllowAnyMethod()
        .AllowCredentials());
});
```

**深入说明：**

- 使用 Cookie 时，跨站请求还需要 CSRF 防护、`SameSite`、`Secure`、`HttpOnly` 等设计；使用 Bearer token 时，不应把长期高权限令牌存储在易受 XSS 影响的位置。
- `AllowAnyOrigin()` 不能与 `AllowCredentials()` 一起使用；生产系统不应用反射任意 `Origin` 请求头。
- API 返回标准 Problem Details 或统一错误格式，使前端能展示字段校验、业务提示和 trace ID。
- 对 `401` 可触发登录/刷新流程；对 `403` 明确告知无权限；对 `429` 读取重试建议并退避；不要把所有失败都显示为“系统异常”。

---

### Q30：如何设计前端状态管理，避免重复请求和数据不一致？

**核心回答：**  
区分服务端状态与纯 UI 状态。服务端状态（订单列表、用户资料）由查询缓存工具管理其加载、缓存、失效、重试和重新获取；UI 状态（弹窗是否打开、当前 tab、临时表单输入）保持在组件或轻量全局状态中。不要把所有东西都塞进一个全局 store。

**深入说明：**

- 写操作成功后根据影响范围精确失效或更新相关缓存，避免“每次保存后全站刷新”。
- 列表数据的查询键应包含筛选条件、排序和分页参数；否则不同条件可能错误复用缓存。
- 乐观更新能提升体验，但要有失败回滚、并发编辑处理和服务端最终校验；资金、库存等高风险业务应更保守。
- 复杂表单用明确的提交状态、字段错误映射和离开页面保护。后端仍是最终规则执行者。

---

### Q31：大型前端页面加载慢时，怎样系统定位与优化？

**核心回答：**  
先以真实用户指标和浏览器性能工具确认瓶颈是在网络、JavaScript、渲染、接口、图片还是第三方脚本，再采取相应措施。常见优化包括路由级代码分割、延迟加载、压缩资源、图片响应式处理、列表虚拟化、减少不必要重渲染、API 分页/批量化和缓存策略。

**排查顺序：**

1. 通过 RUM、Web Vitals 和浏览器 Network 面板看 LCP、INP、CLS、资源瀑布和 API 耗时。
2. 用 Performance/React Profiler 或对应框架工具判断长任务与重渲染来源。
3. 对比首屏、缓存命中与弱网环境，确认问题是否来自前端包、CDN、后端 API 或第三方。
4. 每次优化后用相同环境和指标验证收益，防止把可维护性换成没有实际收益的复杂代码。

---

## 八、综合架构与独立问题解决

### Q32：请设计“客户上传订单文件，系统校验并供运营人员审核”的 Azure 架构。

**核心回答：**  
我会把同步交互与异步处理分开：Web 前端部署在 Static Web Apps 或独立前端托管，ASP.NET Core API 部署在 App Service；API 负责 Entra ID 身份授权、创建上传会话和签发短期受限上传凭据；文件进入 Blob `incoming` 容器；Event Grid / Blob 事件触发 Function 做安全扫描、格式校验和幂等处理；处理结果和审计状态写入 Azure SQL；运营后台通过 API 查询状态并审核。ADF 负责历史文件回填和定期汇总入湖，而非承担实时单文件业务事务。

```text
Browser SPA
    | Entra ID + HTTPS
    v
App Service (.NET API) ---- Azure SQL (订单、状态、审计)
    | short-lived delegated upload access
    v
Blob Storage: incoming -> validated / rejected / archive
    | event
    v
Azure Function ---- Service Bus (异步后续动作 / 死信)
    |
    +---- Application Insights / Log Analytics

ADF: 历史回填、批量清洗、数据湖/报表编排
```

**深入说明：**

- **安全**：Managed Identity + RBAC、Key Vault、私有终结点、禁用不必要的公共访问；前端永不持有服务端密钥。
- **可靠性**：Function 幂等处理，输入与输出不可变，Service Bus 重试和死信，数据库并发控制，失败文件可定位、可重放。
- **性能**：大文件不经 App Service 转发；队列削峰；Function 并发受下游 SQL/第三方服务承载能力约束。
- **可运维性**：每个上传分配 correlation ID，API、Blob 元数据、消息与数据库审计记录它；仪表盘展示成功率、处理时延、失败原因和积压量。

---

### Q33：线上 API P95 延迟从 300ms 升至 5 秒，你会如何在 30 分钟内处理？

**核心回答：**  
先建立影响范围和止血措施，再用遥测定位，不凭直觉直接改代码。目标是在保护业务正确性的前提下恢复服务，并保留证据完成后续根因分析。

**30 分钟行动框架：**

| 时间 | 行动 | 目的 |
|---|---|---|
| 0-5 分钟 | 确认告警、影响 API/区域/用户比例、最近发布与依赖状态；建立事件沟通频道 | 避免误判和重复操作 |
| 5-10 分钟 | 看 Application Insights 的请求、依赖、异常、实例、CPU/内存；比较变更前后 | 定位是应用、依赖还是容量 |
| 10-15 分钟 | 采取低风险缓解：回滚最近版本、扩容、启用限流/缓存、暂停非关键任务 | 优先恢复用户体验 |
| 15-25 分钟 | 以 Trace ID 深挖慢请求；检查 SQL、外部 HTTP、线程池、连接池、队列与配置变化 | 形成可验证假设 |
| 25-30 分钟 | 验证指标恢复，记录时间线、影响、缓解措施与待办 | 让恢复可证明、可复盘 |

**深入说明：**

- 若依赖 SQL 耗时激增，先检查慢查询、连接池耗尽、锁等待和资源利用率；扩 Web 实例可能使 SQL 更糟。
- 若是下游第三方失败，使用超时、熔断、有限重试和降级，不能无限重试放大雪崩。
- 恢复后编写 RCA：触发条件、根因、为何未提前发现、短期修复、长期预防及负责人/截止日期。事故复盘强调系统改进，不归咎个人。

---

### Q34：数据管道出现重复数据和部分批次失败，如何拆解并修复？

**核心回答：**  
我会先冻结会继续扩大问题的自动重跑或下游消费，保留失败批次证据；然后将问题分成“重复从何处产生”“目标写入是否幂等”“水位何时推进”“如何安全补数”四部分。修复不能只删除重复行，必须防止下一次再次发生。

**排查与修复：**

1. 根据批次 ID、源主键、目标加载时间、ADF Run ID 对账，确定重复是源端重复、抽取边界重叠、Pipeline 重试还是目标 merge 非幂等。
2. 检查水位是否只在整批成功后更新；确认是否采用了固定上界及正确的时区/精度。
3. 在 Staging 层保留原始批次，目标层通过唯一键 + `MERGE` / upsert 或版本规则实现可重跑。
4. 对已污染数据，依据权威源和审计记录制定回滚/去重脚本，在预生产验证并经过审批后执行。
5. 增加行数对账、唯一性检测、失败告警、重放工具和演练。

**面试加分点：**  
明确提出“删除生产数据前先备份并在只读副本/预生产验证”，“补数操作也要有批次 ID 与审计记录”。这体现了对数据修复风险的控制。

---

### Q35：如何解释 CAP、最终一致性与 Saga/Outbox，并将它们用于订单场景？

**核心回答：**  
分布式系统出现网络分区时，无法同时保证强一致性和完全可用性。订单、库存、支付属于多个独立边界时，通常采用本地事务保证单服务内部一致，再通过可靠事件实现最终一致性。Saga 将跨服务流程拆为一系列可补偿步骤；Outbox 保证“业务数据提交”和“待发布事件记录”在同一数据库事务中完成，避免数据库已更新但消息未发出的双写问题。

**订单示例：**

1. 订单服务在本地事务中创建 `Pending` 订单，并写入 `OrderCreated` Outbox 记录。
2. 后台发布器可靠地将 Outbox 事件发到 Service Bus，并记录发布状态。
3. 库存服务幂等地预留库存，成功后发出 `InventoryReserved`，失败则发出拒绝事件。
4. 订单服务据此进入待支付或取消状态；若后续支付失败，Saga 触发释放库存的补偿命令。

**深入说明：**

- Saga 不是免费午餐：补偿未必能物理“回滚”（例如邮件已发送），所以需要业务上可接受的状态机、审计和用户沟通。
- 事件消费者必须幂等、可重试、可观测。Outbox 表也需要清理、监控积压和处理毒消息。
- 不要把分布式事务当默认方案；先确定业务是否真的需要跨边界强一致性，以及是否能通过预留、状态机和人工介入满足需求。

---

### Q36：请举例说明你如何独立解决一个陌生技术问题。

**推荐回答结构：STAR + 证据链**

> **S（情境）**：某次上线后，文件处理任务在高峰期积压，运营人员等待时间显著增加。  
> **T（任务）**：我负责在不丢数据、不破坏正在处理订单的前提下，恢复处理时效并找出根因。  
> **A（行动）**：我先用队列长度、Function 执行耗时、失败率和依赖追踪区分“生产变慢”与“消费变慢”；发现瓶颈是下游 API 的限流，而 Function 重试策略导致并发请求进一步放大。随后我将处理流程改为受控并发、指数退避和死信隔离；为每个文件引入幂等键和可重放状态；在预生产用回放数据验证处理量和正确性后分阶段上线。  
> **R（结果）**：积压在可预期时间内消化，失败文件可以准确定位和重放；随后把队列积压、429 比例和端到端时延加入告警，并完成运行手册与复盘。

**回答要点：**

- 说清楚如何定义问题和成功标准，而非只说“我查日志修好了”。
- 说明使用了哪些数据和工具验证假设，哪些备选方案被否决及原因。
- 同时覆盖正确性、风险控制、沟通和长期预防，体现“落地执行”而不只是技术名词。
- 结果尽量量化，但只能使用真实指标；无法披露数字时可描述相对改善和验证方法。

---

## 九、面试收尾：反问与准备清单

### 可以反问面试官的问题

1. 团队当前在 Azure 上最关键的业务链路是什么？实时事件处理、数据平台和 Web API 的比例如何？
2. Functions、App Service、ADF 分别承担哪些职责？目前最大的稳定性、成本或交付挑战是什么？
3. 团队的发布流程是否使用 IaC、部署槽、自动化测试和可观测性门禁？
4. 前后端 API 契约如何管理？是否已有 OpenAPI、契约测试和版本弃用规范？
5. 这个岗位在前三个月最希望解决的具体问题和衡量标准是什么？

### 面试前准备清单

- [ ] 准备 2 个真实项目的架构图：入口、认证、数据流、异步链路、监控与故障路径。
- [ ] 为每项 Azure 服务准备“为什么选它、如何安全访问、如何扩缩、失败如何恢复、如何监控”的回答。
- [ ] 准备 1 个性能问题、1 个线上故障、1 个跨团队协作和 1 个技术选型案例，均用 STAR 结构表达。
- [ ] 复习 C# 异步、DI 生命周期、EF Core 查询与并发、ASP.NET Core 认证授权及 API 设计。
- [ ] 能在白板上画出：SPA -> API -> Blob/Queue/Function -> 数据库/ADF -> 监控告警 的端到端架构。
- [ ] 确认所有项目数字、职责边界和技术细节真实可追溯；不会的内容坦诚说明，并展示学习与验证方法。

---

## 快速答题原则

优秀回答通常遵循以下顺序：

1. **先给结论**：用一句话说明选择或判断。
2. **讲关键权衡**：性能、可靠性、安全、成本、交付速度之间如何取舍。
3. **落到实现**：说明 Azure 服务、.NET 机制、数据模型或流程如何协同。
4. **说明失败路径**：重试、幂等、死信、回滚、监控与告警。
5. **用真实结果收尾**：量化指标、验证方式或可复盘的业务价值。

掌握“为什么这样设计、失败时怎么办、如何证明它有效”，比背诵服务定义更能体现 .NET + Azure 全栈工程师的实战能力。
