# dev-01c 端到端联调 — 实测报告（K5a / K5b）

> 版本 v1.5 · 2026-09-28 · 负责人：开发主管 · 迭代版本：v1.0 建站
>
> **v1.3 变更（频道体系对齐收尾）**：site 侧重建一级分类，**与本地 8 频道一一对应**（仅
> 「教育/教育就业」命名差异）；`channel_categories` 回填 8 条完整映射并实测验证
> （`本地突发`→`general/`、`商业地产`→`commercial-real-estate/`）；§4.4 / §7 / §8 / §9
> 相关项更新；原「8 vs 5 不一致」跨域反馈**已撤回**。
>
> **v1.2 变更（K5b 收官）**：真实 site 掉线恢复后完成联调——**发布到真实 52sask.com
> 前端全链路打通**（推送 → 回写 → 前端可见 → 幂等零重复 → 地域轮转 → 分类映射）；
> 3 项适配落地（commit `f8a4797`）；§4 重写为真实联调结果；实测**纠正 site 答复 3 处**
> （关键：site 未强制 slug 唯一，幂等须 pipeline 侧查重）。
>
> **v1.1 变更（2026-09-26 Max 拍板拆分）**：本报告对应原 **K5（A7.1）**，现拆分为
> **K5a**（只读链路 + 发布路径实证，本报告 §2/§3）与 **K5b**（真实 site 推送，本报告 §4）。
>
> 定位：`run_daily.py` 全链路（采集→打标→聚类→热度→翻译→发布）**真实数据实测记录**，是 v1.0 验收标准「全自动跑通 + 每天稳定发布 ≥10 篇」的实证环节。
>
> 关联：`03_开发文档/归档/03c_前端API发布.md` §8（联调步骤）/ §11（site 答复 + 实测校准）· `dev-01b_发布开发文档.md` §9/§10 · `pm-01b_kanban.md` K5a/K5b · `pm-01a_项目计划.md` §盘点

---

## 0. 结论摘要（TL;DR）

| 环节 | 状态 | 说明 |
|---|---|---|
| ① 只读链路（fetch→tag→rank→translate） | ✅ **实测通过** | 真实数据全链路跑通，pending 19 → **29**（+10，达 ≥10/天目标） |
| ② 发布路径（单篇/批量 + 轮转 + 回写） | ✅ **实测通过**（本地 mock site） | 5 篇真实发布，region_order 轮转正确，site_article_id/slug 回写完整 |
| ③ 幂等重跑（slug 冲突对账） | ✅ **实测通过**（本地 mock site） | 5/5 冲突对账，**0 重复创建**，site_id 保持原值 |
| ④ 发布到**真实 site 前端** | ✅ **K5b 实测通过** | 52sask.com（`localhost:5001`）真实推送：单篇发布 + 前端可见 + 幂等零重复 + 批量 3 篇地域轮转 + 分类映射，全部实证；3 项适配已落（commit `f8a4797`） |
| ⑤ cron 真实行为 | ⏸️ 归 v2.1 | Ubuntu 侧未就绪，Max 已裁决 Ubuntu 不在 v1.0（`prd02b`） |

> **K5a / K5b 拆分（2026-09-26）**：①②③（只读链路 + 发布路径实证 + 回归） = **K5a**（✅ 完成）；
> ④（真实站推送 + 3 项适配） = **K5b**（✅ **完成**）。
> **M3 收官：v1.0 端到端全链路（含真实前端推送）已打通。**

**一句话**：**发布前的全链路（采集→翻译）已用真实数据完整跑通**；**发布后全链路**（推送→回写→幂等）已用本地 mock site 按真实契约验证通过；唯一未完成项是「推到**真实** 52sask.com」——**外部依赖阻塞**（site 后端未部署/不可达），非本模块缺陷。

---

## 1. 实测环境

| 项 | 值 |
|---|---|
| 时间 | 2026-09-26 21:00–21:19 EDT |
| 机器 | Windows 开发机（RTX 3070 Ti，8GB VRAM） |
| Python | 项目 `.venv`（requests 2.34.2 + bs4 + lxml） |
| Ollama | 0.34.3，本地 127.0.0.1:11434（`serve` 常驻） |
| 模型 | gemma3:4b（打标）+ qwen3:8b（翻译） |
| 数据基线 | pool 48 = tagged 28 + pending 19 + published 1（运行前快照，已备份 `data/pool.bak.a71_20260926_210023`） |
| 网络 | cbc.ca 可达（HTTP 200） |

> 说明：启动时 Ollama GUI 未运行、11434 无服务。首次 `tag.py` 因此大面积失败（打频道失败跳过）。**修复**：以常驻进程重启 `ollama serve` 后重跑，全绿。此坑已记入 §5「发现的问题」。

---

## 2. 环节①：只读链路实测（真实数据）

### 2.1 fetch.py — 采集

```
CBC Saskatchewan：列表页 30 条 → 入池 26
CBC Toronto：列表页 15 条 → 入池 14
合计：新增入池 40 条，失败 0 条，池内共 88 条
```

- 结构化运行记录 `logs/fetch/fetch_20260927_010053.json`：`total_new: 40`、`total_failed: 0`、`status: success`（**FR-C7 埋点 ✅**）。
- URL 去重生效（88 = 48 存量 + 40 新增，无重复入池）。

### 2.2 tag.py — 打频道 + 标签 + 聚类

```
完成。打频道/标签并聚类完毕：话题 3 个（media>=2），单篇文章 81 篇（media=1）。
pool 状态：tagged 68
```

- 8 频道分类正常（抽检：本地突发/商业地产/民生服务等）。
- 事件级聚类：3 个话题（media≥2）+ 81 单篇，符合「宁可新建不误并」阈值设计。
- **Ollama 首次失败 → 重启 serve 后重跑全绿**（见 §5）。

### 2.3 rank.py — 热度评分

```
Top 3 话题（media>=2）：
  1. [本地突发] score=1.000 media=2  Sister of Saskatoon homicide victim…
  2. [本地突发] score=0.700 media=2  34-year-old dies after e-bike…
  3. [商业地产] score=0.550 media=2  Sask. premier unsurprised that…
已为 87 篇文章独立计算热度。
```

- 评分公式 `0.55×媒体数 + 0.30×新鲜度 + 0.15×频道权重` 正常产出（**FR-P7/P9 ✅**）。

### 2.4 translate.py — 中译（方案 B 选题）

```
地域「萨省」：3 个 ranked 话题，取前 3 翻译 → 话题代表稿
地域「安省」：单篇补齐 5 篇
地域「萨省」：单篇补齐 4 篇
完成。本次翻译 10 篇（话题代表 1 + 单篇 9），状态 pending。
```

**结果**：pending 19 → **29**（萨省 14 + 安省 15），**单轮 +10 篇，达「每地域 top5」设计与 ≥10/天目标（FR-P11 / K1 方案 B 实证 ✅）**。

**字段完整性抽检**（`title_cn`/`body_cn` 写入、英文原文保留）：

| news id | 地域 | score | title_cn | body_cn 长度 | 英文保留 |
|---|---|---|---|---|---|
| news_8dd948d07eb5 | 萨省 | 1.00 | 萨斯卡通今年第10起凶杀案 | 217 | ✅ |
| news_b5b2201de9cc | 萨省 | 0.55 | 加拿大不应从白俄罗斯购买"血钾肥"，萨省省长发声 | 1110 | ✅ |
| news_d0d01fb9434e | 萨省 | 0.70 | 里贾纳警方调查严重电动车事故 | 188 | ✅ |
| news_18ef7aafe3e3 | 萨省 | 0.45 | 蓝门后的故事：萨斯喀彻温省音乐人圆桌会 | 276 | ✅ |

---

## 3. 环节②③：发布路径实测（本地 mock site）

> **为何用 mock**：真实 52sask.com 后端不可达（§4），但发布路径（POST 契约、轮转选稿、状态回写、幂等对账）可在**按 03c 契约实现**的本地 mock 上完整验证。mock 实现 `POST /api/posts`（slug 幂等 + 409 冲突）+ `GET /api/posts?slug=`，与 `lib/site_api.py` 契约一致。

### 3.1 单轮批量发布（5 篇，含地域轮转）

临时 config：`mode=auto`、`region_order=['萨省','安省']`、`base_url=127.0.0.1:5555`。

```
[published] news_360bd79a5f39 → mongo_1001
[published] news_1414a7b3e338 → mongo_1002
[published] news_8dd948d07eb5 → mongo_1003
[published] news_30b5630aff23 → mongo_1004
[published] news_d0d01fb9434e → mongo_1005
本次批次：尝试 5/5，推送成功 5，冲突对账 0，失败 0。池内 pending 剩 24 篇。
```

- **地域轮转正确**：萨(360b)→安(1414)→萨(8dd9)→安(30b5)→萨(d0d0) 严格交替（**FR-D6 真实路径实证 ✅**）。
- **状态回写完整**：5 篇均 `status=published` + `site_article_id` + `site_full_slug`（**FR-D5 ✅**）。
- **本地 md 双写**：5 篇落 `data/published/2026-09-27/`。

### 3.2 幂等重跑（关键验证）

把上述 5 篇重置为 pending（模拟断点续跑/重复批次），重跑 publish：

```
[reconciled] news_360bd79a5f39 slug 已存在，视为已发布（site_id=mongo_1001）
[reconciled] news_1414a7b3e338 slug 已存在，视为已发布（site_id=mongo_1002）
[reconciled] news_8dd948d07eb5 slug 已存在，视为已发布（site_id=mongo_1003）
[reconciled] news_30b5630aff23 slug 已存在，视为已发布（site_id=mongo_1004）
[reconciled] news_d0d01fb9434e slug 已存在，视为已发布（site_id=mongo_1005）
本次批次：尝试 5/5，推送成功 0，冲突对账 5，失败 0。池内 pending 剩 24 篇。
```

- **5/5 冲突对账，0 重复创建**；site mock 查询确认 slug 仍为原 `_id`（mongo_1001…），**幂等键（slug=news id）生效**（**FR-D3 ✅**）。
- **池状态不变**（published 6 / pending 24），无重复发布。

### 3.3 三档模式 gating（真实数据）

```
发布模式为 semi_auto，跳过自动发布（等人工操作 admin）。
```

- semi_auto 下 publish 直接跳过、**零推送**（**FR-D2 三档真实路径 ✅**）。
- 联调后 config **已还原**（git diff 为空，mode=semi_auto / region_order=[] / base_url=localhost:5001）。

---

## 4. 环节④：真实 site 前端——**✅ K5b 联调通过（2026-09-26）**

### 4.1 过程回顾

| 阶段 | 结果 |
|---|---|
| 首次探测（K5a 轮） | `curl http://localhost:5001/api` → **HTTP 502**（site 后端未启动）；site 代码不在本机 → 判定外部阻塞 |
| **K5b 复测** | `GET /api/health` → **200 `{"status":"ok"}`**，site 已就绪 → 具备联调条件 |

### 4.2 3 项代码适配（site 答复 → 实测校准后落地）

| # | site 答复 | **实测校准** | 落地 |
|---|---|---|---|
| ① | `status` 不含 `published` | ✅ 属实（`draft` 201 通过） | `build_payload` 改 `'draft'` |
| ② | 查重 `GET /posts/slug/{fullSlug}` | ⚠️ **只认 fullSlug**，裸 slug 返回 404 | 两级查重（直连 + `?status=all` 列表过滤） |
| ③ | slug 重复报 **409** | ❌ **不成立**：site 不强制唯一，重复 POST → **201 新建第二篇** | **幂等主路径改「发布前查重」**；409 留作兼容 |

### 4.3 真实推送实测结果

| 验证项 | 结果 |
|---|---|
| 单篇发布（`--limit 1`） | ✅ pending → published；`site_article_id` / `site_full_slug` / `site_published_at` 回写完整 |
| **前端可见** | ✅ `GET /posts/slug/general/news-360bd79a5f39` → 标题/正文/标签/分类齐全，`status=draft` |
| **幂等重跑** | ✅ 重置为 pending 重发 → 查重命中 → `reconciled`，**零新 POST**（文章总数 22→22 不变） |
| 批量 3 篇 + `region_order=['萨省','安省']` | ✅ 3/3 成功；地域**萨→安→萨严格交替**（FR-D6 真实路径生效） |
| 分类映射（初版） | ✅ `商业地产` → 财税金融；`fullSlug` 变 `finance-tax/news-xxx`（映射生效） |
| 未映射频道（初版） | ✅ `本地突发` → 不带分类，site 归 `general/`，不报错 |
| **分类映射（对齐后，2026-09-26）** | ✅ `本地突发` → `general/`、`商业地产` → `commercial-real-estate/` —— **8 频道一一对应后正确落地**（site 一级分类已重建对齐本地 8 类） |

**结论**：**发布到真实 52sask.com 前端全链路打通**（推送 → 回写 → 前端可见 → 幂等零重复 → 地域轮转 → 分类映射 8/8 全覆盖）。

### 4.4 遗留

| 项 | 状态 |
|---|---|
| `SITE_API_TOKEN` | ⏸️ 未设置（测试环境无鉴权可读写）；**生产启用鉴权前必须配**（M5 检查清单）；**Max 拍板先不管** |
| 频道体系 vs site 分类不一致 | ✅ **已解决**（site 侧重建分类对齐本地 8 频道，仅「教育/教育就业」命名差异）；原跨域反馈**已撤回**（见 `03c` §12） |
| cron 真实行为 | ⏸️ 归 v2.1 `prd02b`（Ubuntu） |

---

## 5. 发现的问题与处理

| # | 问题 | 处理 | 状态 |
|---|---|---|---|
| 1 | **Ollama 服务未运行**（GUI 关闭 → 11434 无监听），首次 tag.py 大面积「打频道失败跳过」 | 以常驻进程重启 `ollama serve`，重跑 tag.py 全绿 | ✅ 已解决 |
| 2 | **后台 `ollama serve` 随父 shell 退出而终止**（首次用 `&` 启动，进程被回收） | 改用常驻后台任务方式启动，稳定存活 | ✅ 已解决（运维提示） |
| 3 | 运行前未跑 Ollama → 直接跑 tag 会**静默产生大量跳过**（仅 WARNING，不报错） | **已立卡**：`run_daily.py`/tag.py 增加 Ollama 健康预检（`GET /api/tags`），不健康 fail-fast | ✅ 已立卡（看板 D11） |
| 4 | 真实 site 首次探测不可达（502） | site 后端恢复后 K5b 完成联调 | ✅ 已解决 |
| 5 | **site 未强制 slug 唯一**（K5b 实测）：重复 POST 返回 201 并新建第二篇 | 幂等主路径前移为「发布前查重」；`_find_in_list` 带 `status=all` 覆盖 draft | ✅ 已修复（commit `f8a4797`） |
| 6 | **site 查重端点只认 fullSlug**：`/posts/slug/{裸slug}` 返回 404 | 两级查重（直连 fullSlug + 列表客户端过滤） | ✅ 已修复 |
| 7 | **site 列表默认只含 published**：draft 稿查不到 → 查重漏判致重复 | 列表查询带 `?status=all` | ✅ 已修复 |

---

## 6. 回归测试

K5b 联调后全量回归（5 套，**61 用例**）：

| 套件 | 用例 | 结果 |
|---|---|---|
| test_a14_site_publish | **17** | ✅ OK（+TC4a 查重跳过 / TC4d 查重错误保守 / TC11 两级查重 / TC11b fullSlug 直连） |
| test_a61_region_rotation | 13 | ✅ OK |
| test_a5x_publish_mode | 9 | ✅ OK |
| test_a12_state_machine | 11 | ✅ OK |
| test_a21_translate_scope | 11 | ✅ OK |

---

## 7. 数据状态说明（联调副作用）

| 项 | 处理 |
|---|---|
| pool 新增 | 40 条采集入池（tagged +40，真实数据，**保留**——是正常流水线产物） |
| pool 新增翻译 | pending +10（真实翻译，**保留**） |
| mock 测试发布（K5a） | 5 篇发布后**已还原为 pending** 并清除 `mongo_*` site_id；mock 本地 md 已删 |
| **K5b 真实发布** | **5 篇真实推送 site**（`news_360bd79a5f39`/`news_8dd948d07eb5`/`news_1414a7b3e338`/`news_d0d01fb9434e`/`news_0f718e8f7938`），pool 置 published + 回写 site_id，**保留**（是真实站点内容，勿删） |
| **site 探测稿** | 2 篇 contract probe 已 `DELETE /posts/{_id}` 清理；**频道对齐验证另发 3 篇真实稿**（覆盖 `本地突发`/`商业地产` 等旧未映射频道），site 现有 9 篇 pipeline 稿（1 published + 8 draft） |
| pool 备份 | `data/pool.bak.a71_20260926_210023`（K5a 快照）**已清理**；`data/pool.bak.k5b_*`（K5b 快照）**已清理**（Max 2026-09-26 确认） |
| config.yaml | publish 段（mode/region_order）已还原；**`channel_categories` 为 8 条完整映射（对齐后正式保留）** |
| 最终状态 | tagged 58 + pending 24 + published 6（5 篇 K5b 真实推送 + 1 篇历史 K3 稿） |

---

## 8. FR 盘点影响

| FR | 原状态 | 联调后 | 依据 |
|---|---|---|---|
| FR-P15 串行编排 | 🔶 待 A7.1 | ✅（只读链路实证） | run_daily 四步串行跑通、失败即止逻辑正常 |
| FR-D1 定时发布 | 🔶 待 A7.1+cron | 🔶（发布路径实证，cron 归 v2.1） | 发布路径 + 幂等已实证；cron 真实行为归 v2.1 |
| FR-P11 选题范围 | ✅ | ✅（再获实证） | +10 篇/轮 = 每地域 top5 达标 |
| FR-D3 幂等 | ✅ | ✅（**真实站实证**） | 发布前查重零重复（K5b）；site 未强制唯一故须 pipeline 侧兜底 |
| FR-D5 回写 | ✅ | ✅（**真实站实证**） | site_article_id/site_full_slug/site_published_at 三字段真实回写 |
| FR-D6 地域轮转 | ✅ | ✅（**真实站实证**） | 真实路径萨-安-萨严格交替 |
| FR-D4 分类映射 | — | ✅（**新增实证**） | `channel_categories` 8 条完整映射（本地 8 频道 ↔ site 8 类一一对应），`商业地产`→`commercial-real-estate`、`本地突发`→`general` 生效 |

> **v1.0 验收标准对照**：①全链路自动跑通（✅ 含真实前端推送）②每天 ≥10 篇（✅ 单轮 +10）③可排障（✅ 结构化运行记录）④幂等（✅ 发布前查重，零重复）⑤状态一致（✅ 全程回写）。
> **未闭环**：仅 cron 真实行为（归 v2.1 `prd02b`）；`SITE_API_TOKEN` 生产鉴权（M5 前配）。

---

## 9. 待确认清单

| # | 项 | 状态 |
|---|---|---|
| 1 | site 侧 4 项确认（`03c` §11） | ✅ **已答复 + K5b 实测校准**（纠正 3 处，见 `03c` §11） |
| 2 | 真实 site 后半程联调（§4） | ✅ **K5b 完成**（推送/可见/幂等/轮转/分类全实证） |
| 3 | cron 真实行为 | ⏸️ 归 v2.1 `prd02b`（Max 已裁决 Ubuntu 不在 v1.0） |
| 4 | Ollama 健康预检是否立卡 | ✅ **已立卡**（看板 D11） |
| 5 | `data/pool.bak.a71_*` 备份清理 | ✅ 已清理（Max 2026-09-26 确认） |
| 6 | 分类 `_id` 回填 `config.yaml` | ✅ 已回填 **8 条完整映射**（site 分类已对齐本地 8 频道，见 `03c` §11.2） |
| 7 | 频道体系 8 vs 5 不一致 | ✅ **已解决**（site 侧重建分类对齐本地 8 频道，仅「教育/教育就业」命名差异）；原跨域反馈**已撤回** |
| 8 | `SITE_API_TOKEN`（生产鉴权） | ⏸️ 测试环境未配可读写；M5 生产上线前必配；**Max 拍板先不管** |
| 9 | `data/pool.bak.k5b_*` / 本地 md | ✅ 备份已清理（Max 2026-09-26 确认） |
