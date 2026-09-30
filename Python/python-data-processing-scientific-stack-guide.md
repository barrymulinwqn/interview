# Python 数据处理科学计算工具栈：从基础到进阶

> 面向 Data Scientist、数据分析师和量化研究人员的实战手册。本文围绕 `NumPy`、`pandas`、`SciPy`、`statsmodels`、`scikit-learn`（注意拼写，不是 scikti-learn），并补充常见的数据读取、存储、校验与大数据处理依赖。

## 一、工具如何协同完成数据科学工作

数据科学工作不是只训练一个模型，而是一条可复现的数据链路：

```text
数据源 -> 读取与存储 -> 清洗与转换 -> 探索与统计推断 -> 特征工程
     -> 训练与验证 -> 解释与交付 -> 监控与复现
```

| 阶段 | 首选工具 | 解决的问题 |
|---|---|---|
| 数值计算 | `NumPy` | 多维数组、矩阵运算、随机抽样、广播计算 |
| 表格和时序处理 | `pandas` | 清洗、关联、分组、重塑、时间序列特征 |
| 科学计算与优化 | `SciPy` | 统计检验、分布、插值、优化、稀疏矩阵和信号处理 |
| 统计推断和可解释模型 | `statsmodels` | 回归摘要、置信区间、假设检验、时间序列模型 |
| 机器学习流水线 | `scikit-learn` | 特征预处理、交叉验证、建模、调参和评估 |
| 列式格式和跨引擎数据 | `PyArrow`、`Parquet` | 高效存储、类型保真、和 Spark/Polars/DuckDB 互操作 |
| 超出单机内存的数据 | `Polars`、`Dask`、`DuckDB` | 更快的表格执行、分布式计算、直接查询文件 |
| 数据库和数据质量 | `SQLAlchemy`、`Pandera` | 可靠读写数据库、模式约束和数据质量检测 |

一个可靠的项目通常以 `pandas` 负责业务语义清晰的表格处理，用 `NumPy` 负责高性能数值核心；需要统计结论时使用 `statsmodels` 或 `SciPy`，需要预测和可部署的预处理流程时使用 `scikit-learn`。不要把它们当成相互替代的工具。

---

## 二、环境与项目基线

### 2.1 安装核心依赖

```bash
python -m pip install numpy pandas scipy statsmodels scikit-learn
python -m pip install pyarrow duckdb polars sqlalchemy pandera
python -m pip install matplotlib seaborn jupyter
```

| 依赖 | 适用场景 |
|---|---|
| `pyarrow` | 读写 Parquet、Arrow 类型、和不同计算引擎交换数据 |
| `openpyxl` | 读取或写入 Excel 文件 |
| `duckdb` | 用 SQL 直接查询 CSV、Parquet 或 DataFrame |
| `polars` | 高性能、惰性执行的单机表格处理 |
| `dask[dataframe]` | 数据量超过内存后的并行 DataFrame 工作负载 |
| `sqlalchemy` | 数据库连接和参数化读写 |
| `pandera` | DataFrame 模式校验与数据质量检测 |
| `joblib` | 并行执行和序列化 scikit-learn 模型 |

建议使用虚拟环境，并将经过验证的版本锁定到 `requirements.txt`、`pyproject.toml` 或锁文件中。数据科学项目最难复现的问题之一，往往不是模型，而是依赖和数据版本漂移。

### 2.2 推荐项目结构

```text
project/
├── data/
│   ├── raw/              # 不修改的原始数据
│   ├── interim/          # 中间产物
│   └── processed/        # 可供建模的数据
├── notebooks/            # 探索，不承担生产逻辑
├── src/
│   ├── data.py           # 读取、校验、清洗
│   ├── features.py       # 特征工程
│   ├── train.py          # 训练与验证
│   └── evaluate.py       # 指标和报告
├── tests/
├── requirements.txt
└── README.md
```

原则是：**原始数据不可变、转换逻辑代码化、特征和模型可再生成、实验参数有记录**。Notebook 用于探索和解释，稳定逻辑应迁移到可测试的 Python 模块。

---

## 三、NumPy：数值计算的底座

`NumPy` 的核心对象是 `ndarray`。它适合统一数据类型的密集数值计算，例如矩阵、向量、图像像素和模型输入。相较 Python `list`，它减少了 Python 对象开销，并把大量循环放到经过优化的底层实现中执行。

### 3.1 基础：数组、索引和聚合

```python
import numpy as np

values = np.array([10.0, 12.0, np.nan, 15.0])
matrix = np.array([[1, 2, 3], [4, 5, 6]], dtype=np.float64)

print(matrix.shape)             # (2, 3)
print(matrix[:, 1])             # 第二列: [2. 5.]
print(np.nanmean(values))       # 忽略 NaN 的均值
print(np.nan_to_num(values, nan=0.0))

positive = values[values > 11]
```

常用构造方式：

```python
zeros = np.zeros((3, 4))
sequence = np.arange(0, 10, 2)
grid = np.linspace(0.0, 1.0, num=5)
identity = np.eye(3)
```

数据科学实践中，应优先关注：

- `shape`：每个维度的大小，避免把样本轴和特征轴写反。
- `dtype`：`float64` 精度较高但内存更大；分类编码有时可使用较小整数类型。
- `NaN`：缺失值与数值运算会传播，使用 `np.nanmean`、`np.nanmedian` 等函数时要明确语义。
- `axis`：`axis=0` 通常沿样本维度聚合得到每列统计量，`axis=1` 通常得到每行统计量。

### 3.2 广播：避免 Python 层循环

广播（broadcasting）让形状兼容的数组逐元素计算。下面的标准化不需要显式遍历行：

```python
features = np.array(
    [[10.0, 100.0], [12.0, 80.0], [15.0, 120.0]],
    dtype=np.float64,
)

mean = features.mean(axis=0)
std = features.std(axis=0)
standardized = (features - mean) / std
```

设 `features` 的形状为 $(n, p)$，`mean` 和 `std` 的形状为 $(p,)$。NumPy 会把后者按行逻辑扩展到 $(n, p)$，而不是手工复制数据。

当维度不明确时用 `keepdims=True` 保留维度，降低形状错误风险：

```python
row_total = features.sum(axis=1, keepdims=True)
row_share = features / row_total
```

### 3.3 随机数、View 与数值稳定性

不要依赖全局随机状态。使用 `Generator` 并把随机种子作为实验配置的一部分：

```python
rng = np.random.default_rng(seed=42)
sample = rng.normal(loc=0.0, scale=1.0, size=1_000)
train_indices = rng.choice(len(sample), size=800, replace=False)
```

切片通常返回 view，和原数组共享内存；高级索引通常创建 copy：

```python
scores = np.array([60, 70, 80, 90])
view = scores[1:3]
view[0] = 999
print(scores)  # [ 60 999  80  90]

independent_scores = scores[1:3].copy()
```

对线性方程 $A x = b$，优先使用 `np.linalg.solve(A, b)`，不要显式计算 `np.linalg.inv(A) @ b`。浮点数比较用 `np.isclose` 或 `np.allclose`，而不是直接使用 `==`。可以用 `np.shares_memory(left, right)` 检查数组是否共享内存。

---

## 四、pandas：业务表格和时间序列处理中心

`pandas` 的 `Series` 和 `DataFrame` 提供带标签的表格操作。它特别适合 CSV/Excel/SQL 数据、时间序列、分组统计、关联和特征构造。

### 4.1 基础：读取、检查和明确类型

```python
import pandas as pd

orders = pd.read_csv(
    "data/raw/orders.csv",
    parse_dates=["order_time"],
    dtype={"customer_id": "string", "product_id": "string"},
)

print(orders.head())
print(orders.info())
print(orders.isna().sum().sort_values(ascending=False))
print(orders.duplicated().sum())
```

初次读取数据后，不要立刻开始建模。至少检查：

1. 行数、列数和主键是否唯一。
2. 每列类型是否符合业务含义，日期是否真的被解析为日期。
3. 缺失率、重复率、异常值和不合理取值。
4. 时间范围、时区和数据是否包含未来记录。
5. 标签列产生时间是否晚于特征可得时间，防止数据泄漏。

### 4.2 清洗和赋值

```python
clean_orders = (
    orders.drop_duplicates(subset=["order_id"])
    .assign(
        amount=lambda frame: pd.to_numeric(frame["amount"], errors="coerce"),
        channel=lambda frame: frame["channel"].fillna("unknown").str.lower(),
    )
)

invalid_amount = clean_orders["amount"].lt(0)
clean_orders.loc[invalid_amount, "amount"] = pd.NA

recent = clean_orders.loc[
    clean_orders["order_time"] >= "2026-01-01",
    ["customer_id", "order_time", "amount", "channel"],
].copy()
recent.loc[recent["amount"].isna(), "amount_missing"] = 1
recent.loc[recent["amount"].notna(), "amount_missing"] = 0
```

用 `.loc[rows, columns]` 明确赋值目标；当从原表得到后续要修改的子集时用 `.copy()`。这比关闭 `SettingWithCopyWarning` 更可靠。

填充策略必须与列的业务语义一致：数值缺失可用中位数、组内统计量或模型插补；分类缺失可保留为 `"unknown"`；时序数据只可使用当时已经可见的数据前向填充；目标变量缺失通常不能填充。

### 4.3 关联与聚合

```python
customers = pd.read_parquet("data/raw/customers.parquet")

enriched = clean_orders.merge(
    customers[["customer_id", "segment", "signup_date"]],
    on="customer_id",
    how="left",
    validate="many_to_one",
    indicator=True,
)

assert enriched.loc[enriched["_merge"] == "left_only"].empty
enriched = enriched.drop(columns="_merge")

customer_features = (
    enriched.groupby("customer_id", as_index=False)
    .agg(
        total_amount=("amount", "sum"),
        order_count=("amount", "size"),
        average_amount=("amount", "mean"),
        last_order_time=("order_time", "max"),
    )
)
```

`merge` 前后都应检查行数、主键数量、未匹配比例和重要数值总和。`validate="many_to_one"` 能尽早发现维表键重复导致的样本膨胀。

- `agg` 把每组压缩成汇总结果。
- `transform` 返回原表同样长度的结果，适合组内标准化、排名和占比。
- `pivot_table` 把长表变宽表，`melt` 把宽表变长表。
- 优先使用内置聚合、向量化和 `transform`，不要把 `groupby.apply` 当作默认方案。

### 4.4 时间序列与高级处理

```python
prices = pd.read_parquet("data/raw/prices.parquet")
prices["timestamp"] = pd.to_datetime(prices["timestamp"], utc=True)
prices = prices.sort_values(["symbol", "timestamp"])

prices["return_1d"] = prices.groupby("symbol")["close"].pct_change()
prices["moving_average_20"] = (
    prices.groupby("symbol")["close"]
    .transform(lambda series: series.rolling(20, min_periods=20).mean())
)
prices["target_next_return"] = prices.groupby("symbol")["return_1d"].shift(-1)
```

时序特征的首要约束是：第 $t$ 时刻的特征只能使用 $t$ 时刻之前或当时已发布的信息。不同频率、不同发布时间的数据应使用 `pd.merge_asof(..., direction="backward")` 关联最近的历史数据，而不是按日期字符串粗暴连接。

### 4.5 性能和内存

1. 读取大文件时用 `usecols`、合理 `dtype` 和 `chunksize`，避免无差别加载全部列。
2. 重复的低基数字符串列可转换为 `category`；先测量，不能假设所有字符串列都会节省内存。
3. 优先使用 Parquet 而不是反复读写 CSV；它保留类型、压缩更好，也支持按列读取。
4. 避免在循环中反复 `pd.concat`；先收集批次结果，最后一次性拼接。
5. 使用 `frame.memory_usage(deep=True)` 和真实数据基准识别瓶颈。

---

## 五、SciPy：统计分布、检验与优化

`SciPy` 建立在 NumPy 之上，提供更丰富的科学计算功能。`scipy.stats`、`scipy.optimize`、`scipy.sparse` 是数据科学中最常用的模块。

### 5.1 基础：描述分布和假设检验

```python
from scipy import stats

control = [10.2, 9.8, 10.5, 10.1, 9.9]
treatment = [10.7, 10.4, 10.9, 10.2, 10.8]

result = stats.ttest_ind(control, treatment, equal_var=False)
print(result.statistic, result.pvalue)

confidence_interval = stats.t.interval(
    confidence=0.95,
    df=len(treatment) - 1,
    loc=np.mean(treatment),
    scale=stats.sem(treatment),
)
```

不要只报告 $p$ 值。完整实验结论至少包括：样本量、均值或转化率差异、置信区间、效应量、检验前提和业务影响。$p < 0.05$ 不意味着效果一定有业务价值，也不表示结果可自动复现。

| 问题 | 常见函数 | 关键前提 |
|---|---|---|
| 两独立组均值差异 | `stats.ttest_ind` | 独立性；方差不等时用 Welch 检验 |
| 配对前后差异 | `stats.ttest_rel` | 配对关系正确 |
| 非正态两独立组 | `stats.mannwhitneyu` | 分布形状解释需谨慎 |
| 分类变量关联 | `stats.chi2_contingency` | 期望频数足够 |
| 相关性 | `stats.pearsonr` / `stats.spearmanr` | 线性或单调关系假设 |

### 5.2 进阶：参数估计与约束优化

```python
from scipy.optimize import minimize

expected_return = np.array([0.08, 0.11, 0.06])
covariance = np.array(
    [[0.10, 0.02, 0.01], [0.02, 0.16, 0.03], [0.01, 0.03, 0.08]]
)

def portfolio_variance(weights: np.ndarray) -> float:
    return float(weights @ covariance @ weights)

constraints = {"type": "eq", "fun": lambda weights: weights.sum() - 1.0}
bounds = [(0.0, 1.0)] * len(expected_return)
result = minimize(
    portfolio_variance,
    x0=np.full(len(expected_return), 1 / len(expected_return)),
    method="SLSQP",
    bounds=bounds,
    constraints=constraints,
)

if not result.success:
    raise RuntimeError(result.message)
weights = result.x
```

优化前应把目标函数、变量边界和约束写清楚，并始终检查 `result.success`。若输入收益或协方差略有变化就导致权重巨大跳变，说明问题可能病态，需要正则化、稳健估计或更强的业务约束。

### 5.3 稀疏数据

文本词袋、用户-商品交互等数据大部分元素为零。不要将其转换成密集数组：

```python
from scipy.sparse import csr_matrix

interactions = csr_matrix(
    ([1, 1, 1], ([0, 0, 1], [2, 4, 1])),
    shape=(2, 5),
)
```

`csr_matrix` 适合按行读取和矩阵乘法，`csc_matrix` 适合按列操作。后续模型也必须支持稀疏输入，否则内存优势会在转换时消失。

---

## 六、statsmodels：统计推断、诊断与时间序列

`scikit-learn` 更关注预测性能和工程化接口，`statsmodels` 更强调统计推断：系数、标准误、显著性、置信区间和模型诊断。两者应按问题目标搭配使用。

### 6.1 基础：可解释线性回归

```python
import statsmodels.formula.api as smf

model = smf.ols(
    "monthly_spend ~ age + income + C(channel)",
    data=customer_model_data,
).fit()

print(model.summary())
print(model.conf_int())
```

公式接口 `C(channel)` 会将分类变量正确地视为类别特征。回归摘要中的系数、标准误和 $p$ 值的解释依赖模型前提，不能把“统计显著”直接解释为因果关系。

### 6.2 进阶：稳健标准误和诊断

```python
robust_model = smf.ols(
    "monthly_spend ~ age + income + C(channel)",
    data=customer_model_data,
).fit(cov_type="HC3")

print(robust_model.summary())
```

建模后至少检查残差是否呈现系统性模式、高杠杆点是否支配结论、共线性是否导致系数不稳定，以及结果在不同时间段或人群切片上是否稳定。VIF 是诊断线索，不是机械的删变量规则。

### 6.3 时间序列：趋势、季节性和预测

```python
from statsmodels.tsa.statespace.sarimax import SARIMAX

series = daily_sales.asfreq("D").fillna(0.0)
model = SARIMAX(
    series,
    order=(1, 1, 1),
    seasonal_order=(1, 0, 1, 7),
    enforce_stationarity=False,
    enforce_invertibility=False,
)
fitted = model.fit(disp=False)
forecast = fitted.get_forecast(steps=14)
interval = forecast.conf_int()
```

时间序列不能随机打乱做普通交叉验证。应该按时间前进进行训练/验证切分，并保留预测区间。节假日、价格、促销等外生变量在预测时点是否已知，比选择 ARIMA 阶数更重要。

---

## 七、scikit-learn：从预处理到验证的可复现流水线

`scikit-learn` 提供统一的 `fit`、`transform`、`predict` 接口。它最重要的工程价值不是某一个算法，而是把预处理、模型、交叉验证和调参绑定为一个不会泄漏的流水线。

### 7.1 基础：训练、预测与评估

```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score
from sklearn.model_selection import train_test_split

X = customer_model_data[["age", "income"]]
y = customer_model_data["churned"]

X_train, X_test, y_train, y_test = train_test_split(
    X,
    y,
    test_size=0.2,
    random_state=42,
    stratify=y,
)

model = LogisticRegression(max_iter=1_000)
model.fit(X_train, y_train)
probability = model.predict_proba(X_test)[:, 1]
print(roc_auc_score(y_test, probability))
```

分类任务不应只看 accuracy，尤其是类别不均衡时。可根据业务目标结合 ROC-AUC、PR-AUC、召回率、精确率、F1、校准度和阈值下的成本收益。回归任务常结合 MAE、RMSE、$R^2$ 和残差分布。

### 7.2 标准做法：`ColumnTransformer` + `Pipeline`

下面的流水线在每个训练折中单独拟合缺失值填充器、缩放器和独热编码器，避免把验证集信息泄漏给训练过程：

```python
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler

numeric_features = ["age", "income", "order_count"]
categorical_features = ["channel", "segment"]

numeric_pipeline = Pipeline(
    steps=[
        ("imputer", SimpleImputer(strategy="median")),
        ("scaler", StandardScaler()),
    ]
)
categorical_pipeline = Pipeline(
    steps=[
        ("imputer", SimpleImputer(strategy="most_frequent")),
        ("one_hot", OneHotEncoder(handle_unknown="ignore")),
    ]
)
preprocessor = ColumnTransformer(
    transformers=[
        ("numeric", numeric_pipeline, numeric_features),
        ("categorical", categorical_pipeline, categorical_features),
    ]
)
pipeline = Pipeline(
    steps=[
        ("preprocessor", preprocessor),
        ("classifier", LogisticRegression(max_iter=1_000, class_weight="balanced")),
    ]
)
pipeline.fit(X_train, y_train)
```

不要在执行 `train_test_split` 之前对全量数据 `fit_transform` 标准化、插补或目标编码。这会让测试集的分布信息进入训练过程，导致离线指标虚高。

### 7.3 交叉验证与调参

```python
from sklearn.model_selection import GridSearchCV, StratifiedKFold

cross_validator = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
search = GridSearchCV(
    estimator=pipeline,
    param_grid={
        "classifier__C": [0.01, 0.1, 1.0, 10.0],
        "classifier__penalty": ["l2"],
    },
    scoring="roc_auc",
    cv=cross_validator,
    n_jobs=-1,
    refit=True,
)
search.fit(X_train, y_train)

best_pipeline = search.best_estimator_
print(search.best_params_, search.best_score_)
```

调参时先保留最终测试集；分类问题用 `StratifiedKFold`；同一客户、用户或设备不能同时出现在训练集和验证集时使用 `GroupKFold`；时序问题使用 `TimeSeriesSplit` 或业务定义的滚动窗口，不能使用随机 `KFold`。

### 7.4 进阶：时序验证、阈值和模型持久化

```python
import joblib
from sklearn.model_selection import TimeSeriesSplit, cross_val_score

time_split = TimeSeriesSplit(n_splits=5)
scores = cross_val_score(
    pipeline,
    X_time_ordered,
    y_time_ordered,
    cv=time_split,
    scoring="neg_mean_absolute_error",
)
mae = -scores.mean()

joblib.dump(best_pipeline, "artifacts/churn_pipeline.joblib")
```

模型概率不是天然可靠的业务概率。对于风险排序、营销触达或信贷决策，需检查校准曲线，必要时使用 `CalibratedClassifierCV`。分类阈值不应默认取 `0.5`，而应依据漏判与误判成本、资源容量和实际转化目标确定。

持久化整个 `Pipeline`，而不是只保存分类器。否则线上很容易遗漏训练时的列顺序、填充规则、编码器或缩放器。加载不可信的 pickle/joblib 文件有安全风险；生产环境应使用受控制品仓库，记录 Python 和依赖版本。

---

## 八、数据处理生态的关键补充依赖

### 8.1 PyArrow 与 Parquet：把 CSV 变成可靠数据资产

```python
events.to_parquet(
    "data/processed/events.parquet",
    engine="pyarrow",
    index=False,
    compression="zstd",
)

selected = pd.read_parquet(
    "data/processed/events.parquet",
    columns=["event_time", "user_id", "event_type"],
)
```

相较 CSV，Parquet 的优点包括列式存储、压缩、类型保真和按列读取。它能够减少 I/O、避免日期和空值在文本格式中被错误解析，并与 pandas、Polars、Spark、DuckDB 等工具互通。

### 8.2 DuckDB：直接用 SQL 分析文件

```python
import duckdb

monthly_revenue = duckdb.sql("""
    SELECT
        date_trunc('month', order_time) AS month,
        sum(amount) AS revenue
    FROM 'data/processed/orders/*.parquet'
    WHERE amount > 0
    GROUP BY 1
    ORDER BY 1
""").df()
```

DuckDB 适合本地分析大型 Parquet 数据集、快速验证 SQL 指标、在数据工程和分析语言之间协作。查询文件时仍需要了解数据分区、文件模式和时间字段语义；SQL 不会自动修复脏数据。

### 8.3 Polars：高性能和惰性执行

```python
import polars as pl

monthly = (
    pl.scan_parquet("data/processed/orders/*.parquet")
    .filter(pl.col("amount") > 0)
    .with_columns(pl.col("order_time").dt.truncate("1mo").alias("month"))
    .group_by("month")
    .agg(pl.col("amount").sum().alias("revenue"))
    .collect()
)
```

`scan_parquet` 的惰性模式可让引擎下推列选择和过滤条件，避免读取无关数据。Polars 对表达式、不可变数据转换和并行执行很有优势；若团队已大量依赖 pandas 的复杂索引语义，应先在独立流水线中验证迁移收益。

### 8.4 Dask：数据超过内存时的渐进选择

`Dask DataFrame` 的 API 与 pandas 相似，但并非所有 pandas 操作都能无缝扩展。它适合可分区、可并行的数据转换；全局排序、跨分区 `groupby`、大量小文件和频繁 `.compute()` 都可能很慢。

在引入 Dask 之前，先尝试只读需要的列、用 Parquet 分区、按批处理、使用 DuckDB/Polars，或重新设计聚合逻辑。分布式系统会放大数据倾斜、序列化和网络问题，不能只把 `import pandas as pd` 替换成 Dask。

### 8.5 SQLAlchemy：数据库读写边界

```python
from sqlalchemy import create_engine, text

engine = create_engine("postgresql+psycopg://user:password@host:5432/analytics")

with engine.connect() as connection:
    result = pd.read_sql(
        text("""
            SELECT customer_id, order_time, amount
            FROM orders
            WHERE order_time >= :start_time
        """),
        connection,
        params={"start_time": "2026-01-01"},
    )
```

连接信息应放在环境变量或密钥管理系统中，不能提交到代码库。使用参数化 SQL，避免字符串拼接；大查询要分批读取，并将计算尽量下推给数据库。

### 8.6 Pandera：让 DataFrame 有数据契约

```python
import pandera.pandas as pa
from pandera.typing import Series


class OrderSchema(pa.DataFrameModel):
    order_id: Series[str] = pa.Field(unique=True)
    customer_id: Series[str]
    amount: Series[float] = pa.Field(ge=0)
    order_time: Series[pa.DateTime]


validated_orders = OrderSchema.validate(clean_orders)
```

数据质量校验应尽可能靠近数据边界，例如读取文件、接收 API 响应或写入特征表时。模式不只是 dtype，还应包含非空、唯一、数值范围、允许类别、时间范围和行数变化阈值等业务规则。

---

## 九、端到端范例：从原始订单到可评估模型

下面展示一个缩小版的客户流失预测流程，重点是职责分离和避免泄漏，而不是追求最高分数。

```python
from pathlib import Path

import pandas as pd
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler

data = pd.read_parquet(Path("data/processed/customer_snapshot.parquet"))
data["snapshot_date"] = pd.to_datetime(data["snapshot_date"], utc=True)

target = "churned_next_30d"
features = ["age", "income", "orders_last_90d", "spend_last_90d", "channel", "segment"]
model_data = data.dropna(subset=[target]).copy()

# 标签在快照日后观察；特征仅使用快照日及以前的信息。
train = model_data.loc[model_data["snapshot_date"] <= "2026-06-30"]
test = model_data.loc[model_data["snapshot_date"] > "2026-06-30"]

numeric_features = ["age", "income", "orders_last_90d", "spend_last_90d"]
categorical_features = ["channel", "segment"]
preprocessor = ColumnTransformer(
    transformers=[
        (
            "numeric",
            Pipeline([
                ("imputer", SimpleImputer(strategy="median")),
                ("scaler", StandardScaler()),
            ]),
            numeric_features,
        ),
        (
            "categorical",
            Pipeline([
                ("imputer", SimpleImputer(strategy="most_frequent")),
                ("encoder", OneHotEncoder(handle_unknown="ignore")),
            ]),
            categorical_features,
        ),
    ]
)
pipeline = Pipeline([
    ("preprocessor", preprocessor),
    ("model", LogisticRegression(max_iter=1_000, class_weight="balanced")),
])

pipeline.fit(train[features], train[target])
test_probability = pipeline.predict_proba(test[features])[:, 1]
print(f"test ROC-AUC: {roc_auc_score(test[target], test_probability):.3f}")
```

生产前还需要补足：输入表模式检查和异常数据告警、滚动验证和不同人群切片报告、基线模型对比、阈值下的业务成本收益、特征版本和数据快照记录，以及线上数据漂移和真实标签回流监控。

---

## 十、Data Scientist 的高价值工作方式

### 10.1 将数据处理视为产品的一部分

模型通常只占数据科学交付的一小部分。定义清晰的指标、可信的时间字段、可追溯的数据来源和稳定的特征口径，决定了结论是否可信。每一个聚合和关联都应能回答：数据来自哪里、在什么时点可用、为何这样处理、出现异常怎么办？

### 10.2 先建立基线，再增加复杂度

在复杂梯度提升、深度学习或大规模调参之前，先建立：

1. 可解释的业务规则或历史均值基线。
2. 简单线性/逻辑回归或树模型基线。
3. 明确的时间、用户或实验分组验证策略。
4. 与业务目标绑定的离线指标和阈值策略。

这能快速判断复杂模型是否真正创造增量价值，也为排错提供参照。

### 10.3 区分预测、推断和因果

| 目标 | 重点工具和问题 |
|---|---|
| 预测 | `scikit-learn` 流水线、泛化误差、校准、线上性能 |
| 统计推断 | `statsmodels`、置信区间、标准误、模型前提 |
| 因果分析 | 实验设计、混杂因素、处理分配机制和识别假设 |

一个高 AUC 的模型不证明某个特征“导致”结果变化；一个回归中的显著系数也不自动成为可执行的干预策略。先澄清问题类型，才能选择正确的方法和沟通方式。

### 10.4 可复现、可审计、可协作

- 将数据读取、清洗、特征构造和训练写成小而可测试的函数。
- 使用 Git 管理代码，用固定数据快照或数据版本工具管理训练输入。
- 记录实验参数、指标、随机种子、样本范围、特征列表和模型版本。
- 对关键指标、主键唯一性、数据量突变和空值率建立自动化测试或告警。
- 在交接报告中同时说明有效范围、已知偏差、失败场景和回滚方法。

这些做法让数据科学家从“能跑出一个结果”升级为“能交付可靠、可解释、可持续维护的决策系统”。

---

## 十一、常见反模式检查清单

- [ ] 在全量数据上填充、缩放或编码后才切分训练集和测试集。
- [ ] 用随机交叉验证评估带明显时间依赖的数据。
- [ ] 忽略多对多 `merge` 造成的重复样本和指标膨胀。
- [ ] 把未来发布的字段、最终状态或人工处理结果当作预测时可用特征。
- [ ] 只报告单一分数，不报告切片表现、置信区间或业务成本。
- [ ] 仅保存模型对象，未保存预处理流程、特征定义和依赖版本。
- [ ] 依赖 Notebook 的手工执行顺序，无法从原始数据重新生成结果。
- [ ] 为了“更快”盲目使用 Dask、GPU 或分布式系统，未先定位真正瓶颈。

当上述问题被系统化地避免时，`NumPy`、`pandas`、`SciPy`、`statsmodels` 和 `scikit-learn` 就不只是几个库，而是一套支持探索、推断、预测、协作和生产交付的完整基础能力。