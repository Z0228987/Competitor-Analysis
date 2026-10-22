# 本次需要上传到 z0228987/Competitor-Analysis 的文件（2026-10-23）

## 1. 修币种（会导致线上数值错误，优先传）

| 文件 | 放到 | 说明 |
|---|---|---|
| `fact_financials_combined.csv` | `data/` | 线上 NASN 标成 HKD，实际以人民币披露 → 前端乘 0.855，NASN 全部财务数被低估 14.5%。本地版已改回 CNY，另外含**立讯 2026Q1 资产负债表修正**（线上把 2025Q4 的数当成 2026Q1 重复填了，6 个 BS 科目） |
| `fact_sales_history.csv` | `data/` | NASN 币种 HKD → CNY（8 行） |
| `fact_profit_history.csv` | `data/` | NASN 币种 HKD → CNY（7 行） |

**不要**覆盖 `data/market_data_verified.csv` 和 `data/market_history.csv`：线上的比本地新（线上快照 2026-10-22，本地 2026-10-17），而且 NASN 的**股价确实是港币计价**，currency=HKD 是对的。

## 2. 补 Annual Deep Dive 数据

| 文件 | 放到 |
|---|---|
| `IMPORT_annual_all.csv`（已改，207 行） | `data/` |

## 3. 前端

| 文件 | 放到 |
|---|---|
| `index.html`（根目录那份） | 仓库根 |
| `js/chart.umd.min.js`、`js/papaparse.min.js`、`js/chartjs-plugin-datalabels.min.js` | 已有 `js/` 目录（和 `market_ui_upgrade.js` 放一起） |

---

# 本次补录的 Annual 数据与来源

## customer · Top1 Customer（第一大客户收入占比，%）

| 公司 | 2023 | 2024 | 2025 | 来源 |
|---|---|---|---|---|
| BTL | 38.7 | 45.1 | 39.0 | `Competitor Analysis_BTL.pdf` 招股书补充材料 |
| NASN | 59.2 | 67.2 | 47.2 | 港交所招股书 / 2026 中报 |
| Tuopu | 39.8 | 28.4 | 25.7 | 2025 年报（最大客户特斯拉） |
| Luxshare | 75.2 | 70.74 | 56.68 | 2025 年报 + 港股招股材料 |
| Baolong | — | — | 10.61 | 联合资信 2026 跟踪评级（第一大客户奇瑞） |
| HASCO | — | — | 15.7 | 公司 IR《2025 年度财务分析报告》（上汽大众 276.6 亿元） |

Top5 补充：Tuopu 2023 = 63.4；NASN 2023 = 97.9；Luxshare 2023 = 82.4。

## profile · Plants / RD_Centers / Countries

| 公司 | 新增 | 来源 |
|---|---|---|
| HASCO | Plants 475、Countries 19 | 2025 年报「布局优势」：国内 378 个研发/制造/服务基地 + 海外 97 个；海外 18 国 + 中国 |
| Luxshare | Plants 100、RD_Centers 28、Countries 30 | 2026 跟踪评级（近 30 国超 100 个生产基地）+ 2025 年报（28 个自设研发中心） |
| Tuopu | Countries 11 | 2025 年报：100 余个制造工厂分布于 11 个国家、42 个城市 |

## region · NASN 2025（原先完全没有）

China 603.27 / Overseas 11.24（CNY mn）。来源：招股书「2025 年海外收入 1,124 万元、占 1.8%」，总收入 614,506 千元倒算。

---

# 仍然缺的项（没有可靠公开披露，未编造）

- **customer Top1**：Baolong / HASCO 只有 2025 一年，2023–2024 未披露
- **profile Plants / RD_Centers**：所有公司只有 2025 年；HASCO 的研发中心数未单独披露（年报把研发、制造、服务基地合并成 378 个报）
- **profile Countries**：BTL、NASN 未披露
- **region**：Aumovio 全空（德国上市，分部口径不同）；NASN 只有 2025
- **rd 人均薪酬**：只有 Baolong、Tuopu 有；其余公司年报未单独披露研发人员薪酬，不推算
- **segment**：Aumovio 全空

# 一处需要你确认的口径冲突

- **HASCO 员工数**：Annual 的 `Employees` 2024 = 79,409、2025 = 81,306 是**全口径**（境内在职 + 海外在职，2024 = 52,299 + 27,110）；而主看板 Headcount 明细合计 54,621 是**境内/母公司口径**。两个数都对，只是口径不同。要不要在卡上加一行说明？
- **Luxshare 研发人数**：CSV 现有 2025Q4 = 34,357，2025 年报披露「专职研发人员 48,765 名」。我没改，需要你定用哪个。
