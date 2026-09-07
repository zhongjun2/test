# Wren AI 链路与集成调研

> 调研对象：`wrenai` CLI 0.13.4（`pip install wrenai`，Python 3.14，Windows 本机实测）
> 核心问题：Wren 的项目、配置和外部集成是如何组织的 —— 从 `wren context init` 到 `wren serve mcp`。
> 分析方法：直接读本机 site-packages 源码（`C:\Python314\Lib\site-packages\wren\`）并实跑 CLI help 验证。

## 0. 总览：三层结构

Wren CLI 把所有状态分成**三层文件**，各层职责单一：

| 层 | 位置 | 内容 | 谁写 |
|----|------|------|------|
| ① 项目层（MDL 语义层） | `<project>/`（含 `wren_project.yml`） | models / views / cubes / relationships / knowledge | 用户或 AI agent 编辑（YAML 源文件） |
| ② 编译产物层 | `<project>/target/mdl.json` | 单一 camelCase JSON manifest，喂给 WrenEngine（Rust 内核 wren-core） | `wren context build` 自动生成 |
| ③ 全局配置层 | `~/.wren/`（`WREN_HOME` 可覆盖） | `profiles.yml`（连接配置）、`config.yml`（default_project 等全局偏好）、`.env`（秘密） | `wren profile` 系列命令自动写 |

关键设计：**YAML 是人/AI 编辑的源，`target/mdl.json` 是机器编译产物，连接配置全局共享、项目按名引用。**

---

## 1. 项目生命周期

### 1.1 `wren context init` 创建了哪些文件

源码：`wren/context_cli.py` 的 `init()`（脚手架分支，无 --from-mdl/--from-osi 时）。

在项目目录创建：

```
<project>/
├── wren_project.yml        # 项目元数据：schema_version(=5)/name/version/catalog/schema/data_source
├── relationships.yml       # 空的 join 定义（带注释示例）
├── models/
│   └── example/metadata.yml     # 示例模型（--empty 则跳过）
├── views/
│   └── example_view/
│       ├── metadata.yml         # 示例视图
│       └── sql.yml              # 视图 SQL（statement 字段，多行时独立成文件）
├── cubes/                        # 空目录
├── knowledge/                    # v5 新增的一等"业务知识"骨架
│   ├── knowledge.yml            # schema_version: 1
│   ├── rules/general.md         # 业务规则（LLM 查询生成的约束）
│   ├── glossary/ metrics/ caveats/ sql/   # 各带 .gitkeep
│   └── sql/                     # 确认过的 NL→SQL 对（`wren memory store` 写入）
└── AGENTS.md                    # 给 AI agent 的工作流指引（查数据/改模型的标准步骤）
```

要点：
- `catalog: wren` / `schema: public` 是 **Wren Engine 自己的命名空间**，不是数据库的 catalog/schema；数据库真实位置写在每个 model 的 `table_reference` 里（init 生成的 yml 里有注释专门澄清这一点，是个易混点）。
- `schema_version: 5` 是当前版本；`wren context upgrade` 可做 v1→v2（平文件→目录式）、v4→v5（补 knowledge 骨架）等迁移（`wren/context.py` 的 `plan_upgrade/apply_upgrade`）。
- `--from-mdl <file>` 分支：读入 camelCase MDL JSON（`convert_mdl_to_project()`），反向拆成上面的 YAML 布局。
- 项目发现优先级（`discover_project_path()`）：`--path` 参数 > `WREN_PROJECT_HOME` 环境变量 > 从 cwd 向上找 `wren_project.yml` > `~/.wren/config.yml` 的 `default_project`。

### 1.2 `wren context build` 读什么、产出什么

源码：`context_cli.py:594` → `wren/context.py` 的 `build_manifest()/build_json()/save_target()`。

**读**（全部 snake_case YAML）：

| 输入 | loader | 说明 |
|------|--------|------|
| `wren_project.yml` | `load_project_config` | catalog/schema/data_source（进 manifest 顶层） |
| `models/<name>/metadata.yml`（+ 可选 `ref_sql.sql`） | `load_models` | ref_sql.sql 存在时**优先于** metadata.yml 内联的 ref_sql |
| `views/<name>/metadata.yml`（+ 可选 `sql.yml`） | `load_views` | sql.yml 的 statement 同样优先 |
| `cubes/<name>/metadata.yml` | `load_cubes` | 度量/维度/时间维度 |
| `relationships.yml` | `load_relationships` | 模型间 join |

**产出**：`<project>/target/mdl.json` —— `save_target()` 写入。内容 = 上述全部合并成一个 manifest，再经 `_convert_keys()` 把所有 key 从 snake_case 转 camelCase（`table_reference→tableReference` 等），并盖上 `layoutVersion`（schema_version 5 → layoutVersion 3）。**注意 knowledge/ 不进 mdl.json** —— rules 由 `load_rules()` 在运行时单独读取。

**验证**：`wren context validate` 做结构校验（模型必须有 name/columns、table_reference 与 ref_sql 二选一、relationship 引用的模型必须存在、cube 的 base_object 必须是已定义 model/view 等，`validate_project()`）；build 默认 `--validate` 先验后编译。

### 1.3 `target/mdl.json` 在系统中的角色

它是**YAML 源码与 Rust 引擎（wren-core）之间唯一的接口**：

```
YAML 源(models/ views/ ... )  ──build──▶  target/mdl.json  ──加载──▶  WrenEngine
                                              │
                     ┌────────────────────────┼────────────────────────┐
                     ▼                        ▼                        ▼
              wren --sql / query        wren serve mcp            wren cube query
              （CLI 查询，_build_engine    （MCP 服务，serve_cli     （结构化度量查询，
               读 mdl.json 构引擎）         同样读 mdl.json）          现场重新 build_json）
```

- CLI 的 `--mdl` 选项默认就是 `<project>/target/mdl.json`。
- `wren serve mcp` 启动时硬性要求它存在，否则直接报错提示 "run `wren context build` first"；且会比较源文件与 mdl.json 的 mtime，**源更新但没重新 build 时打 stale 警告**（`serve_cli.py` 的 `_mdl_is_stale()`，监控 `wren_project.yml`、`relationships.yml`、`models/ views/ cubes/` 三个目录）。
- 有趣的例外：MCP 的 `get_mdl`/`query_cube` 等 tool 是**现场从 YAML 重新 build**（`build_json(ctx.project)`）而不是读 target/mdl.json —— 即 MCP 的 schema 工具对"忘了 rebuild"有一定容错，但查询引擎仍用启动时加载的旧 manifest。

---

## 2. 配置管理（profile 体系）

### 2.1 存储机制

唯一存储位置：`~/.wren/profiles.yml`（`WREN_HOME` 环境变量可改到别处），结构：

```yaml
active: onedata          # 全局当前生效的 profile
profiles:
  onedata:
    datasource: postgres
    host: ${PG_HOST}     # 支持 ${VAR} 占位，连接时才从 env/.env 解析
    port: "5432"
    user: ${PG_USER}
    password: ${PG_PW}
```

安全设计（`wren/profile.py`）：
- 写入用**临时文件 + 原子替换**，权限 `0600`（`_save_raw()`）。
- `${VAR}` 只匹配 UPPER_SNAKE_CASE（`_SecretTemplate`，自定义 idpattern + 关 VERBOSE 关掉 IGNORECASE），小写 `${foo}` 一律当字面量——避免误展开真实密码里恰好出现的花括号序列。
- **占位符永远不落盘替换**：`expand_profile_secrets()` 只在连接瞬间解析（来源顺序：`$CWD/.env` → 项目根 `.env` → `~/.wren/.env`，已 export 的 shell 环境变量优先），解析不到就 fail-loud 抛 `MissingSecretError`。
- `wren profile debug` 输出前用字段注册表（`model/field_registry.py` 标记 SecretStr 的字段）+ 名称启发式双重打码，保证不泄露。

### 2.2 `wren profile add` 的四种方式

源码：`wren/profile_cli.py` 的 `add()`（三选一互斥，另有第四种 inline 模式）：

| 方式 | 机制 | 实现 |
|------|------|------|
| `--ui` | 起本地 starlette/uvicorn 服务（自动选端口）自动开浏览器，渲染 `templates/profile_form.html`；选 datasource 后异步拉 `/fields` 局部渲染 `_profile_fields.html`；POST `/save` → `add_profile()` 落盘 → 服务自停 | `wren/profile_web.py` |
| `--interactive` | 终端逐字段问答，字段清单来自**共享字段注册表** `field_registry.get_fields(ds)`：敏感字段 hide_input、文件字段（证书等）读文件 base64、隐藏字段注入默认值 | `profile_cli.py` 的 `_interactive_add()` |
| `--from-file` | 读 JSON/YAML 连接文件，只接受两种 shape：flat（`{datasource, host, ...}`）或 `{datasource, properties: {...}}` 信封；其它嵌套结构直接拒绝 | `_flatten_connection_envelope()` |
| （inline）`--datasource postgres` | 只写 `{datasource: ...}` 最小 profile，提示手工编辑 profiles.yml | — |

保存后默认**自动做连接验证**（`_validate_connection()`：把 profile 转 typed ConnectionInfo → get_connector → `SELECT 1`）；失败仅警告不删除，提示用 `--ui` 重编辑。

另有 `wren profile import dbt --project-dir ...`：读 dbt 的 profiles.yml + dbt_project.yml，把 dbt target 直接转成 wren profile（见 §3）。

### 2.3 `wren context set-profile` 如何关联项目与配置

源码：`context_cli.py:801`。做的事很薄：

1. 校验 profile 存在于 `~/.wren/profiles.yml` 且有 `datasource` 字段；
2. 往项目的 **`wren_project.yml` 写两行**：`profile: <name>` + `data_source: <该profile的datasource>`（`save_project_config()` 按固定字段序重写整个文件）；
3. 若 data_source 变了且已有 target/mdl.json，警告旧 MDL 是按旧方言编译的，需重新 build。

**运行时解析顺序**（`resolve_profile_for_project()`，`wren/profile.py:220`）：

```
wren_project.yml 的 profile: 字段（项目级 pin，优先）
  └─ 不存在 → 报错 fail-loud（不许静默回退，防止连错库）
没有 pin → ~/.wren/profiles.yml 的 active（全局默认）
```

即：**项目 pin > 全局 active**；`wren profile switch` 只改全局 active，不影响已 pin 的项目。

---

## 3. 外部集成

### 3.1 `wren context import dbt`：manifest/catalog → MDL 映射

源码：`wren/dbt.py` 的 `convert_dbt_project_to_wren_project()`；CLI 入口 `context_cli.py` 的 `import_cmd()`。

**输入要求**（`load_dbt_artifacts()`，`dbt.py:261`）：

| 文件 | 必需 | 用途 |
|------|------|------|
| `<dbt>/dbt_project.yml` | ✅ | 项目名、profile 名、target-path |
| dbt `profiles.yml`（`--profiles-path` / `DBT_PROFILES_DIR` / `~/.dbt/`） | ✅ | 解析 target → 数据源类型（`env_var()` 引用会解析） |
| `<dbt>/target/manifest.json` | ✅ | 模型/源的结构与列定义（逻辑层） |
| `<dbt>/target/catalog.json` | ✅ | **数据库真实列**（物理层；缺失时提示先 `dbt docs generate`） |
| `<dbt>/target/run_results.json` | 可选 | dbt test 的 pass/fail 状态 |
| `<dbt>/target/compiled/**/*.sql` | 可选 | 编译后 SQL |

**映射规则**：

- **模型**：manifest 的 `nodes[resource_type=model]`（跳过 ephemeral）→ `models/<name>/metadata.yml`；`table_reference` 取 node 的 database/schema/identifier。**列以 catalog.json 为准**（catalog 有列时 manifest-only 列被忽略），类型经 `parse_type()` 按 dbt adapter 方言（doris→mysql、sqlserver→tsql 特判）转成 MDL 类型。
- **源（sources）**：→ 模型名加 `raw_` 前缀（`raw_<name>`，重名时 `raw_<source_name>_<name>`）。
- **关系**：从 dbt **test** 推导 —— `relationships` 测试（`_apply_dbt_test_enrichment()`）转成 `relationships.yml` 条目，join_type 按 dbt 层次推断（fct_→dim_ 为 MANY_TO_ONE 等）；`not_null`/`unique` 测试转成列的 `not_null`/`is_primary_key` 并推导模型 `primary_key`；`accepted_values` 进列 properties。**特意不**把 relationship 盖到列上（避免引擎把它当 join 解引用隐藏直查能力，源码有注释）。
- **分层标注**：`infer_dbt_layer()` 按 fqn/命名前缀（stg_/int_/fct_/dim_）标 `dbt_layer: staging/intermediate/mart/raw`。
- **知识生成**：写 `knowledge/rules/general.md`（含"已验证约束/关系/数据质量警告"三节，状态来自 run_results）+ 为每个模型生成种子 NL→SQL 对写入 `knowledge/sql/<slug>.md`。
- **绑定**：`wren_project.yml` 里写 `dbt: {project_dir, profile, target}` 记录来源，`schema_version: 5`。
- **profile**：配套 `wren profile import dbt` 把 dbt target 的 host/user/password 等转成 wren profile（BigQuery 的 keyfile 会被 base64 成 credentials）。

**产出**（`write_project_files()`，支持 `--dry-run` 预览、`--force` 覆盖托管路径）：一个完整的标准 wren 项目（models/*/、relationships.yml、knowledge/、wren_project.yml、AGENTS.md）。之后的生命周期与手写项目完全一致（validate → build → 查询/serve）。

### 3.2 `wren serve mcp`：暴露什么、怎么调

源码：`wren/serve_cli.py`（启动）+ `wren/mcp_server.py`（FastMCP 服务定义）。

**启动流程**（`serve_mcp()`）：

1. `discover_project_path()` 定位项目（或 `--project` 显式指定）；
2. 要求 `target/mdl.json` 存在（否则报错）；源文件比它新则打 stale 警告；
3. `--profile` 指定连接（不指定则用项目 pin/全局 active）；`${VAR}` 此刻解析；
4. `_build_engine()` 用 mdl.json + connection_info 构建进程内 **WrenEngine**（不依赖外部 ibis-server/HTTP 引擎）；
5. `run_server()` 按 transport 运行：`stdio`（默认，MCP 客户端拉起本进程）或 `http`（streamable-http，默认 127.0.0.1:8080/mcp）。
6. 启动时向 **stderr** 打印各客户端注册命令（`claude mcp add wren -- wren serve mcp --project <p>` / Cursor 等 IDE 的 `mcpServers` JSON），stdio 模式下 stdout 是协议通道绝不污染。

**暴露面**（`mcp_server.py`，FastMCP "wren"）：

*Tools（17 个，按 flag 条件注册）*：

| 分组 | tool | 作用 | 条件 |
|------|------|------|------|
| 查询 | `run_sql` | 经语义层执行 SQL（限 MDL 模型名；默认 1000 行、硬上限 10000、N+1 探测截断） | 非 `--no-connect` |
| | `dry_run` | 只验不查 | 非 `--no-connect` |
| | `query_cube` | 结构化度量查询（measures/dimensions/filters/time_dimension；`sql_only` 只出 SQL） | 非 `--no-connect` |
| | `dry_plan` | MDL 展开成目标方言 SQL，不连库 | 恒有 |
| context | `get_mdl` / `list_models` / `describe_model` / `get_data_source` | schema 检视（现场重 build YAML） | 恒有 |
| | `list_cubes` / `describe_cube` / `list_functions` | cube 与可用 SQL 函数 | 恒有 |
| knowledge | `get_instructions` / `describe_schema` / `get_context` / `recall_queries` / `list_stored_queries` / `list_knowledge` | 业务规则、全文/语义 schema 检索、NL→SQL 例句召回（memory extra 时 LanceDB 语义检索，否则 token 重叠/全文兜底） | 恒有 |
| 写 | `store_query` | 确认的 NL→SQL 对写 `knowledge/sql/`（markdown 真源）+ LanceDB 索引 | `--allow-write` |

*Resources（5 个）*：`wren://mdl`（JSON）、`wren://instructions`、`wren://project`、`wren://agents`（AGENTS.md）、`wren://knowledge/{name}` 与 `wren://knowledge/{subdir}/{name}`（模板资源，路径逃逸校验，只到一层子目录）。

*Prompts（1 个）*：`wren_workflow` —— 给客户端的 SOP 提示词（读 mdl → get_instructions → recall_queries → dry_run → run_sql/query_cube → store_query），步骤随 `--no-connect`/`--allow-write` 动态增减。

**调用流程（Claude/Cursor 视角）**：

```
客户端注册（claude mcp add / mcpServers 配置）
  → 客户端按需 spawn `wren serve mcp --project …`（stdio）或连 http://host:port/mcp
  → MCP initialize/tools list
  → 用户提问 → 模型按 wren_workflow prompt 依次调：
      list_models/describe_model 或读 wren://mdl   （懂 schema）
      get_instructions + recall_queries            （业务规则 + 例句）
      dry_run → run_sql（或 query_cube）            （验证 → 执行）
      store_query（若 --allow-write 且答案已确认）   （沉淀）
  → 每次查询经 WrenEngine：MDL 名 → 展开/转译目标方言 SQL → 连接器下发数据库
```

---

## 4. 完整文件流转图（init → serve mcp）

```mermaid
flowchart TB
    subgraph GLOBAL["~/.wren/ 全局层 (WREN_HOME)"]
        PROFILES[profiles.yml<br/>active + 各连接 profile<br/>${VAR} 占位不落真值]
        CFG[config.yml<br/>default_project]
        DOTENV_W[.env<br/>秘密注入]
    end

    subgraph DBT["dbt 项目 (可选输入)"]
        DBTP[dbt_project.yml]
        DBTPF[(dbt profiles.yml)]
        MANIFEST[(target/manifest.json)]
        CATALOG[(target/catalog.json)]
        RUNRES[(run_results.json 可选)]
    end

    subgraph PROJECT["<project>/ 项目层 (YAML 源)"]
        WP[wren_project.yml<br/>name/catalog/schema/data_source/profile pin]
        MODELS[models/*/metadata.yml<br/>+ ref_sql.sql]
        VIEWS[views/*/metadata.yml<br/>+ sql.yml]
        CUBES[cubes/*/metadata.yml]
        REL[relationships.yml]
        KNOW[knowledge/<br/>rules/*.md · sql/*.md · knowledge.yml]
        AGENTS[AGENTS.md]
        DOTENV_P[.env 可选]
    end

    subgraph TARGET["编译产物"]
        MDL[target/mdl.json<br/>camelCase manifest + layoutVersion]
    end

    subgraph RUNTIME["运行时"]
        ENGINE[WrenEngine<br/>wren-core Rust 内核]
        MCP[wren serve mcp<br/>FastMCP server]
        CLIENT[Claude / Cursor / Claude Code]
        DB[(目标数据库<br/>postgres 等 20+)]
    end

    %% init
    INIT["wren context init [--empty|--from-mdl]"] -. "脚手架写入" .-> WP
    INIT -.-> MODELS & VIEWS & REL & KNOW & AGENTS

    %% dbt import
    DBTP & DBTPF & MANIFEST & CATALOG & RUNRES --> IMP["wren context import dbt<br/>+ wren profile import dbt"]
    IMP -- "生成项目文件" --> WP & MODELS & REL & KNOW
    IMP -- "dbt target → 连接配置" --> PROFILES

    %% profile
    PADD["wren profile add<br/>--ui / --interactive / --from-file"] -- "原子写 0600" --> PROFILES
    SP["wren context set-profile &lt;name&gt;"] -- "写 profile: + data_source: 两行" --> WP

    %% build
    WP & MODELS & VIEWS & CUBES & REL -- "wren context build<br/>(validate 先行)" --> MDL

    %% serve
    MDL -- "启动加载(缺了报错, 旧了警告)" --> SERVE["wren serve mcp<br/>--profile --allow-write --no-connect"]
    PROFILES -. "pin>active 解析 ${VAR}" .-> SERVE
    DOTENV_W & DOTENV_P -. "env 注入" .-> SERVE
    SERVE --> ENGINE
    MCP --> ENGINE
    SERVE === MCP
    MCP <--> |"stdio / streamable-http<br/>17 tools · 5 resources · 1 prompt"| CLIENT
    KNOW <.-> |"get_instructions / recall_queries<br/>wren://knowledge/*"| MCP
    ENGINE -- "MDL 展开→方言 SQL" --> DB

    %% CLI 查询旁路
    MDL --> Q["wren --sql / query / cube query"]
    Q --> ENGINE
```

**读/写速查表**：

| 命令 | 读 | 写 |
|------|-----|-----|
| `wren context init` | （--from-mdl 时：MDL JSON） | wren_project.yml、relationships.yml、models/example/、views/example_view/、knowledge/ 骨架、AGENTS.md |
| `wren context build` | wren_project.yml、models/、views/、cubes/、relationships.yml | target/mdl.json |
| `wren context validate` | 同上（不写） | — |
| `wren context set-profile` | ~/.wren/profiles.yml | wren_project.yml（profile + data_source 两行） |
| `wren context import dbt` | dbt_project.yml、dbt profiles.yml、target/{manifest,catalog}.json、run_results.json（可选） | 整套 wren 项目（models/、relationships.yml、knowledge/、wren_project.yml、AGENTS.md） |
| `wren profile add`（三模式） | --from-file 的 JSON/YAML；.env | ~/.wren/profiles.yml（原子 0600） |
| `wren profile import dbt` | dbt_project.yml + dbt profiles.yml | ~/.wren/profiles.yml |
| `wren serve mcp` | target/mdl.json（必需）、wren_project.yml（pin）、profiles.yml、.env、knowledge/（tool 现场读）、YAML 源（get_mdl 等现场重 build） | knowledge/sql/*.md（store_query，需 --allow-write） |

## 5. 值得借鉴的设计点

1. **YAML 源 ↔ 编译产物分离**：人/AI 只碰 YAML；引擎只认单一 mdl.json；stale 检测（mtime 比对）兜底忘记 rebuild。
2. **连接配置与项目解耦**：profile 全局一份、项目只 pin 名字，切换环境 = 换 pin 或 switch active；pin 失效 fail-loud 不静默回退。
3. **秘密零落盘**：profiles.yml 只存 `${VAR}` 占位，连接瞬间才从 .env/env 解析，debug 输出双路打码 —— 与我们"秘密只走环境变量"的铁规同构。
4. **知识即文件**：业务规则（rules/*.md）和 NL→SQL 例句（sql/*.md）都是项目内 markdown，可 git、可 review，LanceDB 只是索引副本。
5. **dbt 集成是"一次性编译"而非"运行时依赖"**：import 生成标准 wren 项目后，dbt 侧后续变化不会自动同步（只在 wren_project.yml 留了 dbt 绑定记录）—— 选用时要意识到这是快照式导入。
