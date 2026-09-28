# Java 现场 Coding 高频题与参考答案

> 面向 Java 中高级开发工程师、后端架构师和技术负责人面试。题目覆盖数据结构与算法、Java 集合、并发编程、缓存、限流、重试、熔断、线程池和常见系统设计编码。
>
> 文中的代码片段默认彼此独立，主要使用 JDK 8+ 标准 API。现场作答时不要直接背代码，建议先澄清输入约束、给出朴素方案，再说明优化、复杂度、边界条件和测试策略。

## 目录

1. [现场 Coding 的答题方法](#一现场-coding-的答题方法)
2. [数据结构与算法高频题](#二数据结构与算法高频题)
3. [Java 集合与实用编码题](#三java-集合与实用编码题)
4. [并发编程 Coding 题](#四并发编程-coding-题)
5. [缓存、限流与可靠性 Coding 题](#五缓存限流与可靠性-coding-题)
6. [综合工程 Coding 题](#六综合工程-coding-题)
7. [现场追问与测试清单](#七现场追问与测试清单)
8. [面试前自检](#八面试前自检)

---

## 一、现场 Coding 的答题方法

### 1. 先确认题目边界

写代码前先问清楚：

- 输入是否可能为 `null`、空集合、重复值、负数或超大数据？
- 结果是否要求保持顺序？是否允许修改输入？
- 数据量和时间限制是什么？是否允许额外内存？
- 是否单线程？是否要求线程安全、取消、超时或幂等？
- 失败时返回空结果、异常、错误对象，还是部分结果？

### 2. 推荐答题顺序

1. 用一两个例子复述题意，确认输入和输出。
2. 先写最直接、容易验证的方案。
3. 解释数据结构、循环不变量或并发约束。
4. 写代码，并主动说出边界条件。
5. 给出时间和空间复杂度。
6. 用正常、边界、异常和并发场景验证。
7. 讨论生产环境的超时、日志、监控、限流和回滚。

### 3. 高分实现的共同特征

- 方法职责单一，命名表达业务含义，不用无意义的单字母变量。
- 对输入和返回值的约定清楚，不用 `null` 隐式表达过多状态。
- 业务状态转换有明确不变量，失败路径与成功路径同样完整。
- 并发代码说明锁的范围、可见性、顺序性、取消和资源释放。
- 不为了追求“无锁”而写出难以证明正确的复杂代码。

---

## 二、数据结构与算法高频题

### Q1：反转单链表

**题目：** 给定单链表头节点，原地反转链表并返回新的头节点。

**思路：** 使用 `previous`、`current` 和 `next` 保存反转过程。每次先保存后继节点，再把当前节点指向前驱。

```java
public final class ReverseLinkedList {
    static final class Node {
        int value;
        Node next;

        Node(int value) {
            this.value = value;
        }
    }

    static Node reverse(Node head) {
        Node previous = null;
        Node current = head;

        while (current != null) {
            Node next = current.next;
            current.next = previous;
            previous = current;
            current = next;
        }
        return previous;
    }
}
```

**复杂度：** 时间 O(n)，额外空间 O(1)。

**边界与追问：** 空链表和单节点直接返回；递归写法空间为 O(n)，大链表更适合迭代；如果需要保留原链表，应复制节点而不是原地修改。

### Q2：判断链表是否有环

**题目：** 判断单链表是否存在环，并说明如何找到环的入口。

**思路：** 快慢指针相遇说明有环。相遇后让一个指针回到头部，两个指针每次走一步，下一次相遇点就是环入口。

```java
public final class LinkedListCycle {
    static final class Node {
        int value;
        Node next;

        Node(int value) {
            this.value = value;
        }
    }

    static Node findEntry(Node head) {
        Node slow = head;
        Node fast = head;

        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) {
                Node entry = head;
                while (entry != slow) {
                    entry = entry.next;
                    slow = slow.next;
                }
                return entry;
            }
        }
        return null;
    }
}
```

**复杂度：** 时间 O(n)，额外空间 O(1)。

**边界与追问：** `head == null`、单节点无环和单节点自环都要测试；不能用节点值判断相等，必须比较节点引用。

### Q3：合并两个有序链表

**题目：** 将两个升序单链表合并为一个升序链表，要求尽量复用原节点。

```java
public final class MergeSortedLists {
    static final class Node {
        int value;
        Node next;

        Node(int value) {
            this.value = value;
        }
    }

    static Node merge(Node first, Node second) {
        Node dummy = new Node(0);
        Node tail = dummy;

        while (first != null && second != null) {
            if (first.value <= second.value) {
                tail.next = first;
                first = first.next;
            } else {
                tail.next = second;
                second = second.next;
            }
            tail = tail.next;
        }

        tail.next = first == null ? second : first;
        return dummy.next;
    }
}
```

**复杂度：** 时间 O(m + n)，额外空间 O(1)。

**边界与追问：** `<=` 可以在值相等时保持第一个链表的相对优先级；如果输入链表不保证有序，应该先澄清，不要默默承担排序成本。

### Q4：判断括号是否有效

**题目：** 输入只包含 `()[]{}`，判断括号是否正确闭合和嵌套。

```java
import java.util.ArrayDeque;
import java.util.Deque;

public final class ValidBrackets {
    static boolean isValid(String text) {
        if (text == null || text.length() % 2 != 0) {
            return false;
        }

        Deque<Character> stack = new ArrayDeque<>();
        for (char character : text.toCharArray()) {
            if (character == '(' || character == '[' || character == '{') {
                stack.push(character);
                continue;
            }

            if (stack.isEmpty() || !matches(stack.pop(), character)) {
                return false;
            }
        }
        return stack.isEmpty();
    }

    private static boolean matches(char opening, char closing) {
        return opening == '(' && closing == ')'
                || opening == '[' && closing == ']'
                || opening == '{' && closing == '}';
    }
}
```

**复杂度：** 时间 O(n)，额外空间 O(n)。

**边界与追问：** 空字符串通常视为有效，但要根据题意确认；`ArrayDeque` 不允许存放 `null`，此题不影响。

### Q5：两数之和

**题目：** 给定整数数组和目标值，返回和为目标值的两个元素下标。假设恰好存在一组答案。

```java
import java.util.HashMap;
import java.util.Map;

public final class TwoSum {
    static int[] find(int[] numbers, int target) {
        Map<Integer, Integer> indexByValue = new HashMap<>();

        for (int index = 0; index < numbers.length; index++) {
            int complement = target - numbers[index];
            Integer previousIndex = indexByValue.get(complement);
            if (previousIndex != null) {
                return new int[] { previousIndex, index };
            }
            indexByValue.put(numbers[index], index);
        }
        throw new IllegalArgumentException("No pair found");
    }
}
```

**复杂度：** 平均时间 O(n)，空间 O(n)。

**边界与追问：** 相同数值需要允许使用两个不同下标；如果数组已排序，可使用双指针将额外空间降到 O(1)。注意整数溢出，生产代码可使用 `long` 计算 complement。

### Q6：最长无重复字符子串

**题目：** 返回字符串中不包含重复字符的最长连续子串长度。

```java
import java.util.HashMap;
import java.util.Map;

public final class LongestUniqueSubstring {
    static int lengthOf(String text) {
        Map<Character, Integer> lastIndexByCharacter = new HashMap<>();
        int left = 0;
        int maximumLength = 0;

        for (int right = 0; right < text.length(); right++) {
            char character = text.charAt(right);
            Integer previousIndex = lastIndexByCharacter.put(character, right);
            if (previousIndex != null) {
                left = Math.max(left, previousIndex + 1);
            }
            maximumLength = Math.max(maximumLength, right - left + 1);
        }
        return maximumLength;
    }
}
```

**复杂度：** 时间 O(n)，空间 O(k)，k 为字符集大小或窗口内字符数。

**边界与追问：** Java `char` 表示 UTF-16 code unit，不一定等于一个 Unicode code point；如果题目要求按 Unicode code point 处理，应使用 `codePoints()` 并明确转换成本。

### Q7：最大子数组和

**题目：** 返回非空整数数组的最大连续子数组和。

```java
public final class MaximumSubarray {
    static int maxSum(int[] numbers) {
        if (numbers == null || numbers.length == 0) {
            throw new IllegalArgumentException("Numbers must not be empty");
        }

        int currentSum = numbers[0];
        int maximumSum = numbers[0];
        for (int index = 1; index < numbers.length; index++) {
            currentSum = Math.max(numbers[index], currentSum + numbers[index]);
            maximumSum = Math.max(maximumSum, currentSum);
        }
        return maximumSum;
    }
}
```

**复杂度：** 时间 O(n)，额外空间 O(1)。

**边界与追问：** 全为负数时返回最大的负数，不能把初始值写成 0；如果还要返回区间，需要额外记录当前起点和最佳左右边界。

### Q8：合并重叠区间

**题目：** 合并一组可能重叠的闭区间，例如 `[1,3]` 和 `[3,5]` 是否合并由题目约定决定。下面按闭区间处理。

```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Comparator;
import java.util.List;

public final class MergeIntervals {
    static int[][] merge(int[][] intervals) {
        if (intervals == null || intervals.length <= 1) {
            return intervals == null ? new int[0][] : intervals;
        }

        Arrays.sort(intervals, Comparator.comparingInt(interval -> interval[0]));
        List<int[]> merged = new ArrayList<>();
        int currentStart = intervals[0][0];
        int currentEnd = intervals[0][1];

        for (int index = 1; index < intervals.length; index++) {
            int nextStart = intervals[index][0];
            int nextEnd = intervals[index][1];
            if (nextStart <= currentEnd) {
                currentEnd = Math.max(currentEnd, nextEnd);
            } else {
                merged.add(new int[] { currentStart, currentEnd });
                currentStart = nextStart;
                currentEnd = nextEnd;
            }
        }
        merged.add(new int[] { currentStart, currentEnd });
        return merged.toArray(new int[0][]);
    }
}
```

**复杂度：** 时间 O(n log n)，空间 O(n)；排序是否修改输入要提前说明。

### Q9：滑动窗口最大值

**题目：** 给定数组和窗口大小，返回每个窗口中的最大值。

**思路：** 使用单调递减双端队列保存下标，队首始终是当前窗口最大值的下标。

```java
import java.util.ArrayDeque;
import java.util.Deque;

public final class SlidingWindowMaximum {
    static int[] maxValues(int[] numbers, int windowSize) {
        if (numbers == null || windowSize <= 0 || windowSize > numbers.length) {
            throw new IllegalArgumentException("Invalid window size");
        }

        int[] result = new int[numbers.length - windowSize + 1];
        Deque<Integer> decreasingIndexes = new ArrayDeque<>();

        for (int index = 0; index < numbers.length; index++) {
            while (!decreasingIndexes.isEmpty()
                    && decreasingIndexes.peekFirst() <= index - windowSize) {
                decreasingIndexes.removeFirst();
            }
            while (!decreasingIndexes.isEmpty()
                    && numbers[decreasingIndexes.peekLast()] <= numbers[index]) {
                decreasingIndexes.removeLast();
            }
            decreasingIndexes.addLast(index);

            if (index >= windowSize - 1) {
                result[index - windowSize + 1] = numbers[decreasingIndexes.peekFirst()];
            }
        }
        return result;
    }
}
```

**复杂度：** 时间 O(n)，空间 O(windowSize)。

**边界与追问：** 队列存下标而不是数值，才能判断元素是否过期；相等值保留较新的下标可以减少队列长度。

### Q10：前 K 个高频元素

**题目：** 返回数组中出现频率最高的 K 个元素，顺序不作要求。

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.PriorityQueue;

public final class TopKFrequent {
    static int[] find(int[] numbers, int k) {
        Map<Integer, Integer> frequencyByValue = new HashMap<>();
        for (int number : numbers) {
            frequencyByValue.merge(number, 1, Integer::sum);
        }

        PriorityQueue<Map.Entry<Integer, Integer>> minHeap = new PriorityQueue<>(
                Map.Entry.comparingByValue());
        for (Map.Entry<Integer, Integer> entry : frequencyByValue.entrySet()) {
            minHeap.offer(entry);
            if (minHeap.size() > k) {
                minHeap.poll();
            }
        }

        int[] result = new int[Math.min(k, minHeap.size())];
        for (int index = result.length - 1; index >= 0; index--) {
            result[index] = minHeap.poll().getKey();
        }
        return result;
    }
}
```

**复杂度：** 时间 O(n log k)，空间 O(n + k)。

**边界与追问：** 要先约定 `k <= 0`、`k` 大于不同元素数量和频率相等时的顺序；如果需要线性平均时间，可讨论桶排序或 Quickselect。

### Q11：二叉树层序遍历

**题目：** 返回二叉树每一层的节点值。

```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;

public final class BinaryTreeLevelOrder {
    static final class TreeNode {
        int value;
        TreeNode left;
        TreeNode right;

        TreeNode(int value) {
            this.value = value;
        }
    }

    static List<List<Integer>> traverse(TreeNode root) {
        List<List<Integer>> result = new ArrayList<>();
        if (root == null) {
            return result;
        }

        Deque<TreeNode> queue = new ArrayDeque<>();
        queue.offer(root);
        while (!queue.isEmpty()) {
            int levelSize = queue.size();
            List<Integer> level = new ArrayList<>(levelSize);
            for (int index = 0; index < levelSize; index++) {
                TreeNode node = queue.poll();
                level.add(node.value);
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
            }
            result.add(level);
        }
        return result;
    }
}
```

**复杂度：** 时间 O(n)，队列空间 O(w)，w 为树的最大宽度。

### Q12：验证二叉搜索树

**题目：** 判断一棵二叉树是否满足严格递增的二叉搜索树定义。

```java
public final class ValidateBinarySearchTree {
    static final class TreeNode {
        int value;
        TreeNode left;
        TreeNode right;
    }

    static boolean isValid(TreeNode root) {
        return isValid(root, null, null);
    }

    private static boolean isValid(TreeNode node, Long lowerBound, Long upperBound) {
        if (node == null) {
            return true;
        }
        if (lowerBound != null && node.value <= lowerBound
                || upperBound != null && node.value >= upperBound) {
            return false;
        }
        return isValid(node.left, lowerBound, (long) node.value)
                && isValid(node.right, (long) node.value, upperBound);
    }
}
```

**复杂度：** 时间 O(n)，递归空间 O(h)。

**边界与追问：** 使用 `long` 边界避免节点值为 `Integer.MIN_VALUE` 或 `MAX_VALUE` 时用哨兵值冲突；重复值是否允许要先确认。

### Q13：二叉树最近公共祖先

**题目：** 给定普通二叉树和两个节点，返回它们的最近公共祖先。假设两个节点都在树中。

```java
public final class LowestCommonAncestor {
    static final class TreeNode {
        int value;
        TreeNode left;
        TreeNode right;
    }

    static TreeNode find(TreeNode root, TreeNode first, TreeNode second) {
        if (root == null || root == first || root == second) {
            return root;
        }

        TreeNode leftResult = find(root.left, first, second);
        TreeNode rightResult = find(root.right, first, second);
        if (leftResult != null && rightResult != null) {
            return root;
        }
        return leftResult == null ? rightResult : leftResult;
    }
}
```

**复杂度：** 时间 O(n)，递归空间 O(h)。

**边界与追问：** 如果不保证两个节点存在，需要额外返回 found 状态；如果是 BST，可以利用节点值将复杂度降到 O(h)。

### Q14：数据流中的第 K 大元素

**题目：** 设计一个类，数据持续到达时返回当前第 K 大元素。

```java
import java.util.PriorityQueue;

public final class KthLargestStream {
    private final int capacity;
    private final PriorityQueue<Integer> minHeap = new PriorityQueue<>();

    public KthLargestStream(int capacity, int[] initialValues) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        this.capacity = capacity;
        for (int value : initialValues) {
            add(value);
        }
    }

    public int add(int value) {
        minHeap.offer(value);
        if (minHeap.size() > capacity) {
            minHeap.poll();
        }
        if (minHeap.size() < capacity) {
            throw new IllegalStateException("Not enough values");
        }
        return minHeap.peek();
    }
}
```

**复杂度：** 每次添加 O(log k)，空间 O(k)。

**边界与追问：** 要明确初始元素不足 K 个时的 API 行为；如果要求并发调用，字段和堆操作需要外部同步或专门的并发设计。

### Q15：实现 Trie 字典树

**题目：** 支持插入单词、查询完整单词和查询前缀。

```java
import java.util.HashMap;
import java.util.Map;

public final class Trie {
    private static final class Node {
        boolean word;
        Map<Character, Node> children = new HashMap<>();
    }

    private final Node root = new Node();

    public void insert(String text) {
        Node current = root;
        for (char character : text.toCharArray()) {
            current = current.children.computeIfAbsent(character, ignored -> new Node());
        }
        current.word = true;
    }

    public boolean contains(String text) {
        Node node = findNode(text);
        return node != null && node.word;
    }

    public boolean startsWith(String prefix) {
        return findNode(prefix) != null;
    }

    private Node findNode(String text) {
        Node current = root;
        for (char character : text.toCharArray()) {
            current = current.children.get(character);
            if (current == null) {
                return null;
            }
        }
        return current;
    }
}
```

**复杂度：** 插入和查询 O(L)，L 为字符串长度；空间 O(字符总数)。

**边界与追问：** 大字符集可以讨论数组、压缩 Trie、Unicode code point 和内存占用；删除单词需要清理不再使用的路径。

### Q16：网格最短路径

**题目：** 给定只包含 `0` 和 `1` 的网格，`0` 表示可走，返回从左上角到右下角的最短步数，无法到达返回 `-1`。

```java
import java.util.ArrayDeque;
import java.util.Queue;

public final class GridShortestPath {
    static int shortestPath(int[][] grid) {
        if (grid == null || grid.length == 0 || grid[0].length == 0
                || grid[0][0] == 1) {
            return -1;
        }

        int rowCount = grid.length;
        int columnCount = grid[0].length;
        boolean[][] visited = new boolean[rowCount][columnCount];
        Queue<int[]> queue = new ArrayDeque<>();
        queue.offer(new int[] { 0, 0, 0 });
        visited[0][0] = true;
        int[][] directions = { { 1, 0 }, { -1, 0 }, { 0, 1 }, { 0, -1 } };

        while (!queue.isEmpty()) {
            int[] current = queue.poll();
            int row = current[0];
            int column = current[1];
            int distance = current[2];
            if (row == rowCount - 1 && column == columnCount - 1) {
                return distance;
            }

            for (int[] direction : directions) {
                int nextRow = row + direction[0];
                int nextColumn = column + direction[1];
                if (nextRow >= 0 && nextRow < rowCount
                        && nextColumn >= 0 && nextColumn < columnCount
                        && grid[nextRow][nextColumn] == 0
                        && !visited[nextRow][nextColumn]) {
                    visited[nextRow][nextColumn] = true;
                    queue.offer(new int[] { nextRow, nextColumn, distance + 1 });
                }
            }
        }
        return -1;
    }
}
```

**复杂度：** 时间 O(rows * columns)，空间 O(rows * columns)。

**边界与追问：** 如果每条边权重不同，应改用 Dijkstra；如果只有 0/1 权重，可使用 0-1 BFS。

### Q17：拓扑排序判断课程是否能完成

**题目：** 有 `courseCount` 门课程和先修关系 `[course, prerequisite]`，判断是否存在环。

```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.List;
import java.util.Queue;

public final class CourseSchedule {
    static boolean canFinish(int courseCount, int[][] prerequisites) {
        List<List<Integer>> graph = new ArrayList<>(courseCount);
        for (int course = 0; course < courseCount; course++) {
            graph.add(new ArrayList<>());
        }

        int[] inDegree = new int[courseCount];
        for (int[] prerequisite : prerequisites) {
            int course = prerequisite[0];
            int requiredCourse = prerequisite[1];
            graph.get(requiredCourse).add(course);
            inDegree[course]++;
        }

        Queue<Integer> readyCourses = new ArrayDeque<>();
        for (int course = 0; course < courseCount; course++) {
            if (inDegree[course] == 0) {
                readyCourses.offer(course);
            }
        }

        int completedCount = 0;
        while (!readyCourses.isEmpty()) {
            int course = readyCourses.poll();
            completedCount++;
            for (int nextCourse : graph.get(course)) {
                if (--inDegree[nextCourse] == 0) {
                    readyCourses.offer(nextCourse);
                }
            }
        }
        return completedCount == courseCount;
    }
}
```

**复杂度：** 时间 O(V + E)，空间 O(V + E)。

**边界与追问：** 如果要返回一条可行课程顺序，记录出队顺序；如果有多个合法答案，应说明是否需要字典序。

### Q18：合并 K 个有序链表

**题目：** 合并 K 个升序链表，要求整体保持升序。

```java
import java.util.Comparator;
import java.util.PriorityQueue;

public final class MergeKSortedLists {
    static final class Node {
        int value;
        Node next;

        Node(int value) {
            this.value = value;
        }
    }

    static Node merge(Node[] lists) {
        PriorityQueue<Node> minHeap = new PriorityQueue<>(Comparator.comparingInt(node -> node.value));
        for (Node list : lists) {
            if (list != null) {
                minHeap.offer(list);
            }
        }

        Node dummy = new Node(0);
        Node tail = dummy;
        while (!minHeap.isEmpty()) {
            Node node = minHeap.poll();
            tail.next = node;
            tail = node;
            if (node.next != null) {
                minHeap.offer(node.next);
            }
        }
        return dummy.next;
    }
}
```

**复杂度：** 设总节点数为 N，时间 O(N log K)，空间 O(K)。

**边界与追问：** 也可以两两分治合并，空间复杂度接近 O(1)；如果链表节点需要保持来源信息，应在节点或包装对象中保留元数据。

### Q19：每日温度

**题目：** 给定每天温度，返回每一天之后第几天会出现更高温度，没有则返回 0。

```java
import java.util.ArrayDeque;
import java.util.Deque;

public final class DailyTemperatures {
    static int[] daysUntilWarmer(int[] temperatures) {
        int[] result = new int[temperatures.length];
        Deque<Integer> decreasingIndexes = new ArrayDeque<>();

        for (int index = 0; index < temperatures.length; index++) {
            while (!decreasingIndexes.isEmpty()
                    && temperatures[index] > temperatures[decreasingIndexes.peek()]) {
                int previousIndex = decreasingIndexes.pop();
                result[previousIndex] = index - previousIndex;
            }
            decreasingIndexes.push(index);
        }
        return result;
    }
}
```

**复杂度：** 时间 O(n)，空间 O(n)。

**边界与追问：** 单调栈保存还没有找到答案的下标；相等温度不能弹出，题目要求的是严格更高。

### Q20：数据流中位数

**题目：** 实现 `addNum` 和 `findMedian`，支持数字持续进入后查询中位数。

```java
import java.util.Collections;
import java.util.PriorityQueue;

public final class MedianFinder {
    private final PriorityQueue<Integer> lowerHalf = new PriorityQueue<>(Collections.reverseOrder());
    private final PriorityQueue<Integer> upperHalf = new PriorityQueue<>();

    public void addNum(int value) {
        if (lowerHalf.isEmpty() || value <= lowerHalf.peek()) {
            lowerHalf.offer(value);
        } else {
            upperHalf.offer(value);
        }

        if (lowerHalf.size() > upperHalf.size() + 1) {
            upperHalf.offer(lowerHalf.poll());
        } else if (upperHalf.size() > lowerHalf.size()) {
            lowerHalf.offer(upperHalf.poll());
        }
    }

    public double findMedian() {
        if (lowerHalf.isEmpty()) {
            throw new IllegalStateException("No values available");
        }
        if (lowerHalf.size() == upperHalf.size()) {
            return ((long) lowerHalf.peek() + upperHalf.peek()) / 2.0;
        }
        return lowerHalf.peek();
    }
}
```

**复杂度：** 添加 O(log n)，查询 O(1)，空间 O(n)。

**边界与追问：** 两个堆的大小差不能超过 1，且 lower half 的最大值不大于 upper half 的最小值；计算平均值时要避免整数溢出。

---

## 三、Java 集合与实用编码题

### Q21：实现 LRU 缓存

**题目：** 实现固定容量的 LRU 缓存，`get` 和 `put` 平均 O(1)。

```java
import java.util.LinkedHashMap;
import java.util.Map;

public final class LruCache<K, V> {
    private final int capacity;
    private final Map<K, V> values;

    public LruCache(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        this.capacity = capacity;
        this.values = new LinkedHashMap<K, V>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > LruCache.this.capacity;
            }
        };
    }

    public synchronized V get(K key) {
        return values.get(key);
    }

    public synchronized void put(K key, V value) {
        values.put(key, value);
    }
}
```

**复杂度：** 平均 O(1)。

**边界与追问：** `LinkedHashMap` 的访问顺序来自 `accessOrder = true`；此实现通过 `synchronized` 保证简单线程安全，但高并发场景要讨论锁竞争、分段缓存、Caffeine 和过期策略。缓存值允许 `null` 时，`get` 不能区分不存在和存在空值，应另设结果类型或禁止 null。

### Q22：按频率统计单词并返回前 K 个

**题目：** 输入一批日志单词，忽略大小写，返回频率最高的 K 个单词；频率相同按字典序升序。

```java
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.PriorityQueue;

public final class TopWords {
    static List<String> topK(List<String> words, int k) {
        Map<String, Integer> frequencyByWord = new HashMap<>();
        for (String word : words) {
            String normalizedWord = word.toLowerCase();
            frequencyByWord.merge(normalizedWord, 1, Integer::sum);
        }

        Comparator<Map.Entry<String, Integer>> worstFirst = (first, second) -> {
            int frequencyOrder = Integer.compare(first.getValue(), second.getValue());
            if (frequencyOrder != 0) {
                return frequencyOrder;
            }
            return second.getKey().compareTo(first.getKey());
        };

        PriorityQueue<Map.Entry<String, Integer>> minHeap = new PriorityQueue<>(worstFirst);
        for (Map.Entry<String, Integer> entry : frequencyByWord.entrySet()) {
            minHeap.offer(entry);
            if (minHeap.size() > k) {
                minHeap.poll();
            }
        }

        List<String> result = new ArrayList<>();
        while (!minHeap.isEmpty()) {
            result.add(minHeap.poll().getKey());
        }
        result.sort((first, second) -> {
            int frequencyOrder = Integer.compare(
                    frequencyByWord.get(second), frequencyByWord.get(first));
            return frequencyOrder != 0 ? frequencyOrder : first.compareTo(second);
        });
        return result;
    }
}
```

**复杂度：** 设不同单词数为 U，时间 O(n + U log k)，空间 O(U + k)。

**边界与追问：** 需要明确 locale、标点、Unicode 规范化和空单词处理；海量日志不能全部放内存，应讨论分片统计、外部排序或流式近似算法。

### Q23：合并重叠时间段并计算总时长

**题目：** 输入多个 `[start, end)` 时间段，合并后返回总覆盖时长。假设时间是整数且 `start <= end`。

```java
import java.util.Arrays;
import java.util.Comparator;

public final class CoveredDuration {
    static long calculate(long[][] intervals) {
        if (intervals == null || intervals.length == 0) {
            return 0L;
        }
        Arrays.sort(intervals, Comparator.comparingLong(interval -> interval[0]));

        long currentStart = intervals[0][0];
        long currentEnd = intervals[0][1];
        long total = 0L;
        for (int index = 1; index < intervals.length; index++) {
            long nextStart = intervals[index][0];
            long nextEnd = intervals[index][1];
            if (nextStart <= currentEnd) {
                currentEnd = Math.max(currentEnd, nextEnd);
            } else {
                total += currentEnd - currentStart;
                currentStart = nextStart;
                currentEnd = nextEnd;
            }
        }
        return total + currentEnd - currentStart;
    }
}
```

**复杂度：** 时间 O(n log n)，空间取决于排序实现和是否允许修改输入。

**边界与追问：** 半开区间 `[start, end)` 让相邻区间自然不重复；生产场景要说明时区、夏令时、非法区间和 `long` 溢出。

### Q24：实现字符串反转，正确处理 Unicode code point

**题目：** 反转字符串中的 Unicode code point，不能把 surrogate pair 拆开。

```java
public final class UnicodeReverse {
    static String reverse(String text) {
        int[] codePoints = text.codePoints().toArray();
        StringBuilder result = new StringBuilder(text.length());
        for (int index = codePoints.length - 1; index >= 0; index--) {
            result.appendCodePoint(codePoints[index]);
        }
        return result.toString();
    }
}
```

**复杂度：** 时间 O(n)，空间 O(n)。

**边界与追问：** code point 仍不等于用户感知的 grapheme cluster；如果要正确处理组合字符、emoji 序列和语言规则，需要讨论 `BreakIterator` 或 ICU4J。

### Q25：实现一个可重试的批处理器

**题目：** 将任务分批执行，单个任务失败时最多重试指定次数，返回成功和失败结果。不要因为一个任务失败而丢失其他任务结果。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.function.Function;

public final class BatchProcessor {
    static final class Result<T> {
        final List<T> succeeded = new ArrayList<>();
        final List<String> failed = new ArrayList<>();
    }

    static <T> Result<T> process(
            List<String> taskIds,
            int batchSize,
            int maxRetries,
            Function<String, T> task) {
        if (batchSize <= 0 || maxRetries < 0) {
            throw new IllegalArgumentException("Invalid batch configuration");
        }

        Result<T> result = new Result<>();
        for (int start = 0; start < taskIds.size(); start += batchSize) {
            int end = Math.min(start + batchSize, taskIds.size());
            for (String taskId : taskIds.subList(start, end)) {
                boolean completed = false;
                for (int attempt = 0; attempt <= maxRetries && !completed; attempt++) {
                    try {
                        result.succeeded.add(task.apply(taskId));
                        completed = true;
                    } catch (RuntimeException exception) {
                        if (attempt == maxRetries) {
                            result.failed.add(taskId);
                        }
                    }
                }
            }
        }
        return result;
    }
}
```

**复杂度：** 任务执行成本之外，遍历复杂度 O(n)。

**边界与追问：** 只能对幂等或带幂等键的任务自动重试；生产实现要加入指数退避、随机抖动、可重试异常白名单、超时、取消、审计和死信处理。

---

## 四、并发编程 Coding 题

### Q26：实现有界阻塞队列

**题目：** 不使用 `BlockingQueue`，实现线程安全的有界队列，支持 `put` 和 `take`。队列满时生产者等待，队列空时消费者等待。

```java
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public final class BoundedBlockingQueue<T> {
    private final int capacity;
    private final Deque<T> values = new ArrayDeque<>();
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();

    public BoundedBlockingQueue(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        this.capacity = capacity;
    }

    public void put(T value) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (values.size() == capacity) {
                notFull.await();
            }
            values.addLast(value);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (values.isEmpty()) {
                notEmpty.await();
            }
            T value = values.removeFirst();
            notFull.signal();
            return value;
        } finally {
            lock.unlock();
        }
    }
}
```

**关键点：** 必须使用 `while` 而不是 `if`，因为存在虚假唤醒和多个线程竞争；`signal` 只唤醒一个等待线程，吞吐和公平性要按场景验证。

**追问：** 如何支持超时？可使用 `awaitNanos`；如何关闭？增加 `closed` 状态并唤醒所有等待者；如何避免 `null` 歧义？禁止插入 null 或使用显式结果类型。

### Q27：实现线程安全的单例

**题目：** 给出一种懒加载、线程安全且避免反射破坏的单例实现，并说明更推荐的替代方案。

```java
public enum ApplicationConfig {
    INSTANCE;

    private volatile String environment = "default";

    public String getEnvironment() {
        return environment;
    }

    public void setEnvironment(String environment) {
        this.environment = environment;
    }
}
```

**核心回答：** 枚举单例由 JVM 保证实例化和序列化语义，通常比手写双重检查更可靠。若必须使用类，可以使用静态内部类：

```java
public final class LazyService {
    private LazyService() {
    }

    private static class Holder {
        private static final LazyService INSTANCE = new LazyService();
    }

    public static LazyService getInstance() {
        return Holder.INSTANCE;
    }
}
```

**追问：** 在 Spring 应用中更应让容器管理 bean 的作用域和生命周期，不要把业务对象写成难以测试的全局单例。

### Q28：实现线程安全计数器并比较方案

**题目：** 支持并发 `increment` 和读取总数，说明 `synchronized`、Atomic 和 LongAdder 的取舍。

```java
import java.util.concurrent.atomic.LongAdder;

public final class ConcurrentCounter {
    private final LongAdder value = new LongAdder();

    public void increment() {
        value.increment();
    }

    public long sum() {
        return value.sum();
    }
}
```

**核心回答：** `AtomicLong` 适合需要单值 CAS 语义和精确即时读取的场景；`LongAdder` 通过分散竞争提高高并发累加吞吐，但 `sum()` 是当前各 cell 的汇总，不应把它当作跨多个操作的事务快照；`synchronized` 语义简单，适合临界区较复杂的状态转换。

### Q29：两个线程交替打印数字

**题目：** 两个线程分别打印奇数和偶数，按顺序输出 `1` 到 `limit`。

```java
public final class AlternatingPrinter {
    private final Object monitor = new Object();
    private final int limit;
    private int next = 1;

    public AlternatingPrinter(int limit) {
        this.limit = limit;
    }

    public void printOdd() throws InterruptedException {
        printWhen(true);
    }

    public void printEven() throws InterruptedException {
        printWhen(false);
    }

    private void printWhen(boolean oddTurn) throws InterruptedException {
        while (true) {
            synchronized (monitor) {
                while (next <= limit && (next % 2 == 1) != oddTurn) {
                    monitor.wait();
                }
                if (next > limit) {
                    monitor.notifyAll();
                    return;
                }
                System.out.println(next++);
                monitor.notifyAll();
            }
        }
    }
}
```

**关键点：** 条件检查必须在循环中完成，退出时要唤醒其他线程；`notifyAll` 虽然可能产生更多竞争，但比错误使用 `notify` 更不容易让程序永久等待。

### Q30：使用 CompletableFuture 并行调用两个服务

**题目：** 并行获取用户和订单信息，两个结果都成功后组合；要求使用自定义线程池并处理超时。

```java
import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;

public final class ParallelServiceCall {
    static final class User {
    }

    static final class OrderSummary {
    }

    static final class PageData {
        final User user;
        final OrderSummary orders;

        PageData(User user, OrderSummary orders) {
            this.user = user;
            this.orders = orders;
        }
    }

    static CompletableFuture<PageData> load(
            String userId,
            Executor executor,
            java.util.function.Function<String, User> userLoader,
            java.util.function.Function<String, OrderSummary> orderLoader) {
        CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(
                () -> userLoader.apply(userId), executor);
        CompletableFuture<OrderSummary> orderFuture = CompletableFuture.supplyAsync(
                () -> orderLoader.apply(userId), executor);

        return userFuture.thenCombine(orderFuture, PageData::new)
                .orTimeout(Duration.ofSeconds(2).toMillis(), java.util.concurrent.TimeUnit.MILLISECONDS);
    }
}
```

**关键点：** `thenCombine` 适合两个独立结果合并；不要把阻塞 I/O 扔进公共 ForkJoinPool；要区分超时、取消、部分降级和真正失败。

### Q31：实现一个可取消的定时任务

**题目：** 提交周期任务并允许调用方取消，确保关闭时释放线程资源。

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;

public final class CancellableScheduler implements AutoCloseable {
    private final ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

    public ScheduledFuture<?> scheduleAtFixedRate(
            Runnable task, long initialDelay, long period, TimeUnit unit) {
        return executor.scheduleAtFixedRate(task, initialDelay, period, unit);
    }

    @Override
    public void close() {
        executor.shutdown();
    }
}
```

**关键点：** `cancel(true)` 只是发出中断请求，任务必须响应中断；任务内部不能吞掉 `InterruptedException` 后继续运行；生产代码要考虑未捕获异常导致周期任务停止、线程命名、拒绝策略和优雅关闭。

### Q32：实现线程安全的生产者消费者模型

**题目：** 使用 `ExecutorService` 和前面实现的有界队列，启动多个生产者和消费者，并提供停止机制。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public final class ProducerConsumerDemo {
    static void run(List<String> taskIds) throws InterruptedException {
        BoundedBlockingQueue<String> queue = new BoundedBlockingQueue<>(100);
        ExecutorService executor = Executors.newFixedThreadPool(4);

        for (String taskId : taskIds) {
            executor.submit(() -> {
                try {
                    queue.put(taskId);
                } catch (InterruptedException exception) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        executor.shutdown();
        if (!executor.awaitTermination(30, java.util.concurrent.TimeUnit.SECONDS)) {
            executor.shutdownNow();
        }
    }
}
```

**面试要点：** 这段代码只展示资源管理，真实消费者需要独立生命周期和结束标记。常见做法是 poison pill、关闭状态或 `ExecutorCompletionService`；不能让消费者永久阻塞，也不能在生产者未完成时过早关闭共享资源。

### Q33：避免账户转账死锁

**题目：** 多线程同时在两个账户之间转账，设计锁顺序避免死锁。

```java
public final class SafeTransfer {
    static final class Account {
        final long id;
        private long balance;

        Account(long id, long balance) {
            this.id = id;
            this.balance = balance;
        }
    }

    static void transfer(Account source, Account target, long amount) {
        if (source == target) {
            return;
        }
        Account first = source.id < target.id ? source : target;
        Account second = source.id < target.id ? target : source;

        synchronized (first) {
            synchronized (second) {
                if (amount < 0 || source.balance < amount) {
                    throw new IllegalArgumentException("Insufficient balance");
                }
                source.balance -= amount;
                target.balance += amount;
            }
        }
    }
}
```

**关键点：** 所有转账路径按同一全序获取锁，破坏循环等待条件；锁内只做必要状态修改，不调用外部服务。实际账户系统还需要数据库事务、幂等键、余额版本和审计流水，Java 对象锁不能保证跨进程一致性。

### Q34：实现一个简单的并发安全限流器

**题目：** 实现固定窗口限流：每个 key 在窗口内最多允许指定次数。

```java
import java.time.Clock;
import java.util.HashMap;
import java.util.Map;

public final class FixedWindowRateLimiter {
    private static final class Window {
        long startMillis;
        int count;
    }

    private final long windowMillis;
    private final int limit;
    private final Clock clock;
    private final Map<String, Window> windows = new HashMap<>();

    public FixedWindowRateLimiter(long windowMillis, int limit, Clock clock) {
        this.windowMillis = windowMillis;
        this.limit = limit;
        this.clock = clock;
    }

    public synchronized boolean allow(String key) {
        long now = clock.millis();
        Window window = windows.computeIfAbsent(key, ignored -> new Window());
        if (now - window.startMillis >= windowMillis) {
            window.startMillis = now;
            window.count = 0;
        }
        if (window.count >= limit) {
            return false;
        }
        window.count++;
        return true;
    }
}
```

**复杂度：** 单机内存实现平均 O(1)。

**追问：** 固定窗口存在边界突发问题，可讨论滑动窗口、漏桶、令牌桶和 Redis Lua；分布式限流需要原子操作、时钟策略、key 过期和节点故障处理。

### Q35：安全发布不可变配置

**题目：** 设计一个多线程只读配置对象，配置初始化后不可变，并保证其他线程能看到完整对象。

```java
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

public final class ImmutableConfig {
    private final String environment;
    private final Map<String, String> properties;

    public ImmutableConfig(String environment, Map<String, String> properties) {
        this.environment = environment;
        this.properties = Collections.unmodifiableMap(new HashMap<>(properties));
    }

    public String getEnvironment() {
        return environment;
    }

    public Map<String, String> getProperties() {
        return properties;
    }
}
```

**关键点：** `final` 字段支持构造完成后的安全发布语义；集合需要防御性复制和只读包装，不能只把引用声明为 `final`。如果配置需要动态替换，应使用 `volatile` 引用整体替换不可变对象，而不是并发修改内部 Map。

---

## 五、缓存、限流与可靠性 Coding 题

### Q36：实现带 TTL 的缓存

**题目：** 实现单机缓存，支持写入过期时间、读取过期判断和主动删除。

```java
import java.time.Clock;
import java.util.HashMap;
import java.util.Map;

public final class TtlCache<K, V> {
    private static final class Entry<V> {
        final V value;
        final long expireAtMillis;

        Entry(V value, long expireAtMillis) {
            this.value = value;
            this.expireAtMillis = expireAtMillis;
        }
    }

    private final Clock clock;
    private final Map<K, Entry<V>> entries = new HashMap<>();

    public TtlCache(Clock clock) {
        this.clock = clock;
    }

    public synchronized void put(K key, V value, long ttlMillis) {
        if (ttlMillis <= 0) {
            throw new IllegalArgumentException("TTL must be positive");
        }
        entries.put(key, new Entry<>(value, clock.millis() + ttlMillis));
    }

    public synchronized V get(K key) {
        Entry<V> entry = entries.get(key);
        if (entry == null) {
            return null;
        }
        if (entry.expireAtMillis <= clock.millis()) {
            entries.remove(key);
            return null;
        }
        return entry.value;
    }

    public synchronized void remove(K key) {
        entries.remove(key);
    }
}
```

**追问：** 此实现只在访问时清理过期值，生产环境要考虑最大容量、后台清理、时钟溢出、缓存击穿、穿透、雪崩、热点 key 和多节点一致性；实际项目优先评估 Caffeine 等成熟实现。

### Q37：防止缓存击穿的单飞加载

**题目：** 多个线程同时查询一个不存在或已过期的 key 时，只允许一个线程加载，其余线程等待同一个结果。

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.function.Supplier;

public final class SingleFlightCache<K, V> {
    private final ConcurrentMap<K, CompletableFuture<V>> loading = new ConcurrentHashMap<>();

    public CompletableFuture<V> load(K key, Supplier<V> loader) {
        CompletableFuture<V> created = new CompletableFuture<>();
        CompletableFuture<V> existing = loading.putIfAbsent(key, created);
        if (existing != null) {
            return existing;
        }

        try {
            created.complete(loader.get());
        } catch (Throwable error) {
            created.completeExceptionally(error);
        } finally {
            loading.remove(key, created);
        }
        return created;
    }
}
```

**关键点：** `remove(key, created)` 防止旧加载任务误删新一轮加载；失败也必须移除，否则会永久缓存失败的 Future。生产环境还要限制加载超时、处理空值、区分异常是否缓存，并避免 loader 在持有全局锁时执行。

### Q38：实现指数退避重试

**题目：** 对可重试异常执行有限次重试，等待时间按指数增长并加入随机抖动。

```java
import java.util.concurrent.ThreadLocalRandom;
import java.util.function.Predicate;

public final class RetryExecutor {
    static <T> T execute(
            java.util.concurrent.Callable<T> task,
            int maxRetries,
            long initialDelayMillis,
            Predicate<Exception> retryable) throws Exception {
        Exception lastException = null;
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return task.call();
            } catch (Exception exception) {
                lastException = exception;
                if (attempt == maxRetries || !retryable.test(exception)) {
                    throw exception;
                }

                long exponentialDelay = initialDelayMillis * (1L << Math.min(attempt, 20));
                long jitter = ThreadLocalRandom.current().nextLong(
                        Math.max(1L, exponentialDelay / 2));
                Thread.sleep(exponentialDelay + jitter);
            }
        }
        throw lastException;
    }
}
```

**关键点：** 只重试明确可恢复的错误；指数计算要防溢出并设置最大延迟；不要让大量实例同时在固定时间重试，抖动可以降低同步冲击；写请求必须具备幂等语义。

### Q39：实现简单熔断器

**题目：** 连续失败达到阈值后打开熔断，经过冷却时间允许一次探测请求，成功后关闭。

```java
import java.time.Clock;
import java.util.function.Supplier;

public final class SimpleCircuitBreaker {
    private enum State { CLOSED, OPEN, HALF_OPEN }

    private final int failureThreshold;
    private final long openMillis;
    private final Clock clock;
    private State state = State.CLOSED;
    private int failures;
    private long openedAtMillis;

    public SimpleCircuitBreaker(int failureThreshold, long openMillis, Clock clock) {
        this.failureThreshold = failureThreshold;
        this.openMillis = openMillis;
        this.clock = clock;
    }

    public synchronized <T> T execute(Supplier<T> action) {
        long now = clock.millis();
        if (state == State.OPEN) {
            if (now - openedAtMillis < openMillis) {
                throw new IllegalStateException("Circuit is open");
            }
            state = State.HALF_OPEN;
        }

        try {
            T result = action.get();
            failures = 0;
            state = State.CLOSED;
            return result;
        } catch (RuntimeException exception) {
            failures++;
            if (failures >= failureThreshold) {
                state = State.OPEN;
                openedAtMillis = now;
            }
            throw exception;
        }
    }
}
```

**关键点：** 这是面试用最小模型，不适合直接生产。真实熔断器要区分异常类型、统计时间窗口、限制半开并发探测、提供降级和指标，并避免整个执行过程长期占用锁。

### Q40：实现令牌桶限流

**题目：** 设计单机令牌桶，按固定速率补充令牌，桶有最大容量，每次请求消耗一个令牌。

```java
import java.time.Clock;

public final class TokenBucket {
    private final long capacity;
    private final double refillPerSecond;
    private final Clock clock;
    private double tokens;
    private long lastRefillNanos;

    public TokenBucket(long capacity, double refillPerSecond, Clock clock) {
        if (capacity <= 0 || refillPerSecond <= 0) {
            throw new IllegalArgumentException("Invalid bucket configuration");
        }
        this.capacity = capacity;
        this.refillPerSecond = refillPerSecond;
        this.clock = clock;
        this.tokens = capacity;
        this.lastRefillNanos = clock.millis() * 1_000_000L;
    }

    public synchronized boolean tryAcquire() {
        long nowNanos = clock.millis() * 1_000_000L;
        long elapsedNanos = Math.max(0L, nowNanos - lastRefillNanos);
        tokens = Math.min(capacity, tokens + elapsedNanos / 1_000_000_000.0 * refillPerSecond);
        lastRefillNanos = nowNanos;
        if (tokens < 1.0) {
            return false;
        }
        tokens -= 1.0;
        return true;
    }
}
```

**追问：** `Clock.millis()` 精度有限，生产实现可使用 `System.nanoTime()` 做单调耗时计算；分布式限流需要共享存储的原子脚本、节点时钟、失败策略和 key 过期。

### Q41：实现带幂等键的命令执行器

**题目：** 同一个幂等键重复提交时返回第一次执行结果，不重复执行命令。

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.function.Supplier;

public final class IdempotentExecutor {
    private final ConcurrentMap<String, Object> results = new ConcurrentHashMap<>();

    public <T> T execute(String idempotencyKey, Supplier<T> command) {
        @SuppressWarnings("unchecked")
        T result = (T) results.computeIfAbsent(idempotencyKey, ignored -> command.get());
        return result;
    }
}
```

**关键点：** 这段实现只适合演示内存单机模型。真实系统要校验同一 key 是否对应同一请求参数，持久化执行状态，区分处理中、成功和失败，设置过期时间，并使用数据库唯一约束或 Redis 原子操作防止多节点重复执行。

### Q42：实现一致性哈希的基本模型

**题目：** 将 key 映射到环上的节点，节点增删时尽量少迁移 key。

```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Map;
import java.util.NavigableMap;
import java.util.TreeMap;

public final class ConsistentHashRing<T> {
    private final int virtualNodeCount;
    private final NavigableMap<Long, T> ring = new TreeMap<>();

    public ConsistentHashRing(int virtualNodeCount) {
        this.virtualNodeCount = virtualNodeCount;
    }

    public void addNode(String nodeId, T node) {
        for (int replica = 0; replica < virtualNodeCount; replica++) {
            ring.put(hash(nodeId + "#" + replica), node);
        }
    }

    public T locate(String key) {
        if (ring.isEmpty()) {
            return null;
        }
        Map.Entry<Long, T> entry = ring.ceilingEntry(hash(key));
        return entry == null ? ring.firstEntry().getValue() : entry.getValue();
    }

    private static long hash(String value) {
        try {
            byte[] digest = MessageDigest.getInstance("SHA-256")
                    .digest(value.getBytes(StandardCharsets.UTF_8));
            long result = 0L;
            for (int index = 0; index < Long.BYTES; index++) {
                result = (result << 8) | (digest[index] & 0xffL);
            }
            return result & Long.MAX_VALUE;
        } catch (NoSuchAlgorithmException exception) {
            throw new IllegalStateException(exception);
        }
    }
}
```

**复杂度：** 定位 O(log(V))，V 为虚拟节点数；节点分布取决于 hash 和副本数。

**追问：** 还需要实现删除节点、虚拟节点权重、节点健康检查、数据迁移和一致性验证；一致性哈希不能解决缓存数据一致性本身。

### Q43：实现滑动窗口失败率统计

**题目：** 统计最近一段时间内请求总数、失败数和失败率，用于熔断判断。

```java
import java.util.ArrayDeque;
import java.util.Deque;

public final class SlidingFailureWindow {
    private static final class Event {
        final long timestampMillis;
        final boolean failure;

        Event(long timestampMillis, boolean failure) {
            this.timestampMillis = timestampMillis;
            this.failure = failure;
        }
    }

    private final long windowMillis;
    private final Deque<Event> events = new ArrayDeque<>();
    private int failureCount;

    public SlidingFailureWindow(long windowMillis) {
        this.windowMillis = windowMillis;
    }

    public synchronized void record(long timestampMillis, boolean failure) {
        removeExpired(timestampMillis);
        events.addLast(new Event(timestampMillis, failure));
        if (failure) {
            failureCount++;
        }
    }

    public synchronized double failureRate(long timestampMillis) {
        removeExpired(timestampMillis);
        return events.isEmpty() ? 0.0 : (double) failureCount / events.size();
    }

    private void removeExpired(long timestampMillis) {
        while (!events.isEmpty()
                && events.peekFirst().timestampMillis <= timestampMillis - windowMillis) {
            if (events.removeFirst().failure) {
                failureCount--;
            }
        }
    }
}
```

**关键点：** 生产系统通常使用固定时间桶而不是保存每一条事件，以控制内存和锁竞争；时间戳来源、乱序事件、并发读写、采样和最小请求数都需要定义。

---

## 六、综合工程 Coding 题

### Q44：实现线程安全的延迟任务队列

**题目：** 提交带执行时间的任务，工作线程只在任务到期后取出执行。

```java
import java.util.PriorityQueue;
import java.util.concurrent.TimeUnit;

public final class DelayTaskQueue implements AutoCloseable {
    private static final class Task implements Comparable<Task> {
        final long executeAtNanos;
        final Runnable action;

        Task(long executeAtNanos, Runnable action) {
            this.executeAtNanos = executeAtNanos;
            this.action = action;
        }

        @Override
        public int compareTo(Task other) {
            return Long.compare(executeAtNanos, other.executeAtNanos);
        }
    }

    private final Object monitor = new Object();
    private final PriorityQueue<Task> tasks = new PriorityQueue<>();
    private final Thread worker;
    private boolean closed;

    public DelayTaskQueue() {
        worker = new Thread(this::runLoop, "delay-task-worker");
        worker.start();
    }

    public void submit(Runnable action, long delay, TimeUnit unit) {
        synchronized (monitor) {
            if (closed) {
                throw new IllegalStateException("Queue is closed");
            }
            tasks.offer(new Task(System.nanoTime() + unit.toNanos(delay), action));
            monitor.notifyAll();
        }
    }

    private void runLoop() {
        while (true) {
            Task task;
            synchronized (monitor) {
                while (tasks.isEmpty() && !closed) {
                    waitQuietly();
                }
                if (closed && tasks.isEmpty()) {
                    return;
                }
                task = tasks.peek();
                long remainingNanos = task.executeAtNanos - System.nanoTime();
                if (remainingNanos > 0) {
                    waitNanos(remainingNanos);
                    continue;
                }
                tasks.poll();
            }
            try {
                task.action.run();
            } catch (RuntimeException exception) {
                exception.printStackTrace();
            }
        }
    }

    private void waitQuietly() {
        try {
            monitor.wait();
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
        }
    }

    private void waitNanos(long nanos) {
        try {
            long millis = nanos / 1_000_000L;
            int nanoseconds = (int) (nanos % 1_000_000L);
            monitor.wait(millis, nanoseconds);
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
        }
    }

    @Override
    public void close() throws InterruptedException {
        synchronized (monitor) {
            closed = true;
            monitor.notifyAll();
        }
        worker.join();
    }
}
```

**关键点：** 新提交更早任务时必须唤醒 worker；等待和取任务都要在条件循环中完成；执行 action 不能持有队列锁；生产实现应使用 `DelayQueue`、线程池、取消句柄、持久化和节点恢复机制。

### Q45：实现最近最少使用且带容量限制的缓存服务

**题目：** 除了 LRU 淘汰，还要限制缓存总权重，例如每个 value 有不同大小。

**参考思路：** 用哈希表定位节点，用双向链表维护访问顺序，再维护当前总权重。访问或写入时把节点移动到链表头，超过权重就从尾部淘汰。

```java
import java.util.HashMap;
import java.util.Map;

public final class WeightedLruCache<K, V> {
    private final class Entry {
        K key;
        V value;
        int weight;
        Entry previous;
        Entry next;
    }

    private final int maximumWeight;
    private final Map<K, Entry> entries = new HashMap<>();
    private final Entry head = new Entry();
    private final Entry tail = new Entry();
    private int currentWeight;

    public WeightedLruCache(int maximumWeight) {
        if (maximumWeight <= 0) {
            throw new IllegalArgumentException("Maximum weight must be positive");
        }
        this.maximumWeight = maximumWeight;
        head.next = tail;
        tail.previous = head;
    }

    public synchronized V get(K key) {
        Entry entry = entries.get(key);
        if (entry == null) {
            return null;
        }
        moveToHead(entry);
        return entry.value;
    }

    public synchronized void put(K key, V value, int weight) {
        if (weight <= 0 || weight > maximumWeight) {
            throw new IllegalArgumentException("Invalid entry weight");
        }
        Entry previous = entries.remove(key);
        if (previous != null) {
            remove(previous);
            currentWeight -= previous.weight;
        }

        Entry entry = new Entry();
        entry.key = key;
        entry.value = value;
        entry.weight = weight;
        entries.put(key, entry);
        addToHead(entry);
        currentWeight += weight;

        while (currentWeight > maximumWeight) {
            Entry leastRecentlyUsed = tail.previous;
            remove(leastRecentlyUsed);
            entries.remove(leastRecentlyUsed.key);
            currentWeight -= leastRecentlyUsed.weight;
        }
    }

    private void moveToHead(Entry entry) {
        remove(entry);
        addToHead(entry);
    }

    private void addToHead(Entry entry) {
        entry.next = head.next;
        entry.previous = head;
        head.next.previous = entry;
        head.next = entry;
    }

    private void remove(Entry entry) {
        entry.previous.next = entry.next;
        entry.next.previous = entry.previous;
    }
}
```

**复杂度：** `get` 和 `put` 平均 O(1)。

**追问：** 线程安全锁会限制吞吐；生产实现还要考虑 TTL、后台淘汰、统计、异步加载、读写锁是否真的有收益以及缓存穿透和击穿。

### Q46：实现批量分页去重

**题目：** 从多个分页接口结果中合并用户，按用户 ID 去重并保留第一次出现的顺序。

```java
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

public final class OrderedDeduplication {
    static final class User {
        final String id;
        final String name;

        User(String id, String name) {
            this.id = id;
            this.name = name;
        }
    }

    static List<User> merge(List<List<User>> pages) {
        Set<String> seenIds = new HashSet<>();
        List<User> result = new ArrayList<>();
        for (List<User> page : pages) {
            for (User user : page) {
                if (seenIds.add(user.id)) {
                    result.add(user);
                }
            }
        }
        return result;
    }
}
```

**复杂度：** 时间 O(n)，空间 O(u)，u 为不同用户数。

**边界与追问：** 需要明确重复记录以哪一页为准；如果 page 来自并发请求，结果顺序要由 page index 重新排序；全量结果过大时应使用分页输出或数据库去重。

### Q47：实现线程安全的批量计数器

**题目：** 高并发记录不同业务 key 的计数，并支持读取某个 key 和快照。

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.atomic.LongAdder;

public final class BatchCounter {
    private final ConcurrentMap<String, LongAdder> counters = new ConcurrentHashMap<>();

    public void increment(String key) {
        counters.computeIfAbsent(key, ignored -> new LongAdder()).increment();
    }

    public long get(String key) {
        LongAdder counter = counters.get(key);
        return counter == null ? 0L : counter.sum();
    }

    public Map<String, Long> snapshot() {
        Map<String, Long> result = new HashMap<>();
        counters.forEach((key, value) -> result.put(key, value.sum()));
        return result;
    }
}
```

**关键点：** 快照不是全局原子快照，不同 key 的读取可能来自不同时间点；如果需要一致快照，要设计 epoch、锁或外部聚合。`computeIfAbsent` 的 mapping function 不应执行慢 I/O 或带不可控副作用。

### Q48：实现安全的字符串模板替换

**题目：** 将模板中的 `${name}` 替换为参数值，缺少参数时保留原占位符；不能执行任意代码。

```java
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class SafeTemplate {
    private static final Pattern PLACEHOLDER = Pattern.compile("\\$\\{([a-zA-Z][a-zA-Z0-9_]*)}");

    static String render(String template, Map<String, String> values) {
        Matcher matcher = PLACEHOLDER.matcher(template);
        StringBuffer result = new StringBuffer();
        while (matcher.find()) {
            String key = matcher.group(1);
            String replacement = values.get(key);
            if (replacement == null) {
                replacement = matcher.group(0);
            }
            matcher.appendReplacement(result, Matcher.quoteReplacement(replacement));
        }
        matcher.appendTail(result);
        return result.toString();
    }
}
```

**关键点：** 必须使用 `Matcher.quoteReplacement`，否则参数中的 `$` 或反斜杠会被当成正则替换语法；模板只做数据替换，不允许把用户输入编译成 Java、SpEL 或脚本执行。

### Q49：实现带超时的并行任务聚合

**题目：** 并发执行多个独立任务，收集成功结果；单个任务超时或失败时记录失败，不影响其他任务完成。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.CompletionService;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorCompletionService;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

public final class ParallelAggregator {
    static <T> List<T> collect(
            List<Callable<T>> tasks,
            ExecutorService executor,
            long timeout,
            TimeUnit unit) throws InterruptedException {
        CompletionService<T> completionService = new ExecutorCompletionService<>(executor);
        for (Callable<T> task : tasks) {
            completionService.submit(task);
        }

        List<T> results = new ArrayList<>();
        long deadline = System.nanoTime() + unit.toNanos(timeout);
        for (int completed = 0; completed < tasks.size(); completed++) {
            long remaining = deadline - System.nanoTime();
            if (remaining <= 0) {
                break;
            }
            Future<T> future = completionService.poll(remaining, TimeUnit.NANOSECONDS);
            if (future == null) {
                break;
            }
            try {
                results.add(future.get());
            } catch (ExecutionException exception) {
                // 生产代码应按任务标识记录失败原因。
            }
        }
        return results;
    }
}
```

**关键点：** 使用 deadline 而不是每个任务都重新等待完整 timeout；超时后应取消尚未完成的 Future，并确认任务是否响应中断；返回部分结果前要让调用方知道结果不完整。

### Q50：设计一个线程安全的库存预占模型

**题目：** 实现单机内存版本的库存扣减，不能让库存变成负数，并说明如何扩展到分布式服务。

```java
import java.util.HashMap;
import java.util.Map;

public final class Inventory {
    private final Map<String, Integer> stockBySku = new HashMap<>();

    public synchronized void initialize(String sku, int quantity) {
        if (quantity < 0) {
            throw new IllegalArgumentException("Quantity must not be negative");
        }
        stockBySku.put(sku, quantity);
    }

    public synchronized boolean reserve(String sku, int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        int available = stockBySku.getOrDefault(sku, 0);
        if (available < quantity) {
            return false;
        }
        stockBySku.put(sku, available - quantity);
        return true;
    }

    public synchronized void release(String sku, int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        stockBySku.merge(sku, quantity, Integer::sum);
    }
}
```

**关键点：** 单机 `synchronized` 只保护一个 JVM。分布式环境要依赖数据库条件更新、乐观锁、Redis 原子脚本或库存服务，并设计订单超时释放、幂等 reservation ID、消息最终一致性、对账和超卖监控。

---

## 七、现场追问与测试清单

### 1. 算法题常见追问

- 能否把时间复杂度从 O(n²) 降到 O(n) 或 O(n log n)？
- 输入为空、只有一个元素、全重复、全负数时结果是什么？
- 是否允许修改输入？排序是否会改变调用方数据？
- 结果是否需要稳定顺序、去重、保留重复项或返回原始下标？
- 数据量超过内存时，是否需要流式、分片、外部排序或近似算法？
- Java `int` 运算是否会溢出？字符串是否需要按 Unicode code point 处理？

### 2. 并发题常见追问

- 共享变量的可见性、原子性和有序性分别如何保证？
- 锁的粒度是什么？是否可能死锁、活锁、饥饿或锁升级？
- 为什么使用 `while` 检查等待条件？如何处理中断？
- 线程池的核心线程数、最大线程数、队列和拒绝策略如何确定？
- 任务超时后如何取消？底层 I/O 和外部服务是否真的支持取消？
- 关闭时如何保证不丢任务、不永久等待和不泄漏线程？

### 3. 缓存与可靠性题常见追问

- 缓存 key 如何设计？TTL、最大容量和失效策略是什么？
- 缓存击穿、穿透、雪崩和热点 key 如何处理？
- 重试是否会放大流量？哪些异常可重试？是否需要幂等键？
- 熔断打开后用户看到什么？半开探测如何限制并发？
- 限流是单机还是分布式？时间窗口和时钟如何处理？
- 监控哪些指标：命中率、P99、拒绝数、重试数、熔断状态和队列长度？

### 4. 最低测试集合

每道现场题至少口头验证以下用例：

1. 正常输入和最小合法输入。
2. 空输入、`null` 或非法参数。
3. 重复值、边界值、极端长度和整数溢出。
4. 任务失败、超时、取消和部分成功。
5. 多线程同时调用、关闭过程和重复提交。
6. 大数据量下的时间、内存和队列增长。

### 5. 推荐的 JUnit 5 测试结构

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.Test;

public class TwoSumTest {
    @Test
    void returnsIndexesForExistingPair() {
        int[] result = TwoSum.find(new int[] { 2, 7, 11, 15 }, 9);
        assertEquals(0, result[0]);
        assertEquals(1, result[1]);
    }
}
```

测试命名应描述行为，不要只写 `test1`。并发测试不能因为“跑过一次”就证明线程安全；要多轮执行、控制并发、验证不变量，并使用超时防止测试永久挂起。

---

## 八、面试前自检

### 算法与 Java 基础

- 我能在不依赖 IDE 自动补全的情况下写出链表、栈、队列、堆、二叉树和图的基本操作。
- 我能解释每道题的循环不变量、时间复杂度和空间复杂度。
- 我能说明 `HashMap`、`ConcurrentHashMap`、`LinkedHashMap`、堆和单调队列的使用边界。
- 我能识别整数溢出、Unicode、空输入、重复值和稳定顺序问题。

### 并发与可靠性

- 我能解释可见性、原子性、有序性、锁、CAS、线程池和 Future 的区别。
- 我能实现带关闭、超时、取消和中断处理的并发组件。
- 我能说明缓存、限流、重试、熔断和幂等之间的关系。
- 我能把单机内存方案扩展到多实例，并指出数据库、Redis、消息队列和网络带来的新问题。

### 现场表达

- 写代码前我会先确认约束，而不是直接假设输入。
- 我会先给可验证的基础实现，再讨论优化和替代方案。
- 我能主动说出失败路径、监控指标、测试策略和资源释放方式。
- 项目案例使用真实数据，明确个人职责，不把团队成果全部归于自己。

> 现场 Coding 的目标不是写出最短代码，而是在有限时间内证明：你能正确建模、写出可读实现、识别边界，并能把算法放进可运行的 Java 系统中。