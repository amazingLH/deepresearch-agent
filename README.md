# DeepResearch Agent

按研究计划搜集证据并生成报告。LangGraph 组织研究和写作子图，Redis Streams 投递任务并传递进度，Milvus 保存可复用事实。

## 验证结果

- 一项研究主题的组件评测记录了 37 条报告引用，其中 36 条得到来源支持，1 条支持较弱。研究计划在需求覆盖、问题清晰度和结构三个维度的综合评分为 4.67/5。
- 本地回归记录中，任务队列、模型配置和知识生命周期三组定向测试共通过 57 项。

## 本地运行

准备 PostgreSQL、Redis Stack、Milvus 及搜索服务。Redis Stack 提供检查点所需的搜索与 JSON 扩展。先确认研究计划，再运行检索与写作。

在 `backend` 创建 Python 3.11 虚拟环境。复制 `.env.example` 为 `.env` 后填写本地服务与密钥，按需导出为进程环境变量。签名密钥请在本地独立生成。

```sh
pip install -e . loguru redis langgraph-checkpoint-redis ipython
python -m agent.db.init_db
uvicorn agent.app:app --host 127.0.0.1 --port 2024
```

在 `frontend` 安装依赖并启动开发服务。根据本机环境配置后端地址，后端示例端口为 2024。

```sh
npm ci
npm run dev
```
