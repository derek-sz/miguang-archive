# 多个动态数组合并取 Top K，如何避免每次全量排序？

很多性能问题并不是因为某次计算特别昂贵，而是因为系统不断重复计算那些其实没有变化的部分。

考虑这样一个实时列表：

- 内存里维护着 \(N\) 条持续变化的记录；
- 每批数据只有少数元素真正发生插入、更新或删除；
- 界面最终只展示排名最靠前的 \(K\) 条；
- 排序字段可能升高，也可能降低；
- 相同分数的元素还必须保持确定顺序。

最直接的实现是每次复制整个数组，完整排序，再截取前 \(K\) 条：

```ts
const top = [...items].sort(compare).slice(0, k);
```

它简单、可靠，时间复杂度为 \(O(N\log N)\)。但当 \(K\ll N\)，而且每批实际变化量 \(\Delta\ll N\) 时，大部分排序工作都在重复维护未变化元素之间的相对次序。

能否只维护排名边界，而不是反复重建全部顺序？

本文介绍一种在真实项目中采用的工程方案：用两个可索引堆把全集分成 `Top` 和 `Overflow` 两个区域，并增量维护二者之间的边界。它不是一种新的基础算法，而是对堆、位置索引、稳定比较器和结果缓存的组合运用。

最终，一次完整快照同步的成本可以从全量排序的：

\[
O(N\log N)
\]

变为：

\[
O\bigl(N+\Delta\log N+K\log K\bigr)
\]

这里仍然保留了 \(O(N)\) 的快照扫描。这个限制很重要，后文会专门解释。

## 一、为什么大小为 K 的单堆不够

处理一次性静态输入时，维护一个大小为 \(K\) 的堆就可以找出 Top-K：

1. 前 \(K\) 个元素进入堆；
2. 后续元素只与当前边界比较；
3. 更好的元素替换堆中最差者。

但动态集合比静态 Top-K 多了两个问题。

第一，当前 Top-K 中的元素可能被删除，或者更新后排名下降。此时谁应该补位？

第二，原本不在 Top-K 中的元素可能更新后排名上升。如何快速知道它是否已经越过边界？

如果只维护 Top-K，自然会失去候补区域的信息。每当入选者离开时，系统仍然需要扫描全集寻找最佳候补。

所以动态 Top-K 实际上要同时回答两个问题：

- 当前入选者中，最差的是谁？
- 当前未入选者中，最好的是谁？

这正是双堆结构的起点。

## 二、把全集分成两个区域

我们把集合划分为：

- `Top`：当前最好的 \(K\) 个元素；
- `Overflow`：其余候补元素。

两个区域分别维护一个边界：

```text
更好                                                    更差

Top 区                                  Overflow 区
[ ...... | 最差入选者 ]  <── 排名边界 ──>  [ 最佳候补 | ...... ]
             ↑                                  ↑
        worst(Top)                      best(Overflow)
```

如果比较器规定“排在前面”的元素更好，那么：

- `Top` 使用反向堆，根节点是最差入选者；
- `Overflow` 使用正向堆，根节点是最佳候补。

只要最佳候补不优于最差入选者，整个跨区顺序就是正确的。反之，交换这两个边界元素，继续检查，直到边界恢复有序。

这套结构的关键不在于“用了两个堆”，而在于把原本隐含在完整排序中的排名分割线变成了一个可以局部维护的对象。

## 三、普通二叉堆还不够

普通二叉堆适合读取和删除根节点，却不擅长删除任意元素。

动态排名中的更新通常是：

1. 找到某个已有元素；
2. 删除旧值；
3. 写入新值；
4. 根据新排名重新插入。

如果不知道元素在堆数组中的位置，删除旧值仍然需要 \(O(N)\) 扫描。因此每个堆还需要一张位置表：

```ts
class IndexedHeap<ID> {
  private values: ID[] = [];
  private positions = new Map<ID, number>();
}
```

堆交换两个节点时，同时更新 `positions`。删除任意元素时：

1. 通过位置表在 \(O(1)\) 时间找到下标；
2. 用数组末尾节点覆盖该位置；
3. 删除末尾节点；
4. 根据覆盖节点与父子节点的关系上浮或下沉。

因此可索引堆能够提供：

| 操作 | 复杂度 |
|---|---:|
| `peek()` | \(O(1)\) |
| `has(id)` | \(O(1)\) |
| `push(id)` | \(O(\log N)\) |
| `pop()` | \(O(\log N)\) |
| `remove(id)` | \(O(\log N)\) |

任意位置删除有一个容易忽略的细节：末尾节点补位后，既可能需要上浮，也可能需要下沉。一个稳妥的实现是先尝试上浮；如果没有移动，再向下调整：

```ts
remove(id: ID): boolean {
  const index = this.positions.get(id);
  if (index === undefined) return false;

  const last = this.values.pop();
  this.positions.delete(id);

  if (last === undefined || index === this.values.length) {
    return true;
  }

  this.values[index] = last;
  this.positions.set(last, index);

  if (!this.bubbleUp(index)) {
    this.bubbleDown(index);
  }
  return true;
}
```

堆中只保存稳定的 `id`，完整对象统一保存在 `items: Map<ID, T>` 中。这样两个堆读取的是同一份最新对象，也不会因为对象引用发生变化而把旧对象残留在堆里。

## 四、正确性来自四个不变量

数据结构是否正确，不能只靠“看起来能工作”。实现过程中需要始终维护四个不变量。

### 1. 分区完整且互斥

每个已知元素必须且只能存在于一个区域：

\[
S=Top\cup Overflow,\qquad Top\cap Overflow=\varnothing
\]

可以使用 `locations: Map<ID, "top" | "overflow">` 记录元素归属。更新或删除时先从原区域移除，再处理新状态。

### 2. Top 容量正确

\[
|Top|=\min(K, |S|)
\]

Top 不满时，不断提升 `Overflow` 中的最佳候补，直到 Top 填满或候补为空。

### 3. 跨区有序

Top 中任何元素都不应该比 Overflow 中任何元素更差。

看似需要比较两个区域中的所有元素，实际上两个堆顶已经分别代表各自最坏和最好的边界。只需保证：

\[
best(Overflow)\not\prec worst(Top)
\]

其中 \(\prec\) 表示“排名更靠前”。

如果最佳候补都无法超过最差入选者，那么其余更差的候补自然也不可能越界。

### 4. 比较器形成确定全序

只按分数排序通常不够。大量元素可能拥有相同主排序值，如果比较器对它们始终返回 `0`，最终顺序就可能依赖插入和更新历史。

这会带来结果抖动、无意义更新，以及难以复现的测试失败。

应该加入稳定且唯一的次级键：

```ts
function compare(a: Item, b: Item): number {
  return descending(a.score, b.score) || ascending(a.id, b.id);
}
```

除了 tie-break，比较器还必须满足一致性、反对称性和传递性。对于 `NaN`、缺失字段等异常值，也应该提前定义明确的排序位置。

## 五、插入、更新与删除

### Upsert：先删除旧值，再插入新值

排序字段可能向任意方向变化。与其分别实现“升高”和“降低”的修复逻辑，一个更简单可靠的策略是：

1. 如果元素已经存在，先从原堆删除；
2. 更新对象表；
3. Top 未满时直接进入 Top；
4. 否则与最差入选者比较；
5. 插入后恢复容量和跨区边界。

```ts
upsert(item: T): void {
  const id = this.getId(item);
  this.removeFromHeap(id);
  this.items.set(id, item);

  const worstTopId = this.topHeap.peek();
  const shouldEnterTop =
    this.topHeap.size < this.limit ||
    worstTopId === undefined ||
    this.comparator(item, this.items.get(worstTopId)!) < 0;

  if (shouldEnterTop) {
    this.topHeap.push(id);
    this.locations.set(id, "top");

    if (this.topHeap.size > this.limit) {
      const demoted = this.topHeap.pop();
      if (demoted !== undefined) this.pushOverflow(demoted);
    }
  } else {
    this.pushOverflow(id);
  }

  this.rebalance();
}
```

### Remove：删除之后让最佳候补补位

删除 Overflow 元素通常不会影响 Top。删除 Top 元素则会留下一个空位，`rebalance()` 会把最佳候补提升上来。

```ts
remove(id: ID): boolean {
  if (!this.items.has(id)) return false;

  this.removeFromHeap(id);
  this.items.delete(id);
  this.rebalance();
  return true;
}
```

### Rebalance：只修复排名边界

```ts
private rebalance(): void {
  while (this.topHeap.size < this.limit && this.overflowHeap.size > 0) {
    const promoted = this.overflowHeap.pop();
    if (promoted === undefined) break;

    this.topHeap.push(promoted);
    this.locations.set(promoted, "top");
  }

  while (true) {
    const bestOverflow = this.overflowHeap.peek();
    const worstTop = this.topHeap.peek();

    if (bestOverflow === undefined || worstTop === undefined) return;

    const boundaryIsOrdered =
      this.comparator(
        this.items.get(bestOverflow)!,
        this.items.get(worstTop)!,
      ) >= 0;

    if (boundaryIsOrdered) return;

    this.overflowHeap.pop();
    this.topHeap.pop();

    this.topHeap.push(bestOverflow);
    this.locations.set(bestOverflow, "top");
    this.pushOverflow(worstTop);
  }
}
```

在每次单元素操作前不变量成立、并且逐项恢复不变量的实现中，一次插入、更新或删除可以保持在 \(O(\log N)\)。

如果选择先批量修改许多元素、最后才统一恢复边界，交换次数就可能不再是常数。复杂度结论必须与具体批处理策略一起描述。

## 六、为什么输出 Top-K 仍然需要排序

二叉堆只保证父节点优先于子节点，不保证内部数组整体有序。因此 `topHeap` 中虽然一定是最好的 \(K\) 个元素，却不能直接作为最终榜单发布。

读取时仍要排序这 \(K\) 个元素：

```ts
getTop(): readonly T[] {
  return this.topHeap
    .ids()
    .map((id) => this.items.get(id)!)
    .sort(this.comparator);
}
```

这部分成本是：

\[
O(K\log K)
\]

当 \(K\ll N\) 时，它通常显著低于排序全集。

还可以记录 Top 区版本号：只有 Top 成员或成员对象发生变化时才重新排序；如果变化完全发生在候补区，则直接返回上一次结果，把这类读取降为 \(O(1)\)。

## 七、接收完整快照时，增量究竟发生在哪里

真实系统未必能直接收到 `insert/update/remove` 事件。很多数据源提供的是周期性完整快照。

此时可以利用稳定标识符和对象引用，把快照转换为增量操作：

```ts
sync(nextItems: readonly T[]): readonly T[] {
  const nextIds = new Set<ID>();

  for (const item of nextItems) {
    const id = this.getId(item);
    nextIds.add(id);

    if (this.items.get(id) !== item) {
      this.upsert(item);
    }
  }

  for (const id of this.items.keys()) {
    if (!nextIds.has(id)) {
      this.remove(id);
    }
  }

  return this.getTop();
}
```

这里需要准确理解“增量”的含义。

为了发现新增和删除，代码仍然要扫描新快照和旧标识符集合，所以无法消除 \(O(N)\) 成本。它增量化的是排序维护，而不是差异发现。

若一批中有 \(\Delta\) 个元素发生更新或删除，一次同步的复杂度为：

\[
O\bigl(N+\Delta\log N+K\log K\bigr)
\]

若上游能够直接提供显式变更事件，则可以去掉完整快照扫描，使成本进一步接近：

\[
O\bigl(\Delta\log N+K\log K\bigr)
\]

### 引用比较的两个前提

用对象引用识别变化，需要上游遵守结构共享：

- 变化元素生成新对象；
- 未变化元素继续复用旧对象；
- 禁止原地修改排序字段；
- 也不要每批无条件重建所有对象。

只满足“不可变更新”还不够。如果每次都为所有元素创建新对象，虽然结果仍然正确，但 \(\Delta\) 会退化为 \(N\)。

无法保证引用语义时，可以改用显式版本号、排序字段指纹或增量事件。

## 八、同一份数据可能需要多套排名索引

真实场景往往不只有一种排序方式。

例如同一候选集合可能允许按评分、增长率或更新时间排序。一个堆的结构由比较器决定，不能在不重建的情况下随意更换比较器。

常见选择有两种：

1. 为高频排序语义各维护一个索引；
2. 只维护当前索引，切换排序时重新构建。

真实项目采用了第一种策略的一种变体：同一份源数据同时喂给多个动态排名索引，读取时根据当前排序语义选择相应结果。

这是一种典型的时间—空间权衡：

- 多索引增加内存和每次更新的维护成本；
- 但排序切换可以立即完成，不必临时重排全集。

选择哪种方式，不应只看排序方式的数量，还要看切换频率、集合规模和每批更新量。

另一个重要边界是：用于合并源数据的 `id -> source position` 索引，不能由 Top-K 的展示位置替代。排名位置会随更新不断变化，而源位置索引承担的是写回原集合的职责。二者应该保持分离。

## 九、引用稳定性：让收益跨过算法边界

增量排名不仅能减少排序，也有机会减少下游传播。

每次生成有序 Top-K 后，可以与上一次结果逐项比较对象引用：

```ts
const unchanged =
  nextTop.length === lastTop.length &&
  nextTop.every((item, index) => item === lastTop[index]);

if (unchanged) return lastTop;

lastTop = nextTop;
return nextTop;
```

假设某个候补元素从 10 分涨到 11 分，但 Top-K 的最低分仍然是 20。内部索引确实发生了更新，对外排名却没有任何变化。

复用原数组可以让依赖浅比较的缓存、订阅系统或 UI 框架跳过无意义的工作。

但这项收益很容易在集成层丢失：

```ts
// 会重新制造数组引用
const output = [...rankIndex.getTop()];
```

因此，引用复用不能只作为数据结构内部的小技巧。它应该被视为一项跨层性能契约：谁拥有数组、调用方能否修改、哪些边界允许复制，都需要明确约定。

如果集成层必须复制数组，那么索引内部的引用稳定仍可作为变化判断依据，但不能宣称下游已经获得了引用复用收益。

## 十、如何证明优化实现没有排错

排名数据结构有大量状态转换，单靠几个手写用例很难覆盖。最有效的测试方法之一，是保留一个简单可信的全量排序实现作为判定器：

```ts
function oracle(items: readonly Item[], k: number): Item[] {
  return [...items].sort(compare).slice(0, k);
}
```

然后使用固定种子的伪随机数生成器，连续制造大量操作：

- 随机插入；
- 随机更新；
- 随机删除；
- 让排序值范围较小，刻意制造大量同分；
- 每一步都将增量结果与全量排序结果比较。

真实实现使用了确定性随机变更，连续执行 1000 次状态转换，并在每一步与全量排序判定器对拍。

这种测试同时具备两个优点：

- 覆盖许多人工难以枚举的边界组合；
- 固定种子让任何失败都可以稳定复现。

还需要单独保护性能契约。例如：

```ts
it("候补变化但 Top-K 不变时复用结果引用", () => {
  const first = index.sync(items);
  const second = index.sync(updateOverflowOnly(items));

  expect(second).toBe(first);
});
```

值相等测试保护功能正确性，引用相同测试保护性能行为。二者不是一回事。

至少还应覆盖：

- 空集合以及 \(K=0\)；
- \(N<K\)、\(N=K\)、\(N>K\)；
- 新元素越过边界；
- Top 元素下降后被降级；
- Overflow 元素上升后被提升；
- 删除 Top 元素后自动补位；
- 大量同分元素；
- 同一元素连续更新；
- 一批快照中同时发生插入、更新和删除；
- 切换比较器后的重建。

## 十一、复杂度与适用边界

| 操作 | 时间复杂度 | 说明 |
|---|---:|---|
| 查询排名边界 | \(O(1)\) | 读取两个堆顶 |
| 单元素插入或更新 | \(O(\log N)\) | 逐项维护不变量 |
| 单元素删除 | \(O(\log N)\) | 位置表定位后调整堆 |
| 输出有序 Top-K | \(O(K\log K)\) | 可通过版本缓存跳过 |
| 完整快照同步 | \(O(N+\Delta\log N+K\log K)\) | 仍需扫描快照 |
| 当前逐项重建 | \(O(N\log N+K\log K)\) | 可用批量分区和 heapify 优化 |
| 空间 | \(O(N)\) | 两个堆、对象表和位置表 |

它适合：

- 全集较大而 \(K\) 较小；
- 单批变化量远小于全集；
- 更新频繁；
- 元素可能被删除；
- 排序字段可能双向变化；
- 系统愿意用额外内存和实现复杂度换取稳定延迟。

它不一定适合：

- \(K\) 已经接近 \(N\)；
- 每批几乎所有对象都会变化；
- 集合很小或更新频率很低；
- 完整排序已经足够快；
- 代码复杂度和长期维护成本高于性能收益。

理论复杂度只能说明增长趋势，不能替代基准测试。JavaScript 引擎中的原生排序高度优化，而 `Map`、多个堆和频繁小对象操作也有固定成本。正式决策应该在目标运行环境和真实数据分布上比较：

- 不同 \(N\)、\(K\) 和 \(\Delta/N\)；
- 全量排序与增量同步耗时；
- 首次构建成本；
- 额外内存；
- Top-K 不变时的下游更新次数；
- 两种方案的性能交叉点。

在没有可靠测量数据时，应该只陈述复杂度和已验证的功能行为，不应把理论优势写成具体的线上加速倍数。

## 结语

动态 Top-K 的核心并不是“用堆代替排序”，而是显式维护排名边界：

- 反向堆暴露最差入选者；
- 正向堆暴露最佳候补；
- 位置表支持任意元素的对数级更新与删除；
- 稳定次级键保证结果确定；
- 只对 \(K\) 个入选元素做最终排序；
- 结果缓存让没有越过边界的变化不再向外扩散。

这套方案最有价值的地方，也许不是把某个 \(O(N\log N)\) 写成更漂亮的公式，而是重新审视计算对象：系统真正关心的不是全集的完整顺序，而是 Top-K 边界是否发生变化。

当工作负载具有“全集较大、结果很小、变化稀疏、更新频繁”这些特征时，维护边界通常比反复重建全部顺序更接近问题本身。
