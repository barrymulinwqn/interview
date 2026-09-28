# Python 现场 Coding 高频题与参考答案

> 面向 Python 中高级开发工程师、后端架构师和技术负责人面试。题目覆盖 Python 基础、数据结构、生成器、装饰器、上下文管理器、线程、进程、asyncio、缓存、限流、重试、幂等和服务端工程设计。
>
> 文中的代码片段默认彼此独立，主要使用 Python 3.9+ 标准库。现场作答时不要直接背代码，建议先澄清输入约束、给出朴素方案，再说明优化、复杂度、边界条件、测试策略和生产环境取舍。

## 目录

1. [现场 Coding 的答题方法](#一现场-coding-的答题方法)
2. [Python 基础与数据结构题](#二python-基础与数据结构题)
3. [算法与实用编码题](#三算法与实用编码题)
4. [Python 特性 Coding 题](#四python-特性-coding-题)
5. [并发与异步 Coding 题](#五并发与异步-coding-题)
6. [缓存、限流与可靠性 Coding 题](#六缓存限流与可靠性-coding-题)
7. [综合工程 Coding 题](#七综合工程-coding-题)
8. [现场追问与测试清单](#八现场追问与测试清单)
9. [面试前自检](#九面试前自检)

---

## 一、现场 Coding 的答题方法

### 1. 先确认题目边界

写代码前先问清楚：

- 输入是否可能为 `None`、空集合、重复值、负数或超大数据？
- 结果是否要求保持顺序？是否允许修改输入？
- 数据量、延迟和内存限制是什么？
- 是否单线程？是否要求线程安全、取消、超时或幂等？
- 失败时返回空结果、异常、错误对象，还是部分结果？
- Python 版本和可使用的第三方库是什么？

### 2. 推荐答题顺序

1. 用一两个例子复述题意，确认输入和输出。
2. 先写最直接、容易验证的方案。
3. 解释数据结构、循环不变量或并发约束。
4. 写代码，并主动说出边界条件。
5. 给出时间和空间复杂度。
6. 用正常、边界、异常和并发场景验证。
7. 讨论生产环境的超时、日志、监控、取消、资源释放和回滚。

### 3. 高分实现的共同特征

- 方法职责单一，命名表达业务含义，不依赖隐式全局状态。
- 对输入和返回值的约定清楚，不用 `None` 隐式表达过多状态。
- 迭代器、文件句柄、锁、任务和连接都有明确的所有权与生命周期。
- 并发代码说明共享状态、锁范围、取消语义和异常传播。
- 重试、缓存和幂等代码说明失败路径，而不是只实现成功路径。

---

## 二、Python 基础与数据结构题

### Q1：两数之和

**题目：** 给定整数列表和目标值，返回和为目标值的两个元素下标。假设恰好存在一组答案。

**思路：** 遍历列表，用字典保存已经见过的值和下标；当前值的补数已经出现时直接返回。

```python
def two_sum(numbers: list[int], target: int) -> tuple[int, int]:
    index_by_value: dict[int, int] = {}

    for index, number in enumerate(numbers):
        complement = target - number
        if complement in index_by_value:
            return index_by_value[complement], index
        index_by_value[number] = index

    raise ValueError("no pair found")
```

**复杂度：** 平均时间 O(n)，空间 O(n)。

**边界与追问：** 相同数值需要允许使用两个不同下标；如果列表已排序，可以使用双指针将额外空间降到 O(1)；如果整数可能超出范围，应明确 Python 整数不会像固定宽度整数那样溢出，但外部协议仍可能有限制。

### Q2：判断括号是否有效

**题目：** 输入只包含 `()`、`[]`、`{}`，判断括号是否正确闭合和嵌套。

```python
def is_valid_brackets(text: str) -> bool:
    if len(text) % 2 != 0:
        return False

    opening_for = {
        ")": "(",
        "]": "[",
        "}": "{",
    }
    stack: list[str] = []

    for character in text:
        if character in "([{":
            stack.append(character)
        elif not stack or stack.pop() != opening_for.get(character):
            return False
        else:
            continue
        if character not in "([{":
            return False

    return not stack
```

**复杂度：** 时间 O(n)，空间 O(n)。

**边界与追问：** 空字符串是否有效要根据题意确认；如果输入可能包含其他字符，应先定义忽略、拒绝还是当作普通文本。现场也可以用 `pairs = {')': '(', ...}` 并把逻辑写得更直观。

### Q3：反转单链表

**题目：** 给定单链表头节点，原地反转链表并返回新头节点。

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass
class ListNode:
    value: int
    next: ListNode | None = None


def reverse_list(head: ListNode | None) -> ListNode | None:
    previous: ListNode | None = None
    current = head

    while current is not None:
        next_node = current.next
        current.next = previous
        previous = current
        current = next_node

    return previous
```

**复杂度：** 时间 O(n)，额外空间 O(1)。

**边界与追问：** 空链表和单节点直接返回；递归写法空间为 O(n)，大链表更适合迭代；如果不能修改输入，应复制节点而不是原地反转。

### Q4：判断链表是否有环并找到入口

**题目：** 判断链表是否存在环；如果存在，返回环的入口节点。

```python
def find_cycle_entry(head: ListNode | None) -> ListNode | None:
    slow = head
    fast = head

    while fast is not None and fast.next is not None:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            entry = head
            while entry is not slow:
                entry = entry.next
                slow = slow.next
            return entry

    return None
```

**复杂度：** 时间 O(n)，额外空间 O(1)。

**边界与追问：** 必须比较节点对象身份而不是节点值；单节点自环应返回自身；如果只需要判断有无环，第一次相遇即可返回。

### Q5：合并两个有序链表

**题目：** 将两个升序链表合并为一个升序链表，尽量复用原节点。

```python
def merge_sorted_lists(
    first: ListNode | None,
    second: ListNode | None,
) -> ListNode | None:
    dummy = ListNode(0)
    tail = dummy

    while first is not None and second is not None:
        if first.value <= second.value:
            tail.next = first
            first = first.next
        else:
            tail.next = second
            second = second.next
        tail = tail.next

    tail.next = first if first is not None else second
    return dummy.next
```

**复杂度：** 时间 O(m + n)，额外空间 O(1)。

**边界与追问：** `<=` 可以在值相等时保留第一个链表的相对优先级；如果输入不保证有序，应先澄清而不是默默排序。

### Q6：最长无重复字符子串

**题目：** 返回字符串中不包含重复字符的最长连续子串长度。

```python
def longest_unique_substring(text: str) -> int:
    last_index_by_character: dict[str, int] = {}
    left = 0
    maximum_length = 0

    for right, character in enumerate(text):
        previous_index = last_index_by_character.get(character)
        if previous_index is not None:
            left = max(left, previous_index + 1)
        last_index_by_character[character] = right
        maximum_length = max(maximum_length, right - left + 1)

    return maximum_length
```

**复杂度：** 时间 O(n)，空间 O(k)，k 为字符集大小或窗口内字符数。

**边界与追问：** Python `str` 按 Unicode code point 迭代，但一个 code point 仍不一定等于用户感知的 grapheme cluster；如果需要处理组合字符，要说明 ICU 或专门分词库。

### Q7：最大子数组和

**题目：** 返回非空整数列表的最大连续子数组和。

```python
def maximum_subarray_sum(numbers: list[int]) -> int:
    if not numbers:
        raise ValueError("numbers must not be empty")

    current_sum = maximum_sum = numbers[0]
    for number in numbers[1:]:
        current_sum = max(number, current_sum + number)
        maximum_sum = max(maximum_sum, current_sum)
    return maximum_sum
```

**复杂度：** 时间 O(n)，额外空间 O(1)。

**边界与追问：** 全为负数时返回最大的负数，不能把初始值设为 0；如果还要返回区间，需要记录当前起点和最佳左右边界。

### Q8：合并重叠区间

**题目：** 合并一组可能重叠的闭区间，例如 `[1, 3]` 和 `[3, 5]` 应合并。

```python
def merge_intervals(intervals: list[tuple[int, int]]) -> list[tuple[int, int]]:
    if not intervals:
        return []

    sorted_intervals = sorted(intervals)
    merged: list[tuple[int, int]] = [sorted_intervals[0]]

    for start, end in sorted_intervals[1:]:
        current_start, current_end = merged[-1]
        if start <= current_end:
            merged[-1] = current_start, max(current_end, end)
        else:
            merged.append((start, end))
    return merged
```

**复杂度：** 时间 O(n log n)，空间 O(n)。

**边界与追问：** 需要明确是闭区间还是半开区间；这里不修改输入，若允许修改可原地排序；非法区间 `start > end` 应拒绝、交换还是由调用方保证。

### Q9：滑动窗口最大值

**题目：** 给定整数列表和窗口大小，返回每个窗口中的最大值。

```python
from collections import deque
from dataclasses import dataclass


def sliding_window_maximum(numbers: list[int], window_size: int) -> list[int]:
    if window_size <= 0 or window_size > len(numbers):
        raise ValueError("invalid window size")

    result: list[int] = []
    decreasing_indexes: deque[int] = deque()

    for index, number in enumerate(numbers):
        while (
            decreasing_indexes
            and decreasing_indexes[0] <= index - window_size
        ):
            decreasing_indexes.popleft()
        while (
            decreasing_indexes
            and numbers[decreasing_indexes[-1]] <= number
        ):
            decreasing_indexes.pop()
        decreasing_indexes.append(index)

        if index >= window_size - 1:
            result.append(numbers[decreasing_indexes[0]])

    return result
```

**复杂度：** 时间 O(n)，空间 O(window_size)。

**边界与追问：** 双端队列保存下标而不是值，才能判断过期元素；相等值保留较新的下标可以减少队列长度。

### Q10：前 K 个高频元素

**题目：** 返回列表中频率最高的 K 个元素，顺序不作要求。

```python
from collections import Counter
import heapq


def top_k_frequent(numbers: list[int], k: int) -> list[int]:
    if k <= 0:
        return []
    frequency = Counter(numbers)
    return [number for number, _ in heapq.nlargest(k, frequency.items(), key=lambda item: item[1])]
```

**复杂度：** 常见实现时间 O(n + u log k)，空间 O(u)，u 为不同元素数。

**边界与追问：** 频率相同时是否需要稳定或字典序，需要先定义；海量数据不能全部放内存时，可以讨论分片统计、外部排序或近似 Top K。

### Q11：二叉树层序遍历

**题目：** 返回二叉树每一层的节点值。

```python
from collections import deque


@dataclass
class TreeNode:
    value: int
    left: TreeNode | None = None
    right: TreeNode | None = None


def level_order(root: TreeNode | None) -> list[list[int]]:
    if root is None:
        return []

    result: list[list[int]] = []
    queue: deque[TreeNode] = deque([root])
    while queue:
        level: list[int] = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.value)
            if node.left is not None:
                queue.append(node.left)
            if node.right is not None:
                queue.append(node.right)
        result.append(level)
    return result
```

**复杂度：** 时间 O(n)，队列空间 O(w)，w 为树的最大宽度。

### Q12：验证二叉搜索树

**题目：** 判断一棵二叉树是否满足严格递增的二叉搜索树定义。

```python
def is_valid_bst(root: TreeNode | None) -> bool:
    def validate(node: TreeNode | None, lower: int | None, upper: int | None) -> bool:
        if node is None:
            return True
        if lower is not None and node.value <= lower:
            return False
        if upper is not None and node.value >= upper:
            return False
        return validate(node.left, lower, node.value) and validate(node.right, node.value, upper)

    return validate(root, None, None)
```

**复杂度：** 时间 O(n)，递归空间 O(h)。

**边界与追问：** 重复值是否允许需要先确认；如果树非常深，递归可能触发 Python 递归深度限制，可以改成显式栈；如果是 BST 的查找而不是验证，可利用排序性质降低访问范围。

### Q13：二叉树最近公共祖先

**题目：** 给定普通二叉树和两个节点，返回它们的最近公共祖先。假设两个节点都在树中。

```python
def lowest_common_ancestor(
    root: TreeNode | None,
    first: TreeNode,
    second: TreeNode,
) -> TreeNode | None:
    if root is None or root is first or root is second:
        return root

    left_result = lowest_common_ancestor(root.left, first, second)
    right_result = lowest_common_ancestor(root.right, first, second)
    if left_result is not None and right_result is not None:
        return root
    return left_result if left_result is not None else right_result
```

**复杂度：** 时间 O(n)，递归空间 O(h)。

**边界与追问：** 如果不保证两个节点存在，需要额外返回 found 状态；如果是 BST，可以利用节点值将复杂度降到 O(h)。

### Q14：网格最短路径

**题目：** 给定只包含 `0` 和 `1` 的网格，`0` 表示可走，返回从左上角到右下角的最短步数，无法到达返回 `-1`。

```python
from collections import deque


def grid_shortest_path(grid: list[list[int]]) -> int:
    if not grid or not grid[0] or grid[0][0] == 1:
        return -1

    rows = len(grid)
    columns = len(grid[0])
    queue: deque[tuple[int, int, int]] = deque([(0, 0, 0)])
    visited = {(0, 0)}
    directions = ((1, 0), (-1, 0), (0, 1), (0, -1))

    while queue:
        row, column, distance = queue.popleft()
        if row == rows - 1 and column == columns - 1:
            return distance

        for row_delta, column_delta in directions:
            next_row = row + row_delta
            next_column = column + column_delta
            if (
                0 <= next_row < rows
                and 0 <= next_column < columns
                and grid[next_row][next_column] == 0
                and (next_row, next_column) not in visited
            ):
                visited.add((next_row, next_column))
                queue.append((next_row, next_column, distance + 1))

    return -1
```

**复杂度：** 时间 O(rows * columns)，空间 O(rows * columns)。

**边界与追问：** 如果边权不同，应使用 Dijkstra；如果只有 0/1 权重，可以使用 0-1 BFS；生产输入要校验每一行长度是否一致。

### Q15：拓扑排序判断依赖是否有环

**题目：** 输入任务数量和依赖关系 `(task, prerequisite)`，判断是否可以完成全部任务。

```python
from collections import deque


def can_complete_tasks(
    task_count: int,
    dependencies: list[tuple[int, int]],
) -> bool:
    graph: list[list[int]] = [[] for _ in range(task_count)]
    in_degree = [0] * task_count

    for task, prerequisite in dependencies:
        graph[prerequisite].append(task)
        in_degree[task] += 1

    ready = deque(index for index, degree in enumerate(in_degree) if degree == 0)
    completed = 0
    while ready:
        task = ready.popleft()
        completed += 1
        for next_task in graph[task]:
            in_degree[next_task] -= 1
            if in_degree[next_task] == 0:
                ready.append(next_task)

    return completed == task_count
```

**复杂度：** 时间 O(V + E)，空间 O(V + E)。

**边界与追问：** 如果需要返回执行顺序，记录出队顺序；如果需要字典序最小的顺序，将队列换成最小堆；生产系统还要保存失败任务和依赖原因。

### Q16：合并 K 个有序迭代器

**题目：** 给定 K 个升序迭代器，按升序输出所有元素，并且不要求把所有元素一次性放入内存。

```python
import heapq
from collections.abc import Iterable, Iterator
from typing import TypeVar


T = TypeVar("T")


def merge_sorted_iterables(iterables: list[Iterable[T]]) -> Iterator[T]:
    iterators = [iter(iterable) for iterable in iterables]
    heap: list[tuple[T, int]] = []

    for index, iterator in enumerate(iterators):
        try:
            heapq.heappush(heap, (next(iterator), index))
        except StopIteration:
            continue

    while heap:
        value, index = heapq.heappop(heap)
        yield value
        try:
            heapq.heappush(heap, (next(iterators[index]), index))
        except StopIteration:
            pass
```

**复杂度：** 总元素数为 N、迭代器数为 K 时，时间 O(N log K)，堆空间 O(K)。

**边界与追问：** 元素必须可比较；如果相同值需要稳定排序，堆元素中加入来源序号和来源内序号；生成器消费失败时要说明异常是否传播。

---

## 三、算法与实用编码题

### Q17：按条件稳定去重

**题目：** 保留列表中每个元素第一次出现的顺序，返回去重结果。

```python
from collections.abc import Hashable, Iterable
from typing import TypeVar


T = TypeVar("T", bound=Hashable)


def unique_in_order(values: Iterable[T]) -> list[T]:
    seen: set[T] = set()
    result: list[T] = []
    for value in values:
        if value not in seen:
            seen.add(value)
            result.append(value)
    return result
```

**复杂度：** 平均时间 O(n)，空间 O(u)，u 为不同元素数。

**边界与追问：** 如果元素不可哈希，需要通过序列化、键函数或线性比较实现，复杂度会改变；如果要按业务字段去重，应增加 `key: Callable` 而不是强制对象实现 hash。

### Q18：扁平化嵌套列表

**题目：** 将任意深度的嵌套列表扁平化，要求使用生成器，避免一次性创建完整结果。

```python
from collections.abc import Iterator
from typing import Any


def flatten(values: list[Any]) -> Iterator[Any]:
    for value in values:
        if isinstance(value, list):
            yield from flatten(value)
        else:
            yield value
```

**复杂度：** 遍历元素总数为 n 时，时间 O(n)，额外空间为递归深度 O(d)，结果不计入内存。

**边界与追问：** 深度过大时递归会触发限制，可使用显式栈；是否展开 tuple、set、字符串和自定义 Iterable 需要定义，不能把字符串当作字符列表无限拆开。

### Q19：最长连续数字区间

**题目：** 给定无序整数列表，返回最长连续整数区间的长度，要求平均 O(n)。

```python
def longest_consecutive(numbers: list[int]) -> int:
    values = set(numbers)
    longest = 0

    for value in values:
        if value - 1 not in values:
            length = 1
            while value + length in values:
                length += 1
            longest = max(longest, length)

    return longest
```

**复杂度：** 平均时间 O(n)，空间 O(n)。每个连续值只会在一个起点上被扩展。

**边界与追问：** 重复值不影响结果；如果要求保留区间起止点，需要定义多个最长区间时的选择规则。

### Q20：按字母异位词分组

**题目：** 将字符串列表中的字母异位词分组。

```python
from collections import defaultdict


def group_anagrams(words: list[str]) -> list[list[str]]:
    groups: defaultdict[tuple[str, ...], list[str]] = defaultdict(list)
    for word in words:
        key = tuple(sorted(word))
        groups[key].append(word)
    return list(groups.values())
```

**复杂度：** 设单词平均长度为 L，时间 O(n * L log L)，空间 O(n * L)。

**边界与追问：** 只包含小写英文字母时可使用 26 维计数将单词构造 key 的成本降到 O(L)；大小写、空格、Unicode 规范化和标点是否忽略需要明确。

### Q21：实现 Trie 字典树

**题目：** 支持插入单词、查询完整单词和查询前缀。

```python
class TrieNode:
    def __init__(self) -> None:
        self.children: dict[str, TrieNode] = {}
        self.is_word = False


class Trie:
    def __init__(self) -> None:
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        current = self.root
        for character in word:
            current = current.children.setdefault(character, TrieNode())
        current.is_word = True

    def contains(self, word: str) -> bool:
        node = self._find_node(word)
        return node is not None and node.is_word

    def starts_with(self, prefix: str) -> bool:
        return self._find_node(prefix) is not None

    def _find_node(self, text: str) -> TrieNode | None:
        current = self.root
        for character in text:
            current = current.children.get(character)
            if current is None:
                return None
        return current
```

**复杂度：** 插入和查询 O(L)，L 为字符串长度；空间 O(字符总数)。

**边界与追问：** 大字符集会增加内存；删除单词需要清理不再使用的路径；海量词典可讨论压缩 Trie、DAWG 或外部索引。

### Q22：流式统计大文件中的单词频率

**题目：** 统计文本文件中的单词频率，要求不一次性读入整个文件。

```python
from collections import Counter
import re


WORD_PATTERN = re.compile(r"[A-Za-z0-9_]+")


def count_words(path: str) -> Counter[str]:
    counts: Counter[str] = Counter()
    with open(path, encoding="utf-8") as file:
        for line in file:
            counts.update(match.group(0).lower() for match in WORD_PATTERN.finditer(line))
    return counts
```

**复杂度：** 时间 O(n)，其中 n 为字符数；Counter 空间 O(u)，u 为不同单词数。

**边界与追问：** 如果不同单词数也超过内存，需要分片统计后合并、外部排序或 Count-Min Sketch；生产实现要讨论编码、异常行、符号规则、文件轮转和压缩文件。

### Q23：解析 JSON Lines 日志并按状态码聚合

**题目：** 流式读取每行一个 JSON 对象的日志，统计每个 HTTP 状态码的数量，坏行不能阻断整个文件。

```python
import json
from collections import Counter
from collections.abc import Iterable
from typing import Any


def status_counts(lines: Iterable[str]) -> tuple[Counter[int], int]:
    counts: Counter[int] = Counter()
    invalid_lines = 0

    for line in lines:
        try:
            record: dict[str, Any] = json.loads(line)
            status = int(record["status"])
        except (json.JSONDecodeError, KeyError, TypeError, ValueError):
            invalid_lines += 1
            continue
        counts[status] += 1

    return counts, invalid_lines
```

**复杂度：** 时间与输入大小线性相关，额外空间为不同状态码数量。

**边界与追问：** 不应用 `except Exception` 静默吞掉所有错误；坏行是否要记录行号、采样日志或进入死信文件；状态字段类型和范围要做 schema 校验。

### Q24：实现批量分页迭代器

**题目：** 给定一个返回分页结果的函数，返回一个惰性迭代器，直到没有下一页为止。

```python
from collections.abc import Callable, Iterator
from typing import TypeVar


T = TypeVar("T")


def paginate(
    fetch_page: Callable[[int, int], tuple[list[T], bool]],
    page_size: int = 100,
) -> Iterator[T]:
    if page_size <= 0:
        raise ValueError("page_size must be positive")

    page_number = 1
    while True:
        items, has_next = fetch_page(page_number, page_size)
        yield from items
        if not has_next:
            return
        page_number += 1
```

**复杂度：** 除数据访问成本外，迭代器额外空间 O(1)，当前页数据空间由调用方决定。

**边界与追问：** 服务端分页可能出现重复或遗漏，稳定排序和 cursor pagination 通常比 offset 更可靠；要处理空页但 `has_next=True`、最大页数、请求超时和取消。

### Q25：实现一个可合并的区间迭代器

**题目：** 输入两个已经按起点排序的区间迭代器，惰性输出合并后的非重叠闭区间。

```python
from collections.abc import Iterable, Iterator


def merge_interval_iterators(
    first: Iterable[tuple[int, int]],
    second: Iterable[tuple[int, int]],
) -> Iterator[tuple[int, int]]:
    combined = sorted([*first, *second])
    if not combined:
        return

    current_start, current_end = combined[0]
    for start, end in combined[1:]:
        if start <= current_end:
            current_end = max(current_end, end)
        else:
            yield current_start, current_end
            current_start, current_end = start, end
    yield current_start, current_end
```

**复杂度：** 此版本时间 O(n log n)、空间 O(n)，因为用排序合并；如果要利用两个输入本身有序且保持流式，应实现双路归并，额外空间 O(1)。

**面试要点：** 现场应主动指出函数名虽包含 iterator，但当前实现仍物化输入；如果题目强调超大数据或真正惰性，必须改成双指针迭代器方案。

---

## 四、Python 特性 Coding 题

### Q26：实现带参数的计时装饰器

**题目：** 实现 `@measure_time`，记录函数耗时，同时保留原函数名称、文档和签名元信息。

```python
from functools import wraps
from time import perf_counter
from collections.abc import Callable
from typing import Any, TypeVar


R = TypeVar("R")


def measure_time(function: Callable[..., R]) -> Callable[..., R]:
    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> R:
        started_at = perf_counter()
        try:
            return function(*args, **kwargs)
        finally:
            elapsed = perf_counter() - started_at
            print(f"{function.__qualname__}: {elapsed:.6f}s")

    return wrapper
```

**关键点：** 使用 `functools.wraps` 保留元信息；使用 `finally` 确保异常路径也能记录；生产代码不应直接 `print`，应注入 logger，并考虑采样、敏感参数脱敏和异步函数兼容。

### Q27：实现可配置重试装饰器

**题目：** 对指定异常进行有限次重试，等待时间指数增长并加入抖动。

```python
from functools import wraps
import random
import time
from collections.abc import Callable
from typing import Any, TypeVar


R = TypeVar("R")


def retry(
    attempts: int,
    delay: float = 0.1,
    retryable: tuple[type[Exception], ...] = (Exception,),
) -> Callable[[Callable[..., R]], Callable[..., R]]:
    if attempts <= 0:
        raise ValueError("attempts must be positive")

    def decorate(function: Callable[..., R]) -> Callable[..., R]:
        @wraps(function)
        def wrapper(*args: Any, **kwargs: Any) -> R:
            for attempt in range(attempts):
                try:
                    return function(*args, **kwargs)
                except retryable:
                    if attempt == attempts - 1:
                        raise
                    backoff = delay * (2**attempt)
                    time.sleep(backoff + random.uniform(0, backoff / 2))
            raise AssertionError("unreachable")

        return wrapper

    return decorate
```

**关键点：** 只重试明确可恢复的异常；写操作必须具备幂等语义；生产实现要支持最大延迟、取消、日志、指标和可注入 sleep 函数，便于测试。

### Q28：实现缓存装饰器并处理可哈希参数

**题目：** 缓存函数结果，参数不可哈希时给出明确错误，不要把可变对象偷偷转成不可靠字符串。

```python
from functools import wraps
from collections.abc import Callable
from typing import Any, TypeVar


R = TypeVar("R")


def memoize(function: Callable[..., R]) -> Callable[..., R]:
    cache: dict[tuple[Any, ...], R] = {}

    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> R:
        try:
            key = args, tuple(sorted(kwargs.items()))
            if key not in cache:
                cache[key] = function(*args, **kwargs)
            return cache[key]
        except TypeError as error:
            raise TypeError("all cache key arguments must be hashable") from error

    return wrapper
```

**关键点：** 参数顺序规范化、返回值可变、缓存无界、递归重入、线程安全和 TTL 都是生产问题；如果不想重复实现，应优先使用 `functools.lru_cache`，并明确它要求参数可哈希。

### Q29：实现上下文管理器测量代码块耗时

**题目：** 实现 `with measure("name"):`，无论代码块成功还是异常，都记录耗时，并原样传播异常。

```python
from contextlib import contextmanager
from time import perf_counter
from collections.abc import Iterator


@contextmanager
def measure(name: str) -> Iterator[None]:
    started_at = perf_counter()
    try:
        yield
    finally:
        elapsed = perf_counter() - started_at
        print(f"{name}: {elapsed:.6f}s")
```

**关键点：** `finally` 适合资源释放和记录；不在上下文管理器中吞掉异常，除非 API 明确要求；对于异步代码需要实现 `@asynccontextmanager` 并使用 `async with`。

### Q30：实现可复用的上下文管理器切换目录

**题目：** 临时切换当前工作目录，退出后恢复原目录，即使代码块抛出异常也要恢复。

```python
import os
from contextlib import contextmanager
from collections.abc import Iterator


@contextmanager
def change_directory(path: str) -> Iterator[None]:
    previous_path = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(previous_path)
```

**关键点：** 当前工作目录是进程级全局状态，多线程程序中不应并发使用这种上下文管理器。更好的服务端设计是向 API 传递绝对路径，不依赖进程 cwd。

### Q31：实现惰性批处理器

**题目：** 将任意可迭代对象按指定大小分批输出，不能先把全部数据转成列表。

```python
from collections.abc import Iterable, Iterator
from typing import TypeVar


T = TypeVar("T")


def batched(values: Iterable[T], size: int) -> Iterator[list[T]]:
    if size <= 0:
        raise ValueError("size must be positive")

    batch: list[T] = []
    for value in values:
        batch.append(value)
        if len(batch) == size:
            yield batch
            batch = []
    if batch:
        yield batch
```

**复杂度：** 时间 O(n)，额外空间 O(size)。

**边界与追问：** 生成器返回的 batch 不应在下一轮复用同一个可变列表；Python 3.12 的 `itertools.batched` 可直接使用，但面试中仍要能解释实现。

### Q32：实现带取消的异步上下文管理器

**题目：** 模拟异步资源连接，进入时建立连接，退出时关闭；退出时不吞掉异常。

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator


class AsyncConnection:
    async def open(self) -> None:
        pass

    async def close(self) -> None:
        pass


@asynccontextmanager
async def connection_scope() -> AsyncIterator[AsyncConnection]:
    connection = AsyncConnection()
    await connection.open()
    try:
        yield connection
    finally:
        await connection.close()
```

**关键点：** 真实连接关闭也可能超时，需要定义取消和强制释放策略；如果 `open` 失败，不能调用未成功初始化资源的 close；资源所有权应明确由上下文管理器还是调用方负责。

---

## 五、并发与异步 Coding 题

### Q33：实现线程安全计数器

**题目：** 支持多个线程并发递增和读取计数，要求结果不丢失。

```python
import threading


class ThreadSafeCounter:
    def __init__(self) -> None:
        self._value = 0
        self._lock = threading.Lock()

    def increment(self, amount: int = 1) -> int:
        with self._lock:
            self._value += amount
            return self._value

    def value(self) -> int:
        with self._lock:
            return self._value
```

**关键点：** 即使某些 Python 实现存在 GIL，也不能把复合操作当作跨实现、跨版本的业务原子性保证；锁保护的是读取、修改、写回的完整临界区。

**追问：** CPU 密集型任务通常考虑 `multiprocessing` 或进程池；I/O 密集型任务可考虑线程池或 asyncio；共享计数还要讨论锁竞争和分片计数。

### Q34：实现有界线程安全队列

**题目：** 不使用 `queue.Queue`，实现 `put` 和 `get`。队列满时生产者等待，队列空时消费者等待。

```python
from collections import deque
import threading
from typing import TypeVar


T = TypeVar("T")


class BoundedQueue:
    def __init__(self, capacity: int) -> None:
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self._capacity = capacity
        self._values: deque[T] = deque()
        self._condition = threading.Condition()

    def put(self, value: T) -> None:
        with self._condition:
            while len(self._values) >= self._capacity:
                self._condition.wait()
            self._values.append(value)
            self._condition.notify()

    def get(self) -> T:
        with self._condition:
            while not self._values:
                self._condition.wait()
            value = self._values.popleft()
            self._condition.notify()
            return value
```

**关键点：** 必须使用 `while` 检查条件，因为存在虚假唤醒和多个线程竞争；生产环境优先使用标准库 `queue.Queue`，它还提供超时、task tracking 和更成熟的关闭语义。

### Q35：线程池并行处理并保留输入顺序

**题目：** 使用线程池并行处理一组 I/O 任务，返回结果顺序与输入一致，单个异常要能被调用方发现。

```python
from concurrent.futures import ThreadPoolExecutor
from collections.abc import Callable
from typing import TypeVar


T = TypeVar("T")
R = TypeVar("R")


def map_in_order(
    values: list[T],
    function: Callable[[T], R],
    max_workers: int,
) -> list[R]:
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        return list(executor.map(function, values))
```

**关键点：** `executor.map` 保留输入顺序，但消费结果时可能等待前面的慢任务；如果需要先完成先处理，使用 `as_completed`，再自行记录输入索引。线程池大小要受下游连接池、CPU、文件描述符和限流约束。

### Q36：异步并发调用并设置超时

**题目：** 并发执行多个异步任务，整体超时后取消未完成任务，并返回成功结果。

```python
import asyncio
from collections.abc import Awaitable, Callable
from typing import TypeVar


T = TypeVar("T")


async def gather_with_timeout(
    factories: list[Callable[[], Awaitable[T]]],
    timeout: float,
) -> list[T]:
    tasks = [asyncio.create_task(factory()) for factory in factories]
    try:
        completed, pending = await asyncio.wait(
            tasks,
            timeout=timeout,
            return_when=asyncio.ALL_COMPLETED,
        )
        for task in pending:
            task.cancel()
        if pending:
            await asyncio.gather(*pending, return_exceptions=True)
        results: list[T] = []
        for task in tasks:
            if task in completed and not task.cancelled():
                error = task.exception()
                if error is None:
                    results.append(task.result())
        return results
    except asyncio.CancelledError:
        for task in tasks:
            task.cancel()
        await asyncio.gather(*tasks, return_exceptions=True)
        raise
```

**关键点：** `asyncio.wait` 返回的集合没有顺序；若要求输入顺序，应按任务列表读取已完成任务的结果，并定义未完成任务是否返回部分结果。取消只是向协程传播 `CancelledError`，底层 HTTP 客户端也要正确关闭连接。

### Q37：使用 Semaphore 限制异步并发量

**题目：** 并发处理大量 URL，但同时最多允许 `limit` 个请求。

```python
import asyncio
from collections.abc import Awaitable, Callable
from typing import TypeVar


T = TypeVar("T")


async def limited_gather(
    values: list[str],
    fetch: Callable[[str], Awaitable[T]],
    limit: int,
) -> list[T]:
    if limit <= 0:
        raise ValueError("limit must be positive")

    semaphore = asyncio.Semaphore(limit)

    async def run(value: str) -> T:
        async with semaphore:
            return await fetch(value)

    return await asyncio.gather(*(run(value) for value in values))
```

**关键点：** 信号量限制的是客户端同时进入临界区的协程数，不代表服务端一定只收到这么多请求；还要设置连接超时、读取超时、重试上限和整体预算。大量输入时可以进一步限制已创建 task 的数量。

### Q38：实现异步重试装饰器

**题目：** 为异步函数增加有限次重试、指数退避和抖动。

```python
import asyncio
from functools import wraps
import random
from collections.abc import Awaitable, Callable
from typing import Any, TypeVar


R = TypeVar("R")


def async_retry(
    attempts: int,
    delay: float = 0.1,
    retryable: tuple[type[Exception], ...] = (Exception,),
) -> Callable[[Callable[..., Awaitable[R]]], Callable[..., Awaitable[R]]]:
    if attempts <= 0:
        raise ValueError("attempts must be positive")

    def decorate(function: Callable[..., Awaitable[R]]) -> Callable[..., Awaitable[R]]:
        @wraps(function)
        async def wrapper(*args: Any, **kwargs: Any) -> R:
            for attempt in range(attempts):
                try:
                    return await function(*args, **kwargs)
                except asyncio.CancelledError:
                    raise
                except retryable:
                    if attempt == attempts - 1:
                        raise
                    backoff = delay * (2**attempt)
                    await asyncio.sleep(backoff + random.uniform(0, backoff / 2))
            raise AssertionError("unreachable")

        return wrapper

    return decorate
```

**关键点：** 不能捕获并重试 `CancelledError`；只重试明确可恢复的异常；重试次数、最大延迟和总耗时要有预算；非幂等写操作必须使用幂等键。

### Q39：实现异步生产者消费者模型

**题目：** 使用 asyncio 实现有界队列，多个生产者写入、多个消费者处理，收到停止信号后优雅退出。

```python
import asyncio
from collections.abc import Awaitable, Callable
from typing import TypeVar


T = TypeVar("T")


async def consume_items(
    items: list[T],
    process: Callable[[T], Awaitable[None]],
    worker_count: int,
    queue_size: int,
) -> None:
    queue: asyncio.Queue[T | None] = asyncio.Queue(maxsize=queue_size)

    async def producer() -> None:
        for item in items:
            await queue.put(item)
        for _ in range(worker_count):
            await queue.put(None)

    async def worker() -> None:
        while True:
            item = await queue.get()
            try:
                if item is None:
                    return
                await process(item)
            finally:
                queue.task_done()

    workers = [asyncio.create_task(worker()) for _ in range(worker_count)]
    await producer()
    await queue.join()
    await asyncio.gather(*workers)
```

**关键点：** 哨兵数量要与消费者数量匹配；`task_done` 必须放在 finally；异常策略需要明确，是停止全部消费者、记录失败后继续，还是进入死信队列；真实生产者通常也是异步流而不是已有列表。

### Q40：解释 GIL，并为 CPU 密集任务选择并发模型

**题目：** 给定 CPU 密集型函数，设计并行执行方案。

```python
from concurrent.futures import ProcessPoolExecutor
from collections.abc import Callable
from typing import TypeVar


T = TypeVar("T")
R = TypeVar("R")


def process_in_parallel(
    values: list[T],
    function: Callable[[T], R],
    workers: int | None = None,
) -> list[R]:
    with ProcessPoolExecutor(max_workers=workers) as executor:
        return list(executor.map(function, values))
```

**核心回答：** CPython 的 GIL 会限制同一进程中多个线程同时执行 Python 字节码，因此 CPU 密集型纯 Python 任务通常考虑进程、释放 GIL 的 native 库或改进算法；I/O 密集型任务可使用线程或 asyncio。进程池要求函数和参数可序列化，进程创建、数据复制和跨进程通信都有成本。

**追问：** Windows 和 macOS 的进程启动方式、模块顶层副作用、`if __name__ == "__main__"`、任务粒度和共享内存都要考虑。

---

## 六、缓存、限流与可靠性 Coding 题

### Q41：实现 TTL 缓存

**题目：** 实现单机内存缓存，支持写入 TTL、读取过期判断、删除和容量限制。

```python
from collections import OrderedDict
from dataclasses import dataclass
from time import monotonic
from typing import Generic, TypeVar


K = TypeVar("K")
V = TypeVar("V")


@dataclass
class CacheEntry(Generic[V]):
    value: V
    expires_at: float


class TtlCache(Generic[K, V]):
    def __init__(self, capacity: int) -> None:
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self._capacity = capacity
        self._entries: OrderedDict[K, CacheEntry[V]] = OrderedDict()

    def get(self, key: K) -> V | None:
        entry = self._entries.get(key)
        if entry is None:
            return None
        if entry.expires_at <= monotonic():
            del self._entries[key]
            return None
        self._entries.move_to_end(key)
        return entry.value

    def put(self, key: K, value: V, ttl: float) -> None:
        if ttl <= 0:
            raise ValueError("ttl must be positive")
        self._entries[key] = CacheEntry(value, monotonic() + ttl)
        self._entries.move_to_end(key)
        while len(self._entries) > self._capacity:
            self._entries.popitem(last=False)

    def remove(self, key: K) -> None:
        self._entries.pop(key, None)
```

**关键点：** 此版本不是线程安全的；应使用锁或成熟库 Caffeine/Python 对应实现；TTL 使用单调时钟而不是 wall clock，避免系统时间回拨；生产环境还要考虑过期清理、缓存击穿、空值、统计和多节点一致性。

### Q42：防止缓存击穿的 Single Flight

**题目：** 多个协程同时加载同一个 key 时，只允许一个协程执行 loader，其他协程等待同一个结果。

```python
import asyncio
from collections.abc import Awaitable, Callable
from typing import Generic, TypeVar


K = TypeVar("K")
V = TypeVar("V")


class SingleFlight(Generic[K, V]):
    def __init__(self) -> None:
        self._inflight: dict[K, asyncio.Future[V]] = {}
        self._lock = asyncio.Lock()

    async def do(self, key: K, loader: Callable[[], Awaitable[V]]) -> V:
        async with self._lock:
            future = self._inflight.get(key)
            if future is None:
                future = asyncio.get_running_loop().create_future()
                self._inflight[key] = future
                owner = True
            else:
                owner = False

        if not owner:
            return await future

        try:
            result = await loader()
            future.set_result(result)
            return result
        except BaseException as error:
            future.set_exception(error)
            raise
        finally:
            async with self._lock:
                if self._inflight.get(key) is future:
                    del self._inflight[key]
```

**关键点：** `finally` 中用对象身份删除，避免旧任务误删新一轮加载；失败不能永久缓存异常 Future；生产实现要处理 owner 被取消、等待者取消是否影响 owner、超时、空值和异常是否缓存。

### Q43：实现固定窗口限流器

**题目：** 每个 key 在时间窗口内最多允许指定次数，要求单进程内线程安全。

```python
from dataclasses import dataclass
import threading
from time import monotonic


@dataclass
class Window:
    started_at: float
    count: int = 0


class FixedWindowLimiter:
    def __init__(self, window_seconds: float, limit: int) -> None:
        if window_seconds <= 0 or limit <= 0:
            raise ValueError("invalid limiter configuration")
        self._window_seconds = window_seconds
        self._limit = limit
        self._windows: dict[str, Window] = {}
        self._lock = threading.Lock()

    def allow(self, key: str) -> bool:
        now = monotonic()
        with self._lock:
            window = self._windows.get(key)
            if window is None or now - window.started_at >= self._window_seconds:
                window = Window(started_at=now)
                self._windows[key] = window
            if window.count >= self._limit:
                return False
            window.count += 1
            return True
```

**关键点：** 固定窗口在边界处可能产生突发流量；生产实现要清理长期不活跃 key，分布式场景要讨论 Redis 原子操作、key 过期、节点时钟和失败策略；限流拒绝应有可观察指标。

### Q44：实现令牌桶限流器

**题目：** 令牌按固定速率补充，桶有最大容量，每次请求消耗一个令牌。

```python
import threading
from time import monotonic


class TokenBucket:
    def __init__(self, capacity: float, refill_per_second: float) -> None:
        if capacity <= 0 or refill_per_second <= 0:
            raise ValueError("invalid bucket configuration")
        self._capacity = capacity
        self._refill_per_second = refill_per_second
        self._tokens = capacity
        self._last_refill = monotonic()
        self._lock = threading.Lock()

    def try_acquire(self, tokens: float = 1.0) -> bool:
        if tokens <= 0 or tokens > self._capacity:
            raise ValueError("invalid token amount")
        with self._lock:
            now = monotonic()
            elapsed = now - self._last_refill
            self._tokens = min(
                self._capacity,
                self._tokens + elapsed * self._refill_per_second,
            )
            self._last_refill = now
            if self._tokens < tokens:
                return False
            self._tokens -= tokens
            return True
```

**追问：** 令牌桶控制平均速率和突发容量，漏桶更强调平滑输出；分布式限流需要共享原子状态；锁内计算和外部调用必须分离。

### Q45：实现有限重试执行器

**题目：** 对可重试异常执行有限次重试，支持指数退避、最大延迟和抖动。

```python
import random
import time
from collections.abc import Callable
from typing import TypeVar


T = TypeVar("T")


def execute_with_retry(
    operation: Callable[[], T],
    attempts: int,
    initial_delay: float,
    maximum_delay: float,
    retryable: tuple[type[Exception], ...],
) -> T:
    if attempts <= 0 or initial_delay < 0 or maximum_delay < 0:
        raise ValueError("invalid retry configuration")

    for attempt in range(attempts):
        try:
            return operation()
        except retryable:
            if attempt == attempts - 1:
                raise
            delay = min(maximum_delay, initial_delay * (2**attempt))
            time.sleep(delay + random.uniform(0, delay / 2))

    raise AssertionError("unreachable")
```

**关键点：** 只重试明确可恢复的异常；总超时应包含执行时间和等待时间；`delay == 0` 时随机抖动也应为 0；对于 POST 等写操作，重试前必须确认幂等语义。

### Q46：实现简单熔断器

**题目：** 连续失败达到阈值后进入 OPEN，冷却后允许一次 HALF_OPEN 探测，成功后恢复 CLOSED。

```python
from enum import Enum, auto
import threading
from time import monotonic
from collections.abc import Callable
from typing import TypeVar


T = TypeVar("T")


class CircuitState(Enum):
    CLOSED = auto()
    OPEN = auto()
    HALF_OPEN = auto()


class CircuitBreaker:
    def __init__(self, failure_threshold: int, recovery_seconds: float) -> None:
        if failure_threshold <= 0 or recovery_seconds < 0:
            raise ValueError("invalid circuit configuration")
        self._threshold = failure_threshold
        self._recovery_seconds = recovery_seconds
        self._state = CircuitState.CLOSED
        self._failures = 0
        self._opened_at = 0.0
        self._lock = threading.Lock()

    def call(self, operation: Callable[[], T]) -> T:
        with self._lock:
            now = monotonic()
            if self._state is CircuitState.OPEN:
                if now - self._opened_at < self._recovery_seconds:
                    raise RuntimeError("circuit is open")
                self._state = CircuitState.HALF_OPEN

        try:
            result = operation()
        except Exception:
            with self._lock:
                self._failures += 1
                if self._failures >= self._threshold:
                    self._state = CircuitState.OPEN
                    self._opened_at = monotonic()
            raise
        else:
            with self._lock:
                self._failures = 0
                self._state = CircuitState.CLOSED
            return result
```

**关键点：** 这是面试用最小模型；真实熔断器要使用时间窗口或滑动桶统计、限制 HALF_OPEN 并发探测、区分异常类型、提供降级和指标，并避免整个慢操作占用状态锁。

### Q47：实现带幂等键的命令执行器

**题目：** 相同幂等键重复提交时返回第一次执行结果，不重复执行命令。

```python
import threading
from collections.abc import Callable
from typing import Any, TypeVar


T = TypeVar("T")


class IdempotencyStore:
    def __init__(self) -> None:
        self._results: dict[str, T] = {}
        self._lock = threading.Lock()

    def execute(self, key: str, command: Callable[[], T]) -> T:
        with self._lock:
            if key in self._results:
                return self._results[key]
            result = command()
            self._results[key] = result
            return result
```

**关键点：** 这段实现会在锁内执行命令，只适合演示原子性，不适合生产，因为慢命令会阻塞所有 key。真实系统要记录 `PROCESSING/SUCCEEDED/FAILED` 状态，持久化结果，校验同一 key 的请求参数，设置 TTL，并用数据库唯一约束或 Redis 原子操作防止多实例重复执行。

### Q48：实现单机滑动窗口失败率统计

**题目：** 统计最近一段时间内请求总数、失败数和失败率，为熔断或告警提供基础数据。

```python
from collections import deque
import threading
from time import monotonic


class SlidingFailureWindow:
    def __init__(self, window_seconds: float) -> None:
        if window_seconds <= 0:
            raise ValueError("window_seconds must be positive")
        self._window_seconds = window_seconds
        self._events: deque[tuple[float, bool]] = deque()
        self._failures = 0
        self._lock = threading.Lock()

    def record(self, failed: bool) -> None:
        with self._lock:
            now = monotonic()
            self._remove_expired(now)
            self._events.append((now, failed))
            if failed:
                self._failures += 1

    def failure_rate(self) -> float:
        with self._lock:
            self._remove_expired(monotonic())
            if not self._events:
                return 0.0
            return self._failures / len(self._events)

    def _remove_expired(self, now: float) -> None:
        deadline = now - self._window_seconds
        while self._events and self._events[0][0] <= deadline:
            _, failed = self._events.popleft()
            if failed:
                self._failures -= 1
```

**关键点：** 事件量很大时逐条保存会消耗内存并产生锁竞争，生产系统通常使用固定时间桶、采样或指标系统；还要定义最小请求数、乱序事件、统计快照和分布式聚合。

---

## 七、综合工程 Coding 题

### Q49：实现优雅关闭的异步任务调度器

**题目：** 提交异步任务，限制并发数，关闭时不再接受新任务并等待已有任务结束。

```python
import asyncio
from collections.abc import Awaitable, Callable
from typing import Any


class AsyncTaskRunner:
    def __init__(self, concurrency: int) -> None:
        if concurrency <= 0:
            raise ValueError("concurrency must be positive")
        self._semaphore = asyncio.Semaphore(concurrency)
        self._tasks: set[asyncio.Task[Any]] = set()
        self._closed = False

    def submit(self, operation: Callable[[], Awaitable[Any]]) -> asyncio.Task[Any]:
        if self._closed:
            raise RuntimeError("runner is closed")

        async def run() -> Any:
            async with self._semaphore:
                return await operation()

        task = asyncio.create_task(run())
        self._tasks.add(task)
        task.add_done_callback(self._tasks.discard)
        return task

    async def close(self) -> None:
        self._closed = True
        if self._tasks:
            await asyncio.gather(*tuple(self._tasks), return_exceptions=True)
```

**关键点：** `submit` 与 `close` 之间存在并发竞态，生产实现应在同一事件循环中约束调用顺序，或增加异步锁；关闭策略要明确是等待、取消还是超时后强制取消；异常不能被 `return_exceptions=True` 后无记录地吞掉。

### Q50：实现稳定分页游标

**题目：** 给定按 `(created_at, id)` 升序排列的记录，使用游标获取下一页，避免 offset 分页在插入数据时重复或遗漏。

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass(frozen=True)
class Record:
    created_at: datetime
    record_id: int


def next_page(
    records: list[Record],
    cursor: tuple[datetime, int] | None,
    page_size: int,
) -> tuple[list[Record], tuple[datetime, int] | None]:
    if page_size <= 0:
        raise ValueError("page_size must be positive")

    start = 0
    if cursor is not None:
        for index, record in enumerate(records):
            if (record.created_at, record.record_id) > cursor:
                start = index
                break
        else:
            return [], None

    page = records[start:start + page_size]
    if not page:
        return [], None
    next_cursor = (page[-1].created_at, page[-1].record_id)
    return page, next_cursor
```

**关键点：** 此实现假设输入已经按唯一稳定排序键排序，并且使用列表线性扫描；数据库实现应将游标条件下推为 `(created_at, id) > (...)`，配合索引。排序键必须唯一或由唯一 ID 补充，否则会遗漏相同时间的记录。

### Q51：实现异步批量去重请求

**题目：** 同一批请求中相同 key 只调用一次异步 loader，返回结果按原输入顺序排列。

```python
import asyncio
from collections.abc import Awaitable, Callable
from typing import TypeVar


K = TypeVar("K")
V = TypeVar("V")


async def batch_load_unique(
    keys: list[K],
    loader: Callable[[K], Awaitable[V]],
    concurrency: int,
) -> list[V]:
    if concurrency <= 0:
        raise ValueError("concurrency must be positive")

    unique_keys = list(dict.fromkeys(keys))
    semaphore = asyncio.Semaphore(concurrency)

    async def load(key: K) -> V:
        async with semaphore:
            return await loader(key)

    unique_values = await asyncio.gather(*(load(key) for key in unique_keys))
    values_by_key = dict(zip(unique_keys, unique_values))
    return [values_by_key[key] for key in keys]
```

**复杂度：** 去重和重建结果 O(n)，最多对 u 个不同 key 发起请求。

**边界与追问：** loader 失败时是整批失败还是返回逐项错误；key 是否可哈希；跨请求去重需要 Single Flight 或缓存；结果映射不能假设 loader 返回值本身可哈希。

### Q52：实现配置对象的不可变快照

**题目：** 配置加载后不可变，读取方只能获得一致快照；动态刷新时整体替换配置。

```python
from dataclasses import dataclass
from types import MappingProxyType
from typing import Mapping


@dataclass(frozen=True)
class AppConfig:
    environment: str
    values: Mapping[str, str]

    @classmethod
    def create(cls, environment: str, values: Mapping[str, str]) -> "AppConfig":
        return cls(environment, MappingProxyType(dict(values)))


class ConfigHolder:
    def __init__(self, config: AppConfig) -> None:
        self._config = config

    def get(self) -> AppConfig:
        return self._config

    def replace(self, config: AppConfig) -> None:
        self._config = config
```

**关键点：** `frozen=True` 只限制 dataclass 字段赋值，嵌套可变对象仍可能被修改；`MappingProxyType` 提供只读视图，但构造时必须复制源字典。CPython 中简单引用替换通常是原子的，但跨实现和复合刷新流程仍应使用锁或明确发布协议。

### Q53：实现安全的字符串模板替换

**题目：** 将模板中的 `${name}` 替换为参数值，缺少参数时保留占位符，不能执行任意代码。

```python
import re
from collections.abc import Mapping


PLACEHOLDER = re.compile(r"\$\{([A-Za-z][A-Za-z0-9_]*)\}")


def render_template(template: str, values: Mapping[str, str]) -> str:
    def replace(match: re.Match[str]) -> str:
        key = match.group(1)
        return values.get(key, match.group(0))

    return PLACEHOLDER.sub(replace, template)
```

**关键点：** 使用白名单格式进行数据替换，不要把用户输入交给 `eval`、`exec`、Jinja 不受控环境或任意表达式解释器；如果模板用于 HTML、SQL 或 shell，还要使用对应上下文的安全编码，而不是普通字符串替换。

### Q54：实现批量任务的失败隔离与重试

**题目：** 将任务按批次处理，每个任务独立重试；一个任务最终失败不影响其他任务，并返回成功和失败结果。

```python
from collections.abc import Callable
from dataclasses import dataclass, field
from typing import TypeVar


T = TypeVar("T")


@dataclass
class BatchResult:
    succeeded: list[T] = field(default_factory=list)
    failed: list[str] = field(default_factory=list)


def process_batch(
    task_ids: list[str],
    process: Callable[[str], T],
    max_retries: int,
) -> BatchResult:
    if max_retries < 0:
        raise ValueError("max_retries must not be negative")

    result = BatchResult()
    for task_id in task_ids:
        for attempt in range(max_retries + 1):
            try:
                result.succeeded.append(process(task_id))
                break
            except Exception:
                if attempt == max_retries:
                    result.failed.append(task_id)
    return result
```

**关键点：** 这里捕获异常是为了演示失败隔离，生产实现必须记录异常类型和 traceback，并使用可重试异常白名单；写操作必须有幂等键；还要考虑超时、退避、死信、取消和批次级审计。

### Q55：实现简单的库存预占模型

**题目：** 实现单进程内存版本的库存扣减，不能让库存变成负数，并说明如何扩展到分布式服务。

```python
import threading


class Inventory:
    def __init__(self) -> None:
        self._stock: dict[str, int] = {}
        self._lock = threading.Lock()

    def initialize(self, sku: str, quantity: int) -> None:
        if quantity < 0:
            raise ValueError("quantity must not be negative")
        with self._lock:
            self._stock[sku] = quantity

    def reserve(self, sku: str, quantity: int) -> bool:
        if quantity <= 0:
            raise ValueError("quantity must be positive")
        with self._lock:
            available = self._stock.get(sku, 0)
            if available < quantity:
                return False
            self._stock[sku] = available - quantity
            return True

    def release(self, sku: str, quantity: int) -> None:
        if quantity <= 0:
            raise ValueError("quantity must be positive")
        with self._lock:
            self._stock[sku] = self._stock.get(sku, 0) + quantity
```

**关键点：** 单进程锁只保护一个 Python 进程。分布式环境要使用数据库条件更新、乐观锁、Redis 原子脚本或独立库存服务，并设计 reservation ID、超时释放、幂等、消息最终一致性、对账和超卖监控。

---

## 八、现场追问与测试清单

### 1. Python 基础题常见追问

- 可变默认参数为什么危险？如何使用 `None` 哨兵或不可变默认值？
- `is` 与 `==` 的区别是什么？为什么判断 `None` 要用 `is None`？
- 生成器何时开始执行？异常何时抛出？如何关闭生成器？
- 浅拷贝和深拷贝的成本与共享语义是什么？
- 迭代器被消费后还能否重复使用？函数是否应该接收 Iterable 还是 Sequence？
- 类型注解是否会在运行时强制检查？如何配合 mypy 或 Pyright？

### 2. 并发题常见追问

- GIL 对 I/O 密集和 CPU 密集任务分别有什么影响？
- 什么时候使用线程、进程、asyncio 或第三方分布式任务系统？
- 共享变量的可见性、原子性和锁范围如何保证？
- 条件变量为什么要使用 `while` 而不是 `if`？
- 任务超时后如何取消？底层 I/O 是否真正支持取消？
- 关闭时如何保证不丢任务、不永久等待和不泄漏线程或连接？

### 3. 缓存与可靠性题常见追问

- 缓存 key、TTL、最大容量和失效策略如何设计？
- 缓存击穿、穿透、雪崩和热点 key 如何处理？
- 重试是否会放大流量？哪些异常可重试？是否需要幂等键？
- 熔断打开后用户看到什么？半开探测如何限制并发？
- 限流是单机还是分布式？固定窗口的边界突发如何处理？
- 监控哪些指标：命中率、P99、拒绝数、重试数、熔断状态和队列长度？

### 4. 最低测试集合

每道现场题至少口头验证以下用例：

1. 正常输入和最小合法输入。
2. 空输入、`None` 或非法参数。
3. 重复值、边界值、极端长度和异常类型。
4. 任务失败、超时、取消和部分成功。
5. 多线程或多协程同时调用、关闭过程和重复提交。
6. 大数据量下的时间、内存、队列和缓存增长。

### 5. 推荐的 pytest 测试结构

```python
def test_two_sum_returns_indexes_for_existing_pair() -> None:
    assert two_sum([2, 7, 11, 15], 9) == (0, 1)
```

测试命名应描述行为，不要只写 `test_1`。异步测试应使用项目配置的异步测试插件；并发测试不能因为“跑过一次”就证明线程安全，应多轮执行、控制并发、验证不变量，并使用超时防止测试永久挂起。

---

## 九、面试前自检

### Python 与算法

- 我能在不依赖 IDE 自动补全的情况下写出链表、栈、队列、堆、二叉树和图的基本操作。
- 我能解释每道题的循环不变量、时间复杂度和空间复杂度。
- 我能说明 `dict`、`set`、`deque`、堆、生成器和迭代器的使用边界。
- 我能识别可变默认参数、迭代器消费、递归深度、异常吞噬和资源未释放问题。

### 并发与可靠性

- 我能解释 GIL、线程、进程、asyncio、线程池和进程池的适用边界。
- 我能实现带关闭、超时、取消和中断处理的并发组件。
- 我能说明缓存、限流、重试、熔断和幂等之间的关系。
- 我能把单机内存方案扩展到多实例，并指出数据库、Redis、消息队列和网络带来的新问题。

### 现场表达

- 写代码前我会先确认约束，而不是直接假设输入。
- 我会先给可验证的基础实现，再讨论优化和替代方案。
- 我能主动说出失败路径、监控指标、测试策略和资源释放方式。
- 项目案例使用真实数据，明确个人职责，不把团队成果全部归于自己。

> 现场 Coding 的目标不是写出最短代码，而是在有限时间内证明：你能正确建模、写出可读实现、识别边界，并能把 Python 代码放进可运行、可观测、可维护的系统中。