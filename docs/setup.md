# 配置与本地运行

## 环境与依赖服务

使用Python 3.11。前端包含React、TypeScript及Vite，可用Node.js 22环境安装。后端需要PostgreSQL、Redis、Milvus和模型/搜索服务。Redis检查点依赖Search与JSON模块，可使用包含这些模块的Redis服务；普通精简Redis镜像不一定满足要求。

示例端口为后端`2024`、PostgreSQL `5432`、Redis `6379`、Milvus `19530`。Milvus Standalone应连同配套etcd、对象存储和持久化卷一起准备。

## 安装后端

```bash
cd backend
python -m venv .venv
# Windows PowerShell：.venv/Scripts/Activate.ps1
# Linux/macOS：source .venv/bin/activate
python -m pip install -e . loguru redis ipython
```

`loguru`和`ipython`对应源码直接导入；`redis`对应队列、会话与缓存。项目描述文件位于[pyproject.toml](../backend/pyproject.toml)。

复制`backend/.env.example`为`backend/.env`，在`backend`作为工作目录启动。不要把密钥、数据库数据或检查点内容加入公开仓库。

## 配置表

| 名称 | 用途 |
| --- | --- |
| `APP_TOKEN` | 聊天兼容接口密钥；部分服务可复用 |
| `LLM_BASE_URL` | 兼容API根地址，应与账号地域匹配 |
| `AVAILABLE_MODELS` | JSON模型列表，包含`model_id`、`display_name`、`icon`等字段 |
| `QUERY_GENERATOR_MODEL` | 查询与计划角色模型 |
| `REFLECTION_MODEL` | 研究反思默认模型 |
| `ANSWER_MODEL` | 写作默认模型 |
| `EMBEDDING_API_KEY` / `EMBEDDING_BASE_URL` | Embedding独立凭据和地址，未填可回退聊天配置 |
| `EMBEDDING_MODEL` / `EMBEDDING_DIM` | 向量模型与维度，必须和已有集合一致 |
| `DASHSCOPE_API_KEY` / `MCP_APP_ID` | DashScope搜索应用凭据与应用ID |
| `DATABASE_URL` | PostgreSQL异步连接串 |
| `REDIS_URL` / `MILVUS_URI` | Redis与Milvus服务地址 |
| `NUMBER_OF_INITIAL_QUERIES` | 角色配置的初始查询数量 |
| `MAX_RESEARCH_LOOPS` | 角色配置的研究循环上限 |
| `KB_LIFECYCLE_MODE` | `off/inform/freshness/lifecycle` |
| `WEB_SEARCH_MAX_QPS` | 进程内搜索限流，默认12 |
| `RERANKER_KB_ENABLED` / `RERANKER_WEB_ENABLED` | 知识库和Web精排开关，默认关闭 |

`AVAILABLE_MODELS=[]`是合法空列表，会导致前端没有可选项。请填入自己账号实际可用的模型，例如：

```dotenv
AVAILABLE_MODELS=[{"model_id":"YOUR_MODEL_ID","display_name":"Research Model","icon":"Zap","icon_color":"yellow-400"}]
QUERY_GENERATOR_MODEL=YOUR_MODEL_ID
REFLECTION_MODEL=YOUR_MODEL_ID
ANSWER_MODEL=YOUR_MODEL_ID
```

[Configuration.from_runnable_config](../backend/src/agent/configuration.py)对同名角色字段采用“环境变量优先、请求configurable其次、默认值最后”。页面提交的`reasoning_model`还涉及图状态字段传播，不能只凭界面选择就判断所有角色均已切换；固定角色模型可直接使用上述三个变量。

## 搜索、Embedding与精排

搜索通过[base_agent.py](../backend/src/agent/base_agent.py)中的DashScope应用封装调用。`MCP_APP_ID`需要对应已配置搜索能力的应用，不是任意聊天模型ID，也不是本地MCP服务器地址。

知识库向量模型应与`EMBEDDING_DIM`一致。已有集合维度不会因修改`.env`自动迁移；更换维度时使用独立集合或在备份后重建。精排默认关闭，开启后需另有TextReRank模型权限和匹配凭据。

## 初始化与启动

先创建连接串中指定的PostgreSQL数据库。然后在`backend`执行：

```bash
python -m agent.db.init_db
uvicorn agent.app:app --host 127.0.0.1 --port 2024
```

[初始化脚本](../backend/src/agent/db/init_db.py)创建用户及线程关联表、索引和本地示例账号`zhangsan`、`lisi`，密码各自与用户名相同。仅用于本地初始化，开放服务前应改为自己的账号配置并替换默认凭据。不要对已有数据使用脚本的`--drop`选项。

前端启动：

```bash
cd frontend
npm ci
npm run dev
```

[vite.config.ts](../frontend/vite.config.ts)将相对路径代理到`http://127.0.0.1:2024`。前端API基地址为空字符串，依赖同源或开发代理；更换后端端口需同步代理。登录Cookie也依赖正确的同源配置。

## 两种后端入口

`uvicorn agent.app:app`运行自建FastAPI服务，提供账号、`/api/research`和事件流。`langgraph dev`则读取[langgraph.json](../backend/langgraph.json)，用于图开发与LangGraph平台接口。当前React研究页面主要使用自建研究接口；两种入口的图执行、生命周期与平台线程API不能直接混为一谈。

## 排查顺序

| 现象 | 检查 |
| --- | --- |
| 模型下拉列表为空 | `AVAILABLE_MODELS`不是空数组，JSON格式正确 |
| 登录失败 | PostgreSQL已初始化、Redis会话连接及默认账号是否已修改 |
| 研究提交503 | Redis任务流连接与应用日志 |
| 检查点初始化失败 | Redis Search/JSON模块是否齐全 |
| 搜索失败 | 应用ID、搜索能力配置、凭据及限流 |
| Milvus维度不一致 | `EMBEDDING_DIM`、模型输出及既有集合Schema |
| 页面只显示计划 | 是否确认计划并携带相同任务编号再次提交 |
| SSE没有最终报告 | 阶段错误事件、worker日志、是否处于`task_paused` |

前端页面、队列、事件和模型各有独立失败路径，先定位失败阶段再更换配置，避免把所有错误都归为模型问题。
