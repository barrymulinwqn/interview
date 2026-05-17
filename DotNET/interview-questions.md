# .NET 全栈 / SharePoint / Vue 面试题汇总

> 涵盖 .NET 全栈工程师、SharePoint 开发、Vue.js 前端三大方向，每题附详细解释。

---

## 目录

1. [.NET 基础与 C#](#一net-基础与-c)
2. [ASP.NET Core / Web API](#二aspnet-core--web-api)
3. [数据库与 ORM (EF Core)](#三数据库与-orm-ef-core)
4. [微服务与架构设计](#四微服务与架构设计)
5. [SharePoint 开发](#五sharepoint-开发)
6. [Vue.js 前端](#六vuejs-前端)
7. [综合全栈场景题](#七综合全栈场景题)

---

## 一、.NET 基础与 C#

### Q1：值类型和引用类型的区别？

**答：**

| 特性 | 值类型 | 引用类型 |
|------|--------|---------|
| 存储位置 | 栈（Stack） | 堆（Heap），栈上存引用 |
| 赋值行为 | 复制整个值 | 复制引用（指针） |
| 默认值 | 0 / false / struct 默认值 | null |
| 代表类型 | int, double, bool, struct, enum | class, string, array, delegate |
| 性能 | 分配/释放快 | GC 管理，开销稍大 |

**详细解释：**  
值类型变量直接包含数据，赋值或传参时会产生独立副本，互不影响。引用类型变量保存的是对象在堆上的地址，多个变量可以指向同一个对象，修改一个会影响另一个。  
`string` 虽是引用类型，但因其不可变性（immutable），表现上类似值类型。

---

### Q2：`ref`、`out`、`in` 关键字的区别？

**答：**

- **`ref`**：传引用，调用前必须初始化，方法内可读可写。
- **`out`**：传引用，调用前不需要初始化，方法内必须赋值后才能返回。
- **`in`**（C# 7.2+）：传只读引用，避免大型 struct 复制，方法内不允许修改。

```csharp
void Demo(ref int a, out int b, in int c)
{
    a = a + 1;   // 可修改
    b = 10;      // 必须赋值
    // c = 5;    // 编译错误，只读
}
```

**详细解释：**  
`out` 常用于 `TryParse` 等方法返回多个值；`ref` 用于需要回写的场景；`in` 主要用于性能优化，避免复制大型 struct。

---

### Q3：什么是装箱（Boxing）和拆箱（Unboxing）？有何性能影响？

**答：**  
- **装箱**：将值类型转换为 `object` 或接口类型，会在堆上分配内存并复制数据。  
- **拆箱**：将 `object` 显式转换回值类型，需要类型检查和数据复制。

```csharp
int i = 42;
object o = i;       // 装箱
int j = (int)o;     // 拆箱
```

**性能影响：**  
频繁装拆箱会造成大量堆内存分配，增加 GC 压力，在热路径（高频调用）中应使用泛型（`List<int>` 而非 `ArrayList`）来避免。

---

### Q4：解释 `async/await` 的工作原理，`Task` 与 `Thread` 的区别？

**答：**

**async/await 原理：**  
编译器将 `async` 方法转换为状态机（State Machine）。遇到 `await` 时，方法"挂起"并将控制权交还给调用方，不阻塞线程；当等待的操作完成后，状态机继续从挂起点执行。

**Task vs Thread：**

| 特性 | Thread | Task |
|------|--------|------|
| 级别 | OS 级线程 | 抽象的异步操作 |
| 资源消耗 | 高（~1MB 栈内存） | 低，由线程池管理 |
| 返回值 | 不直接支持 | `Task<T>` 原生支持 |
| 取消 | 需手动实现 | `CancellationToken` |
| 推荐场景 | CPU 密集型长时任务 | I/O 操作、并发任务 |

```csharp
// 推荐：使用 async/await + Task
public async Task<string> GetDataAsync()
{
    var result = await httpClient.GetStringAsync("https://api.example.com");
    return result;
}
```

---

### Q5：什么是委托（Delegate）、事件（Event）、Lambda 表达式？

**答：**

- **委托**：类型安全的函数指针，可封装方法引用。
- **事件**：基于委托的发布-订阅模式，`event` 关键字限制外部只能 `+=` / `-=`，不能直接调用。
- **Lambda**：匿名方法的简洁语法，本质上被编译为委托实例。

```csharp
// 委托
Func<int, int, int> add = (a, b) => a + b;

// 事件
public event EventHandler<DataEventArgs> DataReceived;
DataReceived?.Invoke(this, new DataEventArgs { Value = 42 });
```

---

### Q6：解释 `IDisposable` 与 `using` 语句，什么时候需要实现它？

**答：**  
当类持有非托管资源（文件句柄、数据库连接、网络连接等）时需实现 `IDisposable`。`using` 语句保证无论是否发生异常，`Dispose()` 都会被调用。

```csharp
// 标准实现模式（Dispose Pattern）
public class ResourceHolder : IDisposable
{
    private bool _disposed = false;
    private FileStream _stream;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
                _stream?.Dispose(); // 释放托管资源
            // 释放非托管资源
            _disposed = true;
        }
    }
}

// 使用
using var holder = new ResourceHolder();
```

---

### Q7：LINQ 的延迟执行（Deferred Execution）是什么？

**答：**  
LINQ 查询在定义时不会立即执行，只有在实际枚举结果时（如 `foreach`、`.ToList()`、`.Count()` 等）才执行。这称为延迟执行。

```csharp
var query = list.Where(x => x > 5); // 此时不执行
// ...修改 list 的代码...
var result = query.ToList(); // 此时才执行，会包含修改后的数据
```

**好处：** 可以组合多个操作而只遍历一次；**风险：** 多次枚举同一查询会多次执行，应及时 `.ToList()` 缓存。

---

## 二、ASP.NET Core / Web API

### Q8：ASP.NET Core 的请求处理管道（Middleware Pipeline）是如何工作的？

**答：**  
中间件是按注册顺序形成一条管道，每个中间件可以：
1. 在调用 `next()` 之前处理请求
2. 调用 `next()` 将控制权传递给下一个中间件
3. 在 `next()` 返回后处理响应

```csharp
// Program.cs
app.UseHttpsRedirection();   // 中间件1
app.UseAuthentication();     // 中间件2
app.UseAuthorization();      // 中间件3
app.MapControllers();        // 终端中间件

// 自定义中间件
app.Use(async (context, next) =>
{
    // 请求前处理
    Console.WriteLine("Before");
    await next(context);
    // 响应后处理
    Console.WriteLine("After");
});
```

**注意顺序很重要**：例如 `UseAuthentication` 必须在 `UseAuthorization` 之前。

---

### Q9：依赖注入（DI）的三种生命周期区别？

**答：**

| 生命周期 | 注册方法 | 创建时机 | 典型场景 |
|---------|---------|---------|---------|
| Transient | `AddTransient` | 每次注入都创建新实例 | 无状态服务、轻量级操作 |
| Scoped | `AddScoped` | 每个 HTTP 请求一个实例 | DbContext、当前用户上下文 |
| Singleton | `AddSingleton` | 应用启动时创建，全程共享 | 配置、缓存、日志 |

**陷阱：** 不要在 Singleton 中注入 Scoped 服务（Captive Dependency），会导致 Scoped 服务被长期持有，引发数据污染问题。

---

### Q10：RESTful API 设计原则是什么？如何实现版本控制？

**答：**

**RESTful 原则：**
- 无状态（Stateless）：每个请求独立，服务器不保存会话状态
- 统一接口：使用标准 HTTP 方法（GET/POST/PUT/DELETE/PATCH）
- 资源导向：URL 表示资源，不是动作（`/users/1` 而非 `/getUser?id=1`）
- 合理使用 HTTP 状态码（200/201/400/401/403/404/500）

**版本控制方式：**

```csharp
// 方式1：URL 路径
[Route("api/v1/[controller]")]

// 方式2：查询参数
// GET /api/users?api-version=1.0

// 方式3：请求头
// api-version: 1.0

// 使用 Microsoft.AspNetCore.Mvc.Versioning
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});
```

---

### Q11：JWT 认证的工作原理？

**答：**  
JWT（JSON Web Token）由三部分组成：`Header.Payload.Signature`

1. 用户登录 → 服务端验证凭据 → 生成 JWT（包含用户信息和过期时间）→ 返回客户端
2. 客户端存储 JWT（通常在 localStorage 或 httpOnly Cookie）
3. 后续请求在 `Authorization: Bearer <token>` 头中携带
4. 服务端验证签名有效性和过期时间，无需查数据库

```csharp
// 生成 JWT
var token = new JwtSecurityToken(
    issuer: "myapp",
    audience: "myapp",
    claims: new[] { new Claim(ClaimTypes.Name, user.Name) },
    expires: DateTime.Now.AddHours(1),
    signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256)
);
```

**安全注意：** 
- 不要在 Payload 中存放敏感信息（Payload 只是 Base64 编码，非加密）
- 使用足够强的密钥（256 位以上）
- 设置合理的过期时间，配合 Refresh Token 使用

---

## 三、数据库与 ORM (EF Core)

### Q12：EF Core 中 Code First 迁移的工作流程？

**答：**

```bash
# 1. 创建迁移
dotnet ef migrations add InitialCreate

# 2. 查看生成的 SQL（可选）
dotnet ef migrations script

# 3. 应用迁移
dotnet ef database update

# 4. 回滚到指定迁移
dotnet ef database update PreviousMigrationName
```

**详细解释：**  
EF Core 比较当前模型与上一次迁移快照，生成差异的迁移文件（`Up` 方法执行变更，`Down` 方法回滚）。迁移历史记录在数据库的 `__EFMigrationsHistory` 表中。

---

### Q13：N+1 查询问题是什么？如何解决？

**答：**  
N+1 问题：查询 N 条主记录，再对每条记录额外执行 1 次查询获取关联数据，共执行 N+1 次查询。

```csharp
// ❌ N+1 问题
var orders = context.Orders.ToList();
foreach (var order in orders)
{
    var customer = order.Customer; // 每次触发一次 SQL 查询
}

// ✅ 使用 Include 预加载（Eager Loading）
var orders = context.Orders
    .Include(o => o.Customer)
    .ToList(); // 只执行 1 次 JOIN 查询

// ✅ 或使用显式加载（需要时才加载）
context.Entry(order).Reference(o => o.Customer).Load();

// ✅ 或使用投影避免加载不需要的字段
var orders = context.Orders
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .ToList();
```

---

### Q14：数据库事务与 EF Core 中的使用方式？

**答：**

```csharp
// 方式1：SaveChanges 内置事务（同一 DbContext 的操作自动在事务中）
context.Orders.Add(order);
context.Payments.Add(payment);
await context.SaveChangesAsync(); // 原子性提交

// 方式2：显式事务（跨多次 SaveChanges）
using var transaction = await context.Database.BeginTransactionAsync();
try
{
    context.Orders.Add(order);
    await context.SaveChangesAsync();

    context.Invoices.Add(invoice);
    await context.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

---

## 四、微服务与架构设计

### Q15：SOLID 原则是什么？

**答：**

| 原则 | 全称 | 核心思想 |
|------|------|---------|
| S | Single Responsibility | 一个类只做一件事 |
| O | Open/Closed | 对扩展开放，对修改关闭 |
| L | Liskov Substitution | 子类可以替换父类而不破坏程序 |
| I | Interface Segregation | 接口应小而专，不强迫实现不需要的方法 |
| D | Dependency Inversion | 依赖抽象而非具体实现 |

**示例（D 原则）：**
```csharp
// ❌ 违反 DIP
public class OrderService
{
    private SqlOrderRepository _repo = new SqlOrderRepository(); // 依赖具体实现
}

// ✅ 遵循 DIP
public class OrderService
{
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo; // 依赖抽象
}
```

---

### Q16：什么是 CQRS 模式？适用场景？

**答：**  
CQRS（Command Query Responsibility Segregation）将读（Query）和写（Command）操作分离到不同的模型中。

- **Command**：改变系统状态，不返回数据（或只返回操作结果）
- **Query**：读取数据，不改变系统状态

**适用场景：**  
- 读写比例差异极大的系统
- 读模型和写模型优化方向不同
- 结合 Event Sourcing 使用

```csharp
// MediatR 实现 CQRS
public record GetOrderQuery(int Id) : IRequest<OrderDto>;
public record CreateOrderCommand(OrderDto Dto) : IRequest<int>;

public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    public async Task<OrderDto> Handle(GetOrderQuery request, CancellationToken ct)
        => await _readDb.Orders.FindAsync(request.Id);
}
```

---

## 五、SharePoint 开发

### Q17：SharePoint Framework (SPFx) 的架构是什么？

**答：**  
SPFx 是微软官方的 SharePoint 客户端开发框架，基于现代 Web 技术：

- **运行时**：浏览器端执行，无需服务器端代码
- **技术栈**：TypeScript + React/Vue/Angular（任选），webpack 打包
- **开发组件类型**：
  - **Web Parts**：页面上的可视化组件（类似 Webpart 的现代版）
  - **Extensions**：Application Customizer（页眉/页脚）、Field Customizer、Command Sets
  - **Adaptive Card Extensions（ACE）**：Viva Connections 卡片

**开发流程：**
```bash
# 1. 创建项目
yo @microsoft/sharepoint

# 2. 本地调试（连接真实 SharePoint）
gulp serve

# 3. 打包
gulp bundle --ship
gulp package-solution --ship

# 4. 部署到应用目录（App Catalog）
```

---

### Q18：SPFx 中如何调用 SharePoint REST API 和 Microsoft Graph API？

**答：**

```typescript
// 调用 SharePoint REST API（使用 SPHttpClient）
import { SPHttpClient, SPHttpClientResponse } from '@microsoft/sp-http';

const response: SPHttpClientResponse = await this.context.spHttpClient.get(
  `${this.context.pageContext.web.absoluteUrl}/_api/web/lists`,
  SPHttpClient.configurations.v1
);
const data = await response.json();

// 调用 Microsoft Graph API（使用 MSGraphClientV3）
const graphClient = await this.context.msGraphClientFactory.getClient('3');
const me = await graphClient.api('/me').get();

// 调用 PnPjs（推荐，封装更友好）
import { spfi, SPFx } from "@pnp/sp";
import "@pnp/sp/webs";
import "@pnp/sp/lists";

const sp = spfi().using(SPFx(this.context));
const lists = await sp.web.lists();
```

**权限配置：** 在 `package-solution.json` 中声明所需 API 权限，租户管理员需在 SharePoint 管理中心审批。

---

### Q19：SharePoint 内容类型（Content Type）和列表模板有什么区别？

**答：**

| 特性 | 内容类型（Content Type） | 列表模板（List Template） |
|------|------------------------|------------------------|
| 定义 | 列/行为的可重用集合 | 预配置的列表结构 |
| 继承 | 支持继承层次 | 不支持继承 |
| 跨站点 | 可通过内容类型中心共享 | 只能在同一网站集 |
| 典型用途 | 定义文档类型、元数据字段 | 快速创建标准列表 |

**内容类型示例（XML）：**
```xml
<ContentType ID="0x010100..." Name="Project Document" 
             Group="Custom Content Types">
  <FieldRefs>
    <FieldRef ID="{...}" Name="ProjectCode" Required="TRUE"/>
    <FieldRef ID="{...}" Name="Department"/>
  </FieldRefs>
</ContentType>
```

---

### Q20：什么是 SharePoint 权限管理模型？

**答：**  
SharePoint 采用层次化权限模型：

```
租户 → 网站集（Site Collection）→ 网站（Site）→ 列表/库 → 列表项/文档
```

- **权限继承**：默认情况下子级继承父级权限
- **权限断开**：可在任意级别打断继承，单独设置权限
- **内置权限级别**：完全控制、设计、编辑、参与、读取、受限读取

**代码操作权限（CSOM）：**
```csharp
// 打断权限继承
list.BreakRoleInheritance(copyRoleAssignments: false, clearSubscopes: true);
ctx.ExecuteQuery();

// 授予权限
var user = ctx.Web.EnsureUser("user@domain.com");
var roleDefinition = ctx.Web.RoleDefinitions.GetByName("Contribute");
var binding = new RoleDefinitionBindingCollection(ctx) { roleDefinition };
list.RoleAssignments.Add(user, binding);
ctx.ExecuteQuery();
```

---

### Q21：SharePoint 搜索架构与自定义搜索方案？

**答：**  
SharePoint 搜索核心组件：
- **爬网组件**（Crawler）：抓取内容
- **内容处理组件**：提取、处理、丰富内容
- **索引组件**：存储倒排索引
- **查询处理组件**：解析并优化查询
- **分析处理组件**：使用行为数据提升相关性

**常见自定义方案：**
```typescript
// KQL（Keyword Query Language）搜索
const searchResults = await sp.search({
  Querytext: "ContentType:\"Project Document\" AND Department:IT",
  SelectProperties: ["Title", "Author", "ProjectCode"],
  SortList: [{ Property: "LastModifiedTime", Direction: 1 }],
  RowLimit: 50,
  RefinementFilters: ["Department:\"IT\""]
});
```

---

## 六、Vue.js 前端

### Q22：Vue 2 和 Vue 3 的核心区别？

**答：**

| 特性 | Vue 2 | Vue 3 |
|------|-------|-------|
| 响应式系统 | `Object.defineProperty` | `Proxy`（支持数组/动态属性） |
| API 风格 | Options API | Composition API + Options API |
| 性能 | 基准 | 比 Vue 2 快 ~1.3-2x，内存减少 ~50% |
| TypeScript | 有限支持 | 原生支持 |
| Tree-shaking | 不支持 | 支持（按需导入） |
| Fragment | 不支持（必须单根） | 支持多根节点 |
| Teleport | 无 | 内置 `<Teleport>` |
| 生命周期钩子 | `beforeDestroy/destroyed` | `onBeforeUnmount/onUnmounted` |

---

### Q23：Vue 3 Composition API 与 Options API 的对比？何时选用？

**答：**

```vue
<!-- Options API -->
<script>
export default {
  data() { return { count: 0 } },
  computed: {
    doubled() { return this.count * 2 }
  },
  methods: {
    increment() { this.count++ }
  },
  mounted() { console.log('mounted') }
}
</script>

<!-- Composition API (setup 语法糖) -->
<script setup>
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
const increment = () => count.value++
onMounted(() => console.log('mounted'))
</script>
```

**选用建议：**
- **Options API**：适合简单组件、团队有 Vue 2 背景
- **Composition API**：适合复杂逻辑复用、大型项目、TypeScript 项目，逻辑可按功能组织而非按选项类型散落

---

### Q24：Vue 响应式原理（Vue 3 Proxy 机制）？

**答：**  
Vue 3 使用 `Proxy` 拦截对象的 `get` 和 `set` 操作：

```javascript
// 简化版响应式原理
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key, receiver) {
      track(target, key);  // 收集依赖（记录哪个 effect 在读取这个属性）
      return Reflect.get(target, key, receiver);
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver);
      trigger(target, key); // 触发更新（通知所有依赖此属性的 effect 重新执行）
      return result;
    }
  });
}
```

**相比 Vue 2 `Object.defineProperty` 的优势：**
- 可检测数组索引变化和 `length` 变化
- 可检测新增/删除属性（Vue 2 需要 `Vue.set`）
- 惰性代理，性能更好

---

### Q25：Vue 3 中 `ref` 和 `reactive` 的区别？

**答：**

| 特性 | `ref` | `reactive` |
|------|-------|-----------|
| 适用类型 | 任意类型（基本类型必须用 ref） | 仅对象/数组 |
| 访问值 | 需要 `.value`（模板中自动解包） | 直接访问属性 |
| 解构 | 解构后保持响应式 | 解构后**丢失**响应式 |
| 替换整体 | 可以（`ref.value = newObj`） | 不可以（会丢失响应式） |

```javascript
const count = ref(0)
count.value++  // 需要 .value

const state = reactive({ count: 0, name: 'Alice' })
state.count++  // 直接访问

// ❌ reactive 解构丢失响应式
const { count } = state  // count 不再响应式

// ✅ 使用 toRefs 保持响应式
const { count } = toRefs(state)
```

---

### Q26：Vue 的虚拟 DOM（Virtual DOM）和 Diff 算法？

**答：**  
虚拟 DOM 是真实 DOM 的 JavaScript 对象描述，渲染时先比较新旧 VDOM 差异（Diff），只更新变化的部分。

**Vue 3 Diff 算法优化（快速 Diff）：**
1. **头尾双指针预处理**：先处理相同的头部和尾部节点
2. **最长递增子序列（LIS）**：对中间乱序节点，找到最长稳定序列，最小化 DOM 移动次数
3. **静态提升（Static Hoisting）**：编译时将静态节点提升为常量，跳过 Diff

```javascript
// key 的重要性：帮助 Diff 算法识别节点身份
// ❌ 不要用 index 作为 key（列表增删时导致错误复用）
<li v-for="(item, index) in list" :key="index">

// ✅ 用唯一稳定 ID
<li v-for="item in list" :key="item.id">
```

---

### Q27：Pinia vs Vuex 的区别？Pinia 基本使用？

**答：**

| 特性 | Vuex 4 | Pinia |
|------|--------|-------|
| API 复杂度 | 高（mutations/actions/getters 分离） | 低（直接修改 state） |
| TypeScript | 需要额外配置 | 原生支持 |
| DevTools | 支持 | 支持 |
| 模块化 | 嵌套模块，命名空间复杂 | 扁平 Store，天然模块化 |
| Vue 3 推荐 | 不推荐（官方推荐 Pinia） | ✅ 官方推荐 |

```typescript
// 定义 Store
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({ name: '', isLoggedIn: false }),
  getters: {
    displayName: (state) => state.name || 'Guest'
  },
  actions: {
    async login(credentials: Credentials) {
      const user = await authApi.login(credentials)
      this.name = user.name
      this.isLoggedIn = true
    }
  }
})

// 组件中使用
const userStore = useUserStore()
userStore.login({ username: 'alice', password: '***' })
console.log(userStore.displayName)
```

---

### Q28：Vue Router 的导航守卫有哪些？如何实现权限控制？

**答：**

```typescript
// 全局前置守卫（最常用）
router.beforeEach(async (to, from) => {
  const authStore = useAuthStore()
  
  if (to.meta.requiresAuth && !authStore.isLoggedIn) {
    return { name: 'Login', query: { redirect: to.fullPath } }
  }
  
  if (to.meta.roles && !authStore.hasRole(to.meta.roles)) {
    return { name: 'Forbidden' }
  }
})

// 路由配置
const routes = [
  {
    path: '/admin',
    component: AdminLayout,
    meta: { requiresAuth: true, roles: ['Admin'] },
    children: [...]
  }
]

// 组件内守卫
onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges.value) {
    return confirm('确定要离开吗？未保存的更改将丢失。')
  }
})
```

---

### Q29：Vue 3 性能优化有哪些手段？

**答：**

```vue
<!-- 1. v-memo：跳过不必要的子树更新 -->
<div v-for="item in list" :key="item.id" v-memo="[item.selected]">
  <!-- 只有 item.selected 变化时才重新渲染 -->
</div>

<!-- 2. v-once：只渲染一次（静态内容） -->
<header v-once>{{ staticTitle }}</header>

<!-- 3. shallowRef / shallowReactive：浅层响应式（大型数据） -->
<script setup>
import { shallowRef, defineAsyncComponent } from 'vue'

// 4. 异步组件：按需加载，代码分割
const HeavyComponent = defineAsyncComponent(() =>
  import('./HeavyComponent.vue')
)

// 5. 大列表虚拟滚动（vue-virtual-scroller）
</script>
```

**其他优化：**
- 使用 `computed` 缓存复杂计算
- 合理使用 `watchEffect` vs `watch`（避免不必要的深度监听）
- 生产构建开启 Tree-shaking 和代码压缩

---

## 七、综合全栈场景题

### Q30：设计一个基于 .NET + Vue + SharePoint 的企业文档管理系统，架构如何设计？

**答：**

```
┌─────────────────────────────────────────────────────┐
│  前端（Vue 3 + TypeScript + Vite）                    │
│  - Pinia 状态管理                                     │
│  - Vue Router 权限控制                                │
│  - SPFx Web Part（嵌入 SharePoint 页面）              │
└──────────────┬──────────────────────────────────────┘
               │ REST API / SignalR
┌──────────────▼──────────────────────────────────────┐
│  后端（ASP.NET Core 8 Web API）                       │
│  - Clean Architecture（Domain/Application/Infra）    │
│  - MediatR（CQRS）                                   │
│  - JWT + Azure AD 认证                               │
│  - EF Core + SQL Server                              │
└──────────────┬──────────────────────────────────────┘
               │
       ┌───────┴────────┐
       │                │
┌──────▼──────┐  ┌──────▼──────────────────┐
│ SQL Server  │  │ SharePoint Online        │
│ - 业务数据   │  │ - 文档存储（Document Library）│
│ - 用户权限   │  │ - 元数据（Content Types）  │
└─────────────┘  │ - 搜索索引               │
                 └──────────────────────────┘
```

**关键设计决策：**
1. **认证**：Azure AD SSO，前后端统一身份，SharePoint 权限与系统权限同步
2. **文件存储**：SharePoint Document Library（利用其版本控制、权限管理、搜索能力）
3. **元数据**：Content Type 定义文档分类字段，通过 Graph API 读写
4. **搜索**：调用 SharePoint Search API 或 Microsoft Search，支持全文检索
5. **通知**：SignalR 实时通知 + SharePoint Webhook 监听文件变更

---

### Q31：如何排查和解决 .NET 应用的内存泄漏？

**答：**

**常见内存泄漏原因：**
1. 事件订阅未取消（`+=` 忘记 `-=`）
2. 静态集合无限增长
3. 未实现 `IDisposable` 或未调用 `Dispose`
4. 闭包捕获了大对象
5. 缓存没有过期策略

**排查步骤：**
```bash
# 1. 收集内存转储（dotnet-dump）
dotnet-dump collect -p <pid>

# 2. 分析转储
dotnet-dump analyze <dump_file>
> dumpheap -stat          # 查看各类型对象数量和大小
> dumpheap -type HttpClient  # 查看特定类型
> gcroot <object_address> # 查找对象的 GC 根引用链

# 3. 使用 dotnet-counters 实时监控
dotnet-counters monitor --process-id <pid> System.Runtime
```

**代码层面预防：**
```csharp
// ✅ 取消事件订阅
public class MyViewModel : IDisposable
{
    public MyViewModel(EventSource source)
    {
        source.DataChanged += OnDataChanged; // 订阅
        _source = source;
    }
    
    public void Dispose()
    {
        _source.DataChanged -= OnDataChanged; // 取消订阅
    }
}

// ✅ 使用 WeakReference 避免强引用持有
var weakRef = new WeakReference<ExpensiveObject>(expensiveObj);
```

---

### Q32：前后端接口联调时常见问题及解决方案？

**答：**

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| CORS 跨域 | 浏览器同源策略 | 后端配置 `AddCors`，正确设置允许的 Origin |
| 401 Unauthorized | Token 未携带或过期 | 检查请求头 Authorization，实现 Token 自动刷新 |
| 接口字段大小写不一致 | .NET 默认 PascalCase，JS 期望 camelCase | 配置 `JsonNamingPolicy.CamelCase` |
| 日期时区错误 | 时区不一致 | 统一使用 UTC，前端本地化显示 |
| 大文件上传失败 | 默认请求体大小限制 | 配置 `MaxRequestBodySize`，使用流式上传 |

```csharp
// 解决字段命名和时区问题
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
    });

// CORS 配置
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("https://myapp.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials());
});
```

---

## 附录：面试准备建议

### 技术深度检验点

- **.NET**：异步编程、GC 原理、EF Core 性能优化、依赖注入容器实现原理
- **SharePoint**：SPFx 调试技巧、Graph API 权限申请流程、PnP 库使用、Modern vs Classic 差异
- **Vue**：响应式原理、编译优化（静态提升、Patch Flag）、SSR（Nuxt）、微前端集成

### 常见行为问题

---

#### B1：描述一次你解决复杂技术问题的经历（STAR 法则）

> **STAR 法则**：Situation（背景）→ Task（任务）→ Action（行动）→ Result（结果）

**参考方向（结合 .NET / SharePoint 背景）：**

**Situation（背景）**  
在一个企业内部文档审批系统中，测试环境一切正常，但上线后生产环境每隔约 2 小时就会出现 SharePoint REST API 批量调用失败，返回 `429 Too Many Requests`，导致整个审批流程中断。业务方强烈反映，每天有数十个审批流程被卡死。

**Task（任务）**  
作为负责后端的开发工程师，需要在不影响现有功能的前提下，定位根因并给出可靠的长期解决方案，且要在 3 个工作日内修复上线。

**Action（行动）**
1. **定位根因**：通过 Application Insights 查询请求日志，发现在每 2 小时的批量定时任务执行期间，对 `/_api/web/lists` 的并发请求在短时间内超过了 SharePoint Online 的限流阈值（每 10 分钟 6000 次请求/租户）。
2. **短期止血**：在 SPHttpClient 调用层包裹指数退避重试逻辑（使用 Polly 库），对 `429` 响应自动读取 `Retry-After` 响应头并等待对应时间后重试。
3. **根本优化**：
   - 将并发批量请求改为顺序分批（每批 50 条，批次间加 500ms 延迟）
   - 对高频只读接口（如列表结构、内容类型）引入内存缓存（`IMemoryCache`，TTL 10 分钟），减少重复调用
   - 将定时任务从每 2 小时执行一次拆分为持续低频增量同步
4. **验证**：在预发布环境用压测工具模拟生产流量，确认修复效果后灰度上线。

**Result（结果）**  
上线后 API 限流报错降至零，审批流程中断事件消除。请求量峰值降低了约 70%，系统整体响应时间提升 30%。此方案后来被整理为团队的 SharePoint API 调用规范文档。

**面试表达要点：**
- 强调**数据驱动**：用日志、监控数据定位问题，而不是凭感觉
- 展示**系统思维**：不只是修 Bug，而是同时考虑短期止血和长期优化
- 体现**业务意识**：将技术影响翻译为业务损失（审批中断数量）

---

#### B2：你如何在技术债和新功能之间平衡？

**参考方向：**

**核心立场：技术债不是"要不要还"的问题，而是"何时还、如何还"的优先级管理问题。**

**1. 先把技术债可视化、量化**
- 将技术债整理到 Backlog，明确标注：影响范围、修复成本、不修的风险（如安全漏洞、性能瓶颈、阻碍新功能开发）
- 区分类型：有意的技术债（为赶 deadline 有意为之）vs 无意的技术债（设计不当积累）

**2. 用"利息"概念与业务方沟通**
> "这块代码每次加新功能要多花 2 天调试，每个 Sprint 因此损失约 3 天。现在花 5 天重构，6 周后就回本。"

将抽象的"代码质量"转化为具体的时间成本，让非技术 stakeholder 理解还债的投资回报。

**3. 实际执行策略**

| 策略 | 适用场景 |
|------|---------|
| **Boy Scout Rule**（童子军原则） | 随手修——每次改动相关模块时，顺手改善周边代码 |
| **20% 时间分配** | 每个 Sprint 预留约 20% 容量专项处理技术债 |
| **专项 Sprint** | 技术债积累到严重影响效率时，安排 1-2 个专项清理 Sprint |
| **借新功能机会重构** | 新功能涉及的老模块，在做新功能时一并重构，不单独立项 |
| **绝不碰的原则** | 核心路径上的安全漏洞、已知会导致数据丢失的 Bug，必须立即处理，不做权衡 |

**4. 建立预防机制**
- Code Review 时设置质量门槛，防止新债产生
- 通过 SonarQube、NDepend 等工具量化技术健康指标，纳入团队 Dashboard
- 架构决策记录（ADR）：记录当初为什么妥协，后续还债时有据可查

**面试表达要点：**
- 避免两个极端："永远先还债再开发" 或 "技术债以后再说"
- 强调**沟通能力**：能把技术问题翻译成业务语言
- 展示**主动性**：不是被动等别人安排，而是主动推动技术债管理

---

#### B3：描述一次你与团队意见不合的经历，如何处理？

**参考方向：**

**场景示例：技术选型分歧**

**背景：**  
团队在讨论一个新的内部门户项目的前端框架时，大多数成员（包括团队 Lead）倾向于继续用 Vue 2（因为团队已有经验），而我认为应该直接用 Vue 3 + TypeScript，因为 Vue 2 已进入 EOL（End of Life）阶段，长期维护成本更高。

**我的处理过程：**

1. **先理解对方立场，而非急于说服**  
   私下与团队 Lead 沟通，了解他倾向 Vue 2 的深层原因：担心 Vue 3 学习成本影响交付 deadline，以及现有 Vue 2 组件库迁移工作量。这让我认识到争论不是"谁对谁错"，而是"不同因素的权重如何取舍"。

2. **用数据和具体方案代替主观判断**  
   整理了一份对比文档：
   - Vue 2 EOL 时间表及安全风险
   - Vue 3 学习曲线评估（基于团队现有 Vue 2 经验，预估 1 周上手）
   - 提出过渡方案：使用 `@vue/compat` 兼容模式，支持渐进迁移

3. **提议小范围验证，降低决策风险**  
   建议用第一个模块（非关键路径）作为 Vue 3 试点，两周后评估，再决定整体方案。

4. **尊重最终决定并全力执行**  
   团队最终采纳了试点方案。试点进展顺利后，全员切换 Vue 3。

**结果：**  
项目按时交付，且因 TypeScript 的引入，后续有 3 个原本难以定位的 Bug 在编译阶段就被发现。团队 Lead 事后在复盘中表示这次选型决策是项目中做得最好的地方之一。

**面试表达要点：**
- **避免把自己塑造成"我是对的，我说服了所有人"** 的英雄叙事
- 强调：**先理解 → 再提方案 → 允许验证 → 尊重结果**
- 展示成熟度：即使最终结果不是自己的方案，也能全力执行

---

#### B4：如何保持技术更新，跟进最新的框架和工具？

**参考方向：**

**核心原则：主动、有选择性、系统化地学习，而非被动刷信息流。**

**1. 建立信息输入体系（每周固定时间）**

| 类型 | 具体资源 |
|------|---------|
| 官方动态 | .NET Blog、Vue.js Blog、Microsoft 365 Dev Blog、GitHub Releases |
| 技术社区 | Stack Overflow 周报、CSS-Tricks、DEV.to、掘金 |
| 播客/视频 | .NET Rocks、Vue Mastery、Channel 9、YouTube: Nick Chapsas |
| Newsletter | The Morning Brew(.NET)、VueDose、This Week in React |
| 书籍/课程 | 每季度至少精读 1 本技术书，优先选官方文档而非二手资料 |

**2. 学以致用：用副项目验证新技术**
- 不只是"看完觉得会了"，而是用新技术做一个小工程（哪怕是 Todo App）
- 例如：Microsoft Graph SDK 新版本发布时，用它重写一个内部工具来实践
- 将学习心得整理成技术笔记或博客（教别人是最好的学习方式）

**3. 技术雷达思维（借鉴 ThoughtWorks Technology Radar）**

将技术分为四个象限来决策是否投入时间：

```
采纳（Adopt）   ← 已经成熟，应该在项目中使用（如 .NET 8、Vue 3）
试验（Trial）   ← 值得在项目中小规模尝试（如 .NET Aspire）
评估（Assess）  ← 值得关注，但暂不引入生产（如新的 AI SDK）
暂缓（Hold）    ← 暂不推荐（如 Vue 2 新项目）
```

**4. 利用工作场景驱动学习**
- 遇到性能问题 → 深入研究 EF Core 查询优化和 SQL 执行计划
- SharePoint 新功能上线 → 在测试租户中第一时间体验 Viva Connections、Copilot in SharePoint
- 团队引入新工具时，主动担任"踩坑人"，输出使用报告

**5. 参与技术社区**
- GitHub 关注核心库（`dotnet/aspnetcore`、`vuejs/core`），阅读 PR 和 Issue 了解设计思路
- 参与 Microsoft Q&A、Stack Overflow 回答问题（输出倒逼输入）
- 技术分享：在团队内做 Tech Talk，定期分享新技术调研结论

**面试表达要点：**
- 不要泛泛说"我经常看文档"，要给出**具体的资源、具体的频率、具体的案例**
- 强调**有取舍**：技术更新太快，需要判断哪些值得投入，哪些可以观望
- 体现**分享意识**：个人成长和团队成长结合，才是优秀工程师的特质

---

*最后更新：2026年5月*
