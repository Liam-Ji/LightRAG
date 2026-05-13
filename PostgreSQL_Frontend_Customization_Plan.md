# LightRAG PostgreSQL + 前端定制计划

## 目标

使用源码方式运行 LightRAG，便于定制 WebUI，同时使用 PostgreSQL 作为演示数据的统一存储后端。

最终演示访问地址：

- 后端 API：`http://localhost:9621`
- WebUI: `http://localhost:9621/webui/`
- 存储：PostgreSQL

## 阶段 1：环境确认

确认本地工具：

- Python 3.10+
- `uv`
- Bun
- Docker，或已有 PostgreSQL 实例
- 可访问选定的 LLM 服务
- 可访问选定的 Embedding 服务

当前开发机环境检查结果：

- 系统：Ubuntu 24.04
- Python：已安装 `python3`
- Node.js：已安装
- `uv`：待安装
- Bun：待安装
- Docker：待安装

确认运行模式：

- 开发期：分别运行 `lightrag-server` 和 `bun run dev`。
- 演示期：构建前端，由 `lightrag-server` 在 `/webui/` 统一托管。

导入数据前必须确定：

- LLM provider
- LLM model
- Embedding provider
- `EMBEDDING_MODEL`
- `EMBEDDING_DIM`

演示数据完成索引后，Embedding 模型和维度应视为固定配置。

## 阶段 1.1：开发环境构建

本项目 Python 环境统一使用 `uv` 管理，前端使用 Bun，PostgreSQL 建议用 Docker 启动。

### 安装 uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv --version
```

### 安装 Bun

```bash
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun --version
```

### 安装 Docker

Ubuntu 24.04 可先使用系统包安装 Docker：

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

执行 `usermod` 后需要重新登录终端，再验证：

```bash
docker --version
docker compose version
```

### 安装后端依赖

在仓库根目录执行：

```bash
cd /home/liam/workspace/dev/LightRAG
uv sync --extra api --extra offline-storage
```

### 安装前端依赖

```bash
cd /home/liam/workspace/dev/LightRAG/lightrag_webui
bun install --frozen-lockfile
```

### 启动 PostgreSQL

Docker 可用后，启动 PostgreSQL：

```bash
docker run -d \
  --name lightrag-postgres \
  -p 5432:5432 \
  -e POSTGRES_USER=rag \
  -e POSTGRES_PASSWORD=rag \
  -e POSTGRES_DB=rag \
  -v lightrag_pg_data:/var/lib/postgresql \
  gzdaniel/postgres-for-rag:16.6
```

验证容器状态：

```bash
docker ps --filter name=lightrag-postgres
```

## 阶段 2：PostgreSQL 存储设计

将 LightRAG 的四类存储全部切到 PostgreSQL：

```env
LIGHTRAG_KV_STORAGE=PGKVStorage
LIGHTRAG_DOC_STATUS_STORAGE=PGDocStatusStorage
LIGHTRAG_GRAPH_STORAGE=PGGraphStorage
LIGHTRAG_VECTOR_STORAGE=PGVectorStorage
```

推荐 PostgreSQL 要求：

- PostgreSQL 16.6 或更新版本
- 启用 `pgvector`
- 使用 `PGGraphStorage` 时需要 AGE 支持

推荐容器镜像：

```text
gzdaniel/postgres-for-rag:16.6
```

需要提前规划：

- 数据库名，例如 `rag`
- 数据库用户，例如 `rag`
- 强密码
- Workspace 名称，例如 `demo`
- 持久化数据卷位置
- 最终演示前的备份或快照流程

## 阶段 3：后端配置

Python 环境统一使用 `uv` 管理。首次准备后端环境：

```bash
uv sync --extra api --extra offline-storage
```

后续运行 Python 命令时优先使用：

```bash
uv run <command>
```

在仓库根目录创建或更新 `.env`。

基础存储配置：

```env
WORKING_DIR=./rag_storage
INPUT_DIR=./inputs

LIGHTRAG_KV_STORAGE=PGKVStorage
LIGHTRAG_DOC_STATUS_STORAGE=PGDocStatusStorage
LIGHTRAG_GRAPH_STORAGE=PGGraphStorage
LIGHTRAG_VECTOR_STORAGE=PGVectorStorage

POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=rag
POSTGRES_PASSWORD=change_me
POSTGRES_DATABASE=rag
POSTGRES_MAX_CONNECTIONS=25

WORKSPACE=demo
POSTGRES_WORKSPACE=demo

POSTGRES_VECTOR_INDEX_TYPE=HNSW
POSTGRES_HNSW_M=16
POSTGRES_HNSW_EF=200
```

选定 provider 后，补充 LLM 和 Embedding 配置。

模型选择按项目 README 的推荐标准执行：

- LLM：推荐参数量至少 32B，上下文长度至少 32K，64K 更好；文档索引阶段不建议使用 reasoning 模型。
- Embedding：推荐主流多语言 Embedding，例如 `BAAI/bge-m3` 或 `text-embedding-3-large`。
- Reranker：启用后推荐使用 `mix` 查询模式；推荐 `BAAI/bge-reranker-v2-m3` 或 Jina 等服务商的 reranker。

因此当前规划确认使用硅基流动 SiliconFlow 的 OpenAI 兼容接口，使用 32B 级别 Qwen 作为 LLM，使用项目推荐的 `BAAI/bge-m3` 作为 Embedding。

硅基流动 SiliconFlow 配置：

```env
LLM_BINDING=openai
LLM_MODEL=Qwen/Qwen3-32B
LLM_BINDING_HOST=https://api.siliconflow.cn/v1
LLM_BINDING_API_KEY=replace_with_siliconflow_api_key

EMBEDDING_BINDING=openai
EMBEDDING_MODEL=BAAI/bge-m3
EMBEDDING_DIM=1024
EMBEDDING_TOKEN_LIMIT=8192
EMBEDDING_SEND_DIM=false
EMBEDDING_USE_BASE64=false
EMBEDDING_BINDING_HOST=https://api.siliconflow.cn/v1
EMBEDDING_BINDING_API_KEY=replace_with_siliconflow_api_key
```

如果要启用 reranker，优先使用项目推荐的模型：

```env
RERANK_BINDING=cohere
RERANK_MODEL=BAAI/bge-reranker-v2-m3
RERANK_BINDING_HOST=https://api.siliconflow.cn/v1/rerank
RERANK_BINDING_API_KEY=replace_with_siliconflow_api_key
```

如果硅基流动账号下的模型名称或接口路径有变化，以控制台和官方模型列表为准。模型确定后，需要先用最小请求测试 Chat、Embedding 和 Rerank 接口，再导入演示数据。

阿里云百炼仅作为备用方案记录。当前正式选型为硅基流动。

阿里云百炼示例：

```env
LLM_BINDING=openai
LLM_MODEL=qwen-plus
LLM_BINDING_HOST=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_BINDING_API_KEY=replace_with_dashscope_api_key

EMBEDDING_BINDING=openai
EMBEDDING_MODEL=text-embedding-v4
EMBEDDING_DIM=1024
EMBEDDING_SEND_DIM=true
EMBEDDING_USE_BASE64=false
EMBEDDING_BINDING_HOST=https://dashscope.aliyuncs.com/compatible-mode/v1
EMBEDDING_BINDING_API_KEY=replace_with_dashscope_api_key
```

如果使用中国香港或新加坡地域，`LLM_BINDING_HOST` 和 `EMBEDDING_BINDING_HOST` 改为对应地域的兼容接口地址：

```env
https://dashscope-intl.aliyuncs.com/compatible-mode/v1
```

DeepSeek 官方接口更适合作为 Chat LLM 使用；当前规划不把它作为首选 Embedding provider。若后续使用 DeepSeek 做 LLM，Embedding 仍建议使用项目推荐的 `BAAI/bge-m3`。

## 阶段 4：前端定制计划

开发期间，前后端分开运行。

后端：

```bash
uv run lightrag-server
```

前端：

```bash
cd lightrag_webui
bun run dev
```

开发访问地址：

- 前端开发服务：`http://localhost:5173/`
- 后端服务：`http://localhost:9621`

第一轮定制范围：

- 品牌、标题和描述
- 默认导航或默认入口页面
- 上传和检索页面文案
- 演示流程所需的查询默认参数
- 隐藏演示中不需要的功能入口
- 如果演示包含图谱探索，优化图谱页面展示

第一轮不建议大改前端架构，优先保证演示流程稳定。

## 阶段 5：演示数据导入

导入数据前确认：

- PostgreSQL 正常运行，并且数据可持久化。
- `.env` 已将四类存储全部配置为 PostgreSQL。
- `POSTGRES_WORKSPACE` 已确定。
- `EMBEDDING_MODEL` 和 `EMBEDDING_DIM` 已确定。
- 后端启动时没有存储相关错误。

导入数据后避免修改：

```env
EMBEDDING_MODEL
EMBEDDING_DIM
LIGHTRAG_VECTOR_STORAGE
POSTGRES_WORKSPACE
```

修改这些配置可能需要清空 workspace，或重建数据库表和索引。

## 阶段 6：构建与演示运行

前端定制完成后，构建 WebUI：

```bash
cd lightrag_webui
bun run build
cd ..
```

构建产物会写入：

```text
lightrag/api/webui/
```

然后只运行后端：

```bash
uv run lightrag-server
```

演示地址：

```text
http://localhost:9621/webui/
```

## 阶段 7：验证清单

演示前验证：

- `lightrag-server` 可以成功启动。
- `/health` 响应正常。
- `/docs` 可以访问。
- `/webui/` 可以访问。
- 可以上传样本文档。
- 文档索引可以完成。
- 查询能返回相关结果。
- 如果演示包含图谱页面，图谱视图可以正常打开。
- PostgreSQL 重启后索引数据不丢失。
- 已准备数据库备份或数据卷快照。

## 建议执行顺序

1. 选择 LLM 和 Embedding provider。
2. 确认 Embedding 模型和维度。
3. 安装 `uv`、Bun 和 Docker。
4. 使用 `uv sync --extra api --extra offline-storage` 安装后端依赖。
5. 使用 `bun install --frozen-lockfile` 安装前端依赖。
6. 启动或准备 PostgreSQL。
7. 编写 `.env`。
8. 启动后端并确认健康状态。
9. 启动前端开发服务。
10. 实施前端定制。
11. 导入演示数据。
12. 构建 WebUI。
13. 执行最终演示验证。
