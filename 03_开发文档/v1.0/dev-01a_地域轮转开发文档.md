# dev-01a 地域轮转 — 开发文档（实现级设计）

> 版本 v1.1 · 2026-09-28 · 状态：**已拍板，进入实现** · 负责人：开发主管 · 迭代版本：v1.0 建站
>
> 定位：**实现级设计**——发布选稿环节的**地域轮转**（`publish.py` 消费 `config.yaml` `publish.region_order`，保证各地域发布均衡、避免单省霸榜）；需求侧见 `v1.0_需求清单.md` §5 `FR-D6` / `02a_需求总览.md` §2.3。
>
> 覆盖范围：发布选稿 `select_pending`（现有）+ 地域轮转选取（本轮新增）。`dev-01b_发布开发文档.md` §2 Q2 已定「按 score 降序 + region_tag 轮转」，本文档落其实现。
>
> 关联文档：`02a_需求总览.md`（§2.3 `FR-D6`）· `v1.0_需求清单.md`（§5 `FR-D6` 入选）· `dev-01b_发布开发文档.md`（§2 Q2 / §3.2 数据流 / §9 测试 / §10 验收）· `03f_发布模式三档.md`（§3.5 main() 改造）· `03d_文章发布流程.md`（§2 选稿规则）· `03b_文章模型.md`（`region_tag` 字段）
>
> 命名说明：按 `gov-a_文档规范.md` §2.6，已与 `02a` 功能编码一一映射的文档以 FR 码命名（`FR-<编码>_<功能名>开发文档.md`），**不占 `03<字母>` 序列**；本文档内容属产品 v1.0，故落 `v1.0/`。

---

## 0. 拍板记录

| # | 事项 | 决策 | 来源 |
|---|---|---|---|
| 1 | 地域轮转是否启用 | ✅ 启用；由 `config.yaml` `publish.region_order` 驱动（K2 已落配置字段，本轮实现消费逻辑） | K4 卡 / `FR-D` §2 Q2 |
| 2 | 轮转算法 | ✅ **顺序轮转（round-robin）**，非比例配额——`region_order` 顺序即轮转次序 | 本文档 Q1 |
| 3 | `region_order` 为空 | ✅ **不启用轮转**，回退现有行为（全池 pending 按 score 降序取前 N），与现状完全一致 | 本文档 Q2 |
| 4 | 某地域无稿 | ✅ **跳过该地域继续轮转**，用其余地域的稿子补齐到 N 篇；轮完一轮仍不足 N 则返回已选（不报错） | 本文档 Q3 |
| 5 | 轮转与模式判断先后 | ✅ 轮转在 `mode == 'auto'` 分支**之内**（semi_auto/manual 直接 return，根本不到选稿），不改变 `03f` §3.5 前置结构 | 本文档 Q4 |
| 6 | 每地域配额 | ✅ **不设固定配额**，N 篇在 `region_order` 各地域间**逐席位轮流分配**（第 1 篇给 order[0] 的最高分、第 2 篇给 order[1] 的最高分……循环） | 本文档 Q1 |
| 7 | 未在 `region_order` 中的地域 | ✅ **不参与轮转**（严格以 order 为准）；order 未覆盖地域的 pending 本轮不选，下一轮也不选——**由运营者显式配全地域**（配置即意图） | 本文档 Q5 |
| 8 | order 中地域名拼写不存在于池 | ✅ 静默跳过（等同无稿），记 DEBUG 日志；不告警（配置演进期正常） | 本文档 Q3 |
| 9 | 幂等 | ✅ 不影响——`published`/`rejected` 天然不满足 `status=='pending'`，轮转只在 pending 子集内选取，slug 幂等键不变 | 代码现状 |

---

## 1. 目标与现状

| 项 | 说明 |
|---|---|
| 目标 | 发布选稿在地域间均衡：`region_order` 非空时按顺序轮转取稿；为空时与现状一致 |
| 现状代码 | `select_pending(pool, limit)` 全池 pending 按 `score` 降序取前 N（见 `publish.py` §「select_pending」）；`load_publish_config` 已读取 `region_order`（K2）但**无任何消费逻辑** |
| 本轮缺口 | ① 新增地域轮转选取函数 ② `main()` 选稿处改用轮转函数（order 非空时才生效） ③ 测试 `test/test_a61_region_rotation.py` |

**现状数据快照（2026-09-26 实测）**：pool pending = 萨省 9 + 安省 10 = 19 篇。若不加轮转，按 score 降序取 top N 容易被高分地域霸榜（当前两省分布尚可，但扩展到温哥华后单省霸榜风险上升，故 `FR-D6` 入选 v1.0）。

---

## 2. 设计子问题清单

| # | 子问题 | 决策 | 状态 |
|---|---|---|---|
| Q1 | 轮转算法：顺序轮转 vs 按比例配额 | **顺序轮转（round-robin）**。`region_order` 是显式次序表，语义天然是「轮流」。按比例配额需额外配置权重字段，超出 v1.0 范围（v1.0 判据是「内容不偏废」，轮转已满足）。**分配方式**：生成 N 个席位，逐席位按 order 循环取下一个地域；每个地域内部按 score 降序取第一篇未选中的稿。 | ✅ 已定 |
| Q2 | `region_order` 为空时行为 | **不启用轮转**，`select_pending` 现有行为不变（全池 score 降序取前 N）。理由：K2 落字段时默认 `[]`，若视为「按 score 排地域」会改变现状行为、破坏 K2 回归；空 = 不启用是最直白的语义，也保证向后兼容。 | ✅ 已定 |
| Q3 | 候选池某地域无稿时跳过与回填 | **跳过该地域继续轮转**。例：order=[萨省,安省]，N=5，萨省只剩 1 篇 → 席位分配为 萨省1、安省1、萨省(无→跳过)、安省2、安省3 → 共选 4 篇（rather than 报错或跨地域硬凑）。**总量不足 N 时返回已选篇数**，`main()` 照常发布（与「pending 为空」同类，不视为错误）。池内该地域确实不存在或地域名拼写不符 → 同样跳过，记 DEBUG。 | ✅ 已定 |
| Q4 | 与三档模式判断的先后 | 轮转**在 auto 分支内**：`main()` 先判 `mode != 'auto'` 直接 return（`03f` §3.5 不动），auto 时才读 `region_order` 并走轮转选稿。semi_auto/manual 下根本不执行选稿逻辑，因此轮转不影响人工档。 | ✅ 已定 |
| Q5 | 未在 `region_order` 中的地域如何处理 | **不参与轮转**（严格以 order 为准）。这是**显式配置语义**：运营者配了 order 就表示「只在这些地域间轮转」。风险提示写入 §7 使用说明——配置时必须列全期望覆盖的地域，否则该地域稿子永不发布。若不希望如此，保持 order 为空即回退现状（全地域按 score）。 | ✅ 已定 |
| Q6 | 轮转是否影响幂等 | **不影响**。轮转只在 `status=='pending'` 子集内做选取，不改变 `publish_one` 的 slug 幂等键与状态回写；`published`/`rejected` 天然被排除。 | ✅ 已定 |
| Q7 | 纯函数化（便于测试） | 轮转逻辑抽为纯函数 `rotate_by_region(pool, limit, region_order)`，**不读 config、不碰网络**，入参即 order 列表。`main()` 负责从 `pub_cfg` 取 order 传入。便于单测直接构造池数据。 | ✅ 已定 |

---

## 3. 总体设计

### 3.1 数据流（auto 路径，注入轮转）

```
cron 每小时
   │
   ▼
publish.py main()
   │
   ├─ mode != auto ──► log + return（03f §3.5 不动）
   │
   └─ mode == auto
         │
         ▼
      pub_cfg = load_publish_config()      ← K2 已实现
      limit   = --limit 或 per_hour
      order   = pub_cfg['region_order']    ← 本轮消费
         │
         ├─ order 为空 ──► select_pending(pool, limit)          ← 现状行为
         │
         └─ order 非空 ──► rotate_by_region(pool, limit, order) ← 本轮新增
                                │
                                ▼
                          逐席位轮转取稿 → to_publish
         │
         ▼
      （以下不变）select 结果 → publish_one → 汇总日志
```

### 3.2 函数设计

| 函数 | 职责 | 改动 |
|---|---|---|
| `rotate_by_region(pool, limit, region_order)` | 地域轮转选取：按 order 逐席位循环取各地域最高分 pending 稿，无稿则跳过，返回 ≤limit 篇 | **新增** |
| `select_pending(pool, limit)` | 现状全池 score 降序取前 N | **不动**（order 为空时仍用它） |
| `main()` | 取 `region_order`，非空则调 `rotate_by_region` 替代 `select_pending` | **改造（2 行）** |
| `load_publish_config` / `publish_one` / `build_payload` | — | **不动** |

### 3.3 `rotate_by_region` 实现

```python
def rotate_by_region(pool, limit, region_order):
    """按地域顺序轮转选稿（FR-D6）。

    region_order 为地域轮转次序（如 ['萨省', '安省']）；每个席位按 order
    循环选下一个地域，取该地域内 score 最高且未被选中的 pending 稿。
    某地域无剩余稿则跳过该席位（继续下一地域），不足 limit 时返回已选。

    published/rejected 天然不满足 status=='pending'，不会被选中（幂等隔离）。
    """
    if not region_order or limit <= 0:
        return []

    # 各地域候选（pending，score 降序），一次性分桶
    buckets = {}
    for news in pool.all():
        if news.get('status') != 'pending':
            continue
        region = news.get('region_tag') or '未标注'
        buckets.setdefault(region, []).append(news)
    for region in buckets:
        buckets[region].sort(key=lambda x: x.get('score', 0), reverse=True)

    # 游标位置（每地域下一个待取下标）
    cursors = {region: 0 for region in region_order}
    selected, seen_ids = [], set()

    # 逐席位轮转；总席位 = limit（最多再放大一轮防御空转）
    seat = 0
    while len(selected) < limit:
        region = region_order[seat % len(region_order)]
        idx = cursors.get(region, 0)
        candidates = buckets.get(region, [])
        if idx < len(candidates):
            news = candidates[idx]
            cursors[region] = idx + 1
            if news['id'] not in seen_ids:
                selected.append(news)
                seen_ids.add(news['id'])
        else:
            log.debug('地域轮转：%s 无剩余 pending 稿，跳过该席位', region)
        seat += 1
        # 防御：所有地域都取空后仍不足 limit，退出避免死循环
        if seat > len(region_order) * (limit + 1) + 1:
            break

    return selected
```

> **为何用 `seat > len(order)*(limit+1)+1` 而非「连续一轮全空」判定**：纯游标推进下，最坏情况是某地域无限补位；用总席位上限兜底，逻辑更简单、必终止。正常情况下 `len(selected)` 达 `limit` 即退出。

### 3.4 `main()` 改造（仅选稿一行）

```python
    to_publish = select_pending(pool, limit)
```
改为：

```python
    region_order = pub_cfg.get('region_order') or []
    if region_order:
        to_publish = rotate_by_region(pool, limit, region_order)
        if region_order and not to_publish:
            log.info('地域轮转：region_order=%s 下无可用 pending 稿。', region_order)
    else:
        to_publish = select_pending(pool, limit)
```

> 后续「待发布为空」日志与发布循环**完全不动**。

### 3.5 config 语义（K2 已落，本轮仅补注释）

```yaml
publish:
  region_order: ['萨省', '安省']   # 地域轮转次序；空列表 = 不启用轮转（回退全池 score 排序）
```

---

## 4. 接口契约

无外部接口变更。`config.yaml` `publish.region_order` 由「占位空列表」变为「生效配置」。

---

## 5. 状态与字段变更

无新字段。状态机不变。仅 `config.yaml` 的 `region_order` 注释更新（说明空=不启用、须列全地域）。

---

## 6. 错误处理矩阵

| 场景 | 行为 |
|---|---|
| `region_order` 为空 / 缺省 | 回退 `select_pending`（现状行为），不报错 |
| `region_order` 非空 + 某地域无 pending | 跳过该地域席位，继续下一地域；记 DEBUG |
| `region_order` 非空 + 地域名拼写与池不符 | 同上（等同无稿），记 DEBUG |
| `region_order` 非空 + 全部地域无稿 | 返回 `[]` → `main()` 走「待发布为空」日志，正常退出 0 |
| 轮转后不足 limit 篇 | 返回已选篇数，`main()` 照常发布（非错误） |
| `limit <= 0` | 返回 `[]`（防御，正常 main 不会传 0） |
| mode ≠ auto | 轮转根本不被调用（`03f` 前置 return） |

---

## 7. 代码改动清单

| # | 文件 | 改动 |
|---|---|---|
| 1 | `app/aggregator/publish.py` | ① 新增 `rotate_by_region()` ② `main()` 取 `region_order`，非空时改用轮转选稿（§3.4）③ `_coerce_scalar` + 新增 `_split_inline_list`：支持内联列表 `['萨省','安省']`（原零依赖解析器把列表当字符串，见 §3.6） |
| 2 | `config.yaml` | `publish.region_order` 注释更新（空=不启用；配置须列全地域） |
| 3 | `test/test_a61_region_rotation.py`（新增） | §8 用例（13 个） |
| 4 | `03d_文章发布流程.md` §2 | 选稿规则补「region_order 非空时按该次序轮转（FR-D6）」 |
| 5 | `03f_发布模式三档.md` §3.5 / §10 | 选稿行补 region_order 分支；§10 标注 A6.1 已实现 |

> `select_pending` / `publish_one` / `load_publish_config` / `site_api` 均不动。

### 3.6 零依赖解析器的内联列表支持（实现中发现的必要补丁）

**问题**：`load_publish_config` 在无 PyYAML 环境走 `_parse_simple_config` 兜底，其 `_coerce_scalar` 不识别内联列表，把 `region_order: ['萨省', '安省']` 原样当字符串返回 → `rotate_by_region` 会按字符迭代（`'['`, `'萨'`, …），轮转静默失效。测试环境（无 PyYAML）实测暴露。

**修复**：`_coerce_scalar` 增加内联列表分支（`[...]` → 拆项去引号返回 `list`），新增 `_split_inline_list` 处理引号内逗号；`[]` 返回 `[]`、`[萨省]` 返回 `['萨省']`。**影响面**：仅补强兜底路径，装了 PyYAML 的环境行为不变；对既有 `channel_categories: {}` 等无影响（仍在 `{}` 分支）。

> 覆盖用例：TC10 / TC10b（`region_order` 内联列表解析）。

> ⚠️ 副作用提示：`_coerce_scalar` 现会把形如 `[x, y]` 的**任意**标量解析为列表。config.yaml 中不含此类标量，风险可忽略；如需规避可改为仅在 `region_order` 处特判。

---

## 8. 测试方案

`test/test_a61_region_rotation.py`：unittest + 临时目录 + FakeSiteClient，参考 `test_a5x_publish_mode.py` 风格（零网络/零 Ollama）。

| 用例 | 验证点 |
|---|---|
| TC1 轮转按 order 交替 | order=['萨省','安省']，各 3 篇 pending，limit=4 → 选取次序为 萨-安-萨-安，且每篇为其地域最高分 |
| TC2 order 为空回退现状 | order=[] → 等同 `select_pending`（全池 score 降序取 N） |
| TC3 单地域无稿跳过 | order=['萨省','安省']，萨省 1 篇、安省 3 篇，limit=4 → 选 4 篇（萨1+安3），不报错 |
| TC4 总量不足 limit | order=['萨省','安省']，共 2 篇，limit=5 → 返回 2 篇，无死循环 |
| TC5 全部地域无稿 | order=['BC省']（池中无）→ 返回 `[]`，DEBUG 跳过 |
| TC6 与 per_hour 组合配额 | main() + config region_order + per_hour=4 → client 收到 4 次 create_post，地域交替 |
| TC7 published/rejected 不被选中 | 池中含 published/rejected → 轮转结果不含它们 |
| TC8 order 未覆盖地域不参与 | order=['萨省']，池中安省有更高分稿 → 结果只含萨省 |
| TC9 纯函数 order 单元素 | order=['萨省']，limit=3，萨省 5 篇 → 取萨省 top3（=等地域内 score 降序） |
| TC10 内联列表解析 | `region_order: ['萨省','安省']` → 零依赖解析器返回 list（§3.6） |
| TC10b 内联列表边界 | `[]` → `[]`；`[萨省]` → `['萨省']` |

跑法：`python test/test_a61_region_rotation.py -v`

**回归**：`test_a5x_publish_mode.py`（9）· `test_a14_site_publish.py`（13）· `test_a12_state_machine.py`（11）· `test_a21_translate_scope.py`（11）须全绿（其中 a5x 的 CONFIG_TMPL 已含 `region_order: []`，走「不启用」分支，行为不变）。

---

## 9. 验收标准

| # | 验收项 | 标准 |
|---|---|---|
| 1 | 轮转生效 | config 配 `region_order` 后，发布选稿按该次序在地域间交替取稿 |
| 2 | 空 order 兼容 | `region_order: []` 时行为与现状完全一致（回归 a5x 全绿） |
| 3 | 无稿跳过 | 某地域无 pending 时跳过继续，不报错、不中断 |
| 4 | 幂等不受影响 | published/rejected 不被选中；slug 幂等键不变 |
| 5 | 模式不越界 | semi_auto/manual 下轮转不被调用（沿用 `03f` 前置判断） |
| 6 | 测试 | `test_a61` 全绿 + 4 套回归全绿 |

---

## 10. 不在本轮范围

- 按比例配额 / 地域权重（v1.0 判据为「不偏废」，轮转已满足；配额留后续）
- admin 侧配置 `region_order` 的 UI（`prd02a`，v3.0）
- `region_order` 的在线热更新（沿用「改 config 下次 cron 生效」约定）
- 跨地域回填（Q3 已定：跳过而非用别地域顶替，避免地域语义失真）

---

## 11. 运营提示（写入使用说明/配置注释）

> ⚠️ **配置 `region_order` 时必须列全期望覆盖的地域**。这是「显式轮转」语义：只有出现在 order 中的地域会参与发布选取，未列出地域的 pending 稿将**永不发布**（直到改配置或置空 order）。
> - 想全地域按热度发布 → 保持 `region_order: []`
> - 想地域均衡 → `region_order: ['萨省', '安省']`（后续加温哥华）
> - 想调整优先级 → 越靠前的地域在每轮开头先被取一席（首位地域每轮多得一席）

---

## 12. 待确认清单

无。所有子问题（Q1-Q7）已由开发主管按 L3 实现级自主拍板，均在本文档 §0/§2 记录；无 L2 待拍板项。按 K4 卡定义属「重（标准轮次）」但设计子问题均为实现级（L3），无需升级项目主管。
