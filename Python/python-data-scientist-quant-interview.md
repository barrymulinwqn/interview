# Python Data Scientist / Quantitative Trading Developer 面试题

> 本文根据岗位描述整理：与研究员、投资经理紧密合作，负责新数据接入、交易信号实现、组合优化工具、数据可视化框架和研究平台建设。岗位要求包括 Python、Pandas、NumPy、SciPy、statsmodels、scikit-learn、MySQL/PostgreSQL/MongoDB、Kafka、Linux、Git、Kubernetes、生产系统开发和英文口语。

## 一、岗位画像

这是一个 **Python 数据科学 + 量化交易开发 + 研究平台工程** 的混合岗位，不是只在 Notebook 中训练模型的纯 Data Scientist，也不是只维护基础设施的 DevOps 岗位。面试官通常会同时考察以下能力：

| 能力方向 | 面试官关注点 |
|---|---|
| Python 工程能力 | 代码质量、性能、并发、异常处理、测试、可维护性 |
| 数据处理 | Pandas/NumPy 熟练度、时序数据、缺失值、数据质量和内存优化 |
| 统计与机器学习 | 统计假设、时间序列验证、特征泄漏、模型评估和解释性 |
| 量化基础 | 收益率、风险、交易信号、回测偏差、交易成本和组合优化 |
| 数据平台 | Kafka、数据库、批流一体、数据血缘、幂等和可重放 |
| 生产化 | Linux、Git、CI/CD、监控、部署、容器和 Kubernetes |
| 协作与领导力 | 与研究员/投资经理沟通、拆解需求、推动项目、团队建设 |
| 英语沟通 | 用英文解释技术方案、风险、进度和结果 |

### 一句话自我定位

可以把自己的定位概括为：

> I build reliable Python-based research and trading systems that turn data into reproducible signals and production-ready investment tools.

中文含义是：我使用 Python 构建可靠的研究和交易系统，将数据转化为可复现的信号和可以上线的投资工具。

---

## 二、面试回答主线

遇到开放式系统设计题，不要一上来罗列库和中间件，建议按下面顺序回答：

1. **业务目标**：研究还是实盘，数据频率、延迟、规模、覆盖市场和使用者是谁。
2. **数据链路**：数据来源、接入、校验、标准化、存储、回放和版本管理。
3. **研究链路**：特征、信号、回测、组合构建、风险控制和结果可视化。
4. **生产链路**：信号如何发布、订单如何执行、失败如何重试、如何审计。
5. **质量与风险**：如何避免未来函数、幸存者偏差、重复消费和数据污染。
6. **工程保障**：测试、监控、部署、回滚、权限、成本和团队协作方式。

高分答案要同时讲清楚 **准确性、可复现性、延迟、吞吐、可靠性和业务价值**，不能只说“使用 Pandas 加 Kafka”。

---

## 三、基础面试题

### 3.1 Python 基础

#### 1. Python 中哪些对象可变，哪些不可变？

常见不可变对象包括 `int`、`float`、`bool`、`str`、`bytes`、`tuple` 和 `frozenset`；常见可变对象包括 `list`、`dict`、`set` 和 `bytearray`。

不可变对象不能原地修改，变量重新赋值时通常是绑定到新对象。函数默认参数不要使用可变对象：

```python
# 错误或容易产生意外共享状态

def append_item(item, items=[]):
    items.append(item)
    return items

# 推荐

def append_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

#### 2. `is` 和 `==` 的区别是什么？

`==` 比较两个对象的值是否相等，`is` 比较是否为同一个对象。业务代码中比较值使用 `==`，判断单例对象如 `None` 使用 `is None`。不能依赖小整数或短字符串的缓存行为。

#### 3. Shallow Copy 和 Deep Copy 有什么区别？

浅拷贝只复制最外层容器，内部嵌套对象仍然共享；深拷贝递归复制嵌套对象。对于包含 DataFrame、数组或自定义对象的数据结构，应明确复制成本和共享语义，不能为了“安全”盲目使用 `deepcopy`。

#### 4. List、Tuple、Set、Dict 如何选择？

- `list`：有序、可重复，适合序列。
- `tuple`：有序、不可变，适合固定结构或作为可哈希键的一部分。
- `set`：去重和成员判断，平均查找复杂度通常为 $O(1)$。
- `dict`：键值映射，适合按键访问。

选择数据结构时还要考虑内存、可读性、顺序要求和访问模式。

#### 5. 什么是生成器？为什么适合大数据处理？

生成器使用 `yield` 按需产生结果，不会一次性把完整结果放入内存，适合读取大文件、流式转换和分页处理：

```python
from collections.abc import Iterator


def read_rows(path: str) -> Iterator[str]:
    with open(path, encoding="utf-8") as file:
        for line in file:
            yield line.rstrip("\n")
```

生成器是惰性的，但只能按顺序消费；如果需要多次遍历或随机访问，应使用适合的物化结构。

#### 6. 装饰器、上下文管理器分别解决什么问题？

装饰器用于在不修改函数主体的情况下包装行为，例如计时、重试、权限检查和日志。上下文管理器用于保证资源进入和退出时执行对应逻辑，例如文件、锁、数据库连接和临时目录。

```python
from contextlib import contextmanager


@contextmanager
def transaction(connection):
    try:
        yield
        connection.commit()
    except Exception:
        connection.rollback()
        raise
```

#### 7. 什么是异常处理的好实践？

捕获最具体的异常，不要用裸 `except`；记录足够的上下文但不要泄露密钥；能够处理就处理，不能处理就使用 `raise` 保留 traceback；对外部 API、数据库和消息消费设置明确的超时、重试和死信策略。不要用异常吞掉数据质量问题。

#### 8. Python 的 GIL 是什么？

CPython 的 Global Interpreter Lock 使同一进程中同一时刻通常只有一个线程执行 Python 字节码。I/O 密集型任务可以通过线程或异步提高并发；CPU 密集型任务通常考虑多进程、释放 GIL 的 NumPy/科学计算库，或者把任务分发到外部计算服务。不要简单地说“Python 不能并发”。

#### 9. Threading、Multiprocessing 和 Asyncio 如何选择？

- **Threading**：适合 I/O 密集型任务，复用同步库较方便。
- **Multiprocessing**：适合 CPU 密集型任务，但进程间通信和数据复制有成本。
- **Asyncio**：适合大量网络 I/O，要求调用链使用异步库，不能把阻塞操作直接放入事件循环。
- NumPy、SciPy 等底层实现可能释放 GIL，必须用基准测试确认，而不是凭语言标签判断。

#### 10. 如何优化 Python 程序？

先使用 profiling 找热点，再优化。常用方法包括改进算法复杂度、减少 Python 层循环、使用向量化、批量 I/O、合理缓存、减少重复序列化、控制对象数量和选择合适的数据类型。可以使用 `cProfile`、`py-spy`、`line_profiler` 或内存分析工具，不能只凭感觉优化。

---

### 3.2 NumPy 基础

#### 11. NumPy Array 和 Python List 的区别？

NumPy Array 通常具有统一 dtype、连续或规则的内存布局，并能通过底层实现进行向量化计算；Python List 存储对象引用，类型灵活但数值循环开销更高。NumPy 更适合大规模同类型数值数据，但不代表所有操作都自动更快，数据拷贝和内存布局也要考虑。

#### 12. 什么是 Broadcasting？

Broadcasting 允许形状兼容的数组进行逐元素运算，而不必显式复制小数组。例如一个 `(3, 1)` 数组和一个 `(1, 4)` 数组可以得到 `(3, 4)` 结果。需要注意，Broadcasting 可能产生很大的结果数组，不能忽略峰值内存。

#### 13. View 和 Copy 有什么区别？

View 与原数组共享底层数据，修改可能互相影响；Copy 拥有独立数据。切片、转置和某些 reshape 操作可能返回 view，也可能因为内存不连续而产生 copy。对关键计算要明确是否需要独立副本，并通过 `np.shares_memory` 或文档确认。

#### 14. 如何处理 NaN 和无穷值？

先区分缺失、无效和真实的无穷值，再决定删除、填充、截断还是保留为特征。金融时序不能简单用全局均值填充，否则可能引入未来信息或扭曲分布。处理规则必须只使用当时可获得的信息，并记录数据质量指标。

#### 15. `np.vectorize` 是真正的向量化吗？

`np.vectorize` 主要是 Python 函数的便利包装，通常不等于底层 C 级向量化，也不一定有性能提升。优先使用 NumPy ufunc、布尔索引、广播、聚合和专用函数；复杂逻辑再考虑 Numba、Cython 或批处理。

---

### 3.3 Pandas 基础

#### 16. Series 和 DataFrame 的区别？

Series 是带索引的一维数组，DataFrame 是带行列标签的二维表格。Pandas 的核心优势是标签对齐、缺失值处理、分组、连接、时间索引和表格变换；同时它也会产生隐式对齐、复制和类型转换，需要理解底层行为。

#### 17. `loc` 和 `iloc` 的区别？

`loc` 按标签选择，`iloc` 按整数位置选择：

```python
data.loc[data["symbol"] == "AAPL", ["timestamp", "close"]]
data.iloc[0:10, 1:3]
```

时序和金融数据经常依赖索引，使用前应确认索引是否唯一、是否排序以及选择结果是否为 view 或 copy。

#### 18. 如何避免 `SettingWithCopyWarning`？

使用明确的 `.loc` 赋值，或在确实需要独立数据时显式 `.copy()`：

```python
mask = data["volume"] > 0
data.loc[mask, "return"] = data.loc[mask, "close"].pct_change()

subset = data.loc[mask, ["symbol", "close"]].copy()
```

不要只用关闭 warning 的方式掩盖潜在的数据赋值错误。

#### 19. `merge`、`concat` 和 `join` 如何选择？

- `merge`：按一个或多个键做类似 SQL 的连接。
- `concat`：沿行或列方向拼接数据，适合相同结构的批次合并。
- `join`：常用于按索引连接。

金融数据连接时必须检查键的唯一性、时间对齐、重复行和行数变化：

```python
joined = prices.merge(
    fundamentals,
    on=["symbol", "trade_date"],
    how="left",
    validate="one_to_one",
)
```

`validate` 可以帮助尽早发现意外的一对多连接。

#### 20. `groupby`、`agg`、`transform` 有什么区别？

`agg` 通常把每组聚合成更少的行；`transform` 返回与原数据同长度的结果，适合生成组内特征；`apply` 灵活但通常较慢，且容易隐藏复杂逻辑。能用向量化、`agg` 或 `transform` 时，不要默认使用 `apply`。

#### 21. Pandas 中如何处理时间序列？

先把时间列解析为明确时区的 `datetime64`，统一时区和交易日历，再排序并检查重复时间戳。使用 `resample`、`rolling` 和 `shift` 时，要确认窗口边界和数据是否包含当前值，避免未来数据泄漏：

```python
prices = prices.sort_values(["symbol", "timestamp"])
prices["past_return"] = prices.groupby("symbol")["close"].pct_change(1)
prices["rolling_vol"] = (
    prices.groupby("symbol")["past_return"]
    .rolling(20, min_periods=20)
    .std()
    .reset_index(level=0, drop=True)
)
```

#### 22. 如何降低 DataFrame 的内存使用？

读取时只选择必要列，指定 dtype，重复字符串使用 `category`，分块读取大文件，避免不必要的 copy，及时释放中间对象，并用 `memory_usage(deep=True)` 检查真实占用。优化前要确认精度要求，不能为了省内存随意把价格或收益率转换成过低精度。

#### 23. Pandas 什么时候不适合？

当数据远超单机内存、需要分布式计算、严格流式处理、复杂 SQL 执行计划或极低延迟服务时，Pandas 可能不是合适的核心引擎。可以考虑 Polars、Dask、Spark、数据库计算或专门的流处理系统，但迁移前要验证语义、生态、团队能力和成本。

---

### 3.4 统计、SciPy、statsmodels 和机器学习基础

#### 24. 均值、中位数、方差和标准差分别反映什么？

均值反映总体中心，但容易受极端值影响；中位数对异常值更稳健；方差和标准差描述离散程度。金融收益经常具有偏态、厚尾和波动聚集，不能只依赖均值和标准差判断风险。

#### 25. p-value 和置信区间是什么？

p-value 是在零假设成立时观察到当前或更极端数据的概率，不是“零假设为真的概率”。置信区间表达估计值的不确定性。面试中还要说明样本量、独立性、模型假设、多重检验和经济意义，统计显著不等于交易上有价值。

#### 26. OLS 的基本假设有哪些？

常见假设包括线性关系、误差独立、同方差、无完全多重共线性，以及在进行特定推断时对误差分布的要求。金融时序还常见自相关、异方差、非平稳和结构变化，需要使用稳健标准误、时序模型或重新定义研究问题。

#### 27. 如何识别多重共线性？

观察相关矩阵、VIF、特征值或条件数，并结合领域含义判断。解决方法可以是删除冗余变量、合并特征、正则化、降维或增加数据。共线性不一定影响预测性能，但会让系数解释和统计推断不稳定。

#### 28. ADF 检验用于什么？

Augmented Dickey-Fuller 检验常用于检查单位根，零假设通常是存在单位根，即序列非平稳。检验结果受趋势项、滞后阶数、样本长度和结构突变影响。不能把一次 ADF 结果当成对时序性质的绝对证明。

#### 29. 相关性和因果性有什么区别？

相关性表示变量共同变化，不能单独证明因果关系。因果分析需要实验、准实验、时间顺序、控制混淆变量和明确识别假设。量化研究中一个特征与未来收益相关，也可能来自数据泄漏、共同风险因子、市场状态或偶然样本。

#### 30. 如何划分机器学习训练集、验证集和测试集？

普通独立同分布任务可以随机划分，但时间序列必须按时间顺序划分，避免用未来训练过去。可以使用 walk-forward、expanding window 或 `TimeSeriesSplit`，并在每个训练窗口内独立完成标准化、特征选择和超参数选择。

#### 31. 为什么要使用 Pipeline？

`Pipeline` 把预处理、特征转换和模型绑定在一起，保证训练和预测使用一致步骤，并减少交叉验证中的数据泄漏：

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import Ridge

model = Pipeline([
    ("scale", StandardScaler()),
    ("regression", Ridge(alpha=1.0)),
])
```

任何依赖目标变量或数据分布的步骤，都必须只在训练数据上拟合。

#### 32. 如何处理类别不平衡？

先确认业务目标，不要只看 accuracy。可以使用 precision、recall、F1、PR-AUC、ROC-AUC、成本加权指标和阈值调优；训练时使用 class weight、重采样或合适的损失函数。时间序列场景还要保证采样和验证没有破坏时间关系。

#### 33. 如何判断模型是否过拟合？

观察训练和验证表现差距、不同时间窗口的稳定性、特征重要性变化、残差和敏感性分析。降低复杂度、正则化、减少特征、增加样本、使用正确的时间验证和限制研究自由度都可以帮助控制过拟合。回测中的高 Sharpe 也可能只是过拟合。

#### 34. 如何解释模型？

线性模型可以解释系数，但要注意标准化和共线性；树模型可以用 permutation importance 或 SHAP 等方法分析，但解释仍然依赖数据分布和背景样本。解释预测贡献不等于证明因果关系。对投资决策还要解释稳定性、风险和经济逻辑。

---

## 四、量化交易与组合优化高频题

### 4.1 收益率和风险

#### 35. 简单收益率和对数收益率有什么区别？

简单收益率为：

$$
r_t = \frac{P_t}{P_{t-1}} - 1
$$

对数收益率为：

$$
\ell_t = \log\left(\frac{P_t}{P_{t-1}}\right)
$$

简单收益率可以直接复合；对数收益率在时间上可加。选择哪一种取决于模型和汇总方式，不能混用后再直接比较结果。

#### 36. Sharpe Ratio 如何计算？有哪些陷阱？

常见形式是：

$$
Sharpe = \frac{E[R_p - R_f]}{\sigma(R_p - R_f)}
$$

实际计算时要明确无风险利率、收益频率、年化因子、是否扣除交易成本和收益序列是否独立。高频收益存在自相关、非正态和换手成本，简单乘以平方根时间的年化方式可能失真。Sharpe 不能单独代表策略质量。

#### 37. 最大回撤是什么？

最大回撤是净值相对历史峰值的最大下降：

$$
Drawdown_t = \frac{V_t}{\max_{s \leq t} V_s} - 1
$$

最大回撤反映路径风险，补充了波动率指标无法表达的连续亏损和恢复时间。面试时可以继续说明回撤持续时间、恢复期和压力场景。

#### 38. 什么是交易信号？如何从信号得到仓位？

信号是根据当前或历史可用信息形成的预测、排序或交易倾向；仓位是把信号转化为可执行权重的结果。中间通常还需要风险调整、行业/风格中性化、流动性限制、持仓上限、换手限制、交易成本和订单执行规则。

#### 39. 什么是 look-ahead bias？

Look-ahead bias 是在回测中使用了当时实际不可获得的信息，例如用收盘价计算信号却在同一收盘价成交，或用修订后的财报数据预测更早日期。解决方式包括明确数据可用时间、使用 `shift`、按发布时间对齐、设置交易执行延迟并做端到端审查。

#### 40. 什么是 survivorship bias？

幸存者偏差是数据集只包含今天仍然存在或仍在指数中的资产，忽略退市、破产和被剔除的标的，导致历史表现被高估。正确做法是使用当时可获得的成分股和包含退市标的的历史数据，并记录数据供应商的修订规则。

#### 41. 回测中必须包含哪些成本？

至少考虑佣金、交易所费用、买卖价差、市场冲击、滑点、借券成本、融资成本、汇率影响和容量限制。成本模型应与交易频率、标的流动性、订单类型和市场状态匹配。忽略成本的高收益策略通常没有实际执行价值。

#### 42. 如何验证一个量化策略？

建议按以下顺序：

1. 明确假设和经济逻辑，避免先看结果再编故事。
2. 固化数据截止时间、特征定义、执行规则和成本模型。
3. 使用时间序列训练、验证和测试，必要时做 walk-forward。
4. 在不同市场、行业、时间段和参数扰动下做稳定性分析。
5. 检查换手、容量、最大回撤、尾部损失和交易可执行性。
6. 做纸面交易或仿真交易，再设计小规模上线和持续监控。

#### 43. 什么是组合优化？

组合优化是在收益、风险、相关性、约束和成本之间求解仓位。经典均值-方差问题可以写为：

$$
\min_w \quad \frac{1}{2}w^T\Sigma w - \lambda \mu^T w
$$

其中 $w$ 是权重，$\Sigma$ 是协方差矩阵，$\mu$ 是预期收益，$\lambda$ 控制风险厌恶程度。实际模型还应加入权重上下限、总敞口、行业中性、换手、流动性、杠杆和交易成本约束。

#### 44. 协方差矩阵为什么重要？如何处理不稳定？

协方差矩阵决定组合风险和资产之间的对冲关系。在资产数多、样本少或特征高度相关时，样本协方差可能噪声很大、接近奇异。可以使用 shrinkage、因子模型、滚动窗口、稳健估计或正则化，并通过样本外风险和组合稳定性验证选择。

#### 45. 信号、预测收益和仓位有什么区别？

信号可以是排序、方向判断或概率；预测收益是模型输出的数值估计；仓位还需要考虑风险预算、相关性、约束、交易成本和投资组合整体状态。不能把一个未经校准的模型分数直接当成下单权重。

---

## 五、数据接入、Kafka 和研究平台

### 5.1 数据接入高频题

#### 46. 如何设计市场数据接入系统？

先区分实时行情、历史数据、参考数据、基本面和另类数据的频率与可靠性。建议采用：

1. 原始层保存供应商原始数据，支持审计和重放。
2. 校验层检查 schema、时间戳、重复、缺失、异常价格和交易日历。
3. 标准化层统一字段、时区、证券标识、复权规则和单位。
4. 服务层提供研究查询、实时订阅和回测数据接口。
5. 元数据层记录来源、版本、更新时间、质量指标和血缘。

#### 47. 如何保证数据质量？

建立可自动执行的规则：字段类型、主键唯一性、时间连续性、范围、空值率、重复率、跨源一致性和延迟。质量失败时要区分阻断发布、降级使用和告警继续运行三种策略，并保留坏数据样本和修复记录。

#### 48. 如何保证研究结果可复现？

固定代码版本、依赖版本、数据快照或数据版本、参数、随机种子、交易日历、时区和环境；记录实验输入、输出、指标和生成时间。Notebook 只是探索界面，核心逻辑应沉淀到可测试的 Python 包或服务中。

### 5.2 Kafka 基础

#### 49. Kafka 的核心组件是什么？

Kafka 集群由 Broker 提供存储和服务；Topic 由多个 Partition 组成；Producer 写入消息；Consumer 从 Partition 读取；Consumer Group 让多个消费者分担分区；Offset 表示消费位置。Partition 是并行度、吞吐和顺序保证的基本单位。

#### 50. Kafka 如何保证顺序？

Kafka 只能保证单个 Partition 内的消息顺序。要让同一证券或同一业务实体有序，应使用稳定的 key，使相关消息进入同一 Partition；同时消费者需要按顺序处理。增加分区可能改变 key 到分区的分布和总体并行关系，必须评估影响。

#### 51. Kafka 的 at-most-once、at-least-once 和 exactly-once 是什么？

- **At-most-once**：可能丢消息，但通常不重复。
- **At-least-once**：不轻易丢消息，但可能重复，是常见默认思路。
- **Exactly-once**：需要从生产、Kafka 处理到下游写入的完整链路配合，不能只打开一个配置就保证业务层完全不重复。

实际系统通常使用至少一次投递加幂等消费者，并用业务唯一键、事务或去重表保护下游。

#### 52. 如何处理 Consumer Rebalance？

Rebalance 会重新分配分区，可能导致暂停、重复处理或处理延迟。要合理设置心跳、session timeout、poll interval，避免单次处理时间过长；使用批量处理、手动提交 offset 和幂等写入；必要时使用 cooperative rebalancing。消费者必须能安全地暂停、恢复和重新获得分区。

#### 53. Offset 应该何时提交？

通常在消息被成功处理并完成下游持久化后提交；如果先提交再写下游，进程故障可能造成数据丢失。如果先写下游再提交，重启时可能重复处理，所以需要幂等。提交策略应和业务一致性、吞吐、重放能力和失败恢复时间一起设计。

#### 54. Kafka 和消息队列如何选择？

Kafka 适合高吞吐、可保留、可重放的事件流和多个消费者；SQS 等托管队列更适合任务分发、削峰和较少的运维管理。判断时考虑消息是否需要长期保留、是否需要按 offset 重放、消费者数量、顺序、吞吐、延迟、运维能力和云平台标准。

---

## 六、数据库高频题

### 6.1 SQL 与关系数据库

#### 55. 如何为查询设计索引？

从实际查询的过滤、连接、排序和选择性出发设计索引，注意联合索引的列顺序和最左匹配规则；避免索引过多导致写入、存储和维护成本增加。使用 `EXPLAIN` 或执行计划验证优化器是否使用索引，不能仅凭索引名称判断有效。

#### 56. 事务的 ACID 是什么？

- Atomicity：事务中的操作要么全部成功，要么全部失败。
- Consistency：事务前后满足约束和业务不变量。
- Isolation：并发事务之间按隔离级别呈现受控可见性。
- Durability：提交后数据在故障恢复后仍然持久。

面试时应继续说明数据库隔离级别、锁、死锁、重试和业务幂等。

#### 57. 常见事务隔离级别有哪些？

常见级别包括 Read Uncommitted、Read Committed、Repeatable Read 和 Serializable。隔离越强通常并发能力或锁冲突成本越高，不同数据库的实现细节不同。选择时需要结合脏读、不可重复读、幻读、吞吐和业务一致性要求。

#### 58. 如何排查慢 SQL？

先确认慢的是数据库执行还是网络/应用等待，再查看执行计划、扫描行数、索引、排序、锁等待、统计信息、连接数和数据量变化。解决方案可能是重写 SQL、补充或调整索引、分区、预聚合、缓存、读副本或改变访问模式，而不是盲目增加机器规格。

#### 59. PostgreSQL、MySQL 和 MongoDB 如何选择？

PostgreSQL 适合功能丰富、复杂查询、强一致事务和扩展性要求较高的关系场景；MySQL 适合成熟的 OLTP 和广泛生态；MongoDB 适合文档结构、模式变化较快且访问模式以文档聚合为主的场景。数据库选择应从数据关系、事务、一致性、查询模式、扩展、团队能力和运维成本出发。

#### 60. 如何设计时序数据表？

明确主键、证券标识、事件时间、接收时间、数据版本和数据供应商；常见优化包括按日期或标的分区、合适的复合索引、批量写入、冷热数据分层和压缩。事件时间与接收时间必须同时保留，便于重建当时可见的数据和分析延迟。

### 6.2 NoSQL 和数据访问

#### 61. 什么情况下使用 MongoDB？

当数据天然以文档形式访问，嵌套结构较多，且查询和事务模型与文档聚合匹配时可以考虑 MongoDB。不能因为 schema 灵活就忽略索引、文档大小、数组增长、事务范围、数据一致性和分片键设计。

#### 62. 为什么应用层需要连接池？

频繁建立数据库连接会消耗网络、认证和数据库资源。连接池可以复用连接并控制最大并发，但必须设置连接生命周期、空闲回收、超时、健康检查和故障重连。连接池大小不是越大越好，要和数据库最大连接数、应用副本数和查询耗时共同计算。

---

## 七、生产化、Linux、Git 和 Kubernetes

### 7.1 测试与生产开发

#### 63. 数据科学代码如何测试？

分层测试：

- 单元测试：测试纯函数、边界值、异常和指标计算。
- 数据契约测试：测试 schema、字段类型、主键和时间范围。
- 集成测试：测试数据库、Kafka、外部 API 和文件系统交互。
- 回测回归测试：固定小样本和预期结果，防止信号逻辑悄悄改变。
- 端到端测试：验证从数据接入到结果发布的完整链路。

测试不只验证“代码能运行”，还要验证时间对齐、无未来数据、幂等和数值容差。

#### 64. 如何处理配置和密钥？

代码中不写密码、Token、数据库连接字符串和生产地址。通过环境变量、密钥管理服务或部署平台注入；配置按环境分层，密钥可轮换并限制读取权限。日志、异常和 Notebook 输出也要防止泄露敏感数据。

#### 65. 如何设计日志和监控？

结构化日志应至少包含时间、服务名、版本、环境、请求/追踪 ID、证券或任务 ID、处理延迟和结果状态。监控覆盖吞吐、延迟、错误率、数据新鲜度、缺失率、Kafka lag、数据库连接、容器重启、策略风险和业务成功率。告警必须有阈值、责任人和处理 Runbook。

### 7.2 Linux 基础

#### 66. Linux 进程、线程和常用排查命令？

进程拥有独立的地址空间，线程共享进程资源。常用排查工具包括：

- `ps`、`top`、`htop`：进程和资源使用。
- `free`、`vmstat`：内存和系统活动。
- `df`、`du`、`iostat`：磁盘空间和 I/O。
- `ss`、`lsof`：端口、连接和文件描述符。
- `journalctl`、`systemctl`：服务和系统日志。
- `curl`、`dig`、`nc`：HTTP、DNS 和网络连通性。

排障时先区分 CPU、内存、磁盘、网络、依赖服务和应用逻辑问题，再做针对性处理。

#### 67. 如何排查 Python 服务内存不断上涨？

先确认是实际泄漏、缓存无上限、任务积压、对象生命周期过长还是监控误判。观察进程 RSS、堆对象和 GC，使用 `tracemalloc`、内存 profiler 和分阶段压测定位增长点；检查全局容器、闭包、队列、DataFrame 引用和未关闭资源。修复后要进行长时间稳定性测试。

### 7.3 Git 和交付

#### 68. 如何管理研究代码和生产代码？

将可复用的核心逻辑放入版本化 Python 包，把 Notebook 用于探索和展示；通过 Pull Request、代码审查、自动测试和制品版本进入生产。数据、代码、配置和模型版本都要可追踪。实验分支不能直接成为生产入口。

#### 69. Merge 和 Rebase 的区别？

Merge 保留分支合并历史并创建合并提交；Rebase 把提交重新应用到新的基线，历史更线性但会改写提交 ID。公共分支不应随意 rebase，团队应统一分支、审查、回滚和发布规范。

### 7.4 Docker 和 Kubernetes

#### 70. 为什么要容器化 Python 应用？

容器将代码、运行时和依赖打包，提升环境一致性和部署可重复性。镜像应使用固定基础版本、非 root 用户、多阶段构建、依赖锁定、漏洞扫描和健康检查；不要把密钥写进镜像层。

#### 71. Kubernetes 中 Pod、Deployment、Service 分别是什么？

Pod 是最小调度单元，通常包含一个或多个紧密协作的容器；Deployment 管理无状态 Pod 的副本、滚动更新和回滚；Service 提供稳定的服务发现和负载均衡入口。生产部署还要配置 requests/limits、readiness/liveness probe、PodDisruptionBudget、滚动策略、日志和权限。

#### 72. 如何在 Kubernetes 上部署一个数据处理服务？

先定义镜像、配置和 Secret，再使用 Deployment 或 Job/CronJob 匹配持续服务或批处理任务；设置资源请求和限制、探针、自动扩缩容、重启策略和日志；用 Service 或消息系统连接上下游；为部署配置版本、监控、回滚和故障演练。不要把有状态数据库简单放进集群，除非团队具备明确的存储和运维能力。

#### 73. Kubernetes 上的服务为什么反复重启？

先看 `kubectl describe pod`、容器日志和事件，判断是 OOMKilled、探针失败、镜像拉取失败、配置/Secret 错误、依赖不可达还是应用主动退出。再核对资源限制、启动时间、端口、DNS、Service、网络策略和版本差异。不要仅靠增加重启次数解决根因。

---

## 八、场景设计题

### 74. 如何设计“新数据接入 + 研究查询”平台？

一个可落地的方案可以分为：

1. **接入层**：供应商 API、文件、Kafka 或 SFTP 进入原始区，保留原始 payload 和接收时间。
2. **校验层**：进行 schema、重复、缺失、范围、时区、标识映射和数据新鲜度检查。
3. **处理层**：使用 Python/Pandas 或分布式计算进行标准化、复权、特征生成和版本化。
4. **存储层**：原始数据放对象存储，结构化查询放 PostgreSQL/数仓，实时状态放 Kafka 或缓存。
5. **服务层**：提供研究 API、批量下载、实时订阅和回测接口。
6. **治理层**：记录数据血缘、质量指标、版本、权限、成本和审计信息。

关键追问包括：数据如何补数、如何重放、如何保证版本一致、如何区分事件时间和接收时间、研究结果如何复现。

### 75. 如何实现一个交易信号生产服务？

信号服务读取经过校验的行情和参考数据，按交易日历和时间窗口计算特征，生成带版本的信号，经过风险检查和权限控制后发布到消息主题或下游组合服务。必须定义：

- 信号的计算时间、数据截止时间和执行时间。
- 重复消息和迟到消息如何处理。
- 断点恢复和历史重放如何实现。
- 信号版本、参数和代码如何审计。
- 上游缺数、异常波动或模型失败时如何降级和停止发布。
- 生产指标如何监控，例如延迟、覆盖率、异常值、换手和信号漂移。

### 76. 如何设计组合优化工具？

输入包括预期收益、风险模型、协方差或因子暴露、当前持仓、交易成本和投资限制。优化器输出目标权重、交易差额、约束违约信息和求解状态。生产化时应保存输入快照、模型版本、求解器版本和输出结果；当问题不可行时返回可解释的约束冲突，而不是静默地产生一个看似合理的组合。

### 77. 如何和研究员、投资经理澄清需求？

先把口头目标转换为可验证的输入、输出和验收指标：数据范围、计算频率、允许延迟、基准、风险约束、交易成本、可解释性和上线方式。对于“提高收益”这类模糊目标，要继续追问是提高绝对收益、风险调整收益、容量还是降低回撤。用小样本原型和可视化结果快速确认方向，再拆成可交付的工程任务。

### 78. 研究环境和生产环境如何隔离？

研究环境可以快速试验，但读取生产数据应使用只读权限并保留审计；生产信号只允许经过审查和测试的版本；研究数据快照、代码、参数和模型制品要可追踪；生产发布使用自动化流程和审批。隔离不是禁止研究，而是避免实验代码和不稳定数据直接影响真实交易。

---

## 九、英文面试高频题

岗位要求英文口语流利，至少要能用英文清楚回答下面问题。回答结构建议是：**context -> action -> result -> lesson**。

### 79. Tell me about yourself.

> I am a Python-focused data scientist with strong experience in data processing, statistical modeling, and production engineering. I enjoy working closely with researchers and business stakeholders to turn ambiguous ideas into reliable, testable systems. My strengths are building reproducible data pipelines, validating time-series signals, and taking research code into production with monitoring and automated tests.

### 80. Tell me about a project you are proud of.

回答时说明业务背景、个人负责部分、技术选择、遇到的困难、量化结果和复盘。不要只介绍团队做了什么，要明确使用第一人称说明自己的贡献。

### 81. How do you prevent data leakage in a time-series model?

> I define the information availability timestamp for every feature and label. I use chronological splits or walk-forward validation, fit preprocessing steps only on the training window, and apply an execution delay between signal generation and trading. I also test the pipeline with deliberately shifted data to detect accidental look-ahead bias.

### 82. How do you handle disagreement with a researcher or portfolio manager?

> I first clarify the shared objective and the assumptions behind each proposal. Then I use a small, reproducible experiment with agreed metrics, such as out-of-sample performance, turnover, drawdown, or latency. If the evidence is inconclusive, I document the trade-off and propose a controlled next step instead of turning the discussion into a personal preference.

### 83. How do you explain a complex model to a non-technical stakeholder?

> I start with the decision the model supports, not the algorithm. I explain the key inputs, expected behavior, main risks, and a simple example. I also show out-of-sample performance, failure cases, and how the model is monitored. The goal is to make the decision process transparent without claiming that model explanations prove causality.

### 84. Describe a production incident.

建议按以下结构回答：

- What was the impact?
- How did you detect and contain it?
- What was the root cause?
- How did you restore the service or data?
- What did you change to prevent recurrence?

### 85. How do you prioritize competing requests?

> I evaluate business impact, risk, urgency, dependencies, and effort. For trading or production work, safety and regulatory requirements take priority over convenience. I make the trade-offs visible, agree on milestones with stakeholders, and revisit the priority when new evidence or incidents change the situation.

### 86. Why are you interested in this role?

回答应结合岗位的研究协作、量化交易、Python 工程和平台建设，不要只说“因为贵公司很有名”。可以强调希望把统计和数据能力转化为稳定、可复现、可运营的投资工具。

---

## 十、面试官可能追问的陷阱

| 追问 | 常见低分回答 | 更好的回答方向 |
|---|---|---|
| Pandas 很慢怎么办？ | 换成 NumPy | 先 profiling，再分析数据量、拷贝、类型、连接和算法复杂度 |
| 回测 Sharpe 很高是否说明策略好？ | 是 | 检查泄漏、成本、样本外、容量、回撤和稳定性 |
| Kafka exactly-once 能否保证业务不重复？ | 能 | 区分 Kafka 语义和端到端业务幂等 |
| 数据库连接越多越好吗？ | 越多吞吐越高 | 结合数据库上限、连接池、查询耗时和副本数量计算 |
| 模型准确率 95% 是否优秀？ | 是 | 看类别分布、业务成本、时间外推和基准模型 |
| Kubernetes 会自动保证高可用吗？ | 会 | 说明副本、节点、探针、资源、分区和应用无状态化 |
| Notebook 能直接上线吗？ | 可以 | 核心逻辑需包化、测试、版本化、配置化和可监控 |
| 特征和收益相关就能交易吗？ | 能 | 区分统计相关、经济逻辑、交易成本和可执行性 |
| 发现生产数据错误怎么办？ | 重新跑一遍 | 先止损和标记影响范围，再版本化修复、重放、校验和审计 |

---

## 十一、面试前准备清单

### Python 和数据科学

- [ ] 能解释可变/不可变、生成器、装饰器、上下文管理器和 GIL。
- [ ] 能写出清晰的 Pandas `merge`、`groupby`、`rolling`、`shift` 和时间对齐代码。
- [ ] 能说明 NumPy 广播、view/copy、dtype 和向量化的性能边界。
- [ ] 能解释 p-value、置信区间、平稳性、共线性、过拟合和时间序列验证。
- [ ] 能用 Pipeline 防止预处理和特征选择泄漏。

### 量化与系统设计

- [ ] 能计算并解释收益率、Sharpe、最大回撤、波动率和换手率。
- [ ] 能识别 look-ahead bias、survivorship bias 和交易成本遗漏。
- [ ] 能画出数据接入、研究、回测、信号发布和组合优化的链路。
- [ ] 能说明 Kafka 分区、offset、消费组、重平衡和幂等处理。
- [ ] 能解释 SQL 索引、事务、隔离级别、连接池和数据库选型。

### 生产工程和沟通

- [ ] 能从 Linux 命令、日志、指标和依赖关系定位一个生产问题。
- [ ] 能说明测试、Git、CI/CD、Docker 和 Kubernetes 的交付流程。
- [ ] 能讲清楚一个真实项目中的个人贡献、困难、结果和复盘。
- [ ] 能用英文解释一个模型、一次故障和一个技术取舍。
- [ ] 能回答如何与研究员、投资经理和工程团队协作推进项目。

> 最终目标不是背诵库的 API，而是证明你能把数据质量、统计严谨性、量化逻辑和生产工程连接起来，并对上线后的结果负责。
