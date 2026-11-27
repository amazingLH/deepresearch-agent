# DeepResearch Agent

按研究计划搜集证据并生成报告。LangGraph 组织研究和写作子图，Redis Streams 投递任务并传递进度，Milvus 保存可复用事实。

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
