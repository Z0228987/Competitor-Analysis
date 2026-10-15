# AUMOVIO Competitive Intelligence Dashboard — DESIGN.md

> 本文件如实记录 `index.html` **当前**的设计系统（颜色、字体、间距、组件、图表配色、高亮逻辑），
> 作为下一轮视觉优化的基准文档。先对齐现状，再在此之上提修改方案。

---

## 1. 整体定位

- **产品类型**：内部战略分析看板（SAM China · Strategy & Marketing），对标 7 家汽车零部件/底盘企业的财务与市场数据。
- **视觉基调**：浅色主界面（白底+紫色品牌色），局部嵌入一个独立的"深色 Bloomberg 风"模块（Capital Market Intelligence）。
- **技术栈**：纯静态单文件 HTML + Chart.js 4，无构建流程，所有样式写在 `<style>` 内联块里。

---

## 2. 色彩系统（现状）

### 2.1 CSS 变量（`:root`）

| 变量 | 值 | 用途 |
|---|---|---|
| `--o100` | `#ffb7a1` | Orange 最浅（AUMOVIO 色板） |
| `--o200` | `#ff906e` | Orange 浅 |
| `--o300` | `#ff6938` | Orange 中 |
| `--o400` | `#ff4208` | **Orange 主色**（品牌橙/红） |
| `--o500` | `#da3806` | Orange 深 |
| `--p100` | `#a08ae5` | Purple 最浅 |
| `--p200` | `#7e60dc` | Purple 浅 |
| `--p300` | `#6c57d2` | Purple 中 |
| `--p400` | `#4827af` | **Purple 主色**（品牌紫） |
| `--red` | `= var(--o400)` | 语义别名，站内"红色"即品牌橙 |
| `--purple` | `= var(--p400)` | 语义别名 |
| `--dark` | `#27145f` | 深紫（渐变终点、tooltip 背景） |
| `--soft` | `#f2efff` | 极浅紫（标签底色） |
| `--ink` | `#171326` | 正文主文字色 |
| `--muted` | `#756f86` | 次要文字/坐标轴 |
| `--line` | `rgba(72,39,175,.12)` | 卡片描边 |
| `--shadow` | `0 16px 42px rgba(55,31,127,.11)` | 卡片阴影 |
| `--dn2/3/4` | `rgba(255,255,255,.96) / rgba(255,255,255,.84) / rgba(255,255,255,.66)` | 深色模块专用文字色——**已修复**：原值是偏暗紫灰（`#9389b3`等），现已改为白色系半透明阶梯，解决 Capital Market 模块文字偏暗问题 |
| `--dn5` | `#15102b` | 深色模块专用（极深底色，非文字） |

**已修复的问题**：Chart.js 的图表配置（Capital Market 两张趋势线图的图例/坐标轴）曾直接写 `color:'var(--dn2)'`——但 Chart.js 用 Canvas 绘制文字，不支持解析 CSS 变量字符串，浏览器会静默回退成黑色，这是文字在深色卡片上"看不清"的真正原因。现已把这 4 处替换为具体的 rgba 值。

**仍存在的问题**：`--o*` 和 `--p*` 两条色阶在**同一张图表里经常被同时使用**（例如 Cash Flow Benchmark/Working Capital 紫+橙+第三色混用），尚未按 8.2 节方案收敛为单色相阶梯，详见第 9 节。

**本轮已移除的变量**：`.focus-outline`、`--focus-grey`、`--focus-grey-line` 已从 `:root` 和样式表中整体删除（用户明确表示不需要灰色描边退路方案，选中态统一用"整行/整项高亮"代替）。

### 2.2 公司专属色（`COMPANY_COLORS`，JS 全局常量）——**已更新，当前生效值**

```js
{
  Aumovio:  '#ff4208',  // 品牌橙
  Luxshare: '#4827af',  // 品牌紫
  HASCO:    '#18a9e7',  // 蓝
  Tuopu:    '#12caae',  // 青绿
  Baolong:  '#aea633',  // 橄榄黄
  BTL:      '#bd7150',  // 棕
  NASN:     '#7b4f71',  // 藕紫
}
```

- 这是**全站唯一**的公司配色来源，`companyColor(c)` 读取它，`companyTint(c, alpha)` 由它换算半透明色。
- 复用位置：Executive Overview 气泡图、Revenue Benchmark 柱状图、Capital Market 市场表格/logo/股价趋势线、表格高亮（`tr.sel`）。
- **本轮改动**：不再是"3橙+4紫"的同色系堆叠，已改为 7 个分散在色相环上的颜色（橙/紫/蓝/青绿/橄榄/棕/藕紫），解决了旧版相邻色号区分度低的问题。
- **遗留问题**：`#18a9e7`（蓝）、`#12caae`（青绿）两个颜色不在 AUMOVIO 官方橙/紫色板（`--o100~500` / `--p100~400`）内，属于品牌色板之外新增的颜色，与 8.6 节 Don't 红线冲突，但由于也被 Cash Flow Benchmark / Working Capital / Cash Flow Structure 三张图的系列色复用，牵连较广，暂未处理（见第 9 节）。

### 2.3 高亮 / 选中状态——**已按用户最新要求简化**

- **浅色表格**（`tr.sel`）：通过内联 CSS 变量 `--sel-color / --sel-bg1 / --sel-bg2` 注入该公司专属色，竖线+渐变底色+加粗文字。
- **深色市场表格**（`.market-table tr.sel`）：同逻辑，变量名 `--sel-bg-dark`。
- **柱状图**（Revenue Benchmark、Cash Flow Benchmark、Working Capital）：选中公司用其专属色，其余统一降为中性灰紫 `rgba(148,138,168,.38)` 或坐标轴文字变灰。
- **气泡图**：选中公司描边加粗（3.5px vs 2px），填充色保持公司本色+透明度，未选中不降饱和（与柱状图逻辑不完全一致，但影响较小，未处理）。
- **股价趋势线**：非选中公司线条透明度降至 `4d`（约30%），选中公司保持实色加粗。
- **Benchmark Heatmap 表格**：不做相对排名染色，只做"选中即整行高亮"（见 5.5 节），逻辑与浅色表格 `tr.sel` 完全一致。

**已处理的历史反馈**：
1. ✅ 深色模块文字偏暗——已将 `--dn2/--dn3/--dn4` 的取值改为白色系 `rgba(255,255,255,.96/.84/.66)`，Chart.js 配置项里原先用 `var(--dn2)` 这种 CSS 变量语法（Canvas 无法解析，会静默回退黑色）的 4 处已替换为对应的真实 rgba 值。
2. ✅ "无法用专属色区分选中项时的退路方案"——按用户最新决定，**不再需要** `.focus-outline` 灰色描边方案，已从代码和本文档中移除；选中态统一只用"该行/该项整体高亮"的方式（见 2.3、5.5）。
3. ✅ Benchmark Heatmap 不再使用相对排名三色（绿/黄/红）染色，`heat()` 函数固定返回空字符串，`.good/.mid/.bad/.na` 四个 CSS 类已删除（死代码清理）。
4. ⚠️ **仍未处理**：同一张图表内紫+橙（或紫+橙+第三色）混用的问题（Cash Flow Benchmark、Working Capital、Cash Flow Structure、Revenue/Profit History 四张图），详见第 6、9 节。

---

## 3. 字体系统

- **字体族**：`Inter, "Segoe UI", "Microsoft YaHei", sans-serif`（全站统一，无衬线）。
- **字号层级**（现状，未成体系，散落定义）：

| 用途 | 字号 | 字重 |
|---|---|---|
| H1（页面大标题） | `clamp(29px,3vw,45px)` | 400（未显式设置，继承） |
| Hero 标题 h2 | 24px | 继承 |
| 面板标题 `.pt` | 15px | 继承（约600，浏览器默认h3） |
| KPI 大数字 `.kv` | 31px | 900 |
| KPI 标签 `.kl` | 10px | 850 |
| 表格文字 | 10px | 继承/800（选中行） |
| Eyebrow/小标签 | 11px | 900，大写+字距 |
| 图表坐标轴 | 10px | 400 |
| 市场模块 metric 数字 | 20px | 继承 |

**问题点**：字号梯度过密且随意（29/24/20/15/11/10px 之间没有统一比例），字重两极化（400 或 850/900，缺少中间层级如 600），数字没有使用等宽数字特性（`font-variant-numeric: tabular-nums`），财务表格数字对不齐。

---

## 4. 布局结构

### 4.1 页面骨架

```
.app (grid: 250px侧栏 + 1fr主内容)
├── .side  —— 固定侧栏，深紫渐变背景，品牌Logo + 8个锚点导航
└── main
    ├── .top          —— 页头：Eyebrow + H1 + 状态胶囊
    ├── .hero (#overview) —— 渐变横幅：Executive Snapshot + 公司/期间选择器 + FX输入
    ├── .part-layout（×8，每个版块一个）
    │     ├── 内容区（.kpis / .g2 / .g3 / section）
    │     └── .part-note（右侧/下方可编辑批注卡片）
    └── .footer
```

### 4.2 8 大功能区块（对应侧栏导航锚点）

1. **Executive Overview** `#overview` — 6 张 KPI 卡
2. **Profitability** `#benchmark` — Revenue Benchmark + Margin Benchmark 并排
3. **Historical Trends** `#history` — Revenue Trend + Profit Trend 并排（含全年/季度图例）
4. **Working Capital** `#working-capital` — 单图：AR/AP/NWC 占收入比
5. **Capital Market** `#capital-market` — **唯一深色模块**：ticker跑马灯、snapshot卡、peer表、trend双图
6. **Cash Flow** `#cashflow` — 独立"卡片式小节"（带自己的渐变背景容器 `.cashflow-section`），内含 Benchmark + Conversion 双图
7. **Liquidity & Efficiency** `#liquidity` — 3 图并排（Current Ratio / R&D / SG&A）
8. **Benchmark Table** `#heatmap` — 对比表格，**不做相对排名染色**，仅在选中某公司时整行高亮（逻辑同其他表格的 `tr.sel`）

### 4.3 网格与间距

- 常用 grid：`.g2`（2列）、`.g3`（3列）、`.g4`（4列）、`.kpis`（6列）。
- 卡片圆角不统一：11px（tick分隔无圆角）/ 13px（pill, sig）/ 14px（brief-media）/ 16px（market-dark-card）/ 19px（kpi, panel, part-note）/ 23px（capital-shell, cashflow-section）/ 27px（hero）。
- 阴影统一变量 `--shadow`，但使用密度高（几乎所有白色卡片都带），深色模块另有独立阴影值。

---

## 5. 组件清单

### 5.1 KPI 卡（`.kpi`）
- 白底、圆角19px、阴影、内边距17px。
- 结构：标签（`.kl`，大写11号，--muted）→ 大数字（`.kv`，31px/900，--purple）→ 排名徽章（`.rank`，药丸形，浅紫底深紫字）+ 可选 YoY 趋势（红涨绿跌：`.up`绿 `.down`红橙）。

### 5.2 面板（`.panel`）
- 通用图表容器，白底+圆角19px+阴影。标题区 `.ph`（flex两端对齐）+ 副标题 `.ps`（--muted，10px）。

### 5.3 深色 Capital Market 模块（`.capital-shell`）
- 背景：`radial-gradient(...) + #160b2e` 近黑紫色底。
- 内部卡片 `.market-dark-card`：更深一级 `rgba(16,20,33,.96)` 渐变。
- 文字色：标题白 `#fff`，副标题/说明 `--dn2/--dn3`（**已修复**：现为白色系半透明 `rgba(255,255,255,.96/.84)`，不再是旧版偏暗灰紫）。
- Ticker 跑马灯：横向无限滚动，entries 用绿`#3ddc97`涨 / 橙`--o200`跌。
- Peer 表格：深底白字，选中行用公司色高亮（逻辑同浅色表）。

### 5.4 Cash Flow 独立卡片（`.cashflow-section`）
- 浅紫橙双渐变背景 `linear-gradient(135deg, rgba(72,39,175,.055), rgba(255,66,8,.045))`，圆角23px。
- 内部两张图共用一个父级"卡片式小节"外观，区别于其他区块的"透明背景+独立panel"模式——**这是全站唯一一个区块级别带渐变底色的非深色模块**，风格上略突兀。

### 5.5 表格 / 热力图——**已按用户要求简化**
- `.heat` 药丸样式类与 `heat()` 函数仍保留在代码里（供单元格数字提供统一的圆角药丸容器样式），但 `heat()` 固定返回空字符串，**不再做任何红/黄/绿的相对排名染色**。
- `.good/.mid/.bad/.na` 四个旧配色类已删除（原先的死代码，排名染色逻辑停用后一直未被引用）。
- 选中某公司时，仅通过整行 `tr.sel`（渐变底色+左侧色条+该公司专属色，见 2.3）来标识，这一行为与浅色表格、Capital Market 深色表格完全一致，不再新增'排名优劣'的视觉语义。

### 5.6 悬浮批注面板（`.briefing`）
- 唯一的"浮动+可拖拽+可折叠+可停靠"交互组件，白底、品牌紫描边虚线输入区。

### 5.7 Part Notes（每区块批注卡）
- 统一的富文本编辑卡片，工具栏默认隐藏，hover/focus 时展开（"Scheme A"过渡动画）。

---

## 6. 图表配色现状一览（逐图表）

| 图表 | 系列与颜色 | 问题 |
|---|---|---|
| Revenue Benchmark（柱） | 选中=公司色，其余=`rgba(148,138,168,.38)`灰紫 | OK，单一强调色逻辑 |
| Margin Benchmark（柱，3系列） | 紫`#4827af` + 浅紫`#a08ae5` + 橙`#ff6938` | ⚠️ 紫橙混用，撞色 |
| Growth-Profitability 气泡图 | 每家公司各自专属色 | 7色混合，图例拥挤 |
| Current Ratio / R&D / SG&A（3个单系列小图） | 选中=公司色，其余灰紫 | OK |
| Cash Flow Benchmark（柱，3系列：OCF/ICF/FCF） | 紫 + 橙(.84) + 橙(.82,稍浅) | ⚠️ 紫+双橙，层次含混 |
| Cash Flow Structure（柱，2系列：OCF margin/FCF margin） | 绿`#22c55e` + 紫`#4827af` | 绿色是全站唯一一处"非品牌色"系列色 |
| Working Capital（柱，3系列：AR/AP/NWC） | 紫 + 橙 + 绿`#22c55e` | ⚠️ 紫橙绿三色混合 |
| Revenue/Profit History（柱） | 全年=紫`rgba(72,39,175,.72)`，季度=橙`rgba(255,66,8,.88)`，亏损单独标红`rgba(255,66,8,.94)` | 语义色（年/季）与"亏损警示"色重叠，含混 |
| 市场估值/股价趋势线 | 各公司专属色 | 继承 COMPANY_COLORS 问题 |

---

## 7. 已知问题汇总（更新：已处理 / 仍遗留）

1. ✅ **深色模块整体偏暗**——已修复，`--dn2/--dn3/--dn4` 改为白色系半透明值，且修复了 Chart.js 配置里 `var(--dn2)` 无法被 Canvas 解析的根因问题。
2. ⚠️ **仍遗留：同图表内紫橙撞色**：Cash Flow Benchmark、Working Capital、Cash Flow Structure、Revenue/Profit History 几张图仍存在"紫+橙"或"紫+橙+第三色"混用，尚未收敛成单一色相深浅阶梯。
3. ⚠️ **仍遗留：公司色区分度**：7色已从"3橙4紫"改为橙/紫/蓝/青绿/橄榄/棕/藕紫分散色相（区分度已提升），但其中蓝 `#18a9e7`、青绿 `#12caae` 不在官方橙紫色板内，属于新增颜色，与 8.6 节红线冲突。
4. ✅ **选中态退路方案**：按用户最终决定，**不需要**额外的灰色描边退路方案，已移除 `.focus-outline` 相关代码；所有选中态统一走"该行/该项整体高亮"逻辑。
5. 📝 **圆角、阴影、间距无统一阶梯**，仍未处理，优先级较低，后续如需精细化视觉统一可参考 8.1 思路另行定义 `--radius-xs/sm/md/lg`。

---

## 8. 优化方案（已确认，待实施到 index.html）

> 参考对比了 `awesome-design-md-main`（84 个品牌设计系统写作范例）后归纳出的 5 条可借鉴原则，
> 结合用户本轮反馈（偏暗、紫橙撞色、深色模块文字对比度、选中态退路），形成以下具体决定。

### 8.1 颜色变量改为"按角色命名"，而不是"按色号命名"

现状 `--o100~500 / --p100~400` 是"外观命名"，改色号要满页面找谁在用。改为在**现有色阶之上**叠加一层角色别名，色阶本身不删除（仍可作为角色变量的取值来源）：

| 新角色变量 | 取值 | 用途 |
|---|---|---|
| `--color-brand` | `var(--p400)` `#4827af` | 品牌主紫：Logo、侧栏、按钮、Hero，不用于"数据分类" |
| `--color-accent` | `var(--o400)` `#ff4208` | 品牌强调橙：风险/警示/下跌语义，不用于"数据分类" |
| `--color-ink` | `#171326` | 浅色背景正文 |
| `--color-ink-muted` | `#756f86` | 浅色背景次要文字 |
| `--color-ink-on-dark` | `#ffffff` | **深色背景（Capital Market）主文字，强制纯白** |
| `--color-ink-on-dark-muted` | `rgba(255,255,255,.72)` | 深色背景次要文字（替代现有 `--dn2/--dn3/--dn4`，对比度统一拉高） |
| `--color-surface` | `#ffffff` | 浅色卡片底 |
| `--color-surface-soft` | `#f2efff` | 浅紫色块底（标签、tag） |
| `--color-hairline` | `rgba(72,39,175,.12)` | 分割线/描边 |
旧的 `--o100~500/--p100~400` 色阶保留，作为"数据可视化专用色阶"，改名语义：只给图表分类色使用，不再直接用于品牌/文字场景。

> **本节 8.1 的角色变量表是早期方案，尚未在 index.html 中实施**（`:root` 里目前仍是旧的 `--dn2/3/4` 命名，只是取值已改成白色系，并未重命名为 `--color-ink-on-dark` 等角色名）。并且 `--color-focus-ring` 这一项已因选中态退路方案被取消而废弃。是否值得为其余角色专门做一次变量改名重构，留待后续评估，不属于本轮紧急问题。

### 8.2 "一个主色相 + 深浅阶梯"铁律：图表分类色禁止紫橙混用

现有 3 张问题图表统一改为**单一色相的多级深浅**，不再在同一张图里混合 Orange 系和 Purple 系：

| 图表 | 现状（撞色） | 改为 |
|---|---|---|
| Margin Benchmark（3系列） | 紫`#4827af`+浅紫`#a08ae5`+橙`#ff6938` | 全部 Purple 阶梯：`--p400`（主）/`--p300`/`--p100` |
| Cash Flow Benchmark（OCF/ICF/FCF） | 紫+橙(.84)+橙(.82) | 全部 Purple 阶梯：`--p400`/`--p200`/`--p100` |
| Working Capital（AR/AP/NWC） | 紫+橙+绿 | 全部 Purple 阶梯：`--p400`/`--p300`/`--p100`，NWC 如需强调"净值"可用 `--o400` 单独强调（1系列，不构成"混用"） |
| Cash Flow Structure（OCF margin/FCF margin） | 绿`#22c55e`+紫 | 改为 Purple 深浅两级：`--p400`/`--p200`（去掉非品牌绿色） |
| Revenue/Profit History（年/季） | 紫（年）+橙（季）+橙（亏损警示，和"季度"色重复） | 年=Purple主色，季度=Purple浅一级，**亏损警示单独用 `--color-accent` 橙色+图案（斜纹/描边），不复用"季度"色** |

**公司对比类图表**（Revenue Benchmark 柱、气泡图、股价趋势线）例外：这类图表的"颜色=身份"，不受此约束，继续用 `COMPANY_COLORS`。

### 8.3 深色模块（Capital Market）文字全面提亮

- 所有标题、数值、表格正文：`--color-ink-on-dark`（纯白 `#fff`）
- 次要说明/标签：`--color-ink-on-dark-muted`（`rgba(255,255,255,.72)`，而不是现状偏暗的 `--dn3 #9389b3`/`--dn4 #7a7098`）
- 坐标轴、图例、tooltip 文字同步提升到白色系，不再使用偏紫灰的暗色

### 8.4 选中态退路方案——**已废弃，不再采用**

> 早期方案曾计划新增 `.focus-outline`（半透明浅灰描边框）作为"无法用专属色区分选中项"时的退路。用户最终决定**不需要**这个退路方案：所有选中态统一通过"该行/该项整体高亮"表达（公司专属色 + 渐变底色 + 左侧色条），不再引入额外的灰色描边样式。代码中 `.focus-outline` 类及 `--focus-grey/--focus-grey-line` 变量已全部移除。

### 8.5 财务数字等宽对齐（tabular-nums）

全站数字类元素（KPI `.kv`、表格数字单元格、市场模块 metric 数字）统一加：
```css
font-variant-numeric: tabular-nums;
font-feature-settings: "tnum" 1;
```
解决表格/卡片数字因比例字体宽度不一导致的"数字对不齐"问题，这是金融类页面的通用硬性要求（Stripe/Mercury/NerdWallet 均采用）。

### 8.6 Do's and Don'ts（新增红线，防止后续风格漂移）

**Don't**
- 不在同一张图表里把 Orange 系和 Purple 系同时用作"数据分类色"（⚠️ 仍有4张图违反，见第9节）
- 不在深色模块（Capital Market）使用对比度不足的中性灰文字（已修复）
- 不引入 `COMPANY_COLORS` / AUMOVIO 官方色板之外的新颜色（⚠️ 现状 `#18a9e7`蓝、`#12caae`青绿 违反此条，见2.2/7节）
- 不对 Benchmark Heatmap 做相对排名染色——仅用整行高亮标识选中项（已实施）

**Do**
- 公司识别色只服务于"谁被选中"，图表内多系列对比统一用同一色相深浅阶梯（图表改造部分仍未完成）
- 深色背景文字默认纯白，次要文字用白色透明度分级，不用另一套暗色（已实施）
- 财务数字统一 tabular-nums（已实施）
- 选中态统一用"整行/整项高亮"，不额外引入描边退路方案

---

## 9. 待实施清单（下一步动手改 index.html 时逐条对照）

- [x] Capital Market 深色模块文字提亮（`--dn2/3/4` 改白色系 + 修复 Chart.js `var()` 无法解析导致回退黑色的问题）
- [x] Benchmark Heatmap 去除相对排名染色，改为仅"选中整行高亮"；`.good/.mid/.bad/.na` 死代码已删除
- [x] 选中态退路方案 `.focus-outline` 已按用户决定废弃并从代码中移除（不再需要）
- [x] KPI/表格/市场模块数字已有 `font-variant-numeric: tabular-nums`
- [ ] `:root` 角色变量重命名（8.1，`--dn2/3/4` 等仍是旧命名，只是取值已改）——优先级低，暂缓
- [ ] Margin Benchmark 已改为纯 Purple 阶梯（✅ 已完成）；但 **Cash Flow Benchmark / Working Capital / Cash Flow Structure / Revenue-Profit History 四张图仍是紫+橙+蓝/青绿混用**，未按 8.2 方案收敛（下一轮重点）
- [ ] `COMPANY_COLORS` 里的 `#18a9e7`（蓝）、`#12caae`（青绿）不在官方色板内，是否替换为橙/紫色阶需与用户确认（替换会影响区分度，需权衡）
- [ ] 圆角/阴影/间距统一阶梯（`--radius-xs/sm/md/lg`）——未开始，优先级低

