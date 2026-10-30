# workbuddy-library —— workbuddy资料库 Skill

WorkBuddy 原生「资料库」模块的 Agent Skill 源码副本，来自 WorkBuddy 内置插件
`workbuddy-builtin/skills/library`（version 0.5.16，author csig-x2）。
本副本重命名为 `workbuddy-library`（技能 name：`workbuddy资料库`），以区分上游与本地变体。

资料库是 WorkBuddy 的内容管理、分享协作与轻发布模块。本 skill 覆盖：

- **doc** — 在线文档：读取（含审阅标记）、审阅式编辑、直接编辑、新建
- **database** — 结构化数据表：建表、字段增删改、记录增删改查、CSV 导入
- **page** — 数据/汇报页面：HTML/ZIP 导入、事务式编辑、发布/下线、Page↔Database 关联、md→html
- **manage** — 空间管理：目录树、节点移动/重命名、全库检索、图片上传、节点评论
- **drive** — 网盘文件：上传/覆盖、下载链接
- **link** — 网页剪藏：收藏 http(s) 网页、读取剪藏正文
- **smh** — 智能媒体库：历史媒体节点下载
- **attachment** — 节点附件：上传/下载

## 目录地图

| 路径 | 说明 |
|---|---|
| `SKILL.md` | 总纲：路由规则、鉴权流程、响应约定、安全底线 |
| `manifest.yaml` | CSIG 规范技能清单（版本、模块、能力声明） |
| `runtime_context.py` | 运行模式探测：`mode=sandbox`（auth-proxy 注入身份）/ `mode=client`（connect_open_platform 换票） |
| `space_api.py` | 通用 API 直调入口，运行时读取 `api-manifest.json`；支持 `--help` / `--raw` / `--token-stdin` |
| `_common.py` | 共享工具层：token 读取、HTTP 调用、脱敏、错误退出（业务脚本禁止绕过） |
| `api-manifest.json` | 全量 API 清单（name/method/params 声明） |
| `mutation.md` | 写操作前置规则：目标空间分类、何时停下确认 |
| `error_handling.md` | 错误码行动表（全技能唯一错误映射，模块文档只引用不复制） |
| `<module>/entry.md` | 各模块入口文档：场景判定 → 脚本调用 → 回执格式 |
| `doc/scripts/examples/` | doc 编辑 payload 示例集 + `verify_examples.py` 校验脚本 |
| `doc/scripts/fixtures/` | table_edit_helper 测试用例 |
| `.codebuddy-plugin/` | WorkBuddy 插件安装元数据（bundle 信息） |

### 各模块脚本清单

| 模块 | 脚本 |
|---|---|
| doc | `get_doc_reviews.py`、`submit_review_edit.py`、`submit_doc_edit.py`、`create_doc.py`、`content_validator.py`、`table_edit_helper.py` |
| database | `create_database.py`、`get_database_schema.py`、`get_database_content.py`、`add/update/delete_database_field.py`、`batch_add/update/delete_database_records.py`、`get_database_record.py`、`query_database_record.py`、`import_csv.py`、`_db_types.py` |
| page | `parse_html.py`、`lint_schema.py`、`import_html.py`、`md_to_html.py`、`create_page_transaction.py`、`get_page_upload_url.py`、`commit_page_transaction.py`、`list_page_artifacts.py`、`list_page_publish_artifacts.py`、`download_page_artifacts.py`、`publish_page.py`、`unpublish_page.py`、`page_database_relation.py`、`remap_database_ids.py`、`lint_database_sdk_usage.py` |
| manage | `create_folder.py`、`rag_search.py`、`upload_image.py`、`get_node_comments.py` |
| drive | `upload_drive_file.py`、`get_download_link.py` |
| smh | `get_download_link.py` |
| attachment | `attachment.py` |

### 模块文档索引（entry.md 之外）

| 路径 | 内容 |
|---|---|
| `doc/edit_core.md` / `doc/workflows.md` / `doc/content_contract.md` | doc 编辑核心规则、工作流、内容契约 |
| `doc/action_decision.md` / `doc/batch_replace_sop.md` / `doc/revision_flows.md` / `doc/server_pitfalls.md` | 动作决策、批量替换 SOP、审阅流、服务端陷阱 |
| `doc/tasks/*.md` | 分任务指南：read_create / text_block_edit / complex_edit / table_edit / revision |
| `doc/error_handling.md` | doc 模块错误分流（最终仍按根 `error_handling.md` 行动表执行） |
| `database/params-reference.md` / `database/csv-import-flow.md` | database 参数参考、CSV 导入流程 |
| `page/import-flow.md` / `page/edit-flow.md` / `page/data-page-flow.md` / `page/clone-flow.md` / `page/md-to-html-flow.md` / `page/beautify-flow.md` | page 各流程指南 |
| `page/html-parse-spec.md` / `page/database-sdk-contract.md` / `page/wbp-presentation-contract.md` | page 解析规格、SDK 契约、演示契约 |
| `page/beautify/*.md` | 美化指南与视觉技巧 |

## 运行模式与鉴权（速览）

| 模式 | 鉴权 | 调用形态 |
|---|---|---|
| `sandbox` | auth-proxy 注入身份，不传 token | `python3 "$SKILL_DIR/<script>.py" [args]` |
| `client` | `ToolSearch` → `DeferExecuteTool`（`connect_open_platform`）取 token，stdin 首行传入 | `printf '%s' "<token>" \| python3 "$SKILL_DIR/<script>.py" --token-stdin [args]` |

脚本输出约定：成功 → stdout 结构化文本（`KS_*` 行 / JSON / XML）；失败 → stdout 单行 `{"error":"<脱敏错误>"}` 后 `exit 0`。

## 移植说明（布局约束）

从 WorkBuddy 安装目录复制到本仓库时**未改动目录层级**，原因：

1. **模块脚本的自定位依赖**：所有模块脚本以
   `Path(__file__).resolve().parent.parent` 把技能根目录加入 `sys.path` 来导入
   `_common.py`。若把模块挪到 `modules/` 等更深层级，导入即断。
2. **文档相对引用**：约 25 处模块文档用 `../mutation.md`、`../error_handling.md`、
   `../../mutation.md` 相对引用根级文档；移动将导致全部断链。
3. **根级脚本自定位**：`space_api.py` / `runtime_context.py` 以
   `Path(__file__).parent` 为 ROOT，`api-manifest.json` 必须与之同级。

本副本相对上游的两处适配（行为不变）：

- `SKILL.md`：新增 §0「技能根目录约定」——`${CODEBUDDY_SKILL_DIR}` 不可用时回退到
  SKILL.md 所在目录，使源码仓形态可直接使用；frontmatter 增加 `agent_created: true`。
- `SKILL.md`：章节重编号并补 §10 目录结构总览；全部路由规则、鉴权流程、错误处理、
  安全底线原文保留。

如需以用户技能形式安装：把 `workbuddy-library/` 整目录复制到 `~/.workbuddy/skills/` 即可（平台侧鉴权用的 `skill_id` 仍为上游注册的 `library`，无需改动）。
