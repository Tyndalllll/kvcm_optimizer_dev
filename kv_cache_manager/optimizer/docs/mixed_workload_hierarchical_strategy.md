# 混合 AI Workload 的分层 KV Cache 优化执行策略

## 1. 总体目标

在混合 AI workload 场景下，构建一套从 trace 到存储策略的闭环系统，使系统能够：

1. 从混合 trace 中识别不同访问模式。
2. 将访问模式映射为稳定缓存行为的 workload 类别。
3. 对不同 workload 建立缓存收益模型。
4. 基于收益与成本决定 KV 对象进入哪一层存储。
5. 同时支持离线分析与在线决策。

一句话概括：

> 先识别 workload，再评估 workload 的缓存价值，最后做分层存储决策。

---

## 2. 总体框架（四层）

### Layer 1：Trace 表征层

将原始 trace 转换为可计算样本。

**输入**：
- 请求时间戳
- 输入 / 输出
- prefix trie 匹配关系
- session 信息（如果有）
- token 长度、prefix 长度、输出长度

**输出**：
- request-level 样本
- prefix/node-level 样本
- 时间窗口级统计特征

### Layer 2：Workload 识别层

从混合流量中识别行为稳定的 workload 类型。

**目标**：
- 找出行为相似的请求或 prefix 对象
- 形成 workload taxonomy
- 提供可解释标签

**输出**：
- workload classes
- 每个请求 / prefix 的类别归属
- 每类 workload 的行为画像

### Layer 3：Value / Cost 建模层

对 workload 或 prefix 对象计算缓存价值和层级收益。

**目标**：
- 估计未来命中概率
- 估计命中收益
- 估计对象大小
- 估计迁移成本
- 形成分层放置分数

**输出**：
- value score
- tier score
- eviction / promotion / demotion priority

### Layer 4：分层存储决策层

基于 workload 类型与价值分数执行实际策略。

**目标**：
- 选择 L1 / L2 / L3 / 不缓存
- 设置 TTL
- 设置晋升 / 降级规则
- 动态适应 workload 变化

**输出**：
- 存储层级
- 生命周期策略
- 迁移策略

---

## 3. 方案核心思想：两阶段融合

该方案不是“只做分类”，而是一个两阶段系统：

### 第一阶段：Role-based workload decomposition

回答：

> 这个对象属于哪种访问模式？

作用：建立流量结构理解与可解释性。

### 第二阶段：Cost-based tier placement

回答：

> 这个对象值不值得放高层？应该保留多久？

作用：执行真实的资源优化与在线控制。

因此不是二选一，而是融合：

- Role-based 负责识别
- Cost-based 负责决策

---

## 4. Workload 识别对象：双粒度建模

## 4.1 Request-level 粒度

每个请求一条样本。用于：
- 识别总体流量模式
- 做 workload profiling
- 分析时间分布

优点：直观、易与业务流量对齐。

局限：与缓存对象有一层距离。

## 4.2 Prefix / Node-level 粒度

将 Trie 热点 prefix 节点或子树作为样本。用于：
- 判断哪些 prefix 适合长期保留
- 识别高价值复用子树
- 映射分层存储对象

优点：贴近 KV cache 实体，易映射分层策略。

局限：可解释性略弱，需要附加画像。

## 推荐方式

采用双粒度：
- request-level：用于 role 识别和流量画像
- prefix/node-level：用于价值评估和层级决策

---

## 5. Workload 识别特征框架（五类）

## 5.1 前缀结构特征

核心特征（依赖现有 Trie 能力）：
- matched_prefix_len
- prefix_share_ratio
- prefix_depth
- fanout
- subtree_access_count
- prefix_popularity
- prefix_stability
- branch_entropy

主要区分：
- 模板复用型
- 会话递增型
- 检索拼接型
- 长尾随机型

## 5.2 时间局部性特征

用于回答“多久会再次命中”：
- inter_arrival_time
- reuse_distance_time
- reuse_distance_requests
- burstiness
- hot_survival_time
- last_access_gap
- periodicity_score

主要区分：
- 短 burst 热点
- 稳定长寿命热点
- 周期性回访对象
- 长尾低重访对象

## 5.3 生命周期特征

用于描述活跃周期与衰减过程：
- birth_time
- last_hit_time
- active_duration
- hit_count_before_decay
- ttl_like_duration
- session_boundness

主要区分：
- session 内有效对象
- 跨 session 长期复用对象
- 短命热点对象

## 5.4 容量与收益特征

Cost-based 决策基础：
- input_tokens
- prefix_tokens
- output_tokens
- estimated_kv_size
- estimated_prefill_saving
- hit_probability
- saving_per_byte
- future_value_score

主要区分：
- 小对象高频命中
- 大对象中频命中
- 低价值长尾对象

## 5.5 会话与结构特征（增强可解释）

- session_id
- turn_index
- num_turns_in_session
- cross_session_reuse_ratio
- template_indicator
- retrieval_injection_indicator
- tool_call_indicator
- code_block_indicator

增强对 chat / RAG / agent / code 的可解释性。

---

## 6. 初版 Workload Taxonomy（建议 5 类）

## Class A：模板复用型

**特征**：长共享前缀、低 fanout、命中稳定、生命周期较长。
**来源**：summary / writing / 模板化 QA。
**缓存含义**：适合中高层长期保留。

## Class B：会话递增型

**特征**：session 内前缀持续增长，session 内高复用，session 外低复用。
**来源**：chat / 多轮助手。
**缓存含义**：适合 session-local 高层，session 结束后快速降级或清理。

## Class C：检索拼接型

**特征**：固定指令前缀 + 检索片段插入，结构不稳定，命中碎片化。
**来源**：RAG / retrieval QA。
**缓存含义**：固定骨架值得缓存，检索插入段需细粒度策略。

## Class D：轨迹 / Agent 型

**特征**：请求链长、阶段性复用、中间状态多、生命周期复杂。
**来源**：agent / tool use / workflow orchestration。
**缓存含义**：阶段性短保留，更依赖时间局部性信号。

## Class E：长尾低复用型

**特征**：共享前缀短、命中概率低、生命周期不可预测、单位容量收益低。
**来源**：随机一次性请求。
**缓存含义**：不缓存或短 TTL 缓存。

---

## 7. Workload 识别方法（两步走）

## 7.1 第一步：规则 / 统计粗分类

优先基于稳定指标快速落地：
- prefix_share_ratio
- fanout
- session_boundness
- reuse_distance
- saving_per_byte

示例规则：
- 高共享前缀 + 低 fanout → 模板复用型
- 高 session 绑定 + 前缀持续增长 → 会话递增型
- 中段变化高 + retrieval 痕迹明显 → 检索拼接型
- 长链路 + 阶段性热点 → Agent 型
- 低命中低收益 → 长尾型

目标：快速建立初版 taxonomy，便于人工校验并打通链路。

## 7.2 第二步：聚类 / 学习细分

在粗分类基础上细分：
- 方法 A：K-means / GMM（特征规整、簇数可控）
- 方法 B：HDBSCAN（长尾多、簇数未知、自动噪声识别）
- 方法 C：图聚类 / 社区发现（利用 Trie 共享图）

推荐优先顺序：
1. 规则粗分
2. HDBSCAN 细分
3. 必要时引入图方法

---

## 8. Value / Cost 模型（从识别到分层）

## 8.1 单对象价值

对 prefix/node 定义：

\[
Value = \frac{P(\text{future hit}) \times Saving(\text{hit})}{Size}
\]

即：未来命中概率 × 命中收益 ÷ 占用空间。

## 8.2 加入迁移与存储成本

分层评分：

\[
TierScore = ExpectedSaving - MigrationCost - StorageCost
\]

对 L1/L2/L3 分别估计：
- L1：收益高但容量贵
- L2：收益次高、成本中等
- L3：收益较低、容量最便宜、可长留

## 8.3 Role Prior 与 CostValue 融合

最终评分：

\[
FinalScore = \alpha \cdot RolePrior + \beta \cdot CostValue
\]

- RolePrior：类别先验偏好（如模板型偏长期）
- CostValue：当前对象实时收益价值

示例：
- 模板复用型：偏中高层长期保留
- 会话递增型：偏短期高层
- 长尾型：偏不缓存或低层短保留

---

## 9. 分层存储决策框架

## 9.1 层级抽象

- L1：最快、最贵、容量最小
- L2：中速、中成本
- L3 / cold / spill：最慢、最便宜、容量最大
- 或不缓存

## 9.2 决策动作

系统支持四类动作：
- admit（是否准入）
- promote（是否晋升）
- demote（是否降级）
- evict（是否淘汰）

## 9.3 分类型策略示意

- 模板复用型：admission 更宽松，L2 长驻，高频晋升 L1
- 会话递增型：优先 L1，session 结束后快速降级/淘汰
- 检索拼接型：骨架重点缓存，拼接段谨慎缓存
- Agent 型：阶段热点短保留，promotion 条件更严格
- 长尾型：严格 admission、低 TTL、快速淘汰

---

## 10. 离线到在线实施路线

## Phase 1：离线 Profiling

**目标**：看清混合 workload 结构，验证 taxonomy。
**工作**：特征分布统计、粗分类、类别命中率/生命周期/收益对比。
**产出**：workload 分析报告 + 初版 taxonomy。

## Phase 2：离线策略仿真

**目标**：验证 workload-aware 分层优于统一策略。
**对比**：
- baseline 1：单层统一缓存
- baseline 2：分层但不区分 workload
- method：role-aware + cost-based 分层

**指标**：
- 总命中率
- 分层命中率
- 平均 latency / P99 latency
- migration 次数
- eviction 次数
- saving per byte

**产出**：workload-aware 策略收益评估。

## Phase 3：在线轻量识别

**目标**：请求到来时快速给出 workload 类型和 tier 推荐。
**输入**：prefix 匹配、近期频次、上次访问时间、session 特征、容量压力。
**输出**：workload class、tier recommendation、TTL/priority。

## Phase 4：在线自适应更新

**目标**：应对 workload drift。
**能力**：周期性重估分布、调整 role prior、更新阈值、识别流量切换点。

---

## 11. 三个关键研究问题

1. **按请求分，还是按 prefix 分？**
   - 建议：请求用于画像，prefix/node 用于决策。
2. **taxonomy 按业务名，还是按缓存行为？**
   - 建议：按缓存行为定义，以服务存储优化目标。
3. **最终靠 role 还是 cost？**
   - 建议：role 负责识别与解释，cost 负责优化与控制；采用 role-aware cost-based。

---

## 12. 收敛结论

针对混合 AI workload，不应直接在统一流量上设计分层缓存。
应先基于前缀结构、时间局部性、生命周期和收益特征，将流量分解为稳定 workload 类型，再结合对象级收益模型完成分层决策。

落地顺序：

1. workload decomposition
2. workload taxonomy
3. value/cost modeling
4. tier placement

