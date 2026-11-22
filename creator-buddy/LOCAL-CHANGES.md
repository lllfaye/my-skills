# 本地修改记录（非上游内容）

本目录下的技能实体由 skills-manager 管理（registry 中 `source.type = import`，无上游同步）。
以下改动是本地打的补丁，**重新从 GitHub 导入会覆盖**，覆盖后按本文件重新应用。

最后更新：2026-11-22（第二轮：接入搜狗后备数据源）

---

## A. 数据源可用性总览

| 数据源 | 端点 | 状态 | 能否本地解决 |
| --- | --- | --- | --- |
| 公众号爆款（主） | `onetotenvip.com` | ❌ 无 A 记录，服务下线 | ✗ 代码无法绕过 |
| 公众号标题（备） | `weixin.sogou.com` | ✅ 已接入，实测可用 | ✓ 已完成 |
| 小红书笔记 | `redfox.hk` | ⚠️ 需 `REDFOX_API_KEY` | ✗ 商业 API |
| 全域搜索 | Agent Reach / Guaikei | ⚠️ 需二选一 | ✗ 需外部组件 |
| 抖音 | 自定义 CLI | ⚠️ 需 `DOUYIN_COMMAND` | ✗ 需用户提供 |
| 图表配图 | LabNana | ⚠️ 需 `LABNANA_API_KEY` | △ 可改本地 SVG 替代 |
| 视频字幕 | Groq / whisper | ⚠️ 需 key 或本地安装 | △ 可装 openai-whisper |
| 视频生成 | 火山方舟 Seedance | ⚠️ 需 `ARK_API_KEY` | ✗ 付费 |

---

## 1. `gzh-Skills/baokuan-article-analysis/scripts/daily_sector_trends.py`（改动最多）

### 1.1 假成功（第一轮）

**问题**：后端不可用时脚本静默写出 `items: []`，退出码 0。调用方会把空报告当成「该赛道没有爆款」。

**改动**：写完文件后统计总条数，为 0 时向 stderr 打印 `[BACKEND-UNAVAILABLE]`，
并 `raise SystemExit(2)`（全部关键词报错）或 `3`（部分取到但为空）。

### 1.2 搜狗后备源（第二轮，核心）

**问题**：主后端已死，整条「查赛道」链路不可用，而这是该技能最值钱的功能。

**改动**：新增 `fetch_sogou_keyword()` 及相关辅助函数，并在 `main()` 中做自动降级。

- 触发条件：`fetch_keyword()` 抛异常时自动调用，无需配置
- 解析字段：标题（`<h3><a>`）、摘要（`p.txt-info`）、公众号名（`span.all-time-y2`）、
  发布时间（`timeConvert('ts')`）
- 按 `--days` 本地过滤时间（注意：搜狗的 `tsn` 时间筛选参数已失效，会返回 0 条，
  所以必须抓取后本地过滤）
- `--fallback-pages N` 控制页数（默认 3，每页约 10 条，页间 sleep 0.8s）；`0` 关闭
- 反爬检测：页面含 `antispider` / `请输入验证码` 时抛出明确异常，不当成空结果
- 结构标记：每条 item 增加 `backend`（`api` / `sogou`）与 `metricsAvailable` 布尔值
- report 增加 `dataSources`、`metricsAvailable`，`source` 字段按实际来源改写

**报告层同步改动（避免 0 被当结论）**：

- 顶部 KPI：后备源下「最高阅读/最高分享」替换为「覆盖账号/数据能力」
- 赛道卡片 KPI：替换为「覆盖账号/覆盖日期/数据能力」
- `build_markdown`：后备源下不输出「最高阅读」，改写数据来源说明
- 结束时打印 `[FALLBACK-SOGOU]`，明确「无阅读数、不可判定爆款」

### 1.3 搜狗的两个限流坑（重要）

**坑一：UA 指纹**

UA 里带 `(KHTML, like Gecko)` 会被判定为伪装爬虫，返回约 5KB 验证码页
（含 `antispider.min.js`），而非正常结果页。已改用简版 UA。

对照实验（同一时间窗口、同一 query、只改 UA）：

| 请求头 | 结果页大小 | 结果条数 | 是否拦截 |
| --- | ---: | ---: | --- |
| 最小头（仅 UA + Referer） | 31 KB | 10 | 否 |
| + `Accept: */*` | 31 KB | 10 | 否 |
| + `Accept-Language` | 31 KB | 10 | 否 |
| + `Accept: text/html` | 31 KB | 10 | 否 |
| UA 含 `(KHTML, like Gecko)` | 5 KB | 0 | **是** |

**坑二：IP 级限流**

短时间内密集请求后，搜狗会对来源 IP 返回 **HTTP 302**（162 字节），
连 curl 也一样。此时无论怎么改头都无效，只能等冷却（实测约数分钟到数十分钟）。

已加对策：
- `_sogou_get` 内全局节流：两次请求间隔 ≥2.5s + 0–0.6s 随机抖动
- `_sogou_get_guarded`：遇验证码页退避 12–20s 重试一次
- 仍失败则抛明确异常，提示「5–10 分钟后重试 / 降低 `--fallback-pages`」，
  不当成空结果

**使用建议**：单次运行控制在 2–3 个关键词 × 2 页以内；同一小时内不要反复跑。
需要连续多次查询时，改用 `WebSearch` 更稳。

**实测记录**：

- 成功样本（离线留存 `33KB` 页面）解析 **10/10 条**，标题、公众号名、发布时间、
  摘要、跳转链接全部正确，`backend=sogou`、`metricsAvailable=false`、`dataScore=0`
- 联网实测：`--sector "民生政策=养老金" --days 7` → 6 条，覆盖 1 天；
  `--sector "星座=星座"` → 8 条（含「星座商厦」等噪声）
- 3 页抓取（限流前）：养老金 30 条/近 7 天 30 条，星座 29 条/近 7 天 16 条

**解析逻辑已验证与网络无关**：把网络层替换成离线 HTML 后，`fetch_sogou_keyword()`
仍能正确解析全部字段。所以出现失败时只可能是网络/限流，不是正则失配。

### 1.4 全链路兜底：自动生成检索清单

**问题**：两个数据源都挂时，脚本只打印错误并 `exit 2`，用户/agent 拿到的是死路。

**改动**：新增 `build_search_plan()`。当 `total_items == 0` 时，向输出目录写入
`search-plan.md`，内容为按赛道/关键词展开的可执行检索式，并附必填字段表与三条硬性约束
（不估算阅读量、声明来源、不把搜不到当没有）。

**实测**：`--fallback-pages 0` 强制失败后，`search-plan.md` 正常生成，退出码 2，
stderr 中直接给出该文件路径。

已知噪声：搜狗为全文检索，「星座」会混入「星座商厦」「卫星星座」。关键词越具体越好。

## 2. `gzh-Skills/gzh-explosive-content-detector/scripts/fetch_gzh_trends.py`

**问题**：与 1.1 同类假成功，且 DNS 报错无可执行建议。

**改动**：
- 写出文件后若总条数为 0，打印 `[BACKEND-UNAVAILABLE]` 并 `sys.exit(2)`
- `except` 中若错误含 `nodename` / `Name or service not known` / `timed out`，
  追加后端不可用说明并 `sys.exit(2)`，否则维持 `sys.exit(1)`

**注意**：本脚本**未接入搜狗后备源**。它依赖的低粉爆款榜、阅读量榜等结构化指标
搜狗无法提供，硬接只会产出一份看起来像榜单、实则没有排名的假报告。
需要赛道数据请直接用 `daily_sector_trends.py`。

## 3. `gzh-Skills/global-content-search/src/xiaohongshu/*.js`（三处）

**问题**：无可用后端时 `catch` 只打印 JSON（含 `results: []`）就正常结束，退出码 **0**。
下游只看到空数组，会判断成「该关键词没有内容」，而不是「没连上数据源」。

**改动**：`search-cli.js`、`detail-cli.js`、`post-cli.js` 的 `catch` 末尾各加一行
`process.exitCode = 2;`（用 `exitCode` 而非 `process.exit(2)`，避免截断 stdout 的 JSON）。

**实测**：`node search-cli.js --keyword 星座` → 退出码 2。

## 4. 两处小红书脚本的报错文案

文件：`gzh-Skills/xhs-hotnotes/scripts/fetch_xhs_hot_articles.py`、
`xhs-Skills/space-xhs-hotspot/scripts/fetch_xhs_hot_articles.py`

**改动**：缺 `REDFOX_API_KEY` 时除原有提示外，补充「不付费的替代做法」与
「不要把无密钥当成没有内容」。退出码本来就是 1，未改。

## 5. `gzh-Skills/space-chart-image/scripts/generate_chart.py`

**改动**：缺 `LABNANA_API_KEY` 时补充替代路径（手写 SVG / HTML + Chart.js 截图），
说明「本脚本是 AI 美化视觉风格，不是能不能画图」。

## 6. 根 `SKILL.md`

**改动**：更新「运行环境与后端状态」章节
- 新增搜狗后备源一行，标明已接入
- 新增「搜狗后备源的边界」小节：能拿什么、拿不到什么、适用/不适用、噪声、页数控制
- 降级规则从 4 条扩到 5 条，新增：看到 `[FALLBACK-SOGOU]` 必须写明「标题级数据，无阅读量」
- 后端表补充视频字幕（Groq/whisper）与视频生成（ARK）两行

---

## 未修复项（需外部条件，非代码可解）

| 项 | 原因 | 需要的动作 |
| --- | --- | --- |
| 公众号爆款**指标**（阅读/在看/分享） | `onetotenvip.com` 无 A 记录 | 作者换后端，或购买新榜/清博类 API |
| 小红书数据 | 红狐数据为商业 API | 配置 `REDFOX_API_KEY` |
| 全域搜索 | 依赖 Agent Reach 或 Guaikei | 装 Agent Reach，或配置 `GUAIKEI_API_TOKEN` |
| 抖音 | 无内置后端 | 配置 `DOUYIN_COMMAND` |
| 图表配图 | LabNana 图像 API | 配置 `LABNANA_API_KEY`，或改用本地 SVG |
| 视频生成 | 火山方舟付费 | 配置 `ARK_API_KEY` |

排查记录：`onetotenvip.com` 的 A 记录在 8.8.8.8 / 1.1.1.1 / 223.5.5.5 / 119.29.29.29
四个解析器下均无返回，DoH 同样无结果，`www` / `api` / `skill` 等九个子域亦无解析，
HTTP 与 HTTPS 均返回 `000`。判定为服务下线或解析记录被移除，非本地网络问题。

搜狗验证记录：`weixin.sogou.com/weixin?type=2` 返回 200，单页 10 条；
`tsn=1/2/3/4`（日/周/月/年）全部返回 0 条，说明该时间筛选已失效，须本地按时间戳过滤。
