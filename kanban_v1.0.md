# pm-01b Kanban — 待办清单（含本轮看板）

> 版本 v1.8 · 2026-09-26 · 来源：`01c_功能清单.md` 未完成项（该文已拆分归入 `02a_需求总览.md` / `03a_总体设计.md`）+ 2026-09-24 讨论的 task
>
> v1.8：**K5b PM 验收通过**（亲测 5 套 61 用例全绿 + 真实 site 只读复验 4 项 + 代码抽查 3 项一致）——**M3 收官，v1.0 盘点 24 ✅ / 1 🔶 / 0 ⬜**（仅 FR-D1 cron 归 v2.1）。
>
> v1.7：**A7.1 入板 K5**（2026-09-26 Max 拍板「下一步做」）——run_daily 全链路端到端联调，前置 site 侧 4 项确认。另经 Max 复核确认：D8 ✅、D10 ✅（gov-a v4.11 已收编规范）、retag 修复（`b0d1afc`）已 push 闭环。
>
> v1.6：K4 验收通过（PM 核验：亲测 5 套全绿 + 真实 pool 交替验证 + 抽查一致，FR-D6 §9 六项全过）——**看板 K1-K4 全部 ✅，v1.0 25 条已无 ⬜ 未开发项**。
>
> v1.5：K3 验收通过（PM 核验：亲测 5 套全绿 + 抽查 5 项一致，03g §9 七项全过）；K4 状态同步——开发主管已交付（FR-D6 设计 + `rotate_by_region` + test_a61 13 用例全绿，commit `a9d5a68`），🔍 待 PM 验收。
>
> v1.4：**改名转正（D10 ✅）**——本文件由项目主管改名 `01d_待办清单.md` → `pm-01b_kanban.md`，Max 裁决转正（`gov-a` v4.11 收编 `dev01<字母>` 命名）；全仓活引用已改指 `pm-01a`/`pm-01b`，各文档变更记录与「来源」历史行保留原编号仅作历史标识。
>
> v1.3：新增 §0.5「本轮看板」——**任务派发唯一通道**（认领协议见 `gov-b_开发流程.md` §2.3）；首卡 K1（A2.1）已入板。
>
> 前端 site API **已确认就绪**（2026-09-24），发布功能可直接对接，无需「写本地+适配层」兜底。
>
> v1.2：配合文档编码体系改为**按负责人分段**，更新全部交叉引用（`prd01c`→`prd02a` 等）。
>
> v1.1：A1 全部完成、A3 基本完成、A4 主体完成（A4.2 待 site 确认）、A5.2 完成、admin 使用说明文档就绪；
> 新增「待 site 侧确认」清单；执行顺序改为当前进度视角。

## 0. 阅读说明

- 本清单 = 原 `01c_功能清单.md`（现拆分归入 `02a_需求总览.md` / `03a_总体设计.md`）中所有「未完成 / 待办」项的集中，加上 2026-09-24 讨论出的 5 件 task。
- 分区状态：⬜ 未开始 · 🔶 进行中 · ✅ 完成
- 优先级：P0（阻塞发布，须先做）· P1（发布后必须）· P2（优化/可选）

---

## 0.5 本轮看板（任务派发唯一通道 · 2026-09-25 起）

> 协议见 `gov-b_开发流程.md` §2.3。角色开工只看本区：认领自己名下 ⬜ 的最高优先级卡，开工先置 🔶 并填认领时间。
> 卡状态：⬜ 待办 → 🔶 进行中 → 🔍 待验收（附证据）→ ✅ 完成（PM 核验）；🚫 阻塞（注明原因交回 PM）。
> 卡必须自包含（做什么/怎么做/验收/引用文档）；完成后不删卡，同步回写下方分区。

| 卡 | 任务 | 负责 | 轮次 | 状态 | 任务卡（自包含） |
|---|---|---|---|---|---|
| K1 | A2.1 翻译范围方案 B | 开发主管 | 重（标准轮次） | ✅ 完成（2026-09-26 PM 验收） | **做什么**：翻译选题从「仅真话题」扩为「真话题 + 高分单篇」（需求依据 `02a_需求总览.md` §2.2.3 FR-P11 / `FR-D1_定时发布开发文档.md` §2 Q7 方案 B，已拍板），保证每天 ≥10 篇进入 `pending`。**怎么做**：① 先出 03x 设计文档（按 `gov-a_文档规范.md` §2.2 模板，编号占 03 段下一个可用位），设计子问题至少覆盖：单篇与话题代表稿去重（同 news id 不重复翻）、每地域取数 N 与每日总量关系、单篇是否设 score 门槛、失败重跑幂等；待定项标 ⬜ 升级项目主管拍板。② 拍板后按代码改动清单实现：改动集中在 `app/aggregator/translate.py` 选题段（现状见 `03a_总体设计.md` §8 / `02a_需求总览.md` §2.2.3），翻译写回逻辑（写回 news + 置 pending）不动。③ 测试：新增 `test/test_a21_translate_scope.py`（unittest + 临时目录 + Fake client，参考 test_a12 风格），用例至少覆盖：话题+单篇混合选取、地域均衡、无话题时降级、已 pending/published 不重复选。**验收**：单测全绿 + 真实跑一轮 translate 后 pending 增量 ≥10（或给出达标的地域 N 配置）；卡附测试命令与结果、commit 号。 |
| | | | | **开发证据**：设计文档 `03e_翻译范围方案B.md` v1.0（Q7 方案 A 拍板）；代码 `translate.py` 改造（新增 `select_single_articles`/`_translate_and_save`/`translate_news`，`main` 选题编排，修复 `defaultdict` 未 import bug）；测试 `test_a21_translate_scope.py` 11 用例全绿 + `test_a12` 11/11 回归 + `test_a14` 13/13 回归。命令：`python test/test_a21_translate_scope.py -v`。commit `0a1f432`。<br>**PM 验收（2026-09-26）**：①亲测三套测试全绿（11/11 + 11/11 + 13/13）；②代码抽查与 03e §3 设计一致；③达标配置成立：`--top 5` × 2 地域 = 10 篇/轮 × cron 3 轮 = 30 篇/天 ≥ 10（验收标准 1 的「或」分支）。**遗留**：真实跑一轮未执行（验收时 Ollama 未运行）→ 并入 A7.1 端到端联调复核。 |
| K2 | A5.1+A5.3 发布模式三档 | 开发主管 | 重（标准轮次） | ✅ 完成（2026-09-26 PM 验收） | **做什么**：实现发布模式三档 `manual/semi_auto/auto`（需求依据 `02a_需求总览.md` §2.3 FR-D2/FR-D3 / `FR-D1_定时发布开发文档.md`，起步 semi_auto 已定）+ `config.yaml` 新增 `publish:` 段（mode/per_hour/region_order）。**怎么做**：① 先出 03x 设计文档（编号占 03 段下一个可用位），设计子问题至少覆盖：semi_auto 的人工确认入口依赖 A5.4（admin 手动发布尚未实现）——**请给出 A5.3/A5.4 的实现顺序建议**；`manual/semi_auto` 下 cron 触发 publish 的行为（应直接退出不发布）；`auto` 的直接发布路径；模式切换即改 config 即生效。② 拍板后实现 + 测试 `test/test_a5x_publish_mode.py`（覆盖三档行为）。**验收**：单测全绿；config 改 mode 后行为符合 `FR-D1` §10 / `03f` §9 验收（auto 自动发、semi_auto/manual 跳过等人工）；卡附证据。 |
| | | | | **开发证据**：设计文档 `03f_发布模式三档.md` v1.0（7 项拍板全定，无待拍板）；代码 `publish.py` 改造（新增 `load_publish_config` + `DEFAULT_PUBLISH_CONFIG`/`VALID_MODES`，`main()` 开头加模式判断 mode≠auto→return，`--limit` default 改 None 读 config per_hour，修复 `_coerce_scalar` 不认 `[]`）；`config.yaml` 新增 `publish:` 段（mode: semi_auto, per_hour: 3, region_order: []）；测试 `test_a5x_publish_mode.py` 9 用例全绿（TC1-TC8 + legacy 兼容）；A1.4 回归适配（CONFIG_TMPL 加 publish 段 + tc10b 改用 mode=auto 缺 base_url 场景）13/13 绿；A1.2 11/11 + A2.1 11/11 绿。命令：`python test/test_a5x_publish_mode.py -v`。commit `7739433`。<br>**PM 验收（2026-09-26）**：①亲测 4 套全绿（a5x 9/9 + a12 11/11 + a14 13/13 + a21 11/11）；②代码抽查：load_publish_config 兜底/顶层兼容/非法回退与 03f §3.4 一致，main() 模式判断在 site 配置读取之前（≠auto 连 site 都不读，符合 Q3）；③config publish 段落地核对无误；④03f Q1 正面回答 semi_auto 依赖问题（A5.3 先行，不依赖 A5.4）。**⚠️ 运营提示**：当前 mode=semi_auto 且 admin 写接口未建（A5.4），**暂无人工发布入口**——要发内容需临时切 auto 或等 K3 完成；建议 A7.1 联调时临时切 auto 发 1 篇测试稿。**遗留**：真实 cron 行为验证归 A7.1。 |
| K3 | A5.4+A6.2 admin 手动发布与否决 | 开发主管 | 重（标准轮次） | ✅ 完成（2026-09-26 PM 验收） | **做什么**：admin 新增手动发布（预览/单篇/批量）+ 人工否决（pending→rejected），打通 semi_auto 起步模式下的人工发布通道（需求 `FR-D1` §0 / `prd02a` §FR5；03f Q1 已定 A5.4 独立立项）。**怎么做**：① 先出 03g 设计文档（编号占 03 段下一个可用位），设计子问题至少覆盖：admin 从「全程只读」变「读写」的接口形态（预览/单篇发布/批量发布/否决的 API 设计）、复用 aggregator 的 `publish_one`/`site_api`（跨模块 import 方式）、写操作安全（admin 无鉴权 → 风险声明 + 最小化防护：仅 127.0.0.1 绑定重申、写操作日志）、rejected 后 auto 模式天然跳过的确认（select_pending 只选 pending）、批量发布的失败隔离（单篇失败不影响其余）② 待定项拍板后实现 + 测试 `test/test_a54_admin_publish.py` ③ `04a_admin使用说明.md` 同步新增写操作章节（04a 归开发主管域）。**验收**：单测全绿；admin 页面可预览/单篇发布/批量发布/否决；rejected 文章不被 auto 模式选中；卡附测试证据 + commit 号。 |
**开发证据**：设计文档 `03_开发文档/v2.0/03g_admin手动发布否决.md` v1.0（8 项拍板全定：Q7 同步、Q8 仅否决，项目主管拍板）；代码 `lib_admin.py`（新增，build_site_client/validate_status_transition/write_op_log）+ `main.py`（新增 POST /api/publish + POST /api/status，复用 publish_one）+ `static/index.html`（待选池勾选/发布/否决/批量按钮）+ `requirements.txt` 加 requests；测试 `test_a54_admin_publish.py` 9 用例全绿；回归 a14 13 + a12 11 + a21 11 + a5x 9 全绿；服务实测（只读 200 / 写接口 404·400 正确 / 非法流转拦截）。命令：`app/admin/.venv/Scripts/python.exe test/test_a54_admin_publish.py -v`。commit `f315edf`。<br>**PM 验收（2026-09-26）**：①亲测 5 套全绿（主测 a54 9/9 + 回归 a14 13/13 + a12 11/11 + a21 11/11 + a5x 9/9）；②代码抽查 5 项与 03g §7 一致——`lib_admin` 三函数、`main.py` 两写接口（`publish_one` 仅 401/403 上抛→终止整批语义成立；`/api/status` 仅 pending→rejected，非法流转 400）、`index.html` 按钮权限（仅 pending+有翻译可发布/勾选，否决带理由确认）、`select_pending` 只选 pending（L220-226，Q4 成立）、`04a` §8 写操作章节 + §8.4 安全声明已落（验收第 7 项达成）；③服务亲测：鉴权短路 / 非法流转拦截 / rejected 不被选中 / 日志留痕均复现。**结论：03g §9 验收 7 项全过，K3 通过**。 |
| K4 | A6.1 地域轮转选稿（FR-D6） | 开发主管 | 重（标准轮次） | ✅ 完成（2026-09-26 PM 验收） | **做什么**：发布侧地域轮转——`publish.py` 消费 `config.yaml` `publish.region_order`（K2 已落配置字段；2026-09-26 代码核实：仅 `load_publish_config` 读取、无消费逻辑），保证各地域发布均衡、避免单省霸榜（需求 `v1.0_需求清单.md` §5 `FR-D6` / `02a_需求总览.md` §2.3；v1.0 最后一条未开发 FR，2026-09-26 Max 拍板入板）。**怎么做**：① 先出设计文档 `03_开发文档/v1.0/FR-D6_地域轮转开发文档.md`（FR 命名不占 `03<字母>` 序列，按内容所属版本落 `v1.0/`，依据 `gov-a_文档规范.md` §2.6 + 「编号前缀仅作历史标识」原则；模板 §6.2），设计子问题至少覆盖：轮转算法（region_order 顺序轮转 vs 按比例配额；与 per_hour 组合语义——每轮总数 N 在地域间如何分配）、`region_order` 为空时行为（保持现状全池按 score 选取，空=不启用轮转）、候选池某地域无稿时跳过与回填、与三档模式判断（`main()` 前置判断）的先后关系、轮转不影响幂等（published/slug 键不变）；待定项标 ⬜ 升级项目主管拍板。② 拍板后实现 + 测试 `test/test_a61_region_rotation.py`（unittest + 临时目录风格，参考 test_a5x），用例至少覆盖：region_order 生效时按地域轮转选取、空 order 兜底与现状一致、单地域无稿跳过、与 per_hour 组合配额、published/rejected 不被选中。③ 涉及发布流程描述的 `03d`/`03f` 段落由开发主管同步（03 段归开发主管域）。**验收**：单测全绿 + 回归不破（a5x/a12/a14/a21）；config 配 region_order 后发布选稿按地域轮转、置空行为与现状一致；卡附测试命令与结果、commit 号。 |
| | | | | **开发证据**：设计文档 `03_开发文档/v1.0/FR-D6_地域轮转开发文档.md` v1.0（Q1-Q7 全定，均 L3 实现级自主拍板，无 L2 待拍板）；代码 `publish.py` 改造（新增 `rotate_by_region` 顺序轮转；`main()` auto 路径 region_order 非空走轮转/为空回退 `select_pending`；**顺带修复零依赖解析器**：`_coerce_scalar` 支持内联列表 + 新增 `_split_inline_list`——原把 `['萨省','安省']` 当字符串致轮转静默失效）；`config.yaml` region_order 注释更新；测试 `test_a61_region_rotation.py` **13 用例全绿**。命令：`python test/test_a61_region_rotation.py -v`。回归：a5x 9/9 + a14 13/13 + a12 11/11 + a21 11/11 全绿。**真实 pool 只读验证**：pending 19 篇，`rotate_by_region(pool, 6, ['萨省','安省'])` 输出 萨-安-萨-安-萨-安 严格交替（各取其地域最高分）；空 order 返回 0（回退 select_pending 生效）。文档同步：`03d` §2 选稿规则 + `03f` §3.5/§10 + `03` 段 index v4.7。commit `a9d5a68`。<br>**PM 验收（2026-09-26）**：①亲测 5 套全绿（主测 a61 **13/13** 含 TC6 main() 集成发布 4 篇地域交替 + 回归 a5x 9/9 + a14 13/13 + a12 11/11 + a21 11/11）；②代码抽查 4 项与 FR-D6 §3.3/§3.4/§3.6 一致——`rotate_by_region`（空 order/limit 防御 + 游标轮转 + max_seats 防死循环）、`main()` 分支（非空轮转/空回退 select_pending）、`_coerce_scalar`+`_split_inline_list` 内联列表补丁、`select_pending` 未动；③真实 pool 只读复跑：pending 19（萨9/安10），`rotate(6,['萨省','安省'])` 输出 萨→安 严格交替且各取地域最高分，空 order 返回 0；④文档同步核对：`03d` §2、`03f` §3.5/§10 v1.1 追加注记、config.yaml 注释均落。**结论：FR-D6 §9 验收 6 项全过，K4 通过；v1.0 25 条已无 ⬜ 未开发项**。 |
| K5a | A7.1 端到端联调·**只读链路 + 发布路径实证**（M3 收官·前半） | 开发主管 | 重（标准轮次） | ✅ 完成（2026-09-26 PM 验收） | **做什么**（K5 拆分第一半，2026-09-26 Max 拍板拆分）：① `run_daily.py` 只读链路（fetch→tag→rank→translate）**真实数据**跑通一次 ② 发布路径完整验证（选稿→推送→回写→幂等重跑）——site 未就绪期间用**本地 mock site**（按 `03c` §4 契约）实证 ③ 三档 gating + FR-D6 轮转真实路径实证。**怎么做**：见实测报告 §2/§3。**验收**：链路各环节日志完整、pending→published 全程推进、幂等重跑零重复、回归不破；卡附报告 + 回归结果。 |
| | | | | **开发证据**：实测报告 `03_开发文档/v1.0/A7.1_端到端联调实测报告.md` **v1.2**。**① 只读链路真实数据 ✅**：fetch +40 条（CBC 萨 26/安 14，失败 0，池 48→88，运行记录 status=success）；tag 话题 3 + 单篇 81；rank Top3 score 1.0/0.7/0.55 + 87 篇打分；translate **pending 19→29（+10，每地域 top5 达标）**，抽检 4/4 字段完整。**②③ 发布路径 + 幂等（本地 mock site）✅**：批量 5 篇全成、region_order 萨-安-萨-安-萨 严格交替（FR-D6）、site_id/slug 回写完整（FR-D5）、幂等重跑 **5/5 冲突对账零重复**（FR-D3）、semi_auto 跳过零推送（FR-D2）。回归 5 套 57 用例全绿。**无 commit**（验证轮次，代码零改动）。数据副作用已清理（mock 5 篇还原 pending、md 已删、config 还原）。<br>**PM 验收（2026-09-26）**：①亲测 5 套全绿（a14 17/17 + a61 13/13 + a5x 9/9 + a12 11/11 + a21 11/11，共 61 用例）；②只读链路真实数据复核：fetch +40（失败 0）、translate pending 19→29（+10 达标）；③发布路径/幂等 mock 实证核对；④报告 §2/§3 记录完整、数据副作用已清理。**结论：K5a 通过**（前半程=只读链路 + 契约级发布路径实证）。 |
| K5b | A7.1 端到端联调·**真实 site 推送**（M3 收官·后半） | 开发主管 | 重（标准轮次） | ✅ 完成（2026-09-26） | **做什么**：推真实 52sask.com 前端并验证前端可见 + 幂等重跑（`03c` §11 四项已于 2026-09-26 获 site 侧答复）。**前置**：site 答复暴露 **3 项代码适配缺口**——须先落适配再联调。**怎么做**：① 落 3 项适配（status 改 draft、查重接口适配、分类映射回填）② config 临时 auto ③ 发 1 篇核对回写 + 前端可见 ④ 重跑同篇验证幂等 → mode 还原 semi_auto。**验收**：真实 site 可见测试稿、回写正确、幂等零重复。 |
| | | | | **开发证据（commit `f8a4797`）**：**site 后端已上线**（`/api/health` 200），真实联调完成。**3 项适配已落**：① `build_payload` status→`draft`；② `get_post_by_slug` 两级查重；③ 幂等主路径前移「发布前查重」；④ `channel_categories` 回填（site 5 类）。**⚠️ site 答复实测纠正 3 处**：① 查重端点**只认 fullSlug**（裸 slug 404）；② **site 未强制 slug 唯一**——重复 POST 返 **201 新建第二篇**（非 409！故幂等必须 pipeline 侧查重）；③ 列表默认只含 published，draft 须 `?status=all`。**真实推送实证**：单篇发布 pending→published + 三字段回写完整 + **前端可见**（标题/正文/标签/分类齐全）；**幂等重跑零新 POST**（总数 22→22 不变）；批量 3 篇 + region_order **萨→安→萨严格交替**；分类映射 `商业地产`→财税金融（fullSlug 变 `finance-tax/news-xxx`）生效。测试 a14 **17 用例全绿**（+TC4a/TC4d/TC11/TC11b）；回归 a61 13 + a5x 9 + a12 11 + a21 11 全绿（共 61 用例）。site 探测稿已 DELETE 清理；config/score 已还原（仅 `channel_categories` 保留）。**结论：M3 端到端全链路（含真实前端推送）打通**。<br>**PM 验收（2026-09-26）**：①亲测 5 套 **61 用例全绿**（主测 a14 **17/17** 含新增 TC4a/TC4d/TC11/TC11b + a61 13 + a5x 9 + a12 11 + a21 11）；②真实 site 只读复验 4 项：`/api/health` 200、库中真实中文稿（分类 general 与重建映射一致）、查重端点**裸 slug 404 / fullSlug 200**（两级查重校准独立复核成立）；③代码抽查 3 项与 03c §11 一致——`site_api.get_post_by_slug` 两级查重（直连 + status=all 列表兜底）、`build_payload` status→draft + categoryIds/primaryCategoryId 回填、`publish_one` 幂等前移（发布前查重，失败保守跳过保持 pending 的 fail-safe）；④config 8 分类映射已随 `4fda8b6` 入库，开发证据（22→22 幂等 / 批量轮转 / 探测稿 DELETE 清理）核对无矛盾。**结论：K5b 通过，M3 收官——v1.0 端到端全链路（含真实前端推送）打通**；采纳盘点更新 **24 ✅ / 1 🔶 / 0 ⬜**（仅 FR-D1 cron 归 v2.1）。 |

---

## A. M3 发布功能（核心）

### A1. 落地单一系统重构（P0 · 发布地基）— ✅ 全部完成（2026-09-24）

代码已与定稿设计一致：单一状态机、pool 单一存储、title/body 命名、发布推送前端。

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| A1.1 | 字段重命名：全代码 `title_en`→`title`、`body_en`→`body` | 01§8.1 | ✅ 2026-09-24（含 pool 48 条存量迁移，备份 pool.bak.20260924_a11） |
| A1.2 | 单一状态机 `new→tagged→pending→published/rejected`（废弃 `translated`） | 01§8.2 | ✅ 2026-09-24（测试 test_a12_state_machine.py 11 用例全绿） |
| A1.3 | 废弃 `data/queue/`，翻译结果写回 pool 的 news | 01§8.3 | ✅ 2026-09-24（与 A1.2 同轮，translate.py 直写 pool） |
| A1.4 | publish 回写 `published` + `site_article_id` + `published_at` | 01§8.4 | ✅ 2026-09-24（A4 同轮完成：回写 `site_article_id`/`site_full_slug`/`site_published_at`；`published_at` 保留为原站时间语义，开发文档 03c） |
| A1.5 | front-matter 旧字段 `label`→`channel` | 01§8.5 | ✅ 2026-09-24（顺带加 tags 行、文件名带 id 防同名覆盖） |
| A1.6 | admin 待选池「中文标题」来源改读 news 的 `title_cn`（不再关联 queue） | 01§4.1.4 | ✅ 2026-09-24（/api/translation 也改读 pool；状态筛选下拉更新为五态） |

### A2. 扩大翻译范围（P0 · 保证稿源）

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| A2.1 | 翻译选题改为「真话题 + 高分单篇」（方案 B），每天 ≥10 篇 → **本轮看板 K1** | 01§8.6 / 讨论 task2 | ✅ 2026-09-26（03e 设计 v1.0 + translate.py 改造 + 测试全绿，PM 验收通过，commit 0a1f432；真实跑验证并入 A7.1） |

### A3. 存量数据迁移（P1 · 数据一致性）

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| A3.1 | queue 20 篇翻译结果回写对应 news（`title_cn`/`body_cn`） | 01§8.8 / task3 | ✅ 2026-09-24（20/20 回写，0 失败） |
| A3.2 | pool 的 `translated` 状态 → `pending` | 01§8.8 | ✅ 2026-09-24（pool 现为 tagged 28 + pending 20） |
| A3.3 | 清理残留 `data/topics.bak.*` | 01§8.8 | ✅ 2026-09-26（Max 手动删除，PM 核实 data/ 无残留；queue.migrated 归档亦已清） |
| A3.4 | 废弃/删除 `data/queue/` 目录 | 01§8.3 | ✅ 2026-09-24（归档为 `queue.migrated.20260924` 留底，确认后可手动删） |

### A4. 前端 API 对接（P0 · 前端已就绪 ✅）

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| A4.1 | `POST /api/articles` 幂等发布（`external_id` 键 + Bearer token） | 01§9 | ✅ 2026-09-24（契约更新为真实 `POST /api/posts`；幂等键改用 slug=`news id` 转连字符，external_id 方案作废；Bearer token 可选；见开发文档 03c） |
| A4.2 | 发布前 `GET /api/posts/slug/{fullSlug}` 查重 | 01§9 | ✅ 完成（2026-09-26 K5b，commit f8a4797）：实测校准为**两级查重**（直连 fullSlug + `?status=all` 列表过滤）；且因 **site 未强制 slug 唯一**，幂等主路径前移为**发布前查重**（见 `03c` §11） |
| A4.3 | 回写前端 `article_id` → news 的 `site_article_id` | 01§8.4 | ✅ 2026-09-24（响应 `_id`→`site_article_id`、`fullSlug`→`site_full_slug`，容错解析；测试 test_a14_site_publish.py 13 用例全绿） |

**待 site 侧确认 4 项** → **✅ 已答复（2026-09-26）并经 K5b 真实联调校准（纠正 3 处）**（详见 `03c` §11）：

| # | 事项 | site 侧答复 | **K5b 实测校准（以此为准）** |
|---|---|---|---|
| 1 | `status` 枚举是否含 `"published"` | 不含 | ✅ 属实：`draft` 通过（201）→ `build_payload` 改 `draft` |
| 2 | 有无 slug 查重接口 | 有：`GET /api/posts/slug/{fullSlug}` | ⚠️ **只认 fullSlug**（须 `categoryPath/` 前缀）；裸 slug → **404** |
| 3 | slug 重复提交的实际响应 | 报 **409** | ❌ **不成立**：**site 未强制 slug 唯一**，重复 POST → **201 新建第二篇** → 幂等改为**发布前查重** |
| 4 | 8 频道分类预建并回传 `_id` | 已回传 | ✅ **已对齐**：site 侧重建一级分类，现与本地 8 频道**一一对应**（7 类命名逐字一致 + 1 类「教育/教育就业」语义一致）；`channel_categories` 回填 8 条完整映射并实测验证（`03c` §11.2） |

> **✅ 跨域反馈已解决（撤回）**：原「本地 8 类 vs site 5 类不一致」问题已由 **site 侧重建分类对齐本地 8 频道**解决（2026-09-26）。`channel_categories` 8/8 全覆盖，真实发布验证 `本地突发`→`general/`、`商业地产`→`commercial-real-estate/` 分类正确落地。**本条无需产品侧再处理**（详见 `03c` §12）。

### A5. 发布模式三档（P1）

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| A5.1 | `config.yaml` 加 `publish:` 段（mode / per_hour / region_order）+ `site:` 段 | 01§6.1 | ✅ 2026-09-26（`site:` 段 A1.4 落地 + `publish:` 段 K2 落地：mode/per_hour/region_order，起步 semi_auto） |
| A5.2 | `SITE_API_TOKEN` 走环境变量（不落库） | 01§6.1 | ✅ 2026-09-24（随 A1.4：publish.py 读 `os.environ[token_env]`，未设置仅警告；`.env.example` 已建；token 不出现在 config/git/日志） |
| A5.3 | 实现 manual / semi_auto / auto 三档逻辑 | 01§9 | ✅ 2026-09-26（K2 落地 + PM 验收通过：load_publish_config 兜底/兼容/非法回退 + main() 模式判断前置（≠auto 连 site 都不读），测试 a5x 9/9 + 回归 3 套全绿，commit 7739433；真实 cron 行为验证归 A7.1） |
| A5.4 | admin 手动发布（预览/单篇/批量）+ 否决 | 01§9 | ✅ 完成（2026-09-26 PM 验收，看板 K3：POST /api/publish + POST /api/status + 前端按钮，见 K3 证据） |

### A6. 选稿与状态（P1）

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| A6.1 | 地域轮转选稿（按 region_tag 轮转，避免单省霸榜） | 01§9 | ✅ 完成（2026-09-26 PM 验收，看板 K4：`rotate_by_region` 顺序轮转 + main() region_order 分支 + 13 用例全绿 + 真实 pool 交替验证，commit a9d5a68） |
| A6.2 | `rejected` 人工否决态（下一轮跳过） | 01§9 | ✅ 完成（2026-09-26 PM 验收，看板 K3：POST /api/status 否决 pending→rejected，rejected 不被 select_pending 选中，见 K3 证据） |

### A7. 端到端联调（P0 · M3 验收）

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| A7.1 | `run_daily.py` 全链路（采集→打标→聚类→翻译→发布）真实数据跑通一次 | 03§M3 / task4 | ✅ 完成（拆 **K5a** 只读链路+发布路径实证 ✅ / **K5b** 真实 site 推送 ✅；commit f8a4797。**M3 收官：真实前端推送全链路打通**） |

---

## B. 监控（M4c）

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| B0 | admin 使用说明文档（04a_admin使用说明.md）+ doc/README 过时内容同步 | 本轮附带 | ✅ 2026-09-24（commit ec5785f；启动命令已实测 curl 验证） |
| B1 | admin 发布监控页（展示 `data/published/` 发布记录） | 01§9 / task5 | ⬜ |

---

## D. 长期 / 可选（P2）

| # | 待办 | 来源 | 状态 |
|---|---|---|---|
| D5 | 聚类路径澄清：`match_topic()` 闲置，删或留兜底 | 01§8.7 | ⬜ |
| D6 | admin 待选池补「城市」列（region_city） | 03§M4b | ⬜ |
| D7 | `tokenize_keywords` 加停用词过滤（清理 `of` 等噪音） | 讨论观察 | ⬜ |
| **D11** | **Ollama 健康预检**：`run_daily.py` / `tag.py` 启动时探 `GET {ollama.host}/api/tags`，服务不可用即 **fail-fast**（避免 tag 静默大面积「打频道失败跳过」——仅 WARNING 不报错，产生劣化数据） | 2026-09-26 A7.1 联调发现（K5a） | ⬜ 待办（2026-09-26 Max 拍板立卡） |
| D8 | **文档目录修复**：02 段文件名/位置与规范冲突（v1.0 需求文档编号错位、全局/版本需求未分层；`prd02b_生产上线` 被挪出 02 段），导致 DASHBOARD §3、02 index、01d/03g/01e 的引用断链 | 2026-09-26 目录盘点 | ✅ 已修复（v4.2-v4.9 编号体系重建消化：`prd<版本><字母>` 命名 + `02a` 上移段根 + `prd01b` 移交 03 段 `FR-D1` + `prd01c` 迁 v2.0；仅 `prd02b` v2.1 外置待收编 → 转 **D9**） |
| D9 | **v2.1 上线阶段收编规范**：`pm-0201_生产上线.md` 外置至 `01_项目管理/v2.1_上线/`（三阶段 v1.0→v2.0→v2.1 事实落地；需求层已认账：`v2.0_需求清单`/`02 index` 均注明 `prd02b` 归 v2.1） | 2026-09-26 Max 重构 | ✅ 2026-09-26（Max 授权项目经理代办，`gov-a` **v4.10** 收编：§0.2 分界表加 Ph3 上线 / §0.3 版本表新增 v2.1 行（M5 归位）/ §0.4 总览 / §2.1 目录树与划分原则 / §2.4 索引；`02 index` v4.10 `prd02b` 行链接与标注同步、`03a` 目录树注更新。**`v2.1_上线/` 定位为上线阶段专目录**，后续上线文档默认落此。*记录补正：本行 ✅ 化编辑因改名丢失，据 `gov-a` v4.10 实际完成状态补记*） |
| D10 | **01d/01e 改名断链**：两文件被改名 `pm-01b_kanban.md` / `pm-01a_项目计划.md`（2026-09-26 晚，内容完整、编辑基本保留），但 `dev01x` 前缀不在 `gov-a` §2.2 命名格式内、无变更说明，全仓 **32 处 `01d/01e` 引用断链**（gov-a×8、gov-b×7、本文件+pm-01a 自引用、DASHBOARD×4、01 index×3、prd02b/v1.0清单/prd02c/prd02d/prd02e/v2.0清单×6、03a×2、03f×1） | 2026-09-26 晚 盘点 | ✅ 2026-09-26（Max 裁决**转正方案②：采用 dev01x 新名**。`gov-a` **v4.11** 收编 `dev01<字母>` 命名（§0.2/§2.1/§2.2/§2.4/§2.5/§三/§6.3/§七 同步）；全仓 **28 处活引用**改指 `pm-01a`/`pm-01b`（gov-a 17 处编辑、gov-b 11、DASHBOARD 8、01 index 6、00 index/03a/03e/03f/03 index/v1.0 清单 3/v2.0 清单 2/prd02a-e 7、本文件自引用 2）；prd02b/c/d/e 4 处「来源：原 `01d_待办清单.md`」为历史记载**保留原编号**；两文件头部自引用同步。*另：产品经理会话同期叠加 `gov-a` v4.12（新增产品 v3.0），v4.11 收编内容完整保留*） |

---

## 执行顺序建议（v1.1，按当前进度）

**当前数据快照（2026-09-24 晚，admin overview 实测）**：pool 48 = tagged 28 + pending 19 + published 1；
queue 已归档；测试链路 pending→前端已具备，尚未真实联调。

```
✅ 已完成（2026-09-24，commit 5b13bd2/f784f38/2082ae6/cc2dbd2/ec5785f）：
   A1 全部（重构地基） + A3（除 A3.3 待确认） + A4.1/A4.3 + A5.2 + B0

▼ 第一步（P0，现在做）：
   A2.1 翻译范围方案 B   ← 稿源瓶颈：pending 仅 19 篇，每天发布很快耗尽
     ↓
第二步（P0，M3 验收）：
   A7.1 端到端联调（run_daily 全链路 + publish 真实发布 1 篇 + 重跑验证幂等）
   前置：site 侧确认 4 项（见 A4 区下）；联调步骤见开发文档 03c §8
     ↓
第三步（P1，收尾）：
   A5.1 publish 段 + A5.3 三档逻辑 + A6 选稿否决 + B1 发布监控页
     ↓
第四步（上线）：
   M5 部署 已迁 v2.1 上线阶段 → `01_项目管理/v2.1_上线/pm-0201_生产上线.md`（生产检查清单必含 SITE_API_TOKEN；gov-a 收编见 D9）
     ↓
长期：D5-D7（其余长期项已迁 Ph2：`prd02c`/`prd02d`/`prd02e`）
```

> 注：原「A1/A2/A4 合并一次重构」的建议已完成——A1/A4 主体同轮落地（cc2dbd2），
> 仅剩 A2（translate 选题范围）因改动面独立，单独立项执行。





## 8. 代码现状 vs 文档决策的差异清单（M3 动工前必读）

> 差异项已并入待办清单 [`pm-01b_kanban.md`](Projects/ws02_CMS/01_项目管理/v1.0/pm-01b_kanban.md)，此处保留作为开发时对照。

| #    | 差异              | 代码现状                                                     | 文档决策（M3 v1.1）                                   |
| ---- | ----------------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| 8.1  | 字段命名          | ~~全部代码用 `title_en`/`body_en`~~ ✅ 2026-09-24 已落地改为 `title`/`body`（含 pool 存量迁移） | 已定 `title`/`body` + `title_cn`/`body_cn`            |
| 8.2  | 状态机            | ~~new → tagged → translated~~ ✅ 2026-09-24 已切换：new→tagged→pending→published（rejected 待 admin 手动操作落地） | 单一状态机 new→tagged→pending→published/rejected      |
| 8.3  | 翻译落点          | ~~写 data/queue/~~ ✅ 2026-09-24 已改为直接写回 pool 的 news，queue 目录归档 | 直接写回 pool 的 news                                 |
| 8.4  | publish 回写      | ~~只改 queue 不回写 pool~~ ✅ 2026-09-24 断链已修复（pool 直达 published）；site_article_id 归 A4 | 回写 site_article_id + published_at                   |
| 8.5  | front-matter 字段 | ~~旧概念 `label`~~ ✅ 2026-09-24 已改为 channel + tags        | 应为 channel                                          |
| 8.6  | 翻译选题          | 仅真话题（~2 篇/天）                                         | 方案 B：真话题 + 高分单篇（≥10 篇/天）                |
| 8.7  | 聚类路径          | 实际用 similarity.py 算法；ollama match_topic() 闲置         | 明确是否保留 LLM 判定为兜底（或删除）                 |
| 8.8  | 数据存量          | pool 48（28 tagged+20 translated）、queue 20、published 0、topics 46、topics.bak 残留 | 存量迁移：queue 翻译结果回写 news，translated→pending |

---

## 
