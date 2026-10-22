# AUMOVIO 竞争情报项目 — 完整知识文档
> 最后更新：2026-10-22 | 适用于下次对话直接喂给 AI 使用
> 本次更新：Education Mix + R&D Headcount & Compensation 两张新图表、Annual Deep-Dive 自动从 `data/` 目录读取 CSV（免手动 Import）、Annual Deep-Dive 9 张卡片各自配专属年份/季度下拉框（含镜像联动修复）、Revenue/Profit Trend 新增 QoQ 环比箭头（与原有 YoY 同比并列）、次要图表 Notes 改为按需展开的"💬 Add comment"开关、Capital Markets 区块移除 Notes 栏、**全站交互/UI 走查并修复 13 项问题**（自动滚动跳转、破坏性操作二次确认、Notes 工具栏触屏可发现性、R&D 维度绑定、CSS 死代码、Import 按钮误选提示、时间维度分组标识、Annual Deep-Dive 单卡折叠、FX 自动生效、Peer Trend 一键恢复等）

---

## 一、项目全景

### 两个仓库，分工明确

| 仓库 | 用途 |
|---|---|
| `financial_auto_project_v3`（本地） | 数据处理流水线：PDF下载→标准化→瘦身→归档 |
| `Z0228987/Competitor-Analysis`（GitHub） | 前端看板：`index.html` + `data/` 目录下的 CSV，GitHub Pages 托管 |

**工作流约定**：前端改动直接在本地 workspace 改 `index.html`（以 GitHub 上最新版为基准下载），确认后整体推回仓库根目录即上线（Pages 自动生效）。本地与线上版本可能存在时间差，改版前先核对 `raw.githubusercontent.com` 上的最新文件。

### 覆盖公司（7家）

| 公司 | 交易所 | 币种 | 备注 |
|---|---|---|---|
| Aumovio | 欧洲 | EUR | 德国公司，IFRS准则，季报PDF手动放入根目录 |
| BTL 伯特利 | 深交所 | CNY | |
| Baolong 保隆 | 深交所 | CNY | |
| HASCO 均胜/华域 | 上交所 | CNY | |
| Luxshare 立讯 | 深交所 | CNY | |
| NASN | 港交所 | 财务=CNY，股价/市值=HKD | 2026-08 上市，报表以人民币列示（营收 6.15 亿元与招股书/媒体口径一致）；走 convert_nasn.py 单独处理 |
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

> **2026-10-22 新增：Annual Deep-Dive / 手动录入板块的自动加载。**
> 页面加载时会额外 `fetch('data/IMPORT_manual_all.csv')` 和 `fetch('data/IMPORT_annual_all.csv')`（`cache:'no-store'`），
> 拿到后分别复用现成的 `importData()`（季度板块：equity/headcount/education/rd/productline）和
> `adImportData()`（年度板块：profile/customer/region/segment）自动灌入数据，**不再需要每次手动点 "Import CSV"**。
> - 两个文件若本地 `data/` 目录暂不存在，`fetch` 失败会被静默吞掉（`.catch`），不报错、不中断其他模块加载，手动 Import 按钮仍可作为后备方案。
> - 维护方式：以后只需要更新 GitHub 仓库 `data/` 下的 `IMPORT_manual_all.csv` / `IMPORT_annual_all.csv` 两个文件（字段结构见 3.2 节、3.5 节的 CSV 契约），同步到本地 `data/` 目录后刷新页面即可看到最新数据。

> ⚠️ **重要**：`init()` 用 `fc=PATH.fin.length` 动态计算偏移量，不能硬编码
> `x[0].concat(x[1])`，否则后续变量全部错位，Trend/Capital Market/Heatmap/Cash Flow 全部无数据。

### 汇率换算规则

- AUMOVIO（EUR）：界面 `eurRate` 输入框（默认 8.35）
- NASN：`hkdRate` 输入框（默认 0.855）**只作用于股价/市值（HKD）**；财务数据报表本身是 CNY，companyRate 对 `currency=CNY` 的 NASN 行返回 1 不折算
- 其余公司：固定 1（本身人民币）

### 看板模块（12 个，页面自上而下顺序；2026-10-19 重排为「总览 → 增长 → 盈利 → 定位 → 现金 → 营运 → 流动性 → 资本市场 → 结构数据 → 年报 → 附表」的分析叙事线）

| # | 导航/section id | 数据来源 | 核心内容 |
|---|---|---|---|
| 01 | Executive Overview `overview` | fin+sales+profit | Hero（公司/季度/汇率控制）+ **Executive Signals（8 张大数字卡）** |
| 02 | Historical Trends `history` | sales+profit | 年度(FY)/季度(Q)营收与净利润走势 |
| 03 | Profitability `benchmark` | fin | 营收对标、毛利率/营业利润率/净利率 |
| 04 | Positioning `positioning` | fin | 增长-盈利气泡图（四象限，综合 02+03） |
| 05 | Cash Flow `cashflow` | fin | **合并单图**：OCF/ICF/融资现金流柱（左轴 CNY bn）+ OCF/自由现金流利润率线（右轴 % of revenue） |
| 06 | Working Capital `working-capital` | fin | AR/AP/净营运资本 |
| 07 | Liquidity & Efficiency `liquidity` | fin | 流动比率、研发强度、SG&A 强度 |
| 08 | Capital Markets `capital-market` | market+marketHistory | 股价、市值、PE、PB、YTD、近1月趋势 + A股底盘零部件行业 PE/PB 均值对比（见下方） |
| 09 | Ownership & People `ownership` | **localStorage + MANUAL_SEED** | 股权结构 + 员工构成 + **Education Mix（学历构成饼图，2026-10-22 新增）** + **R&D Headcount & Compensation（研发人数+人均薪酬柱线图，2026-10-22 新增）** 四张图（手动录入，见 3.6 节） |
| 10 | Product Line `product-line` | **localStorage** | 关注产品线的营收柱 + ASP 线（手动录入） |
| 11 | Annual Deep-Dive `annual-deep-dive` | **localStorage，2026-10-22 起支持从 `data/` 自动加载** | 年报口径四卡（见 3.5 节） |
| 12 | Benchmark Table `heatmap` | fin | 6 项指标同业相对排名热力图（数据附录，收尾） |

> **2026-10-22：Annual Deep-Dive 9 张卡片的年份/季度选择器已从「顶部统一下拉」改为「逐卡片旁挂专属下拉框」**，不用再上下滚动对照。三组联动分组：
> - **季度组 A**：`ownPeriod`（Shareholding Structure）↔ `ownPeriodRd`（R&D Headcount & Compensation）
> - **年度组 B**：`annPeriod`（Headcount Mix）↔ `annPeriodEdu`（Education Mix）↔ `annPeriodPl`（Product Line）
> - **年度组 C**：`adYear`（Company Profile）↔ `adYearCustomer`（Customer Concentration）↔ `adYearRegion`（Regional Performance）↔ `adYearSegment`（Product Segment P&L）
> 同组内任一下拉框都是真实双向联动（选项列表、当前值、`change` 事件三者都同步），不是摆设镜像；每次 `boot()`/`render()` 都会重新同步一次。

### Capital Markets · 行业 PE/PB 均值对比卡（2026-10-19 新增）

- 位置：Capital Markets 区块内，紧跟 Peer Market Snapshot / Market Metrics Explained 两张卡之后，Share Price Trend 之前
- 卡片 id：`industryBenchCard`，4 个指标格 + 1 行手动输入区：
  - **Peer Avg P/E / P/B**：自动计算，取 `market_data_verified.csv` 中 5 家 A 股标的（BTL、Baolong、HASCO、Luxshare、Tuopu，**不含 Aumovio 和 NASN**）当期 pe/pb 的简单平均，随 CSV 更新自动刷新，无需手动维护
  - **Industry Avg P/E / P/B**：手动录入的"A股底盘与汽车零部件行业"整体估值均值（例如申万行业指数或东方财富板块估值），每个报告期录入一次即可，不随公司切换变化
- 手动录入方式：在 `industryPeInput` / `industryPbInput` 两个输入框填数字，点击 Save；存入 `localStorage['ciIndustryBench_<period>']`（如 `ciIndustryBench_2026Q2`），按报告期隔离，不同季度互不覆盖
- 实现要点：
  - JS 函数 `renderIndustryBench()`：在 `renderMarket()` 末尾调用，随公司/期间切换、FX 调整自动重新计算 Peer Avg 并读取该期已存的 Industry Avg
  - `avg(arr)`：简单平均工具函数
  - 不进流水线、不进 CSV、不影响 `market_data_verified.csv` 的数据契约——这是看板本地展示层的手动基准线，和股权/员工/产品线手动板块同一思路（localStorage，按期间 key 隔离）
- 每季度维护：打开 Capital Markets 卡片，手动从行业数据源（申万/东财等）查到当季度底盘与汽车零部件行业平均 PE、PB，填入两个输入框并点 Save

### Historical Trends · Revenue/Profit Trend 的 QoQ 环比箭头（2026-10-22 新增）

- Revenue Trend 和 Profit Trend 两张柱状图，原来每根柱子上只显示 **YoY（同比）** 箭头，现在同时显示 **QoQ（环比）** 箭头
- 新增两个纯函数：
  - `prevQKey(key)`：给定一个季度 key（如 `2025Q3`），返回上一季度 key（`2025Q2`；跨年正确处理 `2025Q1`→`2024Q4`）
  - `qoqSeries()`：按 `prevQKey` 逻辑给 Revenue、Profit 各算一条 QoQ 变化率序列
- 显示效果：柱子上方两行 `▲/▼ YoY xx.x%` + `▲/▼ QoQ xx.x%`，tooltip 同步两行
- **自动适配未来数据**：纯函数式计算，不依赖写死年份；只要新增数据仍是 `YYYYQ[1-4]` 格式，QoQ 会自动算出，无需改代码
- 纯年度数据（非季度格式）算不出上一季度时自动跳过 QoQ 显示（不报错），只显示 YoY

### 次要图表 Notes 改为按需展开（2026-10-22 新增）

- **保留不变**：Executive Signals 的 Notes 栏（`data-note-key="overview-notes"`）始终固定显示
- **移除**：Capital Markets 区块的 Notes 栏已整体去掉，该区块改为占满整行宽度的单列布局
- **其余 6 个区块**（History / Profitability / Positioning / Cash Flow / Working Capital / Liquidity / Benchmark Table）的 Notes 栏改为「💬 Add comment」开关：
  - 默认收起：Notes 栏收窄成 34px 竖条，只露出一个 💬 按钮，图表区域占满剩余空间
  - 点击展开：Notes 栏展开回原 286px 宽度，自动聚焦编辑框；再点一次收起
  - **已有笔记内容的区块默认自动展开**（不会把已写内容藏起来），按钮同时高亮（橙色边框）提示"这里有笔记"；清空内容后高亮自动取消
  - 展开/收起状态记忆在 `localStorage`，刷新页面保持上次的选择

### Executive Signals（2026-10-19 重构后的首屏核心）

- 位置：Hero 之后、其他模块之前，是**打开页面看到的第一组数字**
- 8 张卡，按「规模 → 盈利链 → 现金 → 资产负债」排列：
  1. **Revenue scale（主卡，占2列，48px 大数字，单位 B CNY）**
  2. Gross margin → 3. Operating margin → 4. Net margin → 5. Cash conversion → 6. Working capital → 7. R&D intensity → 8. Current ratio
- 每张卡：大数字 + 排名 pill（`#1 / 7`，第一名紫色填充）+ vs 同业中位数偏离（pp）+ 一句话解读 + 底部排名进度条（#1 满格→#7 空格）
- 异常 flag（橙色）：`REVENUE DOWN` / `OPERATING LOSS` / `NET LOSS` / `OCF < PROFIT` / `BELOW 1.0x`
- 颜色语义只有一条线：**紫=常态、橙=警示、青绿=正向**
- 原 KPI 卡行（Peer metric detail）已删除，其独有指标（毛利率/营业利润率/流动比率）已并入信号卡；Executive 区只保留一个通栏写字区（`data-note-key="overview-notes"`，旧 signals-notes 会自动迁移）

### 核心计算公式

| 指标 | 公式 | 备注 |
|---|---|---|
| 毛利率 | `gross / revenue` | gross 优先取 IS_GROSS_PROFIT，没有则 revenue-\|cost\| |
| 营业利润率 / 净利率 | `op / revenue`、`np / revenue` | |
| OCF Margin | `ocf / revenue` | |
| 流动比率 | `ca / cl` | |
| 研发强度 / SG&A强度 | `rnd / revenue`、`(\|sell\|+\|admin\|) / revenue` | |
| 自由现金流 | `ocf + icf` | ⚠️近似值，非严格定义 |
| 同比增长率 | `本期/去年同期 - 1` | |
| YoY pp | `本期比率 - 去年同期比率` | 单位百分点，非增长率 |

### ⚠️ 3个已知计算"坑"

1. **FCF 是近似值**：用 `ocf+icf` 代替严格的 `ocf-capex`
2. **NASN 净利润替代**：净利润缺失时用 `-|税前亏损|` 近似
3. **热力图是相对排名**：颜色是当期所有公司的排名着色，不是绝对达标标准

---

## 三、手动录入板块（2026-10-19 新增）

### 3.1 三个板块与数据模型

所有数据存浏览器 localStorage，键格式 `公司代码|季度`（如 `HASCO|2026Q2`），**编辑后才落盘；未编辑时显示预置默认值（MANUAL_SEED），不写存储**。

| 板块 | localStorage 键 | 值结构 |
|---|---|---|
| Shareholding Structure（股权环形图） | `ciEquityStructure` | `{"HASCO|2026Q2":[{n:"均胜集团",v:33.57,c:-0.42},...]}`，c=环比 pp 变动，可省略 |
| Headcount Mix（员工构成环形图） | `ciHeadcountMix` | 同上（c 不用），单位：人数 |
| 产品线名称 | `ciProductLineName` | `{"HASCO":"Thermal management module"}`，按公司记忆 |
| 产品线数据 | `ciProductLineData` | `{"HASCO|2026Q2":{rev:531.4, price:3760}}`，rev=百万元CNY，price=ASP 元 |

交互细节：
- 顶部全局「公司+季度」选择器驱动三个板块；股权/员工板块另有**独立期间下拉 `#ownPeriod`**（= 全局期 ∪ 该公司已录期）
- 员工构成是**年报口径**：自动显示 ≤ 当前期的最近一个有数据年份（`hcPeriod` 只认合计>0 的期），副标题显示 `FYxxxx annual report · N employees · YoY%`
- 股权图例带变动徽标：紫色 `+` 增持、珊瑚色 `−` 减持；TOTAL ≠ 100% 标警示色
- 每张卡 Edit 展开行编辑器（名称+数值，可增删行、Clear quarter）
- 产品线季度轴 = 全局下拉选项 ∪ 已录数据键（下拉里没有的季度也能显示）；已录季度以 chip 展示、可单删

### 3.2 CSV 数据契约（Import / Export / Template 共用）

```
block,company,period,name,value,revenue,price,change
equity,HASCO,2026Q2,均胜集团,33.57,,,-0.42
headcount,HASCO,2025Q4,生产人员,35259,,,
productline,HASCO,2026Q2,热管理模块,,531.4,3760,
```

- `block`：`equity` / `headcount` / `productline`（也接受中文：股权/员工/产品线）
- `company`：支持中文与代码，自动映射（ALIAS 表）：均胜电子→HASCO、立讯精密→Luxshare、拓普集团→Tuopu、保隆科技→Baolong、伯特利→BTL、安波福→Aumovio、拿森→NASN；也接受 `600699`/`600699.SH` 等代码
- `change`：股权环比 pp 变动，可留空；**首期必须留空**（否则显示假变动）
- 文件编码：UTF-8（带 BOM 也可）；自动识别 GBK（Excel 另存不会乱码）
- 同一 `block+company+period` 的导入会**整组替换**已有数据；Export 会导出预置+手录的全部数据（含 SEED）

### 3.3 导入体验（2026-10-19 修复）

- 导入成功后**自动跳转**到有数据的公司+最新期间并平滑滚动到板块（`applyImport`），toast 提示 `Imported 262 rows · 5 companies · showing BTL 2026Q2`
- 板块标题下有一排**公司 chips**（`#coChips`）：有点 = 该公司有数据（当前选中高亮紫色），虚线灰 = 无数据，点击即切换公司
- 选中公司无数据时空态会提示 `data on file: 2025Q4 / 2026Q1 / 2026Q2`（该公司其他有数据的期间）
- `#company` 下拉为空时（如 file:// 打开、data CSV 加载失败）会用公司池兜底填充，手动板块仍可用

### 3.4 预置数据（MANUAL_SEED，2026-10-19 采集）

| 数据 | 覆盖 | 来源 |
|---|---|---|
| 股权结构 | 5 家A股 × 2024Q4–2026Q2 共 7 期，前五大股东+其他股东（含 pp 变动） | 东方财富 datacenter `RPT_F10_EH_HOLDERS`（十大股东历史） |
| 员工构成 | FY2024 + FY2025 年报口径，按职能（生产/销售/技术研发/财务/行政） | 年报 PDF（`pdf.dfcfw.com/pdf/H2_{art_code}_1.pdf`，pdfplumber 抽"专业构成"表），明细合计与报告总人数全部核对一致 |

- ⚠️ **知了财报有人机验证墙**（curl 和 headless 浏览器都过不去），以后别再尝试直接抓它
- PDF 解析锚点：必须以 **"专业构成类别"** 为准（"专业构成"首次出现在小节标题里会误匹配）；跨页表拼接 idx..idx+3 页
- 生成脚本在本地 `.workbuddy/tmp/em/`（`build_equity.py` / `parse_hc.py` / `make_seed.py`），输出 `05_output/manual_blocks/manual_blocks_A股股权与员工.csv`（262行）+ `manual_blocks_template.csv`
- MANUAL_SEED 直接内联在 index.html 里（`window.MANUAL_SEED`），**更新数据 = 重新生成 seed 并替换该行**

### 3.6 Education Mix & R&D Headcount and Compensation（2026-10-22 新增）

位置：Ownership & People 板块，紧跟 Shareholding Structure / Headcount Mix 之后，两张图并排。

| 图表 | 画布 id | 类型 | 数据模型 key | 值结构 | 联动下拉 |
|---|---|---|---|---|---|
| Education Mix | `eduPie` | 环形图（Doughnut），风格与 Shareholding/Headcount 两张饼图一致 | `ciEducationMix` | `{"公司\|期间":[{n:"Bachelor",v:46},...]}`，`v`=占比%，默认4档 Below bachelor / Bachelor / Master / PhD，可在 Edit 面板增删改 | `annPeriod`（主） / `annPeriodEdu`（镜像，年度组 B） |
| R&D Headcount & Compensation | `rdChart` | 柱状图(人数,左轴) + 折线图(人均薪酬,右轴) 组合图，复用 Product Line 板块 `plChart` 的双轴实现 | `ciRdHeadcount` | `{"公司\|期间":{headcount:1499, comp:270}}`，`headcount`=研发人数(人)，`comp`=人均薪酬(CNY 千元/年) | `ownPeriod`（主） / `ownPeriodRd`（镜像，季度组 A） |

- CSV 契约并入 3.2 节同一张导入表：`block` 新增 `education`（教育构成）/ `rd`（研发人数薪酬）两个取值（也接受中文别名）；`education` 行用 `name+value`（同 equity/headcount 的 `%`/人数填法），`rd` 行的 `name` 固定写 `headcount` 或 `comp`，`value` 填对应数字
- `importData()`/`exportCsv()`/`TEMPLATE` 三处均已同步支持这两个新 block，Export 时会一并导出
- 关键 bug 修复记录（2026-10-22）：
  1. `importData()` 最初完全不识别 `education`/`rd`，CSV 导入会被静默跳过 → 已修复
  2. `periodsAsc()`（R&D 图表的时间轴来源函数）最初没有扫描 `db.rd`，即使数据存进去了柱状图也拿不到横轴 → 已修复，现在会正确合并 `db.rd` 的所有历史期间
- 真实数据来源：`Competitor Analysis for Reference/education-rd-import.csv`（逐字核对 4 家公司 PDF 原文：NASN/Baolong/Tuopu/BTL 的学历构成 + 研发人数/人均薪酬，Tuopu 因原文未披露具体研发人数，只有教育构成数据、未纳入 rd 行）

### 3.5 Annual Deep-Dive 年度深度板块（2026-10-19 新增，现位于 Product Line 之后、Benchmark Table 之前）

- 独立 IIFE `annualDeepDive()`（主脚本之后第二个 `<script>`），与季度板块完全解耦：独立年份下拉 `#adYear`、独立 localStorage keys、独立 CSV
- 四张折叠卡（`data-card`，点击 `adive-head` 开合，`.is-open` 控制）：Company Profile（Plants/RD_Centers/Employees/Countries 大数字）、Customer Concentration（Top5 客户占比条形）、Regional Performance（中国 vs 海外营收+毛利率）、Product Segment P&L（分部营收/毛利率/占比表）
- 数据模型（4 个 key）：`ciAdProfile / ciAdCustomer / ciAdRegion / ciAdSegment`，结构 `"Company|Year": [{n, v?, rev?, gm?}]`；profile/customer 用 `value`，region/segment 用 `revenue`+`gm`（**营收单位 CNY mn**）
- CSV 契约：`block,company,year,name,value,revenue,gm`；block ∈ `profile/customer/region/segment`（接受中文别名）；company 走 AD_ALIAS（含代码 `600699.SH` 等）；UTF-8/GBK 自动识别；**同名 name 行覆盖不追加**
- 模板/导出按钮复用 `#manualToast`；导入成功自动切到导入的公司+最新年份并滚动定位
- 导航项 `#navAnnual`：无任何年度数据时加 `.is-ghost`（半透明），提示该板块还没启用
- 公司 chips（`#adChips`）与季度板块 `#coChips` 行为一致；切公司时若无该年度数据自动跳到该公司最新有数年度
- region/segment 允许只填 name 不填数字（如模板里的分部名占位行），导入不丢弃

---

## 四、设计系统（2026-10-19 版，玻璃质感）

### CSS 变量（`:root`，9 阶 ramp，低饱和）

```
紫(主)  --p50 #EEEDFE / --p400 #7F77DD / --p600 #534AB7 / --p800 #3C3489 / --p900 #26215C
珊瑚(警示/下跌) --c200 #F0997B / --c400 #D85A30 / --c600 #993C1D
青绿(正向) --t200 #5DCAA5 / --t400 #1D9E75 / --t600 #0F6E56
金(强调) --a200 #FAC775 / --a400 #BA7517
蓝 --b400 #378ADD
文字 --ink #2B2750 / --text #4A4763 / --muted #6E6B84 / --faint #918DA6
--up=青绿 --down=珊瑚（涨跌语义）
```

### 玻璃配方（所有卡片统一）

```css
background: linear-gradient(180deg,rgba(255,255,255,.76),rgba(255,255,255,.54));
backdrop-filter: blur(22px) saturate(145%);
border: 1px solid rgba(255,255,255,.78);      /* 白色高光描边 */
box-shadow: 0 22px 44px -30px rgba(43,39,80,.42), inset 0 1px 0 rgba(255,255,255,.86);
```
页面 body 铺三团 radial 渐变色雾（紫/青/珊瑚）+ `#F6F5FB` 底色，颜色透过半透明卡片渗出 = 透明感来源。阴影一律极浅。

### 公司色（JS `COMPANY_COLORS`，与 CSS ramp 同源）

```
Aumovio #D85A30 · Luxshare #534AB7 · HASCO #378ADD · Tuopu #1D9E75
Baolong #BA7517 · BTL #D4537E · NASN #639922
```
工具函数：`companyColor(c)`、`companyTint(c, alpha)`（转 rgba 半透明背景）。指标类图表（非公司对比）用同色相深浅阶梯；公司对比类图表颜色=身份，允许多彩。

### 布局

- `.app` = grid `246px + 1fr`；侧边栏 sticky 100vh，玻璃底
- **可折叠目录**：`#sideToggle` 手柄吸附侧栏右缘；`body.nav-collapsed` 时 246px→78px（grid 过渡动画），导航只显示 01–09 数字 chip，hover 弹出 `data-label` 浮标；状态存 `localStorage('ciNavCollapsed')`
- ≤960px：侧栏变顶部横向滚动条，手柄隐藏；`.side/.nav` 必须有 `min-width:0`，否则横向导航会把页面撑宽（已踩过）
- 侧栏品牌 logo 是白字 → `.brand` 用紫渐变底（p600→p900）
- 侧栏导航顺序与页面 section 顺序一致（11 项），scrollspy 用 IntersectionObserver 同步选中

### 历史色板（官方 AUMOVIO Power BI JSON，备用权威来源）

`dataColors` 32 色与语义色（bad=#D82822 / good=#32CB70 / neutral=#FFD600）仍有效，改色优先从这里取值；**不要靠截图 OCR 识别色值（不可靠，已废弃）**。当前版本色板与官方 32 色是两套并行体系：CSS ramp 是主动设定的低饱和体系，官方色板留作品牌校准参考。

---

## 五、数据处理流水线（run.bat / run_pipeline.py）

> ⚠️ **版本差异提示（2026-10-18 核实）**：本地 v3 副本的 `03_scripts/` 共 12 个脚本，
> `run_pipeline.py` 实际到 **Step 4d 结束**，输出目录是 `02_processed_data/dashboard_data/`；
> 未找到下述 Step 4e/4f 与 `dashboard_ready/`。以另一台机器上的最新版本为准，此节描述以那台机器为权威。

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
        ├── Step 4e：filter_dashboard_fields.py        （以另一台机器版本为准）
        │           → dashboard_ready/（瘦身版：8列，按item_code白名单过滤）
        ├── Step 4f：archive_dashboard_history.py      （以另一台机器版本为准）
        │           → 历史累积 → dashboard_ready/fact_financials_combined.csv（含全历史）
        └── Step 5：质量报告
```

### 数据血缘图

```
【自动化（run.bat）】
PDF（深交所/上交所）→ raw_extracted/ → fact_financials_all.csv ┐
PDF（AUMOVIO，手动放入根目录）→ aumovio_financials.csv        ┤→ 瘦身/合并
final_facts/_history/<Q>/（手动归档上季度）                    ┘→ fact_financials_combined.csv

【人工维护（run.bat 不覆盖）】
GitHub workflow 自动抓取 → market_data_verified.csv + market_history.csv

【手动录入（不走流水线，直接在看板录入/CSV导入）】
股权结构 / 员工构成 / 产品线 → 浏览器 localStorage（可导出CSV备份）

【部署】
combined + sales_history + profit_history → 手动推送到 Competitor-Analysis/data/ → GitHub Pages
```

### 5.1 随机抽查质检（`03_scripts/qa_spot_check.py`，2026-10-19 新增）

> 目的：不重跑全量校验，每次随机抽几格，用**另一条独立抽取链路**反查我们的 PDF 解析结果；
> 一旦对不上，就说明抓取/映射程序有缺陷。

```
venv/Scripts/python.exe 03_scripts/qa_spot_check.py            # 每家公司随机 1 格（默认）
venv/Scripts/python.exe 03_scripts/qa_spot_check.py -n 2       # 每家公司 2 格
venv/Scripts/python.exe 03_scripts/qa_spot_check.py --company Luxshare --period 2026Q1 --all
venv/Scripts/python.exe 03_scripts/qa_spot_check.py --official --download   # 附巨潮官方PDF并定位页码
```

- **比对源**：A 股 5 家 → 东财 F10 结构化接口（`RPT_F10_FINANCE_GINCOME/GBALANCE/GCASHFLOW`，独立解析链路）；NASN → 东财港股 F10（IS/BS）；Aumovio 与 NASN 现金流量表无结构化源，不纳入自动抽查
- **校验维度**：数值（相对容差 1e-6）、币种、期间；额外识别单位差（×100/千/万/亿）与符号相反（`FAIL_UNIT` / `FAIL_SIGN`）
- **随机但收敛**：每次随机抽，优先抽台账里没抽过的格子；`02_processed_data/qa/spot_check_ledger.json` 记录覆盖进度，日志 `spot_check_log.csv` 逐条留痕。指定 `--seed` 时为纯随机可复现
- **不一致时**：脚本自动给出巨潮资讯（官方披露平台）的定期报告 PDF 链接；`--download` 会下载并在原文定位该数字所在页码与上下文，用于裁定"是我们解析错"还是"第三方口径不同"
- **退出码**：0 全过；1 有 FAIL（抓取程序可能有缺陷）；2 无可比对样本
- 2026-10-19 首轮 376 格可自动核对格子**全量比对：0 不通过**（过程中发现并修正 3 个真实问题，见变更记录）

---

## 六、关键文件说明

### 本地项目（financial_auto_project_v3）

| 文件/目录 | 说明 |
|---|---|
| `index.html` | **看板前端最新工作副本**（改完推 GitHub 根目录） |
| `data/` | 从 GitHub 下载的 5 个 CSV 镜像，供本地起 http 服务预览 |
| `00_source_registry/mapping_display_4.csv` | 科目映射表：原始科目名 → 标准 item_code |
| `00_source_registry/fact_sales_history.csv` | 手工补的历史年度营收 |
| `02_processed_data/final_facts/` | 完整大表（所有科目） |
| `02_processed_data/dashboard_data/` | 本地 Step 4d 输出（sales/profit_history 等） |
| `05_output/manual_blocks/manual_blocks_A股股权与员工.csv` | 预置数据的 CSV 版（262行），可 Import 进看板 |
| `05_output/manual_blocks/manual_blocks_template.csv` | 空模板 |
| `.workbuddy/tmp/em/` | 股权/员工数据采集脚本（东财+年报PDF），可复跑 |
| `.workbuddy/tmp/index_before_v*.html` | 每轮 UI 改版前的备份 |

### dashboard 8 列结构

```
company | statement_type | period | item_code | item | item_en | value | currency
```
- `currency`：AUMOVIO=EUR，其余=CNY（**NASN 财务数据也是 CNY**，报表以人民币列示；只有 market_data 里 NASN 股价/市值为 HKD）；去重键 `(company, statement_type, period, item_code)`

### GitHub 仓库（Competitor-Analysis/data/）

| 文件 | 来源 | 说明 |
|---|---|---|
| `fact_financials_combined.csv` | 流水线 | **主数据文件**（⚠️ 当前线上版本仅 459 行/19 个 item_code，A股只有 2025Q1/Q2/Q4+2026Q1/Q2，缺 2025Q3 与 2024 全年，历史趋势靠 sales/profit 两张历史表支撑） |
| `fact_sales_history.csv` / `fact_profit_history.csv` | 流水线 | 年度+季度历史趋势 |
| `market_data_verified.csv` / `market_history.csv` | GitHub workflow | 股价（每日自动更新） |

---

## 七、每季度操作手册

```
1. 把新季度 PDF 放入根目录（AUMOVIO）或 01_raw_data_*/ 目录（A股/港股）
2. 把上季度的 final_facts/*.csv 手动归入 final_facts/_history/<上季度>/
3. 双击 run.bat（Step 0 加 --download 参数才会自动下载 PDF）
4. 检查 terminal 输出，确认各 Step 正常完成
5. 推送 data/ 三个财务 CSV 到 GitHub；workflow 自动更新股价
6. 手动板块数据（股权/员工）：新季度直接在看板上选公司+季度录入，
   或更新 MANUAL_SEED + 重新生成 CSV；录完 Export CSV 留备份
6b. Capital Markets 行业 PE/PB：打开看板 Capital Markets 区块，在"A-Share Chassis & Auto-Parts Industry Benchmark"卡片填入当季度行业平均 PE/PB（来源如申万行业指数/东方财富板块估值），点 Save（存 localStorage，按季度隔离，不随 CSV 更新）
7. index.html 有改动时一并推送根目录
```

---

## 八、已知问题与坑

### 流水线 / 数据

| 问题 | 状态 | 说明 |
|---|---|---|
| `venv` 指向旧项目 | ⚠️ | 跑脚本用 `PYTHONPATH=旧venv/Lib/site-packages` + 管理版 python |
| mapping 中 row_type='header' 不一定是真标题 | ✅已处理 | 标准化代码只删无数值的 header 行，勿回退 |
| A股 Q1 对比列是上年同期数但列头写期末日期 | ✅已处理 | 非 BS 报表强制 derive_comparison_period |
| 半年度列头（"2026年半年度"→Q2）解析 | ✅已处理 | 否则 IS/CF 期间整体左移一期 |
| combined 表期间覆盖不全 | ⚠️待补 | 缺 2025Q3/2024 全年；A股Q1 数据看板未同步 |
| NASN 源头缺 2025Q2/2026Q2 | ⚠️ | 线上历史表有，本地 workspace 无法复现 |
| 净利润口径 | ⚠️ | combined 主表 IS_NP（含少数股东）vs 历史表归母优先，两处不同口径 |
| CCC 分析层（DSO/DPO/DIO） | 📝设计稿 | `05_output/analysis_layer_design.md`；依赖的 BS_CA_INV 等不在 19 个 item_code 白名单，落地需先扩白名单 |
| `00_source_registry/mdna_notes.csv` / `product_lines.csv` | ⚠️ | 实为 xlsx 改后缀，CSV 读取会报错 |
| 知了财报 | 🚫 | 有人机验证墙，抓不到，用东财 F10 + 巨潮/东财年报 PDF 替代 |

| A股 Q1 资产负债表列序 | ⚠️待修 | 立讯 2026Q1 的 6 个 BS 科目全部取到了上期（2025-12-31）列，已按官方 PDF 手工更正；根因在列序判定（期末列在左仍取错），重跑流水线可能复发，建议加回归用例 |
| 自动抽查覆盖盲区 | ℹ️ | Aumovio 全部科目、NASN 现金流量表没有结构化第三方源，`qa_spot_check.py` 不纳入；这两类只能靠 `--official`/人工看 PDF |
| NASN 报表币种 | ✅已修 | 报表以人民币列示（东财港股 F10 CURRENCY=CNY；营收 6.15 亿元与证券时报/中国商报/cibo 口径一致；上市/交易货币为港元是另一回事）。2026-10-19 底表 HKD→CNY，并同步修前端 `companyRate`：NASN 且 `currency=CNY` 返回 1 不再折算（此前一律 ×0.855，低估 ~14.5%） |

### index.html 前端坑（改版时注意）

| 坑 | 说明 |
|---|---|
| 全局函数名 = 元素 id | `function kpis()` 与 `id="kpis"` 同名时，全局绑定被函数占用，`kpis.innerHTML` 赋到函数对象上，**静默不渲染**。同类 id 一律 `document.getElementById()` 访问 |
| `[hidden]` 被 display 覆盖 | `.pie-empty`/`.entry` 等 `display:grid/flex` 的元素必须配 `.xxx[hidden]{display:none}`，否则 hidden 属性失效、空态常显 |
| 移动端横向溢出 | flex/grid 子项要 `min-width:0`（.side/.nav 踩过） |
| 图例 Total 行 | `.pie-legend li.lg-total` 需单独 `grid-template-columns`（4列图例与2列 Total 不同构） |
| 全 0 模板行 | 股权/员工首次渲染给 0 值模板行，判断"有数据"必须按合计>0（`blockHas`/`hcPeriod`），否则 0 行抢占期间选择 |
| 变动基准 | 股权环比基准取上一期**全部十大股东**映射，避免"进入前五"被误判为增持；首期 change 必须为空 |
| 主应用异步填充下拉 | 手动板块脚本用 MutationObserver 监听 `#company/#period` 的 childList 重建后重渲染 |

---

## 九、变更记录

### 2026-10-22（本轮）

| 改动 | 详情 |
|---|---|
| Education Mix + R&D Headcount & Compensation 新图表 | Ownership & People 板块新增两张图：学历构成饼图（`eduPie`，`ciEducationMix`）+ 研发人数/人均薪酬柱线组合图（`rdChart`，`ciRdHeadcount`）；CSV 契约新增 `education`/`rd` 两个 block；真实数据见 `Competitor Analysis for Reference/education-rd-import.csv`（逐字核对 NASN/Baolong/Tuopu/BTL 四家 PDF 原文，Tuopu 研发人数未披露故未纳入）；详见 3.6 节 |
| 修复两个阻断性 bug | `importData()` 原不识别 `education`/`rd` block（CSV 导入被静默跳过）；`periodsAsc()` 原不扫描 `db.rd`（R&D 柱状图拿不到横轴数据）；两处均已修复，`exportCsv()`/`TEMPLATE` 同步支持 |
| Annual Deep-Dive 自动从 `data/` 加载 | 页面加载时自动 `fetch('data/IMPORT_manual_all.csv')` + `fetch('data/IMPORT_annual_all.csv')` 并复用 `importData()`/`adImportData()` 自动灌入，不再需要每次手动点 Import；文件不存在时静默回退到手动 Import，不影响其他模块 |
| Annual Deep-Dive 9 张卡片各配专属年份/季度下拉框 | 原先靠顶部统一下拉框控制全部卡片，上下滚动对照困难；改为逐卡片旁挂专属下拉框，分 3 组联动（季度组 A：Shareholding+R&D；年度组 B：Headcount+Education+Product Line；年度组 C：Profile+Customer+Region+Segment）；发现并修复了最初 6 个"镜像"下拉框只有 HTML 没有 JS 绑定、选中不生效的 bug，现在组内任一下拉框都真双向联动 |
| Revenue/Profit Trend 新增 QoQ 环比箭头 | 原只有 YoY 同比箭头，新增 `prevQKey()`/`qoqSeries()` 两个纯函数计算环比，柱子上方 YoY/QoQ 两行箭头并列显示；季度格式数据自动支持未来期间，年度数据自动跳过不报错 |
| 次要图表 Notes 改为按需展开 | Capital Markets 区块整体移除 Notes 栏（占满整行）；其余 6 个次要区块（History/Profitability/Positioning/Cash Flow/Working Capital/Liquidity/Benchmark Table）Notes 栏改为默认收起的「💬 Add comment」开关，有内容自动展开+高亮；Executive Signals 的 Notes 栏保持固定显示不变 |

### 2026-10-22（交互/UI 走查，本轮追加）

全站交互走查共发现 13 项不符合人类操作习惯的问题，按严重度分三档，已全部修复：

**🔴 严重**

| # | 问题 | 修复方式 |
|---|---|---|
| 1 | 自动滚动跳转——`data/` 自动加载成功后无条件 `scrollIntoView`，导致每次刷新页面都被强制拽到 Annual Deep-Dive | `applyImport()` 新增 `doScroll` 参数：仅用户**手动点击** Import 按钮时才滚动；自动 fetch 路径传 `false`，不再打断刷新体验 |
| 2 | 清空类操作无二次确认——Clear quarter/year、行内 `×`、Product Line 删除行/按年删除、Notes Clear 均一键永久删除且无撤销 | 统一加 `confirm()` 弹窗；行内 `×` 仅当该行有实际内容时才弹确认（空行直接删，不打扰） |
| 3 | Notes 格式工具栏 + Clear 按钮仅 hover 才显示，触屏设备无法发现 | 新增常驻「⋯」按钮（`hover:none` 媒体查询下显示），点击可手动固定展开工具栏 |

**🟡 中等**

| # | 问题 | 修复方式 |
|---|---|---|
| 4 | 4 套独立「期间」概念并存，容易误以为联动 | 顶部全局 period 旁加常驻提示文案；Annual Deep-Dive 9 个下拉框加彩色小标签区分三组（蓝 `Qtr`/绿 `Year·A`/粉 `Year·P`） |
| 5 | R&D 卡片绑定季度选择器，但数据本质按年，Q1-Q3 误显示"无数据" | 改绑到年度组（`annPeriodRd`），`slotB()`/`annPeriods()` 同步识别 `rd`，与 Headcount/Education/Product Line 一致 |
| 6 | Import Annual / Import Manual 按钮并排，选错只提示"No usable rows found" | 检测到列名对不上目标 block（如传了 `year` 列却点 Import Manual）时，明确提示"是否点错了按钮" |
| 7 | CSS 死代码——`.chart.history`/`.chart.pl`/`.chart.positioning`/`.exec-block` grid 各有两份重复定义，后者覆盖前者 | 清理重复规则，每个 class 只保留一份生效定义 |
| 8 | 原生 `title` 提示 hover-only，触屏不可见 | 随 #4 一并改为常驻可见标签 |

**🟢 轻微**

| # | 问题 | 修复方式 |
|---|---|---|
| 9 | Annual Deep-Dive 单卡折叠此前形同虚设——`.adive-card.is-open` 类从未真正控制显示（CSS 无条件 `display:block`） | 补上 `.adive-card:not(.is-open) .adive-body{display:none}`；9 张卡片标题栏新增可点击折叠箭头（▾/▸）+ 独立展开状态记忆（localStorage），下拉框/按钮点击不误触折叠 |
| 10 | FX 汇率输入框改完数字要手动点"Apply FX"才生效，和其他控件行为不一致 | 输入框失焦(`change`)或按回车时自动触发 Apply FX |
| 11 | Cash Flow Y 轴缩放滑块记忆值疑似跨公司残留 | 核查后确认 `render()` 已在切换公司时重置 `_cfUserMax=null`，无需改动 |
| 12 | Peer Trend 图例可逐个隐藏公司，但无一键恢复 | 新增"Show all"按钮，清空隐藏集合、恢复全部公司曲线 |
| 13 | 侧边栏折叠后切换公司要点两下 | 维持现状（鼠标浮窗展开+聚焦为刻意设计），走查中顺带发现并修复了 #9 更严重的单卡折叠失效问题 |

### 2026-10-19（本轮）

| 改动 | 详情 |
|---|---|
| Executive Signals 前移重构 | 8 张大数字卡（营收主卡 2 列 48px），排名 pill + vs 中位数偏离 + 排名进度条 + 异常 flag；KPI 行删除、指标并入；Executive 区只留一个写字区 |
| 全局玻璃质感 UI | 9 阶低饱和 ramp + 半透明白卡 + backdrop blur + 三团色雾底；Capital Markets 从深色 Bloomberg 风改为浅色玻璃；JS 公司色/图表网格色同步 remap |
| 可折叠侧边栏 | 手柄 + 78px 细条态 + 数字 chip + hover 浮标 + localStorage 记忆；导航重排为与页面顺序一致（12 项，新增 Annual Deep-Dive） |
| 手动录入三板块 | 股权/员工双环形图 + 产品线柱线混合图；行编辑器、独立期间下拉、chip 删除、Export/Import/Template |
| A股真实数据预置 | 5 家 × 7 期股权（含 pp 变动）+ FY2024/2025 员工构成；来源东财 F10 + 年报 PDF，全部核对 |
| CSV 导入体验修复 | 导入后自动切到有数据的公司+最新期间并滚动定位；公司 chips 一览数据覆盖；toast 提示；空态提示该公司其他有数据期间；GBK/UTF-8 自动识别；`#company` 空时兜底填充 |
| 隐藏 bug 修复 | `function kpis` 与 `id="kpis"` 同名导致 KPI 从未渲染；`.pie-empty[hidden]` 失效；Total 行错位；首期假变动 +18.24 |
| 股价趋势图坐标/图例修复 | 趋势图 tick/标题/图例原为白色（深色卡遗留）在浅色卡上不可见 → 改主题色 `#6E6B84` + 网格 `rgba(83,74,183,.10)` + 坐标轴线；Chart.js 内置图例换成图表下方 HTML 网格图例（3 列、色条对齐、点击显示/隐藏数据集、选中公司高亮，状态存 `window.__trendHidden`）；tooltip 补 titleColor/bodyColor |
| Capital Markets 新增行业 PE/PB 均值卡 | `industryBenchCard`：自动算 5 家 A 股同业（BTL/Baolong/HASCO/Luxshare/Tuopu）PE/PB 简单平均，并提供两个输入框手动录入"行业整体"PE/PB 均值（每季度一次，存 `localStorage['ciIndustryBench_<period>']`）；不进 CSV、不进流水线，纯前端展示层 |
| Cash Flow 双图合并 | Cash Flow Benchmark（OCF/ICF/融资现金流柱）与 Cash Conversion Benchmark（OCF/FCF 利润率柱）信息重复 → 合并为一张全宽组合图：3 组柱（左轴 CNY bn）+ OCF margin 实线 / Free CF margin 虚线（右轴 %）；删除第二张画布与 `cashFlowStructureChart/cfb/cfs`，改用 `cashFlowChart`+`charts.cf` |
| 新增随机抽查质检 | `03_scripts/qa_spot_check.py`：每家公司随机抽格 → 对照东财 F10（独立链路）→ 不一致时自动给巨潮官方 PDF 链接并可下载定位页码；带覆盖台账与日志，退出码可接 CI |
| 修正 NASN 币种标注 | 底表/销售历史/利润历史里 NASN 89+8+7 行 currency 由 HKD 改为 **CNY**（报表以人民币列示，数值与东财港股 F10 完全一致）；此前前端按 0.855 折算，导致 NASN 全部财务数字被低估约 14.5% |
| Annual Deep-Dive FY2025 数据 | 生成 `05_output/manual_blocks/annual_deep_dive_FY2025.csv`（61 行，6 家公司全覆盖）：从 5 家 A 股 FY2025 年报 PDF（`02_processed_data/qa/annual_2025/`，巨潮下载）提取员工数/前五客户占比/分地区收入毛利率/分产品收入毛利率；NASN 取自内部竞品 deck。A 股财务明细以年报原文为准，Plants/RD_Centers 部分来自竞品 deck（Tuopu 100+/4、BTL 17/8、Baolong 12/15、NASN 3/1）；HASCO 员工 81,306=国内 54,621+海外 26,685（年报 p32 注）；region 收入单位 CNY mn、gm %；Node 模拟导入 61/61 行 0 skipped，headless 浏览器端到端验证渲染正常 |
| 前端 NASN 汇率分支修复 | `companyRate` 原对 NASN 非 HKD 兜底也返回 hkdRate → 币种改 CNY 后仍被 ×0.855；已加 `cu==='CNY'→1` 分支，hero 汇率控件文案同步改为"NASN QUOTE FX · HKD/CNY（仅股价/市值）"；截图验证 2026Q2 营收 0.37B CNY 正确 |
| 修正 Luxshare 2026Q1 资产负债表 | 6 个 BS 科目取到了**上期（2025-12-31）列**（如流动资产合计 196.02bn 应为 212.61bn）；已按一季报 PDF 第 5–6 页更正。⚠️ 流水线列序缺陷仍在，重跑可能复发 |
| 统一 NASN 税前亏损符号 | `IS_LOSS_BEFORE_TAX` 7 期由正数改为负数，与 `IS_NP` 及外部源一致（前端 `-Math.abs()` 兜底逻辑不受影响） |
| 板块顺序重排 | 12 个板块按分析叙事线重排：总览 → 历史趋势 → 盈利 → 增长-盈利定位 → **现金流** → 营运资本 → 流动性 → 资本市场 → 股权/员工 → 产品线 → 年度深度 → 基准表（附录收尾）；导航编号 01–12 同步重排（导航高亮按 nav 顺序=DOM 顺序映射，两者必须一致） |
| Annual Deep-Dive 年度板块 | 年报口径四卡（公司概况/客户集中度/中国vs海外/产品分部 P&L），独立年份选择 + CSV Import/Export/Template + 公司 chips + 折叠卡 + 无数据时导航半透明；详见 3.5 节 |

### 2026-10 前期（历史记录，见 git log 与旧版文档）

数据源统一为 combined 单文件、init() 偏移动态计算修复、公司色全局统一、选中行高亮改公司色、单色阶梯、tabular-nums、Valuation Positioning/Executive Market Signals 删除等。

---

## 十、本项目相关文档索引

| 文档 | 内容 |
|---|---|
| `PROJECT_KNOWLEDGE.md`（本文件） | 项目全貌、数据契约、设计系统、手动板块、变更记录 |
| `DESIGN.md` | 旧设计系统记录（官方色板 JSON、通用原则、红线清单），部分已被第四节取代 |
| `awesome-design-md-main/` | 外部设计参考库（Mercury/Stripe 等范式） |
| `05_output/analysis_layer_design.md` | CCC/异动信号分析层设计稿（未实现） |
