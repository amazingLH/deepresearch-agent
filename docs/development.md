# 开发与验证

## 接口与载荷

| 接口 | 用途 |
| --- | --- |
| `GET /api/models` | 模型选择列表 |
| `POST /api/login` | 用户名密码登录并设置会话Cookie |
| `GET /api/whoami` | 当前会话身份 |
| `POST /api/logout` | 删除会话并清理Cookie |
| `POST /api/research` | 入队并返回任务编号和事件流地址 |
| `GET /api/research/{task_id}/stream` | SSE研究事件 |

初次研究载荷：

```json
{
  "messages": [{"type": "human", "content": "研究某类开发工具的产品能力与商业模式"}],
  "initial_search_query_count": 2,
  "max_research_loops": 2,
  "plan_status": "unconfirmed"
}
```

计划生成后读取`task_paused`事件。用户确认后复用返回的`task_id`，提交`plan_status="confirmed"`、已确认的`plan`与确认消息。新任务编号意味着另一研究状态，不能用来续接原计划。

## 事件与界面

[useResearchStream.ts](../frontend/src/lib/useResearchStream.ts)负责POST任务和创建EventSource，[ActivityTimeline](../frontend/src/components/ActivityTimeline.tsx)展示阶段。worker把子图节点名翻译成前端事件，例如`generate_queries → generate_query`、`web_search → web_research`、`critique → reflection`。

文本事件包含`token.text`和`token.node`，阶段事件携带节点输出；`task_paused`表示等待用户，`finalize_answer`表示报告结束，`error`表示失败。接口受理、图节点完成和最终报告完成是三个不同时间点。

增加新图节点时，需要检查状态字段、事件映射和界面渲染是否同步。修改事件时也要考虑Redis已有历史事件与旧前端的兼容。

## 建议阅读路径

先看[app.py](../backend/src/agent/app.py)与[task_queue.py](../backend/src/agent/task_queue.py)，然后读[主图](../backend/src/agent/graph.py)、[状态](../backend/src/agent/state.py)、[研究子图](../backend/src/agent/sub_agents/research_agent.py)、[写作子图](../backend/src/agent/sub_agents/writer_agent.py)。最后再看[模型代理](../backend/src/agent/base_agent.py)、[结构化解析](../backend/src/agent/post.py)和[知识库](../backend/src/agent/kb/fact_store.py)。

## 功能验证矩阵

| 场景 | 预期证据 |
| --- | --- |
| 初次主题输入 | 生成计划，尚未开始完整研究 |
| 修改与确认 | 相同任务状态续接，研究范围与确认计划一致 |
| 多查询搜索 | 查询、搜索摘要及来源分别归并 |
| 搜索证据不足 | 补充查询或达到轮数上限后退出 |
| 写作修订 | 结构化反馈影响下一版草稿，修订次数有限 |
| 引用还原 | 短链接替换为真实来源，正文引用可追踪 |
| 知识时效 | 新旧事实分别验证四种生命周期模式 |
| 模型异常 | 429/5xx与401/400走不同分类与重试路径 |
| 浏览器断开 | 区分图任务是否继续、事件是否重放与是否已过期 |
| worker崩溃 | 检查Pending消息，不能误判为已经自动恢复 |

## 评测指标与统计口径

引用核验覆盖11个研究主题的361条报告引用，引用支持率为`95.01%`。每个核验单元对应具体主张、来源地址和原文片段，检查来源是否充分支持主张；链接可访问不等于内容有支撑。部分支持、不支持和来源不可访问分别保留分类，不从分母中静默剔除。

研究计划质量平均得分为`4.71/5`，从需求覆盖、问题清晰度和结构质量评价。计划质量与报告审稿使用不同量表，不能将0至5分与0至10分直接混合平均。

工程回归通过322项定向测试，覆盖任务队列、模型配置及知识生命周期，检查消息消费、模型参数优先级和历史事实时效规则。工程测试以唯一测试实例计数，重复执行不增加独立用例数量；失败、错误与跳过单独统计。

原始运行日志、来源快照和完整实验报告在受控环境保存，不纳入公开仓库。统计协议与结果汇总方式见[评测模块](../benchmarks/evaluation/README.md)。

## 异常与重试

[exceptions.py](../backend/src/agent/exceptions.py)区分临时和永久错误，[模型适配](../backend/src/agent/llm/llm.py)把429、服务端错误、网络超时映射为可重试类，把认证和无效请求映射为不可重试类。代理层进行有限次数退避。

模型SDK重试、搜索缓存和任务队列重试不是同一层。当前worker在失败后ACK，不会因内部存在模型重试就自动重投整个研究任务。新增恢复策略时应定义已完成节点是否复用、事件是否重复及用户如何识别失败任务。

## 扩展建议

新增搜索源时，保持标题、摘要、原URL和来源标识一致；新增研究状态时明确标量覆盖还是列表累加；增加Writer检查时使用独立可验证条件，避免只靠同模型自评。增加多worker前先改消费者标识、Pending认领、任务owner及幂等处理。
