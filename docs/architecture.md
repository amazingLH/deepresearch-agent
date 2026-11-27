# 系统架构与研究工作流

## 主图与计划确认

[graph.py](../backend/src/agent/graph.py)以`OverallState`构建主图，顺序为计划、确认、研究子图、写作子图。计划未确认时走`awaiting_plan_confirmation`并结束当前执行，状态由检查点保存；前端提交同一任务编号与已确认计划后再次进入图。

这条计划交互通过状态路由和再次提交实现，不是调用LangGraph `interrupt`暂停。`plan_status`控制计划是否生成，`plan_messages`保留计划相关消息，修改范围后可回到重新规划。

## 状态与并行归并

[state.py](../backend/src/agent/state.py)定义消息、计划、查询、来源、研究轮数、证据缺口、草稿、审稿意见及修订次数。消息使用`add_messages`归并；查询、搜索摘要和来源列表使用`operator.add`，让并行分支返回的结果累加。

列表归并保留分支产物，但不会自动去重，也不会自动限制上下文长度。当前系统主要通过查询数量、研究轮数、修订次数与搜索摘要来控制任务规模，不是完整Token预算调度器。

## 研究子图

[research_agent.py](../backend/src/agent/sub_agents/research_agent.py)先生成带理由的查询，再用`Send`分发独立搜索。搜索节点访问缓存或外部搜索服务，把标题、摘要和链接压成供模型读取的材料；来源使用短引用映射，降低重复长URL对上下文的影响。

反思节点输出`is_sufficient`、`knowledge_gap`及`follow_up_queries`。证据充分或达到最大轮数则退出，否则按补充查询再次分发。模型反思失败时视为证据不足，仍受轮数限制约束。最大轮数防止无限搜索，不证明证据已经完整。

搜索实现包含同步SDK/模型调用，因此节点分发能力不应被直接解释为所有底层I/O都已无阻塞。需要根据具体服务延迟和线程/异步边界分析吞吐。

## 写作与审稿循环

[writer_agent.py](../backend/src/agent/sub_agents/writer_agent.py)依次执行`outline → draft → critic_review → cite_and_polish`。Critic使用[CritiqueResult](../backend/src/agent/tools_and_schemas.py)约束评分、问题位置、严重程度和建议，Writer把反馈加入下一轮修订。

退出条件是：明确可润色；评分至少8分；至少修订一次且评分至少6分；或达到默认3次修订上限。评分是`0～10`量表。当前审稿失败时会降级直接润色，并使用默认分值；所以节点中的`critic_score`不能直接作为独立质量证据。

润色后根据来源映射替换短链接，保留在正文使用的引用。链接还原解决可追溯性，不代表来源内容已经被独立核实；人工检查还需确认来源确实支持对应陈述。

## 事实知识库与时效

[FactExtractor](../backend/src/agent/kb/extractor.py)从搜索摘要提取事实；[FactStore](../backend/src/agent/kb/fact_store.py)保存`fact_text`、`source_url`、`research_topic`、`confidence`、`created_at`和`fact_category`。Embedding生成向量，Milvus承担相似检索。查询时可按置信度和年龄过滤，并可启用重排。

[生命周期配置](../backend/src/agent/kb/lifecycle.py)提供四种模式：

| 模式 | 行为 |
| --- | --- |
| `off` | 关闭年龄标记、过滤和衰减 |
| `inform` | 提示事实年龄，但不过滤或衰减 |
| `freshness` | 按研究新鲜度使用7/30/180天阈值 |
| `lifecycle` | 市场7天、产品30天、战略90天、技术180天，历史类别不按TTL过期 |

年龄过滤是查询阶段的逻辑，不是自动物理删除。历史类别虽然不按TTL排除，当前衰减实现仍有回退时间尺度，不能解释为永远不降权。时效策略减少旧事实误用，但未知时间、错误来源和模型提取错误仍需单独治理。

## Redis的四种职责

1. `research:tasks`是任务流，消费者组为`research-workers`。
2. `research:events:<task_id>`保存阶段、文本、错误及结束事件。
3. Redis Checkpointer保存图状态，代码配置7天并读时续期。
4. 账号会话保存24小时并续期，搜索缓存默认1小时。

[task_queue.py](../backend/src/agent/task_queue.py)以固定消费者名`worker-1`每次读一条新任务。当前没有Pending消息接管，且执行失败后在`finally`仍会ACK。因此“使用Consumer Group”不等于已有可靠失败重试、崩溃自动恢复或exactly-once语义。

## SSE与任务续接

提交接口立即返回`task_id`与`stream_url`。worker把图节点更新和部分Token写入事件流，SSE读取后输出`data:`帧。终止事件包括`finalize_answer`、`task_paused`和`error`。

事件流最大保留约500条并设置一天过期。读取器接受`Last-Event-ID`，但HTTP输出当前未发送标准SSE `id:`字段，前端也没有完整持久化事件游标；所以不能声称断线后自动精确续读。浏览器重连可能重放事件，任务检查点续接和页面事件续读是两件事。

## 业务数据与访问控制

PostgreSQL保存用户和`UserThread`关联，密码使用bcrypt；Redis存Cookie对应的会话。中间件保护非公开路径。当前任务提交、原任务编号复用和事件读取还需补完整owner校验；“已登录”并不自动证明有权读取任意任务编号。

公开部署还需为参数数量、研究轮数、模型费用和输出大小设置服务端上限，并规划数据库、图状态、事件与知识索引各自的保留与备份策略。
