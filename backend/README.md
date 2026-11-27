# DeepResearch Agent Backend

后端包含FastAPI研究接口、LangGraph主图与研究/写作子图、Redis任务及事件流、PostgreSQL账号和Milvus事实知识库。

请从[项目说明](../README.md)开始，安装与密钥配置见[配置与运行](../docs/setup.md)，模块交互见[系统架构](../docs/architecture.md)，接口和验证见[开发与验证](../docs/development.md)。

在本目录配置`.env`后，执行`python -m agent.db.init_db`初始化本地数据库，再执行`uvicorn agent.app:app --host 127.0.0.1 --port 2024`启动自建API服务。
