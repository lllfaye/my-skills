# 本地修改记录（非上游内容）

本目录下的技能实体由 skills-manager 管理（registry 中 `source.type = import`，无上游同步）。
以下改动是本地打的补丁，**重新从 GitHub 导入会覆盖**，覆盖后按本文件重新应用。

最后更新：2026-11-22

---

## 1. `gzh-Skills/baokuan-article-analysis/scripts/daily_sector_trends.py`

**问题**：后端不可用时脚本静默写出 `items: []` 的 `data.json` / `report.html`，退出码 0。
调用方会把空报告当成“该赛道没有爆款”，属于**假成功**。

**改动**：写完文件后统计总条数，为 0 时向 stderr 打印 `[BACKEND-UNAVAILABLE]` 与降级建议，
并 `raise SystemExit(2)`（全部关键词都报错）或 `3`（部分取到但结果为空）。

## 2. `gzh-Skills/gzh-explosive-content-detector/scripts/fetch_gzh_trends.py`

**问题**：同上，且 DNS/连接失败的报错没有可执行建议。

**改动**：
- 文件写完后若总条数为 0，打印 `[BACKEND-UNAVAILABLE]` 并 `sys.exit(2)`
- `except` 分支中，若错误信息含 `nodename` / `Name or service not known` / `timed out`，
  追加后端不可用说明并 `sys.exit(2)`，否则维持 `sys.exit(1)`

## 3. 根 `SKILL.md`

**改动**：新增「运行环境与后端状态（2026-11-22 实测）」章节，含
- 运行约定：脚本路径相对技能根目录，执行前需 `cd`
- 各后端可用性表（哪个死了、缺哪个密钥）
- 降级规则四条：退出码 2 ≠ 没有爆款；改用 WebSearch 并声明来源；不编造数字；不静默降级

---

## 未修复项（需要外部条件，非代码问题）

| 项 | 原因 | 需要的动作 |
| --- | --- | --- |
| 公众号爆款数据 | `onetotenvip.com` 无 A 记录（NS 仍在阿里云 DNS，但无解析记录） | 作者更换后端，或自建数据源 |
| 小红书热门笔记 | 红狐数据为商业 API | 购买并配置 `REDFOX_API_KEY` |
| 全域搜索 | 依赖 Agent Reach 或 Guaikei | 装 Agent Reach，或配置 `GUAIKEI_API_TOKEN` |
| 抖音 | 无内置后端 | 配置 `DOUYIN_COMMAND` |
| 图表配图 | LabNana 图像 API | 配置 `LABNANA_API_KEY` |

排查记录：`onetotenvip.com` 的 A 记录在 8.8.8.8 / 1.1.1.1 / 223.5.5.5 / 119.29.29.29
四个解析器下均无返回，DoH 同样无结果；`www` / `api` / `skill` 等九个子域亦无解析。
判定为服务下线或解析记录被移除，非本地网络问题。
