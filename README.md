# 掌柜智库 app

基于 Python、FastAPI 与 LangGraph 的知识库导入和查询代码，包含外部服务客户端、模型封装及页面示例；已接入 Spec Kit。

## 仓库与运行目录

源码使用 `app.*` 导入，当前目录是名为 `app` 的 namespace 包。请把仓库克隆到名为 `app` 的目录，并从它的父目录运行服务。

```bash
git clone https://github.com/YanghaoCode/knowledge-base.git app
cp -n app/.env.example .env
```

预期布局：

```text
工作目录/
├── .env           # 私密配置，来自模板并填写真实值
├── prompts/       # 当前仓库未提供，需补齐实际 .prompt 文件
├── logs/          # 运行时日志
└── app/           # 本仓库
    ├── conf/
    ├── import_process/
    ├── query_process/
    ├── .agents/
    └── .specify/
```

不要把 `.env` 放在 `app/` 内。`utils/path_util.py` 会向上查找 `.env`，将其所在目录作为 `PROJECT_ROOT`；放错位置会改变提示词等资源路径。通常无需设置 `PROJECT_ROOT`。需要覆盖时，在启动进程的环境中设置为工作目录的绝对路径。

## 启动前需要补齐

当前代码未附完整的依赖清单或锁文件，也未附实际 `prompts/`、模型权重与外部服务配置。请依据原项目环境补齐；本文不提供未经验证的依赖版本。

- 安装项目实际依赖，准备所用 BGE-M3 与 reranker 模型，并填写模型路径和设备。
- 将实际提示词放到父目录 `prompts/`，文件名需要与 `core/load_prompt.py` 及业务节点调用一致。
- 编辑父目录 `.env`，填写模型 API、MinerU、百炼 MCP、Milvus、MinIO、MongoDB、Neo4j 等相应配置。
- `LLM_DEFAULT_TEMPERATURE` 必须填写有效数值，代码会直接转换成 `float`。布尔配置格式见模板注释。
- `LOG_*` 项在模板中保持注释，沿用代码默认值；确需覆盖时先填写有效值，再取消注释。
- 模型下载脚本含原环境的 Windows 本地路径，运行前应按自己的目录调整。

`.env.example` 仅展示配置项，不是可直接运行的配置，不含可用凭据。真实密钥通过本地 `.env` 或进程环境提供，勿提交到 Git。

## 启动服务

补齐上述条件后，在 `app/` 的父目录分别打开终端运行：

```bash
python -m uvicorn app.import_process.api.import_server:app --host 127.0.0.1 --port 8000
```

```bash
python -m uvicorn app.query_process.api.query_server:app --host 127.0.0.1 --port 8001
```

导入服务端口为 8000，查询服务端口为 8001。启动命令依据源码中的 ASGI 入口编写；当前缺少配套运行材料，尚未完成端到端启动和业务验收。

## Spec Kit 与开发验证

在 Codex 中打开 `app/`，参考 `AGENTS.md` 与 `SPEC_KIT.md`。首次可用 `$speckit-constitution` 建立项目原则，再通过 `$speckit-specify` 描述功能需求。`.agents/` 与 `.specify/` 是项目协作文件，应随代码提交。

`test/` 当前包含真实流程演示脚本，会在顶层执行业务流程，可能连接外部服务或加载模型；不应直接当作隔离单元测试批量收集。配置和资源补齐后，应分别验证导入、查询、错误处理及相应外部服务连接。
