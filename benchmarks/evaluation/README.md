# 评测协议与结果汇总

本模块固定评测范围、记录结构和统计口径。评测范围是输入配置，测量结果由逐条观察记录计算；本模块不内置准确率、F1或模型评分常量。

## 数据来源与执行边界

汇总器只读取带原始证据文件引用的JSONL记录，不调用模型、不向业务接口写入，也不发送邮件。每次报告保存输入文件、协议和证据文件的SHA-256指纹。源记录仍需来自实际评测执行，文件存在与哈希一致不能单独证明记录内容正确。

`scope_complete`只表示记录数量与范围配置匹配，不代表所有任务通过。零分母输出null；缺少原始证据、重复ID或不合法分数会终止汇总。使用`--strict-scope`要求完整覆盖配置范围。

## 运行

在仓库根目录使用Python 3.11及以上版本运行。仅使用标准库，不依赖数据库或模型凭据。

```bash
python benchmarks/evaluation/aggregate_metrics.py --project deepresearch --records PATH_TO_RECORDS --output PATH_TO_REPORT
python -m unittest discover -s benchmarks/evaluation -p "test_*.py" -v
```

Copilot另外传入`--inventory`指定与本次实验一致的工具Schema目录。DeepResearch工程回归传入`--junit`指定同次pytest执行生成的JUnit XML；计数由唯一测试ID与执行状态计算。

## 所有记录共同字段

| 字段 | 含义 |
| --- | --- |
| record_id | 全局唯一的观察记录编号 |
| kind | 观察类型，见各项目说明 |
| case_id | 固定测试用例或引用核验单元编号 |
| source_artifact | 相对JSONL文件的原始请求/响应、工具轨迹或审计文件路径；文件必须存在且非空 |

将原始记录和生成的结果存放在受控目录，公开前清理凭据、候选人资料、邮件和内部知识正文。仓库不要求上传私有实验材料。

## DeepResearch评测范围

| 项目 | 协议数量 | 计数单位 |
| --- | --- | --- |
| 研究主题 | 11 | 不同研究任务 |
| 引用核验单元 | 361 | 具体主张与来源的配对 |
| 工程测试实例 | 322 | 参数化后唯一测试ID |

工程覆盖分配为任务队列120、模型配置90、知识生命周期112，见`protocol.json`。配置数量描述评测范围，实际执行数从JUnit读取；重复运行同一测试不增加唯一实例数。

### 引用记录

`kind=citation`，附带`topic_id`、`claim`、`source_url`、`source_excerpt`、`support_label`和`reason`。标签为supported、partial、unsupported或unavailable；充分支持率只将supported计入分子，其余标签都保留在分母和分布中。来源可访问和来源支持主张是不同检查项。

### 计划评分

`kind=plan`，附带`topic_id`、`judge_model`、`rubric_version`及`dimension_scores`。三个维度为requirement_coverage、question_clarity、structure_quality，范围均为1至5，允许小数。每主题保留一份最终计划评分，先取三维均值再按主题等权平均；不混入独立overall_score或0至10的报告审稿分。

### 工程回归

```bash
cd backend
python -m pytest test/test_task_queue.py test/test_configuration.py test/test_kb_lifecycle.py --collect-only -q
python -m pytest test/test_task_queue.py test/test_configuration.py test/test_kb_lifecycle.py --junitxml=regression.xml
```

在仓库根目录执行汇总：

```bash
python benchmarks/evaluation/aggregate_metrics.py --project deepresearch --records observations.jsonl --junit backend/regression.xml --output report.json
```

报告分开给出收集、通过、失败、错误和跳过数；skip、xfail不能算全部通过。队列消费、图状态续接与SSE事件续读分别测试，不因保存了检查点就推导具备消息接管或外部操作恰好一次能力。
