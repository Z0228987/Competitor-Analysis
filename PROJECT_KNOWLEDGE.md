# AUMOVIO 竞争情报项目 — 完整知识文档
> 最后更新：2026-10 | 适用于下次对话直接喂给 AI 使用

---

## 一、项目全景

### 两个仓库，分工明确

| 仓库 | 用途 |
|---|---|
| `financial_auto_project_v3`（本地） | 数据处理流水线：PDF下载→标准化→瘦身→归档 |
| `Z0228987/Competitor-Analysis`（GitHub） | 前端看板：`index.html` + `data/` 目录下的CSV文件，GitHub Pages 托管 |

### 覆盖公司（7家）

| 公司 | 交易所 | 币种 | 备注 |
|---|---|---|---|
| Aumovio | 欧洲 | EUR | 德国公司，IFRS准则，季报PDF手动放入根目录 |
| BTL | 深交所 | CNY | |
| Baolong 宝龙 | 深交所 | CNY | |
| HASCO 华域 | 上交所 | CNY | |
| Luxshare 立讯 | 深交所 | CNY | |
| NASN 耐世特 | 港交所 | HKD | 港股，单独脚本处理 |
| Tuopu 拓普 | 上交所 | CNY | |

---

## 二、HTML 看板结构（index.html）

### 数据加载（PATH 对象）

```javascript
PATH = {
  fin: ['data/fact_financials_combined.csv'],  // 合并后的统一大表（A股+港股+AUMOVIO）
  sales: 'data/fact_sales_history.csv',         // 营收历史趋势（年度+季度）
  profit: 'data/fact_profit_history.csv',       // 净利润历史趋势
  market: 'data/market_data_verified.csv',      // 股价快照（GitHub workflow 自动更新）
  marketHistory: 'data/market_history.csv'      // 每日收盘价历史（GitHub workflow 自动更新）
}
```

> ⚠️ **重要**：`init()` 函数里用 `fc=PATH.fin.length` 动态计算偏移量，
> 不能硬编码 `x[0].concat(x[1])`，否则所有后续变量全部错位，导致
> Trend / Capital Market / Heatmap / Cash Flow 全部无数据。

### 汇率换算规则

- AUMOVIO（EUR）：用户界面设置的 `eurRate`（默认 8.35）
- NASN（HKD）：用户界面设置的 `hkdRate`（默认 0.855）
- 其余公司：固定 1（本身人民币）

### 8个看板模块

| 模块 | 数据来源 | 核心指标 |
|---|---|---|
| Executive Overview | `fin` + `sales` + `profit` | KPI卡片、营收/利润趋势、气泡图、Executive Signals |
| Profitability | `fin` | 营收对标、毛利率/营业利润率/净利率 |
| Historical Trends | `sales` + `profit` | 年度/季度历史走势 |
| Working Capital | `fin` | AR/AP/净营运资本 |
| Capital Markets | `market` + `marketHistory` | 股价、市值、PE、PB、YTD涨幅 |
| Cash Flow | `fin` | OCF/ICF/FCF对标，现金转化率 |
| Liquidity & Efficiency | `fin` | 流动比率、研发强度、SG&A强度 |
| Benchmark Heatmap | `fin` | 同业热力图（相对排名着色） |

### 核心计算公式

| 指标 | 公式 | 备注 |
|---|---|---|
| 毛利率 | `gross / revenue` | gross 优先取 IS_GROSS_PROFIT，没有则 revenue-\|cost\| |
| 营业利润率 | `op / revenue` | |
| 净利率 | `np / revenue` | |
| OCF Margin | `ocf / revenue` | |
| 流动比率 | `ca / cl` | |
| 研发强度 | `rnd / revenue` | |
| SG&A强度 | `(\|sell\|+\|admin\|) / revenue` | |
| 自由现金流 | `ocf + icf` | ⚠️近似值，非严格定义 |
| 同比增长率 | `本期/去年同期 - 1` | |
| YoY pp | `本期比率 - 去年同期比率` | 单位百分点，非增长率 |

### ⚠️ 3个已知计算"坑"

1. **FCF 是近似值**：用 `ocf+icf` 代替严格的 `ocf-capex`，投资活动净额里包含并购等非资本开支项目
2. **NASN 净利润替代**：净利润缺失时用 `-|税前亏损|` 近似，不是真实净利润
3. **热力图是相对排名**：颜色是当期所有公司的三分位排名，不是绝对达标标准

---

## 三、数据处理流水线（run.bat）

### 完整步骤

```
run.bat
  └── 03_scripts/run_pipeline.py
        ├── Step 0（--download 才跑）：Selenium下载PDF → 01_raw_data_*/
        ├── Step 1：清空 02_processed_data/raw_extracted/
        ├── Step 2：extract_szse_00.py → 深交所PDF提取
        ├── Step 3：extract_data_600.py + extract_common.py → 上交所PDF提取
        ├── Step 4：build_standardized_financials.py
        │           + mapping_display_4.csv（科目映射表）
        │           → fact_IS/BS/CF_all.csv → fact_financials_all.csv
        ├── Step 4b：convert_nasn.py → NASN港股 → 合并进 fact_financials_all.csv
        ├── Step 4c：extract_aumovio.py → AUMOVIO季报PDF → aumovio_financials.csv
        ├── Step 4d：build_dashboard_tables.py
        │           → fact_sales_history.csv + fact_profit_history.csv
        ├── Step 4e：filter_dashboard_fields.py
        │           → dashboard_ready/（瘦身版：8列，按item_code白名单过滤）
        │           → dashboard_ready/fact_financials_combined.csv（两表合并）
        ├── Step 4f：archive_dashboard_history.py
        │           → A. 扫描 final_facts/_history/<季度>/ → 瘦身 → 归档
        │           → B. 叠加本季度 dashboard_ready/ 数据
        │           → C. 写出 dashboard_history/*_history.csv（本地累积备份）
        │           → D. 回写到 dashboard_ready/fact_financials_combined.csv（含全历史）
        └── Step 5：质量报告
```

### 数据血缘图

```
【自动化（run.bat）】
PDF（深交所/上交所）→ raw_extracted/ → fact_financials_all.csv ┐
PDF（AUMOVIO，手动放入根目录）→ aumovio_financials.csv        ┤
                                                                ├→ Step 4e 瘦身
final_facts/_history/<Q>/ （手动归档上季度数据）               ┘   → Step 4f 历史累积
                                                                    → dashboard_ready/fact_financials_combined.csv

【人工维护（run.bat 不覆盖）】
GitHub workflow 自动抓取 → market_data_verified.csv + market_history.csv

【部署】
dashboard_ready/*.csv + fact_sales_history.csv + fact_profit_history.csv
→ 手动推送到 Competitor-Analysis/data/ 目录 → GitHub Pages
```

---

## 四、关键文件说明

### 本地项目（financial_auto_project_v3）

| 文件/目录 | 说明 |
|---|---|
| `00_source_registry/mapping_display_4.csv` | 科目映射表：各公司原始科目名 → 标准 item_code |
| `00_source_registry/item_code_mini.xlsx` | HTML 白名单：25个 item_code，只有这些会进入 dashboard_ready/ |
| `00_source_registry/fact_sales_history.csv` | 手工补的历史年度营收（2021~当季前） |
| `02_processed_data/final_facts/` | 完整大表（所有科目，保留用于 build_dashboard_tables） |
| `02_processed_data/final_facts/_history/<Q>/` | 手动归档的历史季度完整大表 |
| `02_processed_data/dashboard_ready/` | 瘦身版（推送到GitHub的那份） |
| `02_processed_data/dashboard_history/` | 本地历史累积备份（不推送） |
| `03_scripts/filter_dashboard_fields.py` | Step 4e：行过滤（白名单）+ 列裁剪（8列）+ 合并两表 |
| `03_scripts/archive_dashboard_history.py` | Step 4f：历史归档 + 回写全历史到 dashboard_ready/ |

### dashboard_ready/ 产出的8列结构

```
company | statement_type | period | item_code | item | item_en | value | currency
```

- `currency`：AUMOVIO=EUR，NASN=HKD，其余=CNY（filter 脚本自动补）
- 去重键：`(company, statement_type, period, item_code)`

### GitHub 仓库（Competitor-Analysis/data/）

| 文件 | 来源 | 说明 |
|---|---|---|
| `fact_financials_combined.csv` | run.bat → Step 4f 回写 | **主数据文件**，含全历史所有公司 |
| `fact_sales_history.csv` | run.bat → Step 4d | 营收历史趋势 |
| `fact_profit_history.csv` | run.bat → Step 4d | 利润历史趋势 |
| `market_data_verified.csv` | GitHub workflow | 股价快照（真实数据） |
| `market_history.csv` | GitHub workflow | 每日收盘价历史 |
| `aumovio_financials.csv` | ~~已弃用~~ | HTML 已改为读 combined，此文件不再需要更新 |
| `fact_financials_all.csv` | ~~已弃用~~ | 同上 |

---

## 五、每季度操作手册

```
1. 把新季度 PDF 放入根目录（AUMOVIO）或 01_raw_data_*/ 目录（A股/港股）
2. 把上季度的 final_facts/*.csv 手动归入 final_facts/_history/<上季度>/
3. 双击 run.bat（Step 0 加 --download 参数才会自动下载 PDF）
4. 检查 terminal 输出，确认 Step 4e/4f 正常完成
5. 把以下文件推送到 GitHub Competitor-Analysis/data/：
   - fact_financials_combined.csv（含全历史）
   - fact_sales_history.csv
   - fact_profit_history.csv
6. GitHub workflow 会自动更新 market_data_verified.csv 和 market_history.csv
```

---

## 六、已知问题与待办

| 问题 | 状态 | 说明 |
|---|---|---|
| `venv` 损坏 | ⚠️ 待修复 | `venv\Scripts\python.exe` 指向不存在路径，目前 fallback 用系统 python，建议重建 venv |
| 股价数据仍是人工维护 | ⚠️ 部分自动化 | GitHub workflow 已自动抓取，但 workflow 逻辑不在 run.bat 里，是独立维护 |
| AUMOVIO 主表合并来源不明 | ⚠️ 待确认 | `aumovio_financials.csv` 主表（含多季度）不清楚是哪个脚本合并的，已弃用此文件，改用 combined |
| market_data.csv 是假数据 | ✅ 已知 | 标注 ILLUSTRATIVE_DEMO，HTML 不读它，已无影响 |
| FCF 口径不严格 | ✅ 已知 | 用 ocf+icf 近似，团队内部已知晓 |

---

## 七、index.html UI 改动记录（本次对话完成）

| 改动 | 详情 |
|---|---|
| 数据源统一 | `PATH.fin` 从两文件改为 `['data/fact_financials_combined.csv']` |
| `init()` 修复 | 用 `fc=PATH.fin.length` 动态计算偏移量，修复所有模块数据错位问题 |
| 图表配色统一 | 红→紫渐变：Luxshare `#6837d8`，HASCO `#9b6ef0`，Tuopu `#4827af`，Baolong `#c44bff`，BTL `#ff7840`，NASN `#b07aff` |
| 表格选中行高亮 | 左侧红线 + 公司名红色 |
| 导航 active 同步 | scrollspy（IntersectionObserver）随滚动自动更新左侧导航选中状态 |
| Note 区域简化 | 去掉 "MANAGEMENT NOTE" 标题、h4 子标题、空白 head 区域；Clear 按钮绝对定位在右上角，hover 时显示 |
| Note placeholder 清空 | 空白时不显示 "Add interpretation..." 提示文字 |

---

## 八、下一步 UI 优化方向（待做）

- [ ] 加载骨架屏（数据未就绪时显示 skeleton 动画）
- [ ] 响应式适配移动端

---

## 九、UI 配色改造全过程记录（2026-10，本轮对话）

> 背景：用户要求参考 `awesome-design-md-main/` 和官方 AUMOVIO Power BI 主题 JSON，
> 把 `index.html` 的颜色体系从"随意渐变、紫橙撞色"改造成"统一色板、公司色全局一致、
> 深色模块可读"。过程分好几轮，下面按时间顺序记录每一步的结论，避免下次重复踩坑。

### 9.1 设计参考资料

| 资料 | 用途 | 结论 |
|---|---|---|
| `awesome-design-md-main/`（28份金融类DESIGN.md） | 风格选型 | 最终选定 **Mercury**（深浅双世界体系）+ **Stripe**（数字对齐、克制渐变）作为参考基准 |
| `DESIGN_COMMONALITIES_SUMMARY.md` | 横向方法论 | 提炼出5条可落地原则（见9.4），已写入 `DESIGN.md` 第8节 |
| AUMOVIO 官方 Power BI 主题 JSON（用户提供） | **权威色值来源** | 见9.2，这是本项目目前唯一确认过的官方色板，以后改色优先从这里取值 |

### 9.2 AUMOVIO 官方色板（Power BI 主题 JSON，权威来源，务必复用）

```
dataColors（32色，专为分类数据设计）：
 1.#FF4208(橙/品牌主色) 2.#FFB7A1(浅橙) 3.#4827AF(紫/品牌辅色) 4.#A08AE5(浅紫)
 5.#18A9E7(蓝)         6.#A3DDF5(浅蓝)  7.#12CAAE(青绿)        8.#A1F7EA(浅青绿)
 9.#CEC43A(黄橄榄)    10.#ECE9B5(浅黄) 11.#DA3806(深橙)       12.#FF906E(中橙)
13.#341C7D(深紫)      14.#7E60DC(中紫) 15.#1B8DBE(深蓝)       16.#75CCF0(中蓝)
17.#16B69E(深青)      18.#68F3DE(中青) 19.#AEA633(深黄)       20.#E2DD8C(中黄)
21.#168980(深青绿)    22.#293537(深灰) 23.#BB4A4A(暗红)       24.#B59525(暗金)
25.#475052(灰)        26.#6A9FB0(灰蓝) 27.#BD7150(棕橙)       28.#7B4F71(藕紫)
29.#1B4D5C(深蓝灰)    30.#706060(灰棕) 31.#0F5C55(深青)       32.#1C2325(近黑)

语义色：bad=#D82822(红警示)  good=#32CB70(绿正向)  neutral=#FFD600(黄中性)
```

⚠️ 之前通过截图 OCR 识别色值的方式（多次重复识别、结果互相矛盾）**不可靠，已废弃**，
以后有新色板需求，优先要求用户提供 JSON/文字版色值，不要再靠读图识别小字 HEX。

### 9.3 已完成且确认生效的改动

| 改动 | 具体内容 | 状态 |
|---|---|---|
| 颜色源统一 | 删除 `renderMarket()` 内部重复的 `colors={}`，全部改用顶部 `companyColor()`，解决"同一公司在不同模块颜色不同"的问题 | ✅已验证生效 |
| 7家公司专属色（一眼可辨） | Aumovio橙`#ff4208`（固定）、Luxshare紫`#4827af`、HASCO蓝`#18a9e7`、Tuopu青绿`#12caae`、Baolong黄橄榄`#aea633`、BTL棕橙`#bd7150`、NASN藕紫`#7b4f71`——全部取自官方32色板，色相跨度大 | ✅已验证生效 |
| `companyTint(company, alpha)` | 新增工具函数，把公司色转 rgba 半透明，用于高亮背景 | ✅已验证生效 |
| 表格选中高亮 | 从"统一品牌红"改为"该公司自己的颜色"：竖线+文字+淡背景，浅色表格与深色Capital Market表格各一套CSS变量（`--sel-color/--sel-bg1/--sel-bg2` 和 `--sel-bg-dark`） | ✅已验证生效 |
| 指标类图表单色阶梯（非公司对比） | Margin Benchmark/Cash Flow Benchmark/Cash Flow Structure/Working Capital：系列含义是"指标类型"不是"公司"，改用同色相深浅区分，避免紫橙乱撞；Cash Flow Benchmark 的 OCF/ICF/FCF 三色故意选了紫/橙/青绿三个不同色相（因为同图里3个系列必须互相区分） | ✅已验证生效 |
| 股价趋势线 Peer Trend | 选中公司线条加粗不透明，其余公司线条降到30%透明度，做"聚焦"效果 | ✅已验证生效 |
| Revenue/Profit History 图 | 季度柱改浅紫`#a08ae5`，和"全年"柱（深紫）区分层级，不再跟"亏损警示"橙色混淆 | ✅已验证生效 |
| `.focus-outline` 退路样式 | 半透明浅灰描边+浅灰底，供以后无法用强调色区分时使用（目前尚未在实际场景中启用，只是预留了CSS） | ✅已写入CSS，待用 |
| 财务数字等宽对齐 | KPI大数字、表格数字列、市场模块metric数字加 `tabular-nums` | ✅已验证生效 |
| 删除冗余区域 | 用户要求删除"Valuation Positioning估值气泡图"+"Executive Market Signals"卡片区域（HTML容器+对应JS渲染逻辑一并清理，避免报错） | ✅已验证生效 |

### 9.4 DESIGN_COMMONALITIES_SUMMARY 原则 → 本项目落地对照

| 通用原则 | 落地状态 |
|---|---|
| 颜色按角色命名（而非外观命名），如 `--color-accent-brand` | ✅已在 `DESIGN.md` 第8.1节写入角色别名方案，**尚未应用到 index.html 代码**（目前代码仍是 `--o100/--p400` 色号命名），下次可以做这层重命名 |
| 一个主强调色+灰阶，避免同图多个高饱和色乱撞 | ✅已落地到指标类图表（9.3），公司对比类图表例外（颜色=身份，本来就该多彩） |
| 财务数字用 tabular-nums | ✅已落地 |
| 退化方案：无法用颜色区分时用浅灰半透明描边 | ✅CSS已写（`.focus-outline`），**尚未应用到具体场景** |
| Do's and Don'ts 红线清单 | ✅已写入 `DESIGN.md` 第8.6节 |

### 9.5 用户最新反馈（2026-10，尚未处理，下次优先解决）

根据用户提供的最新截图反馈，以下问题**依然存在，尚未修复**：

1. **Capital Market Intelligence 模块仍然偏暗**：虽然已经把 `--dn2/--dn3/--dn4` 的白色透明度从 88/68/48% 提到 96/84/66%，但用户反馈这个区域的文字/元素视觉上还是暗，需要进一步加亮，可能需要重新审查该模块里具体是哪些元素（表头？说明文字？图例？坐标轴？价格趋势线 peerTrend 的配色？）仍在用偏暗的颜色，而不是只调整几个全局变量就够。
   - 排查方向：搜索 `renderMarket` 函数、`.market-dark-card`、`peerTrend` 相关代码，逐个元素检查实际渲染色值，而不是只改CSS变量定义（有可能某些地方是硬编码颜色，没有用变量，所以调变量没生效）。

2. **Heatmap（Benchmark Heatmap 模块）的高亮方式"很丑"**：用户原话反馈，需要重新设计。当前实现是"相对排名三分位着色"（见第二节"3个已知计算坑"的第3条），具体视觉实现方式需要重新审查：
   - 需要找到 heatmap 单元格的 CSS/渲染逻辑（可能是 `.heat-good/.heat-mid/.heat-bad` 这类class，或者是inline style直接赋色）
   - 结合官方色板（good=#32CB70绿、bad=#D82822红、neutral=#FFD600黄，这是官方JSON里自带的语义色，之前没有用上）重新设计单元格底色/文字色/边框，参考 Mercury/Stripe 这类克制风格做法（低饱和度背景色块+深色文字，而不是大面积高饱和纯色块）

3. **临时调试文件需要清理**：本轮对话过程中为了核对代码片段生成了多个临时文件，**这些不是项目正式文件，下次对话可以直接删除**：
   `tmp_find.py` `tmp_find2.py` `tmp_find3.py` `tmp_find_css.py` `tmp_find_peer.py`
   `tmp_find_peertrend.py` `tmp_find_peertrend2.py` `tmp_find_table.py` `tmp_hex.py`
   `tmp_market_bottom.txt` `tmp_peertrend_out.txt`

### 9.6 下次对话建议处理顺序

1. 先删除上面列出的临时调试文件（清理工作区）
2. 排查 Capital Market 模块里"仍然偏暗"的具体元素（极可能是硬编码色值未用变量，或 `peerTrend` 图表的配色函数没有覆盖到）
3. 重新设计 Heatmap 高亮方案，引入官方语义色（good/bad/neutral）+ 低饱和度背景块
4. 待以上确认完成后，再考虑是否要做 9.4 里提到的"颜色角色重命名"这类非紧急重构

---

## 十、本项目相关文档索引

| 文档 | 内容 |
|---|---|
| `PROJECT_KNOWLEDGE.md`（本文件） | 项目全貌、数据流水线、UI改造历史记录 |
| `DESIGN.md` | index.html 当前设计系统现状 + 第8节的改进决定（角色变量、单色阶梯、focus-outline等方案） |
| `awesome-design-md-main/` | 外部参考资料库，28份金融类产品的DESIGN.md范本 |
