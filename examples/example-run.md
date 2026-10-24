# 前车之鉴

**问题**：给部署在多个实例后面的 Node.js / Express Web 服务加限流，先护住登录与写接口，避免突发流量打穿。

**成功标准**：1）超限返回 429 且带可用的限流响应头；2）同一客户端跨实例计数一致，不因实例数放大限额；3）正常流量接入后无明显延迟劣化；4）阈值与作用范围（按 IP / 按登录用户）一处可配置。

**不做**：不做 DDoS 清洗 / WAF；不改造反向代理与网关层；不做租户计费配额；不重写现有鉴权与路由。

## 检索记录

- **预算**：standard（web ≤3、GitHub ≤2）。本轮实际发出 web 3 条单一问题 query、GitHub 2 条，用满未超；另按 search-templates §5 对本轮已命中的 2 个仓库做仓库元数据核验（属读正文，不新增检索次数）。
- **web｜做法**：rate limiting Express best practice → 有结果（8 条）：以「如何在 Node / Express 里加限流」的教程页为主（coreui.io、OneUptime 博客镜像、腾讯云技术百科），夹杂若干 npm 包页。
- **web｜坑**：rate limiting Express pitfalls → 有结果（8 条），本轮质量最高：两个 StackOverflow 的集群 / 负载均衡下限流不生效提问、express-rate-limit 的 issue 与 wiki 排障页、两篇「以为加了限流其实没有」的实践复盘。
- **web｜取舍**：token bucket vs sliding window tradeoff → 有结果（8 条）：arcjet 的四种算法对比、Azion 的算法讲解、redis.io 教程原文（.md）、engineering-handbook 的 tradeoff 章节。
- **GitHub｜找 SKILL.md 形态**：site:github.com rate limiting SKILL.md → 命中 EndogenAI/dogma、SkillMedev/skills、curiositech/windags-skills 等 SKILL.md 路径；同批还混有「客户端被上游限流」的 issue / PR，方向不同，未计入。
- **GitHub｜找现成实现**：site:github.com express-rate-limit rate limiting store → 命中 express-rate-limit 主仓库，以及该组织下三个 store 实现（cluster-memory-store / rate-limit-postgresql / rate-limit-memcached）。
- **证据强度现状**：web 级来源只到标题 / 摘要，本轮没有据此判任何对象「可直接用」；两个仓库的维护状态与许可来自仓库元数据（读正文），其余仓库只到标题 / 摘要。
- **读正文的入口**：https://api.github.com/repos/express-rate-limit/express-rate-limit 与 https://api.github.com/repos/express-rate-limit/cluster-memory-store，两次均 HTTP 200。

### 命中记录（search-templates §4 格式，与「前车之鉴」条目分开放）

- [GH:express-rate-limit/express-rate-limit] — 需改造 — 证据强度：正文已读 — Express 限流中间件主干；元数据显示 MIT 许可、最近推送 2026-10-23，但跨实例计数与反代取真实 IP 要自己补
- [GH:express-rate-limit/cluster-memory-store] — 需改造 — 证据强度：正文已读 — 只把计数共享给同机 node cluster 的各 worker，跨机器仍需外部 store
- [GH:express-rate-limit/rate-limit-postgresql] — 仅作参考 — 证据强度：仅标题 — 跨实例共享 store 的现成选项之一，引入与否取决于本服务已有基础设施
- [GH:express-rate-limit/rate-limit-memcached] — 仅作参考 — 证据强度：仅标题 — 同上，另一种共享 store 选项，迁移成本不同
- [GH:SkillMedev/skills] — 仅作参考 — 证据强度：仅标题 — rate-limit-handler 的 SKILL.md，只看到路径与标题，未读正文
- [web:https://github.com/express-rate-limit/express-rate-limit/wiki/Troubleshooting-Proxy-Issues] — 证据强度：仅标题 — 官方把「代理后取不到真实客户端 IP」单列成排障页，说明这是该中间件的高发配置错误
- [web:https://github.com/express-rate-limit/express-rate-limit/issues/468] — 证据强度：仅标题 — 「限流总比配置多放行 1 次」，阈值边界语义必须先定义
- [web:https://stackoverflow.com/feeds/question/73219781] — 证据强度：仅摘要 — 集群 + 负载均衡下限流不生效的提问，佐证多实例是本问题的高发形态
- [web:https://arcjet.com/learn/rate-limiting-guide] — 证据强度：仅摘要 — token bucket / leaky bucket / sliding window / fixed window 四种算法的对比综述

## 前车之鉴（≤5 条）

1. 多实例 / 多 worker 下限流计数必须进共享 store：官方生态专门提供了「cluster 内共享」与「外部库共享」两类 store，说明「换 store」是这条路线的既定动作 [GH:express-rate-limit/cluster-memory-store]
2. 反向代理 / 负载均衡之后，官方把「取不到真实客户端 IP」单列成排障页，说明这是高发配置错误（仅标题级证据）；「取不到真实 IP 时计数会全落在代理上」属 [先验] 推断 [web:https://github.com/express-rate-limit/express-rate-limit/wiki/Troubleshooting-Proxy-Issues]
3. 阈值边界存在差一语义（「总比配置多放行 1 次」），验收前必须写死「第 N+1 次是否拒绝」 [web:https://github.com/express-rate-limit/express-rate-limit/issues/468]
4. 四种算法的取舍集中在突发容忍、边界精度与状态存储成本，没有脱离流量形态的通用最优 [web:https://arcjet.com/learn/rate-limiting-guide]
5. 该主题在 GitHub 上已有 SKILL.md 形态的资产，但本次只命中文件路径与标题、未读正文，无法判定可直接复用 [GH:SkillMedev/skills]

## 收敛到（≤3 条）+ 推荐理由

- 保留 1、2、3：
  - 1：相关性最高——它是唯一能区分 A / B 两条路线的事实（store 放哪），也是成功标准第 2 条的直接依据；可操作性强，共享 store 已经现成。
  - 2：直接决定计数键（IP 还是登录用户）与是否需要额外配置，且与本任务「不改造网关层」的边界咬合；漏掉会出现限流看似生效、实际全站共用一个桶。
  - 3：直接决定成功标准第 1 条的验收口径，边界差一会把「生效」误判成「没生效」，属可执行层面的必要条件。
- 淘汰 4、5：
  - 4：不适用：本任务要定的是计数 store 放在哪条路线，算法取舍不改变 A / B 任一条路线是否成立，属实现层参数而非路线决策。
  - 5：证据弱——只命中文件路径与标题、未读正文，按命中判定只能到「仅作参考」，不得作为候选主干。
- 口径说明：以上保留 / 淘汰理由只引用前车之鉴 1–5 的内容，未引用任何未列出的信息或印象中的数据。

## 候选流程

- **A**：主干：应用内接入现成限流中间件，把计数 store 换成跨实例共享实现，计数键与阈值集中配置，作用域先覆盖登录与写接口｜优点：覆盖成功标准第 1、2、4 条，跨实例一致与一处配置同时成立，且主干是现成的、不需自研计数逻辑｜风险：触发「部署在反代后但未配置信任真实客户端 IP」→ 后果「计数全落在代理 IP 上，全站共用一个桶，正常用户被误伤」→ 对策：规避（上线前用固定代理头验证计数键，把信任代理配置纳入验收）；触发「每请求多一次共享 store 往返」→ 后果「p99 上升，可能触碰成功标准第 3 条」→ 对策：监控（把 store 调用耗时纳入指标，超阈值再考虑本地缓存分层）；触发「阈值边界差一（前车之鉴 3）」→ 后果「验收时判定限流没生效」→ 对策：明确接受（在成功标准里写死 N+1 次才拒绝，验收按同一口径）
- **B**：主干：沿用中间件默认的进程内内存 store，各实例各自计数，不引入外部存储｜优点：覆盖成功标准第 3 条，零外部依赖、改动与延迟最小｜风险：触发「实例数 > 1」（本任务的前提）→ 后果「每实例各自放行，总额限被放大 N 倍，登录接口仍可被打穿」→ 对策：无有效对策，只能明确接受「成功标准第 2 条不成立」；触发「先用 B、之后再补共享 store」→ 后果「二次改动，前期最小改动的收益被吃掉」→ 对策：规避（一开始就走 A，不中途换 store）

## 选定：A

**理由**：A 覆盖成功标准第 1、2、4 条且可逆——store 侧不达标时切回内存 store 是单点回退；A 与 B 的差别只在 store 形态，选 A 不需要自研，代价只是多一个外部 store 的依赖，与本任务规模同量级。

**放弃 B 因为**：风险无对策——B 唯一风险的触发条件（实例数 > 1）就是本任务的前提，其对策只能是「接受成功标准第 2 条失败」；这不是降级方案，而是在核心验收项上不成立。

## open questions / 未检索到

- 需向用户确认：真实实例数与部署形态（同机 cluster 还是多机器 / 容器）、是否已有 Redis 或共享缓存、是否部署在反向代理之后——均属项目内部信息，不在检索范围。
- 未能检索到 GitHub 上可直接复用的限流 SKILL.md 实现的可用方案 [未检索到]
- 未查（web 预算 3/3 已满）：以登录用户 ID 而非 IP 作为计数键的现成实现
- 未查（web 预算 3/3 已满）：三类共享 store 在该服务现有基础设施下如何取舍
- [GH:express-rate-limit/rate-limit-postgresql] 与 [GH:express-rate-limit/rate-limit-memcached] 只有标题级证据：接口形态、维护状态、与现有基础设施的适配成本均未核对 [未验证]
- 关于 429 之外应带哪些限流响应头、字段名按哪个标准版本走的业界做法，本次检索无可靠来源，见 open questions

---

**示例说明**：本文件对应「场景 = 新对话、无历史上下文；档位 = standard（多步骤实现 / 跨文件改动）」的一次完整记录；文中 [GH:owner/repo] / [web:URL] 标签全部来自该次会话的真实检索（query 与命中见「检索记录」），证据强度按要求只写在检索记录里、不进「前车之鉴」条目，[未验证] 项不参与任何结论；本文件只做决策前的经验输入，不含执行步骤与实现计划。

**施工说明**（不属于示例正文，故置于页脚）：本示例的检索记录经过一次重做——早先一条把「做法 + 坑 + 取舍」合并的复合 query 已作废，最终只保留三段式三条单一问题 query 的结果；早先命中的 MarcoNasi/TRAE-Skills 现已改名为 MarcoNasi/AI-Skills（重定向），未进入本轮检索记录。
