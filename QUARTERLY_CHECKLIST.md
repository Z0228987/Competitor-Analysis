# 每季度财报更新 · 操作待办卡

> 适用项目：`financial_auto_project_v3`（本地流水线） + `Z0228987/Competitor-Analysis`（GitHub 看板）
> 用法：每个季度复制「第二部分 打勾清单」到你的笔记里，照着一条条打勾。
> 详细原理与口径请看 `PROJECT_KNOWLEDGE.md`。

---

## 一、本季度基本信息（开工前先填）

| 项目 | 填写 |
|---|---|
| 报告期（数据键） | `2026Q3` ← 按实际填，格式必须是 `YYYYQ[1-4]` |
| 上季度（归档用） | `2026Q2` |
| 操作人 | |
| 开始日期 | |
| EUR 汇率（看板用，默认 8.35） | |
| HKD 汇率（仅 NASN 股价/市值，默认 0.855） | |

---

## 二、打勾清单（主线）

```
【阶段 A · 准备材料】
[ ] A1  新季度 PDF 放入正确目录
        - AUMOVIO .............. 项目根目录
        - 深交所 BTL/Baolong/Luxshare ... 01_raw_data_00/
        - 上交所 HASCO/Tuopu ........... 01_raw_data_600/
        - NASN ................. 不走 PDF，见阶段 E
[ ] A2  上季度 02_processed_data/final_facts/*.csv
        **复制**归档到 final_facts/_history/<上季度>/，命名如 fact_IS_all_2026_Q2.csv
        共 6 个：fact_financials_all / fact_IS_all / fact_BS_all / fact_CF_all
                 / fact_nasn / aumovio_financials
        ⚠（别用剪切：fact_nasn.csv 和 *_eur.csv 仍被 Step 4b/4d 读取，
          移走会直接报错；Step 4 本来就会覆盖 final_facts/）
        （不做这步 → 历史期间会被覆盖丢失）

【阶段 B · 跑流水线】
[ ] B1  双击根目录 run.bat
        （需要脚本自动下载 PDF 时用命令行 run.bat --download，不稳，建议手动下载）
[ ] B2  终端逐个 Step 确认无 skip / 无报错
        Step 1 清理 → 2/3 PDF 提取 → 4 标准化 → 4b NASN → 4c AUMOVIO
        → 4d 历史趋势表 → 4e 瘦身 → 4f 累积归档 → 5 质量报告

【阶段 C · 数据检查】
[ ] C0  （三季报专有）查 fact_financials_all.csv 是否出现 period=2025Q3 的 IS/CF 行
        - 出现 → Q3 的 YoY 基数已自动补齐（Working Capital 除外，BS 补不出）
        - 未出现 → derive_comparison_period 缺「2025 年前三季度（1-9 月）」列头正则
        详见 PROJECT_KNOWLEDGE.md 8.1 节
[ ] C1  打开 02_processed_data/dashboard_ready/fact_financials_combined.csv
        - 新期间每家公司都有行
        - currency：AUMOVIO=EUR，其余（含 NASN）=CNY
        - 符号：费用为负、亏损为负、毛利 IS_GROSS_PROFIT 必须为正
        - 无重复行（去重键 company+statement_type+period+item_code）
[ ] C2  随机抽查质检
        venv\Scripts\python.exe 03_scripts\qa_spot_check.py -n 2
        退出码 0 = 全过；1 = 有 FAIL 需排查；2 = 无可比样本
        ⚠ 本机 03_scripts/ 下目前没有这个脚本（在另一台机器，同 patch_nasn_selling.py）
          替代方案：用会计恒等式自校，不依赖第三方源，反而更硬。逐家逐期验：
            资产总计 = 负债合计 + 所有者权益合计
            利润总额 = 营业利润 + 营业外收入 − 营业外支出
            净利润   = 利润总额 − 所得税费用
            净利润   = 归母净利润 + 少数股东损益
            经营净额 = 经营流入小计 − 经营流出小计
            期末现金 = 期初现金 + 现金净增加额
          容差 1%；再加符号检查（营收/成本/费用/总资产为正）与 YoY 合理性
[ ] C3  有 FAIL 时：
        venv\Scripts\python.exe 03_scripts\qa_spot_check.py --official --download
        用巨潮官方 PDF 定位页码，判断是「我们解析错」还是「第三方口径不同」
[ ] C4  【必查】Trend 两张表是否包含新季度，且历史期没丢
        文件：02_processed_data/dashboard_data/fact_sales_history.csv
              02_processed_data/dashboard_data/fact_profit_history.csv
        验收标准（以 2026Q3 为例，共应 70 行）：
          - 5 家 A 股每家 11 行：2021–2025 FY + 2025Q1/2025H1/2025Q3
                                        + 2026Q1/2026H1/2026Q3
          - AUMOVIO 6 行、NASN 9 行（走各自数据源，不受影响）
          - 营收累计口径递增：Q1 < H1 < Q3
        ⚠ 这两张表的 period 规则与 combined **故意不同**，不要改成一致：
            combined   → 存 2026Q2，由前端 pLabel 显示成 H1
            history 表 → _norm_period() 直接存字面值 2026H1
[ ] C5  C4 不达标时的单步重生命令（Step 4d）
        venv\Scripts\python.exe 03_scripts\build_dashboard_tables.py
        常见原因：load_fact() 没读到 _history 归档
        → final_facts/ 当季表每次运行被整体覆盖，只剩「本季+对比期」，
          A 股的 Q1/H1 全在 _history/<季度>/ 里，必须一并读入；
          所以 A2 归档这步漏做，Trend 图会直接缺柱子（2026-11-05 已修）

【阶段 D · 推送到看板】
[ ] D1  复制 3 个 CSV 到 Competitor-Analysis/data/——**三个都要推，缺一不可**
        只推 combined 会导致：Historical Trends 没有新季度柱子，
        且期间下拉出现两个重复的 H1（2026-11-05 就踩过这个坑）
        dashboard_ready/fact_financials_combined.csv → data/fact_financials_combined.csv
        dashboard_data/fact_sales_history.csv        → data/fact_sales_history.csv
        dashboard_data/fact_profit_history.csv       → data/fact_profit_history.csv
[ ] D2  提交并推送；GitHub Pages 自动生效
[ ] D2b 【必查】硬刷新（Ctrl+F5）后验证，CDN 缓存可能要等 1–2 分钟
        不要只看页面——直接访问这三个 URL 看线上实际内容：
          .../Competitor-Analysis/data/fact_financials_combined.csv
          .../Competitor-Analysis/data/fact_sales_history.csv
          .../Competitor-Analysis/data/fact_profit_history.csv
        核对：行数与本地一致、含新季度、history 表 H1 行只有 4 行
        页面层再确认：期间下拉无重复 H1，Trend 两图有新季度柱子
[ ] D3  market_data_verified.csv / market_history.csv 不要动（workflow 每日自动更新）
[ ] D4  index.html 本季有改动 → 一并推送到仓库根目录
        （改前先核对 raw.githubusercontent.com 上的最新版，避免覆盖别人改动）

【阶段 E · NASN 专线（港股，不走 PDF 解析）】
[ ] E1  012_Nasn_table/NASN_table.xlsx 三个 sheet 各加一列新期间（数值抄自中报/年报 PDF）
[ ] E2  有新科目 → 在 hk_item_mapping_with_code.csv 补映射
[ ] E3  跑 convert_nasn.py
[ ] E4  跑 patch_nasn_selling.py（NASN 区全量幂等重建）
        ⚠️ 本机 03_scripts/ 下目前没有该脚本，换季前先从另一台机器取回
[ ] E5  复查 NASN：毛利为正、IS_NP/IS_OP 亏损为负、CF_FIN_NET 符号同 PDF 原文

【阶段 F · 手工板块（run.bat 不覆盖）】
[ ] F1  【本季唯一必做】股权 equity —— 5 家 A 股 × 6 行（前5大 + 其他股东）
        → 更新仓库 data/IMPORT_manual_all.csv（页面加载自动 fetch）
        → 或在看板上选「公司+季度」手工录入，录完 Export CSV 留备份
        ⚠ 必须双源验证后才可入库，见 PROJECT_KNOWLEDGE.md 8.2 节：
            源A 官方季报 PDF「前10名股东持股情况」（以此为准，含限售股）
            源B 东财 F10 RPT_F10_EH_FREEHOLDERS（无限售口径，差额=限售股属正常）
        要点：六行合计=100%；含逗号的名字要加引号（"YUAN,YONGBIN"）；
              上期未进前五的 change 留空；前五名单有变动时「其他股东」change 也留空
[ ] F2  以下 block 均为年度口径，季度更新**不用动**：
        headcount 员工 / education 学历 / rd 研发 / productline 产品线
        IMPORT_annual_all.csv（customer/region/segment/profile）
        → 只有年报季才需要更新
[ ] F3  P/E、P/B、股价、市值 —— **无需任何操作**
        由 GitHub workflow 每日自动抓取 market_data_verified.csv
        （早期文档提到的「行业 PE/PB 手动录入卡」经核实并不存在，已废弃）

【阶段 G · 收尾】
[ ] G1  浏览器打开看板，12 个板块逐个滚一遍，确认无空图、无 n/a 异常
[ ] G1b 重点看 Historical Trends：Sales / Profit 两图最右侧应是本季新季度，
        且每家 A 股都能看到 Q1 / H1 / Q3 三根柱子（缺柱→回 C4/C5）
[ ] G2  Executive Signals 8 张卡数值合理，异常 flag（REVENUE DOWN / NET LOSS 等）属实
[ ] G3  更新 PROJECT_KNOWLEDGE.md 的「最后更新」日期 + 变更记录
[ ] G4  口径有变动时，同步更新 PROJECT_KNOWLEDGE.md 2.1 节「各板块数据口径总表」
```

---

## 三、常见坑速查

| 现象 | 原因 | 处理 |
|---|---|---|
| 看板整块无数据、Trend/Heatmap 全空 | `index.html` 的 `init()` 偏移量被硬编码成 `x[0].concat(x[1])` | 必须用 `fc=PATH.fin.length` 动态计算 |
| 历史期间凭空消失 | A2 没归档上季度 `final_facts/` | 从 `_history/` 找回并重跑 4e/4f |
| **Trend 图只剩当季和同期，Q1/H1 柱子消失** | `build_dashboard_tables.py` 的 `load_fact()` 只读 `final_facts/` 当季表，而该表每次运行被整体覆盖；A 股 Q1/H1 在 `_history/` 里没被读到 | 2026-11-05 已修：`load_fact()` 加读 `_history/*/fact_financials_all_*.csv`。若复发，先查 A2 归档在不在，再跑 C5 |
| **期间下拉出现两个 H1** | ① `convert_nasn.py` 的 `PERIOD_MAP` 把第二季度映射成 `2026H1`（应为 `2026Q2`）；② 只推了 combined，两张 history 表还是旧版 | ① 已修；② 按 D1 三个文件一起推。累积表里的旧 H1 行不会自动消失，需一次性清理 |
| **NASN 数值集体偏小约 14.5%** | `filter_dashboard_fields.py` / `archive_dashboard_history.py` 把 NASN 默认币种写成 HKD，前端再 ×0.855 | NASN 财务报表**本身就是 CNY**，两处默认值必须是 CNY（港元只影响股价/市值）。2026-11-05 已修 |
| **某家公司只有 BS、IS/CF 全缺** | 上市公司改了报表标题写法（如立讯 Q3「合并**年初到报告期末**利润表」），`find_title_page()` 精确匹配不中就静默跳过，**流水线仍报成功** | `extract_szse_00.py` 的 `STATEMENTS` 已改为别名列表。遇到新写法继续往里加；平时靠 C1 核对行数发现 |
| 毛利率变负 | NASN 盲用 mapping sign 把正毛利翻负 | 符号规则：**只对 IS_NP/IS_OP 取反**，其余科目透传 |
| AUMOVIO 营业利润和 A 股对不上 | 旧口径用 EBIT（未扣财务费用） | 已修正为 `Earnings before tax` → `IS_OP`，映射已改好，新季度直接跑 |
| 跑脚本报缺包 | `venv` 指向旧项目 | 用 `venv\Scripts\python.exe`，或设 `PYTHONPATH=旧venv\Lib\site-packages` |
| 半年报期间整体左移一期 | 列头「2026年半年度」解析 | 已处理，勿回退 |
| Q3 的 Working Capital 算不出 YoY | A股季报 BS 对比列是上年度末，不是上年同期 | 不是 bug；要补 2025Q3 BS 只能单独跑 2025 三季报 PDF，否则该期只显示 QoQ（见 8.1 节） |
| 巨潮 PDF 下载 404 | `www.cninfo.com.cn/finalpage/...` 不提供附件 | 改用 `static.cninfo.com.cn/<adjunctUrl>` |
| mdna_notes.csv / product_lines.csv 读取报错 | 实为 xlsx 改后缀 | 按 Excel 读 |
| 知了财报抓不到 | 有人机验证墙 | 用东财 F10 + 巨潮/东财年报 PDF 替代 |

---

## 四、关键路径备忘

| 用途 | 路径 |
|---|---|
| 一键入口 | `run.bat` → `03_scripts/run_pipeline.py` |
| 科目映射表 | `00_source_registry/mapping_display_4.csv` |
| 全科目大表 | `02_processed_data/final_facts/` |
| 历史趋势表输出 | `02_processed_data/dashboard_data/` |
| 看板瘦身版（推送用） | `02_processed_data/dashboard_ready/` |
| Trend 表生成器（Step 4d） | `03_scripts/build_dashboard_tables.py` |
| 历史期归档（Trend 表数据源之一） | `02_processed_data/final_facts/_history/<季度>/` |
| 质检脚本 | `03_scripts/qa_spot_check.py`（⚠ 本机缺失，见 C2 替代方案） |
| 质检台账 / 日志 | `02_processed_data/qa/spot_check_ledger.json`、`spot_check_log.csv` |
| 手工板块 CSV 备份 | `05_output/manual_blocks/` |
| 看板前端工作副本 | `index.html`（改完推 GitHub 仓库根目录） |

---

## 五、看板 8 列数据契约（推送前对照）

```
company | statement_type | period | item_code | item | item_en | value | currency
```
- 去重键：`(company, statement_type, period, item_code)`
- `period` 格式 `YYYYQ[1-4]` 或 `YYYYFY`；**数据键始终写 Q2，看板会自动显示成 H1**
- `currency`：AUMOVIO=EUR，其余全部 CNY（NASN 财务数据也是 CNY，只有股价/市值是 HKD）

### ⚠ 两套 period 规则必须分清（常见误改点）

| 文件 | 半年报怎么存 | 谁负责显示成 H1 |
|---|---|---|
| `fact_financials_combined.csv` | 存 `2026Q2` | 前端 `pLabel()` 正则转换 |
| `fact_sales_history.csv` / `fact_profit_history.csv` | 直接存 `2026H1` | `_norm_period()` 在生成时已转好 |

两者**故意不一致**，不要"统一"。但 history 表的 H1 行正常只应有 4 行
（AUMOVIO × 2 期 + NASN × 2 期）；若 7 家公司每家都有 H1（共 14 行），
说明是旧版文件，必须重跑 Step 4d。
