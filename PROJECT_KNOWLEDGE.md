# AUMOVIO 竞争情报项目 — 完整知识文档
> 最后更新：2026-10-19 | 适用于下次对话直接喂给 AI 使用
> 本次更新：Executive Signals 重构、全局玻璃质感 UI、可折叠侧边栏、手动录入三板块（股权/员工/产品线）、A股真实数据预置、CSV 导入体验修复

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
| NASN | 港交所 | HKD | 2026-08 上市，报表货币为人民币，走 convert_nasn.py 单独处理 |
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

> ⚠️ **重要**：`init()` 用 `fc=PATH.fin.length` 动态计算偏移量，不能硬编码
> `x[0].concat(x[1])`，否则后续变量全部错位，Trend/Capital Market/Heatmap/Cash Flow 全部无数据。

### 汇率换算规则

- AUMOVIO（EUR）：界面 `eurRate` 输入框（默认 8.35）
- NASN（HKD）：界面 `hkdRate` 输入框（默认 0.855）
- 其余公司：固定 1（本身人民币）

### 看板模块（11 个，页面自上而下顺序）

| # | 导航/section id | 数据来源 | 核心内容 |
|---|---|---|---|
| 01 | Executive Overview `overview` | fin+sales+profit | Hero（公司/季度/汇率控制）+ **Executive Signals（8 张大数字卡）** |
| 02 | Historical Trends `history` | sales+profit | 年度(FY)/季度(Q)营收与净利润走势 |
| 03 | Profitability `benchmark` | fin | 营收对标、毛利率/营业利润率/净利率、增长-盈利气泡图 |
| 04 | Working Capital `working-capital` | fin | AR/AP/净营运资本 |
| 05 | Positioning `positioning` | fin | 估值定位 |
| 06 | Ownership & People `ownership` | **localStorage + MANUAL_SEED** | 股权结构 + 员工构成两张环形图（手动录入） |
| 07 | Product Line `product-line` | **localStorage** | 关注产品线的营收柱 + ASP 线（手动录入） |
| 08 | Capital Markets `capital-market` | market+marketHistory | 股价、市值、PE、PB、YTD、近1月趋势 |
| 09 | Cash Flow `cashflow` | fin | OCF/ICF/FCF、现金转化率 |
| 10 | Liquidity & Efficiency `liquidity` | fin | 流动比率、研发强度、SG&A 强度 |
| 11 | Benchmark Table `heatmap` | fin | 6 项指标同业相对排名热力图 |

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
- `currency`：AUMOVIO=EUR，NASN=HKD，其余=CNY；去重键 `(company, statement_type, period, item_code)`

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

### 2026-10-19（本轮）

| 改动 | 详情 |
|---|---|
| Executive Signals 前移重构 | 8 张大数字卡（营收主卡 2 列 48px），排名 pill + vs 中位数偏离 + 排名进度条 + 异常 flag；KPI 行删除、指标并入；Executive 区只留一个写字区 |
| 全局玻璃质感 UI | 9 阶低饱和 ramp + 半透明白卡 + backdrop blur + 三团色雾底；Capital Markets 从深色 Bloomberg 风改为浅色玻璃；JS 公司色/图表网格色同步 remap |
| 可折叠侧边栏 | 手柄 + 78px 细条态 + 数字 chip + hover 浮标 + localStorage 记忆；导航重排为与页面顺序一致（11 项） |
| 手动录入三板块 | 股权/员工双环形图 + 产品线柱线混合图；行编辑器、独立期间下拉、chip 删除、Export/Import/Template |
| A股真实数据预置 | 5 家 × 7 期股权（含 pp 变动）+ FY2024/2025 员工构成；来源东财 F10 + 年报 PDF，全部核对 |
| CSV 导入体验修复 | 导入后自动切到有数据的公司+最新期间并滚动定位；公司 chips 一览数据覆盖；toast 提示；空态提示该公司其他有数据期间；GBK/UTF-8 自动识别；`#company` 空时兜底填充 |
| 隐藏 bug 修复 | `function kpis` 与 `id="kpis"` 同名导致 KPI 从未渲染；`.pie-empty[hidden]` 失效；Total 行错位；首期假变动 +18.24 |

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
