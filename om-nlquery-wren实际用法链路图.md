# om-nlquery 里 Wren 的实际用法链路图

> 配套文档：`wren-ai-链路与集成调研.md`（Wren CLI 通用机制）。
> 本文回答：om-nlquery **实际**怎么用 Wren —— 从 PG 库到 LLM 问数服务的完整链路。
> 依据：本仓 `wren_server.py` / `gen_mdl.py` / `deploy/`（entrypoint / k8s / Dockerfile / agent 配置）/ `docs/wren-server.md`，逐文件读源码核对。

## 0. 一句话概括

**om-nlquery = 把 Wren 当"带业务口径的只读 SQL 语义层"，外面包一个 Flask（`wren_server.py`），
用 wren CLI 子进程（query / memory fetch / memory recall）+ GLM 编排出三种问数模式（编排单轮 / 编排多轮 / Claude Agent MCP）。**

与官方用法的对应关系：

| 官方能力 | om-nlquery 用法 | 备注 |
|---|---|---|
| `wren --sql` | `/wren/exec` mode=sql：`wren query --sql ... -o json` | SELECT/WITH/EXPLAIN 白名单 + 只读账号双层保险 |
| `wren memory fetch` | schema 检索（问数 Step 2 + `/wren/exec` mode=fetch） | 召回弱时按表名/列名子串扫 `models/` 兜底 |
| `wren memory recall` | 多轮问数 Step 1.5a 召回已采纳口径 | 用户「采纳」沉淀的问法→SQL 优先复用 |
| `wren memory store` | **不用**（HuggingFace 下载会失败）——直接写 `knowledge/sql/*.md` frontmatter 文件 | 向量库加载推迟到 PR 合入后服务升级统一做（用户决策 2026-09-03） |
| `wren serve mcp` | **仅 Agent 模式**：`deploy/agent/.mcp.json` 配 `wren serve mcp --project /wren`，claude code 以 MCP 工具自主调用 | 编排模式不经过 MCP，直接 subprocess 调 wren CLI |
| `wren context init` | **不用**——项目由 `gen_mdl.py` 从 PG schema 自动生成 + 手工补 knowledge | 反向：先有库再生成 MDL |
| `wren profile add` | 容器 entrypoint 启动时 `--from-file` 注入只读账号（PGPASSWORD 等全 env） | 镜像零真值 |
| `wren context import dbt` | 不用 | 无 dbt 项目 |

## 1. 构建期链路（PG schema → MDL 项目 → 镜像）

```
预览/生产 PostgreSQL (onedata 库, public schema)
        │  gen_mdl.py（本地/CI 跑一次）
        │  psycopg 连库 + wren.type_mapping.parse_type 类型归一化
        │  pg_index/information_schema 抽主键、外键；排除模板占位表名（含 ${）
        ▼
om-nlquery 仓（= Wren 项目，schema_version 5）
├── wren_project.yml        # data_source: postgres, profile: onedata
├── models/<表名>/metadata.yml   # 2347 个模型（一表一模，纯表引用无 ref_sql）
├── relationships.yml            # 外键 → MANY_TO_ONE 关系
├── knowledge/
│   ├── rules/*.md               # 人工写的业务口径（ttfhw.md / openubmc-contribute.md …）
│   ├── sql/*.md                 # 已采纳问法→SQL（frontmatter: nl/sql/source:user）
│   ├── metrics/activated-user.md
│   └── column-comments/*.sql    # COMMENT ON COLUMN 语句（列注释的 git 单一真源）
└── target/mdl.json              # wren context build 产物，直接 COPY 进镜像
        │  docker build（多阶段：deps(root 装 wrenai+node+claude) → runtime(uid 1000)）
        │  COPY models/ knowledge/ target/ wren_project.yml → /wren
        │  COPY deploy/agent/{CLAUDE.md,.mcp.json} → /wren-agent（Agent 模式工作目录）
        ▼
   镜像（零密钥，wren_server.py 在 /usr/local/bin）
```

关键点：**models 是从真库反向生成的**（不是手写），knowledge 才是人工增量；列注释不走 PG 的
col_description（pod 里没 psycopg2），而是解析仓内 `knowledge/column-comments/*.sql` 文件。

## 2. 运行期链路（容器启动 → 三种问数模式）

### 2.1 容器启动（deploy/entrypoint.sh）

```
k8s Secret 注入 env（PGHOST/PGPASSWORD(wren_ro 只读账号)/GLM_API_KEY/…）
   → ① /tmp/conn.yml（临时）→ wren profile add onedata --from-file … --activate → rm（用后即删）
   → ② GLM key 落 /tmp/.glmkey（chmod 600，wren_server env 优先、文件兜底）
   → ③ 后台 wren memory index（emptyDir 首启重建 ~15min：HF 下 embedding 模型 + LanceDB 索引 2347 模型）
   → ④ exec python3 wren_server.py（Flask :8910）
```

### 2.2 请求链路总图

```mermaid
flowchart TB
    UI[datastat wren-console 页<br/>PR #597]
    APIM[APIMagic .ms 转发<br/>/api-gpt/wren/exec|ask<br/>PR #240]
    SRV["wren_server.py (Flask :8910)<br/>/wren/exec /wren/ask /wren/chat<br/>/wren/agent /wren/remember /wren/feedback"]

    subgraph WRENPROJ["/wren（Wren 项目，镜像内置）"]
        MDLY[models/*/metadata.yml]
        RULES[knowledge/rules/*.md]
        SQLMD[knowledge/sql/*.md]
        CCSQL[knowledge/column-comments/*.sql]
        MDLJ[target/mdl.json]
        LDB[.wren/ LanceDB 向量索引<br/>emptyDir, entrypoint 后台建]
    end

    subgraph WRENCLI["wren CLI 子进程（run_wren, cwd=/wren）"]
        QRY["wren query --sql"]
        FETCH["wren memory fetch"]
        RECALL["wren memory recall"]
    end

    GLM["GLM (open.bigmodel.cn)<br/>glm-5.3 / 5.3-flash / 4-flash 三级降级<br/>编排=coding/paas/v4; Agent=api/anthropic"]
    AGENT["claude -p（/wren-agent, uid1000）<br/>.mcp.json → wren serve mcp --project /wren"]
    PG[("PostgreSQL onedata<br/>只读账号 wren_ro")]
    GHPAGES["github om-nlquery<br/>wren-memory 分支长期 PR"]

    UI -->|SSE 直连| SRV
    UI -->|同步转发| APIM --> SRV

    %% 编排模式（/wren/ask 单轮、/wren/chat 多轮）——服务端自己当编排器
    SRV -->|"Step1 规则筛选<br/>load_relevant_rules()"| RULES
    SRV -->|"Step1.5a 召回已采纳口径"| RECALL
    RECALL --> SQLMD
    SRV -->|"Step2 向量找表"| FETCH --> LDB
    SRV -. "召回弱时兜底：<br/>token 扫 models/ 目录" .-> MDLY
    SRV -->|"Step3 生成SQL / Step5 组织答案<br/>（流式 SSE）"| GLM
    SRV -->|"Step4 受控执行<br/>SELECT 白名单校验后"| QRY
    QRY --> MDLJ
    QRY --> PG
    SRV -->|"结果列→列注释<br/>resolve_sql_column_notes()"| CCSQL

    %% Agent 模式（/wren/agent）——LLM 自己当编排器
    SRV -->|拉起子进程| AGENT
    AGENT <-->|"MCP stdio 工具<br/>list_models/run_sql/…"| WRENCLI
    AGENT --> GLM
    AGENT -->|"答案里的 SQL 抽出<br/>受控重跑拿 rows"| SRV

    %% 采纳/反馈闭环
    SRV -->|"/wren/remember 采纳：<br/>写 md + 推 wren-memory 分支开 PR"| SQLMD
    SRV --> GHPAGES
    SRV -->|"/wren/feedback 反馈"| FB[knowledge/feedback/feedback.jsonl]
```

### 2.3 编排模式五步（`/wren/ask` 单轮、`/wren/chat` 多轮，SSE 流式）

`wren_server.py` 官方 usage skill 的工程化实现，**编排逻辑在服务端 Python 里，不在 LLM 手里**：

| 步 | 做什么 | 数据源 | 工程增强（相对官方 skill） |
|---|---|---|---|
| ① | 加载口径规则 | `knowledge/rules/*.md` | `load_relevant_rules()` 按**问题相关性**筛选（官方 `context instructions` 会把无关口径全灌进 prompt）；中文关键词→文件映射兜底 |
| 1.5 | 召回已采纳口径（仅多轮 `/wren/chat`） | `wren memory recall` + `knowledge/sql/` | 用户采纳过的问法→SQL 优先复用；另有**意图识别**：解释类问题（"XX是什么意思"）直接依据口径知识回答，不查库 |
| ② | 检索相关表 | `wren memory fetch`（LanceDB 向量） | 命中表**读 models/<t>/metadata.yml 补完整列清单**（防 LLM 臆造列名）+ 问题英文 token 与表名子串匹配兜底 |
| ③ | LLM 生成 SQL | GLM glm-5.3 流式 | prompt 铁律"只能用清单里列出的表和列"；`sql_head_ok()` SELECT/WITH/EXPLAIN 白名单二次校验 |
| ④ | 受控执行 | `wren query --sql -l 20 -o json` | **执行报错自动带错误信息让 LLM 修正重试 1 次**（常见：臆造列名/枚举值写错） |
| ⑤ | LLM 组织中文答案 | GLM 流式 | 固定结构（结论/口径说明/聚合规则/分析，事实与推测分标）；多轮模式说明与上轮差异 |

最后把结果列映射回中文列注释（`resolve_sql_column_notes`：解析 SQL 里的表名 + 聚合前缀拆解 + 查
column-comments），前端表格直接展示指标解释。

### 2.4 Agent 模式（`/wren/agent`）

与编排模式的本质区别：**编排器从服务端 Python 换成了 claude code 本身**。

```
/wren/agent {"question", "history"}
  → wren_server 拉起子进程：claude -p <prompt> --output-format stream-json
    （cwd=/wren-agent，env: ANTHROPIC_BASE_URL=api/anthropic + ANTHROPIC_AUTH_TOKEN=GLM key）
    /wren-agent/CLAUDE.md    → agent 行为准则（口径要点：scenario 枚举 user/developer、
                               is_current=true、解释类不查数、DISTINCT 查枚举）
    /wren-agent/.mcp.json    → mcpServers.wren = wren serve mcp --project /wren
  → claude 经 MCP stdio 自主用 wren 工具（检索 schema/写 SQL/执行/看报错自我修正）
  → wren_server 解析 stream-json 转成 SSE（tool_use 展示命令、15s 心跳、180s 空闲/900s 总超时）
  → 从最终答案抽 ```sql``` 块，若通过白名单则 wren query 受控重跑 → 结构化 rows 给前端
```

这就是 `wren serve mcp` 在 om-nlquery 里的**唯一**用武之地：不在用户手里，而是作为容器内
claude agent 的工具箱（`deploy/README.md` §6）。

### 2.5 采纳/反馈闭环（口径沉淀）

```
用户点「采纳」→ POST /wren/remember {question, sql}
  → _write_memory_md()：直接写 /wren/knowledge/sql/<主题词>-<时间戳>.md
    （frontmatter: nl/sql/source:user——不调 wren memory store，避免 HuggingFace 下载失败）
  → 后台线程 sync_memory_to_repo()：
      独立克隆（/wren-data/memory-clone，不碰 /wren 工作区）
      fetch main → checkout -B wren-memory → 拷 knowledge/sql/*.md → commit
      → force push wren-memory 分支（每次基于最新 main 重建，无并行写）
      → 调 GitHub API 开/复用长期 PR（合入由人审核，绝不自动合）
  → PR 合入 → 镜像重建部署 → 所有用户的 recall 共享这些已确认口径

用户点「对/错」→ POST /wren/feedback → append knowledge/feedback/feedback.jsonl（口径优化素材）
```

## 3. 安全模型（多层纵深）

```
LLM 生成的 SQL
  ① wren_server.sql_head_ok()    头部白名单：只允许 SELECT/WITH/EXPLAIN
  ② wren query 引擎              MDL 语义层校验（列名必须存在于 models 定义）
  ③ PG 账号层                    wren_ro 只读账号（GRANT SELECT ONLY）
  ④ wren CLI 查询限流            -l 20（MAX_ROWS），agent MCP 侧还有 1000 默认/10000 硬上限
密钥流：
  PG 密码/GLM key/clone token     全部 k8s Secret → env / 挂载文件；镜像与仓零真值
  entrypoint 的 conn.yml          用后即 rm；GLM key chmod 600
  wren profile（容器内）          profiles.yml 由 --from-file 生成（镜像里没有）
```

## 4. 与官方 wren 流程的差异清单（踩坑沉淀）

1. **models 反向生成**：官方 flow 是 init→手写/generate；om-nlquery 用 `gen_mdl.py` 从真库
   information_schema 批量产 2347 个模型，人工只维护 knowledge（规则/注释/例句）。
2. **memory store 绕开**：`wren memory store` 触发 HuggingFace 下载在 pod 里失败 → 改为直接写
   md 文件（frontmatter 兼容），向量索引由维护者合 PR 后服务升级时统一重建。
3. **HF_ENDPOINT 陷阱**：不能设 hf-mirror（308 重定向缺元数据头报 FileMetadataError），
   pod 直连 huggingface.co 才正常。
4. **GLM 通道区分**：Coding Plan key 走普通 paas 端点会 429 code=1113「余额不足」（不是真没钱）；
   编排模式用 coding/paas/v4，Agent 模式用 api/anthropic；三级模型降级（5.3→5.3-flash→4-flash）。
5. **LanceDB 索引在 emptyDir**：pod 重建即丢，entrypoint 后台重建 ~15 分钟（要免重建换 PVC）。
6. **Agent 容器必须非 root**（claude `--dangerously-skip-permissions` 拒绝 root），Dockerfile
   两阶段构建、运行态 uid 1000。
7. **SSE 经 Ingress 必须关 buffering**（`proxy-buffering: off` + 900s 超时），否则流式变攒包。

## 5. 文件读写速查（运行期，容器内 /wren）

| 操作 | 读 | 写 |
|---|---|---|
| `/wren/exec` sql 模式 | target/mdl.json（经 wren query） | — |
| `/wren/exec` fetch 模式 | .wren/ LanceDB；兜底扫 models/*/metadata.yml | — |
| `/wren/ask` / `/wren/chat` | rules/*.md、models/*/metadata.yml、LanceDB、column-comments/*.sql | — |
| `/wren/agent` | 同上（由 claude 经 MCP 间接） | — |
| `/wren/remember` | — | knowledge/sql/*.md；独立克隆 → wren-memory 分支 → PR |
| `/wren/feedback` | — | knowledge/feedback/feedback.jsonl（append） |
| entrypoint 启动 | env（Secret） | ~/.wren/profiles.yml、/tmp/.glmkey、.wren/ 索引 |
