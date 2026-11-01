# Python / Django / PostgreSQL 高频面试题与实战详解

> 面向有实际 Web 后端经验的 Python 工程师、Django 开发者和技术负责人。重点不是罗列 API，而是说明请求如何安全地读写数据、并发冲突如何处理、慢查询如何定位，以及系统如何平滑上线。
>
> 示例以现代 Django 和 PostgreSQL 为背景；部分 API 会随 Django/PostgreSQL 版本演进，生产项目应以锁定版本的官方文档和实测结果为准。代码片段相互独立，省略了与问题无关的字段和导入。

## 目录

1. [面试作答框架与示例系统](#一面试作答框架与示例系统)
2. [Django 高频问题](#二django-高频问题)
3. [PostgreSQL 高频问题](#三postgresql-高频问题)
4. [综合场景题](#四综合场景题)
5. [面试速查清单](#五面试速查清单)

---

## 一、面试作答框架与示例系统

### 1.1 怎样回答工程类问题

一个成熟的回答通常包含以下层次：

1. **先说明约束**：数据规模、并发量、正确性要求、延迟目标、部署拓扑和 Django/PostgreSQL 版本。
2. **给出默认方案**：先讲清楚最简单可行的实现和它适用的条件。
3. **说明正确性边界**：事务范围、并发冲突、异常回滚、幂等和数据完整性由谁保证。
4. **说明性能依据**：展示如何测量查询、索引和锁，而不是凭感觉加缓存或索引。
5. **说明上线与恢复**：迁移是否兼容旧版本、如何监控、如何回滚或重试。

面试时尽量区分“应用层校验”和“数据库最终约束”，也要区分“任务已提交”和“外部副作用已可靠投递”。

### 1.2 示例业务：电商订单服务

本文以一个 Django 电商后端为例：

- 用户浏览商品、查看订单。
- 创建订单时扣减库存，避免超卖。
- 订单提交成功后发送邮件或投递物流任务。
- 订单和明细存放在 PostgreSQL，热点读取可使用缓存。

典型链路：

```text
HTTP 请求
  -> Django URL / Middleware / View
  -> 参数与权限校验
  -> Service 层开启事务
  -> ORM 读写 PostgreSQL
  -> 提交后投递通知任务
  -> 返回响应
```

业务状态、库存和订单必须由数据库事务及约束维护；邮件、支付网关、消息队列等外部操作不能靠数据库事务自动回滚。

---

## 二、Django 高频问题

### Q1：Django 一次请求的处理流程是什么？Middleware、View、Model 各负责什么？

**参考答案：**

请求进入 Django 后，先由 URL resolver 匹配路由，再经过请求阶段的中间件，调用 View。View 负责协调请求、权限、业务服务和响应；Model/ORM 表达持久化数据及查询；响应返回时再经过响应阶段中间件。模板渲染是常见但非必需的步骤，REST API 可以直接返回 JSON。

```text
WSGI/ASGI Server
    -> Middleware（请求方向）
    -> URL Resolver
    -> View -> Service/Domain logic -> ORM -> PostgreSQL
    -> Middleware（响应方向）
    -> HTTP Response
```

**实际应用：**

- 认证、请求 ID、统一安全头适合放在中间件。
- “当前用户是否能取消这笔订单”属于业务授权，应在 View/Service 或权限策略中显式判断，而不是藏在通用中间件。
- 订单创建、扣库存、写支付记录等多步业务逻辑适合放在清晰的 Service 函数中，使其能被 API、后台任务和测试复用。

**常见追问：** 中间件顺序有意义。需要访问用户身份的中间件必须位于认证中间件之后；排查异常时要知道请求和响应方向的执行顺序。同步 WSGI 与异步 ASGI 的部署方式、阻塞调用和并发模型也不能混为一谈。

### Q2：Django `QuerySet` 是什么？它何时真正访问数据库？

**参考答案：**

`QuerySet` 通常是惰性的查询描述。构造和继续过滤一般不会立即执行 SQL；迭代、转换为 `list`、布尔判断、取长度、切片中包含步长等操作可能触发求值。一个已求值的 `QuerySet` 会缓存该实例的结果，但这不是跨请求或跨对象的通用缓存。

```python
orders = Order.objects.filter(status="paid").order_by("-created_at")
# 通常此时还没有执行查询

first_page = list(orders[:20])  # 此处求值并执行查询
```

**实际应用：**

- 在调试日志或模板中意外遍历 `QuerySet`，可能导致原本惰性的查询提前执行。
- 对超大结果集做批处理可使用 `iterator(chunk_size=...)`，避免一次性把所有 ORM 对象放入 Python 内存；需结合驱动、游标和连接池配置验证。
- 不要对同一个大 `QuerySet` 反复调用 `count()`、`len()`、再遍历，先确认各操作会不会多次查询。

**面试重点：** ORM 不是“自动高效 SQL”。理解最终发出的 SQL、执行次数和返回行数，才知道 ORM 写法的实际成本。

### Q3：什么是 N+1 查询？如何使用 `select_related` 和 `prefetch_related`？

**参考答案：**

如果先查出 N 个订单，再逐个访问每个订单的客户或商品，可能产生 1 次主查询加 N 次关联查询，即 N+1 查询。

```python
# 风险写法：遍历时可能为每个订单额外查询 customer
orders = Order.objects.all()
for order in orders:
    print(order.customer.name)

# ForeignKey / OneToOne：通过 JOIN 一次取回关联对象
orders = Order.objects.select_related("customer")

# ManyToMany / 反向 ForeignKey：使用额外批量查询并在 Python 中关联
orders = Order.objects.prefetch_related("items__product")
```

**实际应用：** 订单列表需要显示客户名称和明细商品时：

```python
from django.db.models import Prefetch

orders = (
    Order.objects
    .select_related("customer")
    .prefetch_related(
        Prefetch(
            "items",
            queryset=OrderItem.objects.select_related("product"),
        )
    )
    .order_by("-created_at")[:50]
)
```

`select_related` 适合单值关系，通过 SQL JOIN 扩展行；`prefetch_related` 适合多值关系，通常额外执行查询并在应用层组装。不能机械地预取所有关系：宽 JOIN、大量关联行或不需要的字段也会增加数据库、网络和内存成本。

**如何验证：** 用 Django Debug Toolbar、数据库查询日志或测试中的 `assertNumQueries` 检查请求查询数，并查看 SQL 和返回数据量。

### Q4：`transaction.atomic()` 解决什么问题？什么时候使用 `transaction.on_commit()`？

**参考答案：**

`transaction.atomic()` 将一组数据库操作组成事务：全部成功后提交，异常退出时回滚；嵌套使用通常以保存点实现。事务应尽量短，不能把远程 HTTP 调用、用户输入等待或耗时任务放在持锁事务内。

```python
from django.db import transaction

def create_order(customer, items):
    with transaction.atomic():
        order = Order.objects.create(customer=customer, status="pending")
        OrderItem.objects.bulk_create([
            OrderItem(order=order, product=item.product, quantity=item.quantity)
            for item in items
        ])
        transaction.on_commit(
            lambda: enqueue_order_confirmation(order.pk)
        )
    return order
```

`on_commit()` 注册的回调只会在最外层事务成功提交后执行；事务回滚时不会执行。它适合避免任务在事务提交前就读取不到记录的问题。

**重要边界：** `on_commit()` 本身不是可靠消息队列。如果数据库提交后进程崩溃，任务可能还没投递；若必须保证“订单提交与事件最终都可发布”，使用事务 Outbox：在同一数据库事务中写业务记录和 outbox 事件，由独立任务重试发布，并以事件 ID 做幂等。

### Q5：高并发扣库存如何避免超卖？

**参考答案：**

不能采用“先读库存、应用内判断，再无锁写回”的模式。两个请求可能同时读到库存为 1，然后都认为可以购买。可以用行级锁串行化对同一商品的更新，配合数据库 `CHECK` 约束兜底。

```python
from django.db import transaction

class InsufficientStock(Exception):
    pass

def reserve_one(product_id, quantity):
    if quantity <= 0:
        raise ValueError("quantity must be positive")

    with transaction.atomic():
        inventory = (
            Inventory.objects
            .select_for_update()
            .get(product_id=product_id)
        )
        if inventory.stock < quantity:
            raise InsufficientStock()

        inventory.stock -= quantity
        inventory.save(update_fields=["stock"])
        return inventory.stock
```

`select_for_update()` 必须在事务中使用，并会锁住匹配行直到事务结束。所有竞争路径都应遵循同一套库存规则；库存表还应有 `CHECK (stock >= 0)` 约束，防止其他代码路径写入负数。

如果只需要原子递减，也可以使用条件更新：

```python
from django.db.models import F

updated = Inventory.objects.filter(
    product_id=product_id,
    stock__gte=quantity,
).update(stock=F("stock") - quantity)

if updated == 0:
    raise InsufficientStock()
```

条件更新通常减少读写往返，但复杂业务需要确认锁行为、返回结果和审计字段是否满足要求。订单请求还应有唯一幂等键或唯一业务约束，避免客户端超时重试时重复创建订单。多商品订单要按稳定顺序锁定库存行，降低死锁概率。

### Q6：`select_for_update()` 有哪些注意事项？

**参考答案：**

它通过 PostgreSQL 行级锁协调并发写入，但不会自动解决所有并发问题：

- 必须在事务中使用；事务结束才释放锁。
- 只锁定实际查询到的行。缺失行并不会被普通行锁保护。
- 锁持有时间越长，阻塞和死锁风险越大。
- `skip_locked=True` 常用于多个 Worker 抢占不同待处理任务；它会跳过已锁行，不适合需要完整一致结果的普通业务查询。
- 测试事务行为时应考虑使用 `TransactionTestCase`，因为 `TestCase` 自身通常包在事务中，可能掩盖事务边界问题。

**实际应用：** 多个消费者从待处理任务表领取任务，可在短事务中筛选待处理记录、`select_for_update(skip_locked=True)` 并更新为处理中；真正的耗时处理在提交后执行，同时需要任务租约、失败重试和幂等设计。

### Q7：模型校验和数据库约束有什么区别？

**参考答案：**

Django 的 `validate`、`clean()` 或表单校验提供更友好的用户错误，但不能代替数据库约束。常规 `save()` 不会自动调用 `full_clean()`；并发请求也可能同时通过应用校验。

数据完整性应尽可能由数据库最终保证：

```python
from django.db import models
from django.db.models import Q

class Inventory(models.Model):
    product = models.OneToOneField("Product", on_delete=models.CASCADE)
    stock = models.IntegerField()

    class Meta:
        constraints = [
            models.CheckConstraint(
                condition=Q(stock__gte=0),
                name="inventory_stock_nonnegative",
            ),
        ]
```

唯一性应使用唯一约束或唯一索引，而不是先查询“是否存在”再插入。并发情况下两次“先查后写”都可能认为记录不存在。捕获唯一冲突后，可将数据库异常转换为业务层可理解的错误。

### Q8：`bulk_create()` / `update()` 有什么优缺点？

**参考答案：**

批量操作能减少 SQL 往返，适用于导入、批处理和大量状态更新，但行为不完全等同逐个调用 `save()`：

- `bulk_create()` 和 `QuerySet.update()` 通常不会逐条调用模型 `save()`，相关 `pre_save` / `post_save` 信号也不会像逐条保存那样触发。
- `update()` 直接更新数据库，不会自动执行模型的 `save()` 逻辑或 `auto_now` 赋值逻辑。
- 不同 Django 和数据库版本对批量插入返回主键、冲突忽略/更新等能力有差异。
- 批量写入仍要考虑事务大小、锁持续时间、参数数量和失败后的重试策略。

**实际应用：** 大批量导入时先校验和去重，按有限批次写入；若依赖信号维护搜索索引或审计记录，应显式重构为批次级逻辑，或在同一事务中写入 outbox，而不是假设 bulk 操作会自动触发所有副作用。

### Q9：如何安全地对大型 Django 表做 Schema Migration？

**参考答案：**

生产迁移要考虑表锁、写入流量、旧版本应用兼容性和回滚路径。常用做法是 **Expand–Migrate–Contract**：

1. **Expand**：新增可空字段/新表/新索引，不立即删除旧结构。
2. 发布兼容新旧字段的应用代码；必要时双写。
3. 以可重入、可断点恢复的批次回填历史数据。
4. 验证数据完成后切换读取，并经过一个稳定周期。
5. **Contract**：最后再设为必填或删除旧字段。

在 PostgreSQL 中，`CREATE INDEX CONCURRENTLY` 可降低建索引对正常写入的阻塞，但不能在普通事务块内运行。Django 可用 PostgreSQL 提供的并发索引 migration operation，并将迁移设为非原子迁移；执行前仍要评估磁盘、I/O 和失败后无效索引的清理方式。

**常见陷阱：** 一次性回填数千万行、对热表执行长事务、先发布依赖新列的代码再建列、在仍有旧应用实例时删除列。迁移与应用代码都应向前/向后兼容。

### Q10：Django 异步视图可以直接调用同步 ORM 吗？

**参考答案：**

异步视图适合大量等待 I/O 的工作，但如果调用阻塞式同步代码，会占住事件循环。Django 不同版本对异步 ORM API 的支持逐步演进；事务管理和具体驱动的 async 能力也需要按版本核实。不要仅因为 View 声明为 `async def` 就认为数据库访问天然异步。

如果一组数据库操作需要事务，可把完整事务工作封装在同步函数中，再通过 `sync_to_async` 以线程敏感方式调用：

```python
from asgiref.sync import sync_to_async
from django.db import transaction

def _create_order_sync(customer_id, product_id):
    with transaction.atomic():
        customer = Customer.objects.get(pk=customer_id)
        # 在此完成需要同一事务保障的数据库操作
        return Order.objects.create(customer=customer, product_id=product_id).pk

async def create_order_view(request):
    order_id = await sync_to_async(
        _create_order_sync,
        thread_sensitive=True,
    )(request.user.pk, request.POST["product_id"])
    ...
```

要比较端到端延迟和数据库连接压力。若调用链主要由短小数据库操作构成，同步视图加合适的 worker 可能更简单、更可靠；异步主要解决等待期间的并发利用率，不会让 SQL 本身自动变快。

### Q11：如何测试事务、锁和 `on_commit()`？

**参考答案：**

- 普通模型和视图测试通常用 `TestCase`，执行速度快且会回滚测试数据。
- 需要验证真实提交、事务边界、提交回调或并发锁语义时，考虑 `TransactionTestCase`，并在真实 PostgreSQL 测试环境中运行。
- `on_commit()` 回调应验证提交后执行、回滚后不执行；也可使用 Django 测试提供的回调捕获能力。
- 并发测试需用独立数据库连接/线程或进程构造竞争，并设置超时，避免测试挂死。
- SQLite 不能完整模拟 PostgreSQL 的行级锁、隔离级别和执行计划，关键集成测试不要只依赖 SQLite。

### Q12：Django 缓存与数据库的一致性如何处理？

**参考答案：**

缓存是可丢失、可重建的副本，不应成为唯一事实来源。常见策略是 cache-aside：先查缓存，未命中再查数据库并回填。写操作成功后使缓存失效或更新缓存，但需避免在事务提交前就发布新值。

```python
with transaction.atomic():
    order.status = "paid"
    order.save(update_fields=["status"])
    transaction.on_commit(lambda: cache.delete(f"order:{order.pk}"))
```

**实际应用：** 订单状态变化可在提交后删除详情缓存；高并发场景还需要处理“旧读结果晚到并覆盖新缓存”的竞态，可使用版本号、原子写入或短 TTL。若缓存失效、事件发布必须最终可靠，使用 outbox/重试机制，不要把 `on_commit()` 当成可靠队列。

---

## 三、PostgreSQL 高频问题

### Q13：PostgreSQL 的 MVCC 是什么？为什么需要 VACUUM？

**参考答案：**

PostgreSQL 使用 MVCC（多版本并发控制）：更新或删除不会简单地原地覆盖所有旧版本，而是生成新版本并根据事务快照决定读者能看到哪个版本。这样读通常不需要等待普通写锁，但旧版本会产生空间回收需求。

VACUUM 清理已不再被任何事务快照需要的死元组，并维护可见性信息；autovacuum 根据表活动自动执行。它通常让空间可被 PostgreSQL 复用，不代表文件一定立刻缩小并归还给操作系统。

**实际排障：**

- 长事务、未结束的 idle-in-transaction 会阻止旧版本清理，造成表膨胀、复制槽/WAL 堆积等问题。
- 监控表的 dead tuples、autovacuum 次数与耗时、事务年龄、复制槽和磁盘增长。
- `VACUUM FULL` 会重写表并需要强锁，不是常规在线清理手段。
- 对热点表按实际写入和清理行为调整 autovacuum 参数，不要对所有表照抄一组激进参数。

### Q14：PostgreSQL 默认隔离级别是什么？如何理解并发异常？

**参考答案：**

PostgreSQL 默认是 `READ COMMITTED`。通常每条 SQL 语句开始时取得新的快照，因此同一事务中的两次查询可能看到其他事务已经提交的新数据。它能避免脏读，但某些“读后再写”的业务仍有竞争。

- **`READ COMMITTED`**：适合多数 OLTP；用条件更新、唯一/检查约束和行锁处理具体冲突。
- **`REPEATABLE READ`**：同一事务保持稳定快照；PostgreSQL 使用快照隔离语义，仍需考虑写冲突和序列化失败。
- **`SERIALIZABLE`**：提供更强的串行化保证，但数据库可能中止事务，应用必须能安全重试。

遇到 SQLSTATE `40001`（serialization failure）或 `40P01`（deadlock detected）时，可对**整个可重放事务**做有限次数、带抖动的退避重试。事务中不能包含不可撤销的外部副作用，重试逻辑必须幂等。

**实际应用：** “账户余额不得低于零”不能只用 `READ COMMITTED` 下的 Python 读值判断；可以用条件 `UPDATE ... WHERE balance >= amount`，或锁定账户行，并加数据库 `CHECK` 约束。

### Q15：索引如何工作？何时选择 B-tree、GIN 或部分索引？

**参考答案：**

索引通过额外的数据结构减少扫描，但会占用磁盘、增加插入/更新/删除成本。查询是否使用索引由优化器根据统计信息、选择率、表大小和成本估算决定，不是建了索引就一定会用。

- **B-tree**：默认选择，适合等值、范围和排序查询，例如用户 ID、时间范围。
- **复合 B-tree**：字段顺序要结合常见过滤条件、排序和选择率设计。通常 `(customer_id, created_at)` 适合按客户筛选再按时间查；不能假定它等价于反向顺序。
- **部分索引**：只索引符合谓词的行，例如经常查询的未完成任务，减少索引体积。
- **表达式索引**：适合固定表达式查询，如 `lower(email)`；查询表达式需与索引定义兼容。
- **GIN**：适合 JSONB、数组等包含/倒排查询；会增加写放大，索引操作符和查询写法要匹配。
- **BRIN**：适合物理顺序与取值相关、体量很大的追加型数据，例如按时间近似顺序写入的事件表。
- **`INCLUDE` 覆盖列**：在合适情况下减少回表，但不应把大量列塞进索引。

**实际应用：**

```sql
CREATE INDEX CONCURRENTLY order_customer_created_idx
ON orders (customer_id, created_at DESC);
```

该索引可能适合 `WHERE customer_id = ... ORDER BY created_at DESC` 的订单列表。仍要用真实数据执行计划验证；低选择率字段的单列索引未必有价值。

### Q16：如何使用 `EXPLAIN` / `EXPLAIN ANALYZE` 找慢查询？

**参考答案：**

`EXPLAIN` 展示优化器估算的执行计划；`EXPLAIN ANALYZE` 会实际运行语句并给出实际行数、耗时等信息。比较 estimated rows 与 actual rows，可发现统计信息过期、数据倾斜或估算偏差。

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT id, created_at
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 50;
```

关注：

- `Seq Scan` 是否合理，还是本应有高选择率索引。
- 估算行数和实际行数差异是否很大。
- `Nested Loop` 是否把大量行放大。
- 排序是否落盘、是否出现大量 `shared read`。
- 过滤后丢弃的行数、循环次数和节点实际耗时。

**安全提醒：** `EXPLAIN ANALYZE` 会执行查询；对 `UPDATE`、`DELETE` 或带副作用的函数不能当成只读分析命令。先在安全环境验证，生产写操作需有明确的事务和回滚方案。还应查看 Django 实际生成的 SQL，并用代表性参数测试。

### Q17：死锁如何产生，如何减少？

**参考答案：**

两个事务以相反顺序持有资源时会形成等待环。例如事务 A 锁库存 1 再锁库存 2，事务 B 锁库存 2 再锁库存 1。PostgreSQL 会检测死锁并中止其中一个事务。

**减少方法：**

1. 所有代码路径按稳定顺序访问/锁定多行，例如按主键排序。
2. 保持事务短小，不在事务里调用外部服务或等待用户输入。
3. 为更新和过滤条件创建适当索引，避免锁等待期间扫描过多数据。
4. 捕获并有限重试 `40P01`，同时保证业务幂等。
5. 通过数据库日志、锁等待视图和应用追踪关联出阻塞链路。

死锁重试是最后一道恢复机制，不是替代锁顺序设计的理由。

### Q18：PostgreSQL 连接数为什么不能无限增加？如何连接池化？

**参考答案：**

每个 PostgreSQL 后端连接都有内存和调度成本。过多连接会导致上下文切换、内存消耗和争抢增加，数据库吞吐可能反而下降。

- Django `CONN_MAX_AGE` 控制持久连接复用时间，可减少频繁建连开销，但需结合 worker 数和部署方式估算连接总量。
- PgBouncer 等连接池在应用连接和数据库后端连接之间做复用。事务池化模式下，每个事务结束后连接可能被分配给另一个客户端。
- 事务池化不适合依赖跨事务会话状态的应用逻辑；临时表、会话级设置、会话级 advisory lock、游标和 prepared statement 的行为需根据 PgBouncer/PostgreSQL/Django 版本验证。
- 连接池应配置容量、排队/超时和监控；池不是越大越好。

估算时至少考虑：Web worker/线程数、异步任务 worker、管理命令、迁移进程、只读副本连接、运维预留连接和池化模式。

### Q19：Offset 分页和 Keyset（游标）分页有什么区别？

**参考答案：**

Offset 分页使用 `LIMIT/OFFSET`，实现和跳页方便；页数很深时数据库仍需扫描/跳过大量行，并发插入也可能造成重复或遗漏。Keyset 分页用上一页最后一条记录作为锚点，按稳定、有索引的排序键继续查询，适合深分页和持续翻页。

```python
from django.db.models import Q

page = (
    Order.objects
    .filter(
        Q(created_at__lt=cursor_created_at)
        | Q(created_at=cursor_created_at, pk__lt=cursor_pk)
    )
    .order_by("-created_at", "-pk")[:50]
)
```

`created_at` 可能重复，因此需要主键作为稳定的次级排序键，并创建匹配访问模式的复合索引。Keyset 分页不天然支持任意跳到第 N 页；API 应根据产品需求选择，而不是一律使用游标。

### Q20：JSONB 适合什么场景？为什么不能把所有字段都放进去？

**参考答案：**

JSONB 适合结构变化较快、低频筛选的扩展属性或外部原始载荷；它提供灵活查询，并可使用 GIN 等索引。核心业务字段若承担高频过滤、关联、约束、统计或排序，通常应建成规范的关系列。

**实际应用：** 商品的稳定字段（SKU、价格、库存、状态）用关系列和约束；不同品类少量不稳定展示属性可放 JSONB。若某个 JSONB 键后来成为关键筛选字段，应评估抽取为列或建立有针对性的表达式/GIN 索引。

JSONB 不意味着无模式、无校验或免费查询。需考虑文档大小、索引体积、更新整行版本的成本、字段类型一致性和数据演进。

### Q21：主库和只读副本如何配合？读副本会带来什么问题？

**参考答案：**

只读副本可扩展读吞吐并支持报表，但复制通常有延迟。刚在主库提交的数据，副本可能暂时还看不到。

**实际应用：**

- 写后读、支付确认、订单状态跳转等需要读己之写的请求路由到主库。
- 允许短暂陈旧的商品列表、报表可路由到副本。
- 可使用短期 sticky session/主库读取策略，或基于复制位置设计更强的读后写保证；不要假设“已提交”意味着所有副本立即可见。
- Django Database Router 只能帮助路由，业务仍需表达哪些读取可容忍延迟。

同时监控复制延迟、WAL 积压和副本状态；副本故障切换要有明确的应用连接更新与恢复流程。

### Q22：备份与高可用有什么区别？如何证明备份可用？

**参考答案：**

高可用主要缩短故障期间的服务中断，备份用于恢复误删、损坏、逻辑错误或更大范围故障。副本会复制误操作，因此不能替代独立备份。

- 逻辑备份适合对象级迁移和小规模恢复。
- 基础备份加 WAL 归档可实现时间点恢复（PITR），适合生产恢复目标要求。
- 备份要隔离权限、加密、监控成功/失败并定义保留策略。
- 定期在隔离环境做恢复演练，验证 RPO、RTO、数据一致性和应用可启动性。

面试中应说清楚目标恢复点（RPO）、恢复时间（RTO）、备份覆盖的故障域，以及如何避免备份凭证与生产应用凭证共用。

---

## 四、综合场景题

### Q23：订单创建成功，但邮件或消息队列投递失败，如何处理？

**参考答案：**

数据库事务不能同时原子提交 PostgreSQL 和外部邮件/消息系统。若先发邮件后数据库回滚，会产生“通知了不存在的订单”；若先提交数据库再发邮件，进程可能在两步之间崩溃。

简单场景可用 `transaction.on_commit()`，确保提交后再快速投递。需要可靠最终投递时使用 **Transactional Outbox**：

1. 在同一个 `transaction.atomic()` 中写入订单和 outbox 事件。
2. Worker 按状态和创建时间领取事件，发布到队列。
3. 发布确认后标记已发送；失败则退避重试并监控积压。
4. 消费端用事件 ID 或业务幂等键去重。

该方案通常提供“至少一次”投递；消费者必须能处理重复消息。不要声称数据库、消息队列和邮件服务之间存在天然的全局 exactly-once 事务。

### Q24：某个订单列表接口从 100 ms 变成 3 秒，你会怎么排查？

**参考答案：**

按证据缩小范围，而不是先加缓存：

1. 通过请求追踪确认慢在应用、连接等待、数据库还是外部依赖。
2. 查看 PostgreSQL 慢查询/`pg_stat_statements`，锁定高总耗时、高频或高均值 SQL。
3. 从 Django 捕获实际 SQL 和参数，确认是否 N+1、无谓加载字段或过滤条件退化。
4. 对代表性查询检查 `EXPLAIN (ANALYZE, BUFFERS)`，比较估算/实际行数、扫描行数、排序和 I/O。
5. 检查连接池等待、活动/空闲事务、锁等待、autovacuum、缓存命中、表膨胀及近期发布。
6. 修正查询形状或索引后，在接近真实数据量的环境回归查询耗时、写入成本和并发行为。

**可能的修复：** 加分页、`select_related`/`prefetch_related`、只选择需要的列、调整复合索引、修复统计信息、缩短事务。需要缓存时再定义失效与一致性策略。

### Q25：怎样保证“同一支付请求重试不会重复扣款或重复创建订单”？

**参考答案：**

使用稳定的幂等键，例如客户端请求 ID 或支付平台交易号，并由数据库唯一约束保证并发下只创建一份记录。收到重复请求时返回已有业务结果，而不是再次执行副作用。

基本设计：

```text
payment_request(idempotency_key UNIQUE, status, result, ...)
```

处理时先尝试创建/获取幂等记录，再在事务中更新业务状态；外部支付 API 也应使用对方支持的幂等键。不同服务间的处理使用 outbox、重试和去重。幂等键需要有作用域、保留周期及请求内容冲突检测，不能只用短时间缓存来保证正确性。

---

## 五、面试速查清单

### Django

- 能解释请求链路、中间件顺序、同步与异步边界。
- 知道 `QuerySet` 惰性求值与缓存范围。
- 能识别 N+1，并按关系类型选择 `select_related` 或 `prefetch_related`。
- 把事务范围限定在数据库操作，理解 `on_commit()` 与 Outbox 的差异。
- 用数据库约束保证唯一性、非空和业务不变量。
- 知道批量操作不会等同逐条 `save()` 和信号调用。
- 大表迁移采用兼容旧代码的分阶段发布和可恢复回填。
- 测试锁、事务和 SQL 行为时使用 PostgreSQL 集成环境。

### PostgreSQL

- 理解 MVCC、死元组、VACUUM 和长事务影响。
- 能说明 `READ COMMITTED` 默认行为及序列化失败重试。
- 索引从实际查询和执行计划推导，而不是盲目添加。
- 知道 `EXPLAIN ANALYZE` 会真正执行语句。
- 理解死锁、连接池和副本延迟的应用影响。
- 根据访问模式选择 Offset 或 Keyset 分页。
- 备份、复制和高可用分别定义目标，并实际演练恢复。

### 生产级回答自检

回答一个方案前，快速追问自己：

1. 并发请求同时执行时，数据库最终状态是否仍正确？
2. 客户端超时重试会不会创建重复数据或重复扣款？
3. 事务提交后进程崩溃，消息/通知是否会丢失？
4. 数据量增长 100 倍后，查询和迁移是否仍可接受？
5. 如何用日志、指标、执行计划和测试证明方案有效？
6. 出错时如何重试、回滚、恢复或人工介入？

能把这些边界讲清楚，通常比单纯背诵 Django API 或 PostgreSQL 参数更能体现资深工程能力。
