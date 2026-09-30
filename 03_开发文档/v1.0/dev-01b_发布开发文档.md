# dev-01b 发布模块 — 开发文档（实现级设计）

> 版本 v2.3 · 2026-09-28 · 状态：待评审 · 负责人：开发主管 · 迭代版本：v1.0 建站
>
> 定位：**实现级设计**——发布模块（`publish.py` + 前端 API 对接）的端到端开发设计；需求侧见 `02a_需求总览.md` §2.3 发布模块（`FR-D1`-`FR-D7`）。
>
> 覆盖范围：发布源与选稿（FR-D1）/ 发布模式三档（FR-D2）/ 配置管理（FR-D3）/ 前端 API 对接（FR-D4）/ 状态推进与回写（FR-D5）/ 地域轮转（FR-D6）/ 人工否决（FR-D7）+ 单一状态机合并决策。
>
> 关联文档：`02a_需求总览.md`（§2.3 发布模块）· `03a_总体设计.md`（§4 数据模型、§7 Cron 调度、§13 配置）· `03b_文章模型.md`（字段与状态机）· `03c_前端API发布.md`（接口实现级设计）· `03d_文章发布流程.md`（流程视角）· `03e_翻译范围方案B.md`（FR-P11 稿源）· `03f_发布模式三档.md`（FR-D2 实现）

> v2.0 变更（2026-09-26）：**由「需求文档」重定位为「开发文档（实现级设计）」**，从 `02_产品需求/v1.0/prd01b_自动发布.md` 迁入 `03_开发文档/v1.0/`；原 §3 FR1-FR7 的**需求条目上移** `02a_需求总览.md` §2.3（编号 `FR-D1`-`FR-D7`），本文保留**实现级内容**（拍板记录 / 设计子问题 / 接口契约 / 错误矩阵 / 代码改动 / 测试方案），FR 编号仅作需求映射。

> ⚠️ **v2.1 K2 位置注（2026-09-28，改码不改变行为）**：本文所指发布域代码**物理位置已变化**（`dev-03a_core内核抽取.md`，v2.1 批次 1 首卡），**契约与行为一字未变**：
>
> | 本文原述位置 | 现位置 |
> |---|---|
> | `app/aggregator/publish.py`（发布逻辑 + 配置段 + CLI 混在一起） | `app/core/config.py`（配置段 + 路径常量）· `app/core/publisher.py`（发布域：`publish_one`/`select_pending`/`rotate_by_region`/`build_payload`…）· `app/aggregator/publish.py`（**仅剩 CLI 薄壳**：argparse + 循环） |
> | `app/aggregator/lib/db.py` | `app/core/store.py`（原文件已删除） |
> | `app/aggregator/lib/site_api.py` | `app/core/site_client.py`（原文件已删除） |
> | 脚本内 `sys.path.insert`（脚本范式） | **归零** —— 改为 `pyproject.toml` + `pip install -e .`（editable 安装，单一根 `.venv`） |
>
> 下文 §3/§7 等处的**历史代码位置描述按当时事实保留**（含 `lib/site_api.py`、`queue` 等已退役件），阅读时以上表为准。发布域**行为清单**（幂等 / slug 对账 / 状态回写 / 三档模式 / 地域轮转 / 人工否决）全部不变。

---

## 0. 拍板记录

| # | 事项 | 决策 | 来源 |
|---|---|---|---|
| 1 | 两套 status 合并 | ✅ **合并为单一系统**：只有 `data/pool/`，废弃 `data/queue/` 与 `translated` 状态 | Max 拍板（2026-09-24） |
| 2 | 单一状态机 | ✅ `new → tagged → pending → published / rejected` | 同上 |
| 3 | 起步发布模式 | ✅ `semi_auto` | Max 拍板 |
| 4 | 翻译范围 | ✅ 方案 B（真话题 + 高分单篇），每天 ≥10 篇入 `pending` | Max 拍板（见 `03e`） |
| 5 | 首版是否含图片 | ✅ 不含，纯文字先跑通（图片列 P1） | Max 拍板 |
| 6 | 撤回/下线接口 | ✅ 列 P1，首版不做（见 `prd02e`） | 本文档 §8 |
| 7 | 标签频道完整配置 | ✅ 列后续里程碑（`prd02d`），首版只预留 `slug`/`seo` | 本文档 §8 |
| 8 | 前端 API | ✅ 已就绪（2026-09-24），直接对接，无兜底适配层 | 项目主管确认 |

---

## 1. 目标与现状

| 项 | 说明 |
|---|---|
| 目标 | 打通「本地流水线 → 前端网站」最后一环：把中文稿按模式发布到前端，替换「只写本地 `data/published/*.md`」的临时落点 |
| 现状代码 | `publish.py` 从 `data/queue/` 选 `pending`，写本地 md，**不回写 pool**（断链 bug）；front-matter 用旧字段 `label`；无模式判断、无幂等、无地域轮转 |
| 本轮缺口 | ① 存储合并为 pool 单系统 ② 发布模式三档 ③ 前端 API 幂等对接 ④ 状态回写 + 否决 ⑤ 地域轮转选稿 ⑥ 稿源保证（翻译范围方案 B） |

---

## 2. 设计子问题清单

| # | 子问题 | 决策 | 状态 |
|---|---|---|---|
| Q1 | 两套 status 是否保留 | **合并为单一系统**（详见 §3.1）。断链 bug 的根因是 queue/pool 双写，单系统后自然消失 | ✅ 已定 |
| Q2 | 发布源 | 只从 `data/pool/` 选 `status == 'pending'`；按 `score` 降序 + `region_tag` 轮转 | ✅ 已定 |
| Q3 | 发布模式 | 三档 `manual / semi_auto / auto`，起步 `semi_auto`；实现见 `03f_发布模式三档.md` | ✅ 已定 |
| Q4 | 配置落点 | 静态配置 → `config.yaml`（`publish:` + `site:` 段）；Token → 环境变量 `SITE_API_TOKEN` | ✅ 已定 |
| Q5 | 幂等策略 | `external_id`（= news `id`）为幂等键；发布前 `GET /api/articles?external_id=` 查重，重复跳过 | ✅ 已定 |
| Q6 | 状态回写 | 发布成功 → `published` + 回写 `site_article_id`/`published_at`；失败保持 `pending` 可重试 | ✅ 已定 |
| Q7 | 稿源保证 | 翻译范围选方案 B（真话题 + 高分单篇），每天 ≥10 篇进 `pending`；实现见 `03e` | ✅ 已定 |
| Q8 | 界面/操作入口 | 发布模式三档实现见 `03f`；admin 手动发布与否决见 `prd02a` / `03g`（v2.0） | ✅ 已定 |

---

## 3. 总体设计

### 3.1 核心架构决策：合并两套 status 为单一系统

**问题（现状）**：两套 status 并存，且有一处断链：

| 对象 | 字段 | 取值 |
|---|---|---|
| `data/pool/` 的 news | `news.status` | `new → tagged → translated` |
| `data/queue/` 的 item | `item.status` | `pending → published` |

**断链 bug**：`publish.py` 只改 queue 的 status，**从不回写 pool**，导致 pool 的 news 停在 `translated`，永远到不了 `published`，admin 无法从 pool 看出文章终态。

**合并决策**：

```
new → tagged → pending → published
                    └──> rejected
```

| 决策项 | 内容 |
|---|---|
| 唯一存储 | 只有 `data/pool/`，翻译结果（`title_cn`/`body_cn`）**直接写回 news** |
| 废弃 `data/queue/` | 翻译稿不再单独存一份 |
| 废弃 `translated` | 翻译完成即 `pending`，UI 只显示 `pending` |
| 单一 status | 只有 `news.status` 一个字段，不再有 queue 的 item.status |

**状态机**：

| 状态 | 含义 | 谁写入 |
|---|---|---|
| `new` | 采集入池 | fetch.py |
| `tagged` | 打频道+标签+聚类 | tag.py |
| `pending` | 已翻译、待发布 | translate.py |
| `published` | 已发布到前端 | publish.py |
| `rejected` | 人工否决，下一轮跳过 | admin 手动（`03g`） |

**字段演进（news 一条记录走全程）**：

> 字段命名约定：英文原文 `title` / `body`；中文翻译 `title_cn` / `body_cn`。翻译后四字段并存。

| 状态 | 新增字段 |
|---|---|
| `new` | `title` `body` `url` `source` `region_tag` `fetched_at` |
| `tagged` | `channel` `tags` `summary` `keywords` `region_city` |
| `pending` | `title_cn` `body_cn` |
| `published` | `site_article_id` `published_at` |
| `rejected` | （可选）`rejected_reason` |

> 完整字段表与状态机定稿见 `03b_文章模型.md`。

### 3.2 数据流

```
cron 每小时
   │
   ▼
publish.py ──读 config.yaml 发布模式──► auto？──是──► 从 pool 选 status=pending 前 N（地域轮转）
   │                                                     │
   │ 否（semi_auto/manual）                               ▼
   ▼                                          对每篇调 GET /api/articles 查重
 跳过，等 admin 人工操作                                     │
   │                                                       ▼
   ▼                                                未存在 ──► POST /api/articles
 admin 筛选 pending：                                               │
   预览 / 立即发布 / 批量 / 否决                                     ▼
                                                          news.status → published
                                                          回写 site_article_id
```

### 3.3 模块划分

| 模块 | 职责 | 文档 |
|---|---|---|
| `publish.py` 模式判断 + 选稿 | 读 mode、`select_pending`、编排 | `03f_发布模式三档.md` |
| `lib/site_api.py` | 前端 API 客户端（幂等写、查重、鉴权） | `03c_前端API发布.md` |
| 流程视角（端到端） | 翻译完成 → 推送前端的完整链路 | `03d_文章发布流程.md` |
| 稿源（翻译范围） | 保证 `pending` 供给 | `03e_翻译范围方案B.md` |

---

## 4. 接口契约（前端 API）

前端站只需提供**最小幂等写接口**，其余（去重、状态、调度）由 aggregator 侧管理。

| API | 方法 | 用途 |
|---|---|---|
| `POST /api/articles` | 创建/发布一篇文章（**幂等**） | 核心写接口 |
| `GET /api/articles?external_id=xxx` | 按外部 id 查是否已发布 | 发布前去重 |
| `GET /api/health` | 健康检查 | 可选，发布前探活 |

**请求体（POST /api/articles）**：

```json
{
  "external_id": "news_0f718e8f7938",
  "title": "中文标题",
  "body": "中文正文",
  "channel": "商业地产",
  "tags": ["钾肥", "特朗普"],
  "region_tag": "萨省",
  "source_url": "https://www.cbc.ca/...",
  "published_at": "2026-09-24T10:00:00Z"
}
```

**关键约定**：

| 约定 | 说明 |
|---|---|
| 幂等键 | `external_id`（即 news 的 `id`），重复提交返回「已存在」，不重复创建 |
| 鉴权 | 请求头 `Authorization: Bearer <SITE_API_TOKEN>` |
| 响应 | 返回前端生成的 `article_id`，回写 news 的 `site_article_id` |
| 撤回/下线 | 前端再提供 `DELETE /api/articles/{id}`（或 `POST .../unpublish`），误发纠错（P1，见 `prd02e`） |

**首版范围**：只发纯文字（标题+正文+频道+标签+来源链接），**不含图片**（图片处理为 P1）。

> 完整字段映射、错误处理矩阵、代码改动清单见实现级设计 `03c_前端API发布.md`。

---

## 5. 状态与字段变更

无新增字段（沿用 §3.1 的字段演进）；核心是**存储合并**带来的变更：

| 变更 | 内容 |
|---|---|
| 删除 | `data/queue/` 目录、`item.status`、`translated` 状态 |
| 新增 | 翻译结果直写 news 的 `title_cn`/`body_cn`；发布回写 `site_article_id`/`published_at` |
| config.yaml | 新增 `publish:` 段（mode/per_hour/region_order）+ `site:` 段（base_url/slug） |
| 环境变量 | `SITE_API_TOKEN` |

---

## 6. 错误处理矩阵

| 场景 | 行为 |
|---|---|
| 模式为 `semi_auto`/`manual` | 记日志，`return`（exit 0），不选稿不推送 |
| 前端不可用（`GET /api/health`/网络失败） | 保持 `pending`，下轮重试，记失败日志 |
| 前端返回「已存在」（幂等命中） | 视为成功，回写 `site_article_id`，不重复创建 |
| `POST /api/articles` 失败（4xx/5xx） | 该篇保持 `pending`，继续下一批（失败隔离） |
| `base_url` 缺失（mode=auto） | exit 1（现有行为） |
| mode 非法值 | 回退 `semi_auto` + WARNING，不发布 |
| Token 缺失 | 记错误并跳过发布，不发送请求 |

---

## 7. 代码改动清单

| # | 文件 | 改动 |
|---|---|---|
| 1 | `app/aggregator/publish.py` | ① 选稿改从 pool 取 `pending`（去 queue）② 加模式判断（见 `03f`）③ 幂等查重 + 回写 pool ④ 地域轮转 |
| 2 | `app/aggregator/lib/site_api.py` | 前端 API 客户端（POST/GET/health），Bearer 鉴权（新增，见 `03c`） |
| 3 | `app/aggregator/translate.py` | 翻译结果写回 news 并置 `pending`（去 queue，见 `03e`） |
| 4 | `config.yaml` | 新增 `publish:` 段 + `site:` 段 |
| 5 | `test/test_*.py` | 发布幂等 / 模式三档 / 回写 / 地域轮转用例 |

> 存量迁移：现有 queue 数据 + pool 的 `translated` 状态需迁移为 `pending`（翻译结果回写 news）。

---

## 8. 不在本轮范围

| 项 | 归属 |
|---|---|
| 撤回/下线接口（`DELETE /api/articles/{id}`） | P1 → `prd02e` |
| 首版图片处理 | P1 → `prd02e` |
| admin 手动发布与否决 UI（FR-D7 的操作载体） | v2.0 → `prd02a` / `03g` |
| 标签频道完整后台配置（仅预留 `slug`/`seo`） | 后续里程碑 → `prd02d` |
| 在线改 config（热更新） | 后续；初期改 config 重启 |
| 服务器长期后台 Data Pipeline（Queue/Worker/Retry） | 随生产部署 → `prd02b` |

---

## 9. 测试方案

| 用例 | 验证点 |
|---|---|
| 幂等发布 | 重复提交同 `external_id` 不重复创建 |
| 模式三档 | `auto` 自动发；`semi_auto`/`manual` 跳过（见 `03f` TC1-TC8） |
| 状态回写 | 发布成功后 news → `published` 且含 `site_article_id` |
| 失败隔离 | 单篇失败不阻断其余，失败篇保持 `pending` |
| 地域轮转 | 按 `region_tag` 轮转取稿，避免单一地域霸榜 |
| 存量迁移 | queue/`translated` 正确迁移为 `pending` |

---

## 10. 验收标准

| # | 验收项 | 标准 |
|---|---|---|
| 1 | 单一系统 | 只有 `data/pool/` 一套存储，无 `data/queue/`、无 `translated` 状态 |
| 2 | 发布源 | 只从 pool 选 `pending`，按 score 降序 + 地域轮转 |
| 3 | 状态机 | `new → tagged → pending → published/rejected` 正确流转 |
| 4 | 发布模式 | 三档可配置，auto 自动发、semi_auto 人工确认、手动任意时刻可用 |
| 5 | 配置 | config.yaml 改即生效；Token 走环境变量 |
| 6 | API 对接 | 幂等发布，重复提交不重发，回写 site_article_id |
| 7 | 稿源 | 每天 ≥ 10 篇进入 `pending` |
| 8 | 首版范围 | 纯文字发布跑通，前端就绪后一键切换 |

---

## 11. 待确认清单

| # | 项 | 状态 |
|---|---|---|
| 1 | 前端 API 是否就绪 | ✅ 已确认（2026-09-24） |
| 2 | 撤回接口形态（`DELETE` vs `POST .../unpublish`） | ⏳ 待 site 侧确认（P1，见 `prd02e`） |
| 3 | 存量迁移窗口（queue → pool） | ⏳ 待项目主管排期 |
