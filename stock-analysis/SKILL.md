---
name: stock-analysis
description: 个股深度分析skill，基于westock-partner圆桌讨论框架，输出可视化HTML报告。包含：实时快照、近五年股价中位数、近五年净利润变化、近五年总股本与市值、近五年PE/PB/PS估值曲线、产业趋势/估值/基本面/信号四派圆桌分析、操作路线图；可另出SVG版上传「股票精灵」平台。触发条件：用户提到"分析XX股票"、"XX股票怎么看"、"帮我分析XX"、"上传到股票精灵"等个股分析与报告分发请求。输入：股票名称或代码。输出：完整可视化HTML文件。
description_zh: 个股深度可视化分析
description_en: Stock depth visual analysis
disable: false
agent_created: true
---

# stock-analysis · 个股深度可视化分析

## When to use

当用户需要对某只股票做深度分析时触发，关键词包括：
- "分析XX股票"、"XX股票怎么看"、"帮我分析XX"
- "XX能不能买"、"XX该不该卖"
- 任何涉及个股投资决策的深度分析请求

**不触发**：纯行情查询（用 westock-data kline 取最新收盘）、纯新闻浏览、板块级别讨论。

## 输入

- 股票名称或代码（如"安克创新"或"sz300866"）
- 可选：用户特别关注的维度

## Steps

### 0. 定位 westock-data CLI（⚠️ 必做，路径每次都可能变）

文档里写的内置路径 **通常不存在**。先探测真实入口：

```bash
# 1) 优先找 npx 缓存里的 skillhub 包
ls -d /Users/*/.npm/_npx/*/node_modules/westock-data-skillhub 2>/dev/null
# 2) 备选：全局 / 常见安装位
ls -d /Users/*/.workbuddy/skills/*/westock-data /usr/local/lib/node_modules/westock-data* 2>/dev/null
```

找到后固定用**受管 node** 执行（系统 node 可能版本不兼容），把入口存成变量复用：

```bash
NODE=/Users/luliming/.workbuddy/binaries/node/versions/22.22.2-6/bin/node
CLI=/Users/luliming/.npm/_npx/3d8d88d805f96382/node_modules/westock-data-skillhub/index.js
$NODE $CLI --help          # 先跑 --help 确认命令集，版本差异很大
```

> **版本差异警告**：本 CLI 版本**没有 `quote` / `news` / `risk` 子命令**。
> 最新价与指数一律用 `kline <代码> --period day --limit N` 取最后一根；
> 新闻用 `notice list` / `report list` 或 WebSearch；风险用 `notice list` + `risk` 替代（若无 risk 则用公告/质押数据人工判断）。

### 1. 搜索股票代码

```bash
$NODE $CLI search <股票名称>
```

确认标的身份（公司名、代码、市场、是否为ETF/ADR等），避免混用不同上市主体。

### 2. 查询全量数据

按以下顺序一次性查询所有需要的数据（⚠️ 注意各命令的**市场限制**，A股缺失的命令不要硬调）：

```bash
# 宏观指数（用 kline 取最新一根，不要用 quote）
$NODE $CLI kline sh000001,sh000300,sz399006 --period day --limit 2
$NODE $CLI kline hkHSI,hkHSTECH --period day --limit 2

# 个股实时行情 + 5年K线
$NODE $CLI kline <代码> --period day --limit 300        # 近一年，技术面用
$NODE $CLI kline <代码> --period day --start <5年前今天> --end <今天>   # 五年，分段抓（见下）

# 财报（A股/港股/美股字段名不同，见 Step 5）
$NODE $CLI finance <代码> --num 24        # 要 5 年逐期，期数给足（季度×20 期以上）

# 财报披露日历（找下一次财报日期，非常有用）
$NODE $CLI disclosure <代码>

# 资金面（⚠️ 各子命令市场限制不同）
$NODE $CLI fund flow <代码>              # A股 · 港股
$NODE $CLI fund margin <代码>            # 仅 A股（融资融券）
$NODE $CLI fund north-holding <代码>     # 仅 A股（北向持仓）
$NODE $CLI fund short <代码>             # 仅 港股 · 美股（卖空）
$NODE $CLI fund south-holding <代码>     # 仅 港股（南向持仓）

# 技术指标
$NODE $CLI technical <代码> --indicator macd,kdj,rsi,boll,ma

# 公司简况 / 股东结构 / 分红 / 回购
$NODE $CLI profile <代码>
$NODE $CLI shareholder <代码>
$NODE $CLI dividend list <代码> --years 5     # ⚠️ 必须带 list 子命令
$NODE $CLI buyback <代码>                     # A股 · 港股

# 一致预期（A股 · 港股）/ 机构评级（⚠️ 仅港股 · 美股，A股不可用）
$NODE $CLI consensus <代码>
$NODE $CLI rating <代码>

# 公告 / 研报（替代 news）
$NODE $CLI notice list <代码> --limit 10
$NODE $CLI report list <代码> --limit 10
```

#### ⚠️ A股 K线 250 根上限（必踩）

A股标的 `--limit 1260` 或超长 `--start/--end` **只会返回约 250 根**，静默截断，不会报错。
**解法：按年分段抓，再合并去重。**

```bash
mkdir -p /tmp/<code>
$NODE $CLI kline sz002352 --period day --start 2021-10-29 --end 2022-10-28 > /tmp/<code>/k1.txt
$NODE $CLI kline sz002352 --period day --start 2022-10-29 --end 2023-10-28 > /tmp/<code>/k2.txt
# ...共 5 段
cat /tmp/<code>/k*.txt | grep "^| 20" | sort -u > /tmp/<code>/kall.txt   # 应得 1200+ 根
```

> 注意：zsh 里 `for r in "a b"; do set -- $r` **不会切词**，别用循环变量拆字符串，直接写 5 条显式命令。
> `/tmp` 会被系统清理，写脚本前先 `mkdir -p`。

### 3. 获取宏观新闻

```bash
curl -s 'https://www.cls.cn/nodeapi/telegraphList?app=CailianpressWeb&os=web&rn=10' \
  -H 'Referer: https://www.cls.cn/telegraph'
```

检查新闻时效性（必须是近1-3天内的），否则改用WebSearch。判断是否有系统性风险。

### 4. 计算近五年股价中位数

从K线数据中提取收盘价，按年分组计算中位数：

```javascript
// 从K线数据解析
const byYear = {};
prices.forEach(p => {
  const year = p.date.substring(0, 4);
  if (!byYear[year]) byYear[year] = [];
  byYear[year].push(p.close);
});

function median(arr) {
  const sorted = [...arr].sort((a, b) => a - b);
  const mid = Math.floor(sorted.length / 2);
  return sorted.length % 2 ? sorted[mid] : (sorted[mid - 1] + sorted[mid]) / 2;
}
```

输出：每年中位数 + 年度最高/最低 + 5年整体中位数。

### 5. 提取近五年净利润变化

从 `finance` 查询结果中提取各年度报告的净利润数据。

#### ⚠️ A股 与 港股 财报 schema 完全不同（必踩）

`finance` 输出三张 markdown 表：`**income**` / `**balance**` / `**cashflow**`。
先 `grep -n "^\*\*" finance.txt` 看表名，再 `sed -n '<表头行>p'` 打表头确认字段——**不要凭记忆写字段名**。

| 项目 | 港股 | A股 |
|---|---|---|
| 年报筛选 | `PeriodMark == 12` | **无 PeriodMark**，用 `EndDate.endswith('12-31')` |
| 归母净利 | `ProfitToShareholders` | `NPParentCompanyOwners`（`_Q` 后缀=单季，`_TTM` 后缀=滚动12月） |
| 营业收入 | `OperatingIncome` | `OperatingRevenue` / `OperatingRevenueTTM` |
| 毛利率 | `GrossIncomeRatio` | **无此字段**，算：`(营收 − OperatingCost) / 营收` |
| 加权ROE | `RoeWeighted` | **无此字段**，算：`归母净利 / SEWithoutMI`（资产负债表） |
| 资产负债率 | `DebtAssetsRatio` | **无此字段**，算：`TotalLiability / (TotalCurrentAssets + TotalNonCurrentAssets)` |

解析 markdown 表的可复用片段：

```python
def parse_tables(path):
    tables = {}; cur = None; headers = None
    for line in open(path, encoding='utf-8'):
        m = re.match(r'^\*\*(\w+)\*\*$', line.strip())
        if m:
            cur = m.group(1); tables[cur] = []; headers = None; continue
        s = line.strip()
        if not s.startswith('|'): continue
        cells = [c.strip() for c in s.strip('|').split('|')]
        if all(set(c) <= set('- :') for c in cells): continue   # 跳过分隔行
        if headers is None: headers = cells
        else: tables[cur].append(dict(zip(headers, cells)))
    return tables
```

```javascript
// 关键字段（港股口径）：
//   EndDate          — 报告期，如 2022-12-31
//   ProfitToShareholders  — 归母净利润（港元）
//   NpParentCompanyGr1y   — 归母净利润同比增速(%)
//   OperatingIncome       — 营业收入
//   OperatingRevenueGr1y  — 营收同比增速(%)
//   GrossIncomeRatio      — 毛利率(%)
//   NetProfitRatio        — 净利率(%)
//   RoeWeighted           — 加权ROE(%)
```

输出：近5年年度净利润 + 同比增速表格 + Canvas柱状图（净利润柱+增速折线双轴图）。

**注意事项**：
- 港股只取 PeriodMark=12（年度报告）；**A股改用 `EndDate.endswith('12-31')`**
- 港股货币单位为港元，A股为人民币，美股为美元——需明确标注
- 若某年净利润为负，柱状图用不同颜色（红色）区分
- 增速为负时折线标注绿色（中国惯例：跌=绿）
- **务必单独挑出最新报告期的单季数据**（A股用 `_Q` 后缀字段）。年度数据会掩盖拐点——
  例如某股 FY2025 全年 +9.3% 看着很好，但 2026H1 转 -4.1%、Q2 单季 -15.1%，这才是真正的信号

### 5b. 提取近五年总股本、总市值变化

从 `finance` 和 `kline` 数据中提取各年度的总股本与总市值信息：

```javascript
// 总股本来源：
//   - finance 数据中的 TotalShare / TotalCapitalStock 字段（如有）
//   - 或 profile 数据中的总股本信息
//   - 或从 kline 数据中通过 市值÷股价 反推
//
// 总市值 = 年末收盘价 × 总股本
// 年末收盘价从 kline 数据中取每年最后一个交易日的 close
//
// 关键字段：
//   EndDate          — 报告期
//   TotalShare       — 总股本（股）
//   或从 profile 获取最新总股本，结合历史财报推算
```

输出：近5年总股本 + 总市值变化表格 + Canvas双轴图（总股本柱状图 + 总市值折线图）。

**表格格式示例**：

| 年度 | 总股本(亿股) | 总市值(亿元/亿港元) | 股本同比变化 | 市值同比变化 |
|------|------------|-------------------|------------|------------|
| 2022 | xx.xx | xxxx | — | — |
| 2023 | xx.xx | xxxx | +x.x% | +xx.x% |
| ... | ... | ... | ... | ... |

**图表设计**：
- 左Y轴：总股本（柱状图，蓝色）
- 右Y轴：总市值（折线图，橙色）
- 如果某年总股本发生显著变化（增发/回购/拆股等），需在图表下方用文字注释说明原因

**注意事项**：
- 港股总市值单位为港元，A股为人民币，美股为美元——需明确标注
- 总股本取各年度报告期末的数据，而非平均值
- 如果有增发、回购、拆股、合股等导致股本变动的重大事件，必须在表格下方注释
- 若无法从 finance 数据直接获取历史总股本，可使用 `profile` 获取最新总股本 + `dividend` 获取拆股/合股信息 + `kline` 年末收盘价来推算历史市值

### 5c. 计算近五年 PE / PB / PS 变化曲线

**这是判断「跌的是估值还是业绩」的核心图，不可省略。**

取点方式：**横轴用各期财报的 `InfoPublDate`（披露日），不是报告期末**——这样才是真实的"当时市场看到的估值"。
5 年季度报告约 20 个点，再补一个「当前」点（今天收盘价 + 最新一期 TTM 口径），共 21 点。

```python
# 逐期计算（income 表 + balance 表 + kline）
npttm  = float(r['NPParentCompanyOwnersTTM'])     # TTM 归母净利
revttm = float(r['OperatingRevenueTTM'])          # TTM 营收
npcum  = float(r['NPParentCompanyOwners'])        # 累计归母净利
eps    = float(r['BasicEPS'])                     # 累计 EPS
equity = float(bal[EndDate]['SEWithoutMI'])        # 归母净资产

w_shares = npcum / eps / 1e8       # 加权平均股本（亿股）—— PE 用这个
t_shares = shares_at(EndDate)      # 期末总股本（亿股）—— PB/PS 用这个
price    = 第一个 >= InfoPublDate 的交易日收盘价

PE = price / (npttm/1e8 / w_shares)
PB = price * t_shares / (equity/1e8)
PS = price * t_shares / (revttm/1e8)
```

#### ⚠️ 股本口径必须区分（否则与快照对不上）

- **PE 用加权平均股本**（= 累计归母净利 ÷ 累计 BasicEPS）。数据源披露的 PE(TTM)、EPS(TTM) 都是这个口径。
- **PB / PS 用期末总股本**算总市值。
- 混用会差 5% 左右。若算出的当前 PE 与快照/数据源的 PE(TTM) 偏差 >5%，一定是股本口径错了，**回去改，不要直接改快照数字**。

输出：三项指标的五年最高/最低/中位/当前/当前分位表格 + **三面板堆叠折线图**（各面板独立 Y 轴，画五年中位虚线，末点高亮标数值）。

**解读三板斧（写进正文，不要只给图）**：
1. **PE 下行要拆分子分母**——股价跌多少 vs TTM EPS 涨多少。若 EPS 同期大涨，说明是「估值与盈利双向收敛」，不是纯杀估值。
2. **PS 才是纯估值压缩的铁证**——营收一直在涨而 PS 单边下行，说明市场给每 1 元收入的定价在打折。
3. **PE 跌幅 > PB 跌幅，差额 = ROE 抬升**。反过来说：若后续利润下滑、ROE 回落，**PE 的「便宜」会最先消失**——这就是「低 PE 陷阱」的机制，必须写成风险提示。

### 6. 估值计算（PE Bands / PEG）

**根据增速区间选工具**：
- 增速 >20% → PEG = PE(TTM) ÷ 盈利增速(%)
- 增速 <15% → PE Bands（取近5年PE分位25%/50%/75%，计算价格带）
- 盈利为负 → PS 或其他

### 7. 组织四派圆桌分析

**产业趋势派**：
- 产业链拆解：当前驱动力（资源涨价/扩产升级/路线未定/看不准）
- 最受益环节
- 估值硬约束（PE + 动态PE）
- 候选标的市值分档

**估值派**：
- 宏观层：当前宏观状态 → 权益仓位建议
- 中观层：行业景气度 + 选估值工具 + 为什么
- 微观层：具体数字（PE/PEG + 分位 + 价格带）
- 资金面校验：大股东减持/外资/监管

**基本面派**：
- 行业趋势：体感 → 数据验证
- 管理层靠谱程度：历史周期判断
- 财报检查：营收/净利润/毛利率/负债率
- 估值分位
- 建仓/跟踪建议

**信号派**：
- 四层信号：政策 ✅/⚠️/❌ + 产业 ✅/⚠️/❌ + 资讯 ✅/⚠️/❌ + 资金 ✅/⚠️/❌
- 当前落入哪种组合
- 建议动作

**注意**：输出中不使用专家个人名字（星姐/文仔/钊仔/洲仔），仅使用流派名称（产业趋势派/估值派/基本面派/信号派）。

### 8. 生成可视化HTML

参考模板 `@templates/stock-analysis-template.html`，生成包含以下模块的HTML文件：

1. **头部**：股票名称、代码、实时价格、涨跌幅
2. **实时快照**：市值/PE/成交额/换手率/52周高低（卡片网格）
3. **近五年股价中位数**：表格 + 柱状图（Canvas绘制）
4. **近五年净利润变化**：净利润+同比增速表格 + 柱状折线双轴图（Canvas绘制）
5. **近五年总股本、总市值变化**：总股本+总市值表格 + 柱状折线双轴图（Canvas绘制，左轴股本柱+右轴市值折线）
6. **近五年 PE / PB / PS 变化曲线**（Step 5c）：三项指标五年极值与分位表格 + 三面板堆叠折线图
7. **宏观环境快扫**：主要指数 + 宏观要闻
8. **四派圆桌分析**：每个流派独立卡片（左侧色条区分）
9. **总结**：态度汇总表（每个派别一句话总结核心观点）+ 给小白一句话 + 操作路线图 + 最值得验证的预测 + 最值得重视的风险
10. **免责声明**

**设计要求**：
- 深色主题（背景 #0f1117）
- 卡片式布局，圆角12px
- 流派色条区分：产业趋势蓝/估值橙/基本面绿/信号紫
- 涨跌颜色遵循中国市场惯例：涨红跌绿
- 响应式适配

### 9. 输出文件

- HTML文件保存到当前工作目录
- 文件名格式：`<股票名拼音或英文名>_analysis.html`
- 使用 preview_url 工具预览
- 使用 deliver_attachments 交付

### 10. 可选：上传到「股票精灵」平台

用户说"上传到股票精灵"时，**不要直接把 Canvas 版传上去**——平台可能过滤 `<script>`，Canvas 依赖 JS 执行，会被整片清空。
正确做法：**另生成一份 SVG 版**（纯静态标记，100% 渲染），文件名 `<股票名>_观察纪要_股票精灵.html`。

**SVG 版的硬性要求（缺一不可）**：
1. 完整文档结构：`<!DOCTYPE html>` / `<html>` / `<head>` / `<body>`
2. 简体中文；标题同时出现在 `<title>` 和 `<h1>`
3. 正文必须包含五个小节：**核心结论 / 关键数据 / 近期变化 / 主要风险 /「不构成投资建议」声明**
4. **零 `<script>`、零 `<iframe>`、零外部资源**；样式全部内联在 `<style>`
5. 不确定的数字必须标「估算」或「需核对」
6. **图表数量不能缩水**——三张五年图 + 估值三曲线 + 近期走势，全部要有（只给文字表格会被判"没按规范生成"）

生成方式：把 Canvas 版的图表逻辑改成 Python 生成内联 SVG 字符串，用占位符 `__SVG_XXX__` 塞进模板。
可复用脚本：`/tmp/<code>/gen.py`（`svg_open` + 各图函数）+ `/tmp/<code>/tpl.html`。

**API**：

```bash
# 上传（返回新 id）
curl -s -X POST https://www.luliming.xyz/StockWizard/api/reports \
  -H 'Content-Type: application/json; charset=utf-8' \
  -d '{"title":"...","html":"...","source":"api"}'
# 建议用 Python urllib + json.dumps(ensure_ascii=False)，中文不会转义出错

# 列表
curl -s https://www.luliming.xyz/StockWizard/api/reports

# 删除（更新报告时先传新版再删旧版，避免列表堆重复）
curl -s -X DELETE https://www.luliming.xyz/StockWizard/api/reports/<id>
```

### 11. 写入工作记忆

完成分析后，将执行记录追加到当前项目的 `.workbuddy/memory/YYYY-MM-DD.md`。

## 数据铁律

1. **先查后写**：所有数字来自本次对话的真实查询，严禁从模型训练知识中"补"数据
2. **决策节点交叉验证**：PE用quote直接取 + 股价÷EPS反算，偏差>5%必须标注
3. **时效标注**：盘后2小时以上数据必须标注"数据截止 YYYY-MM-DD HH:MM"
4. **异常主动提**：PE为负/极端、机构覆盖<3家、股东户数暴增等必须主动提及
5. **货币单位正确**：港股港元/美股美元/A股人民币，禁止混用符号
6. **股本口径一致**：PE 用加权平均股本、PB/PS 用期末总股本；自算 PE 与数据源 PE(TTM) 偏差 >5% 必须查口径而不是改数字
7. **拐点优先**：年度增速很好看、但最新单季已转负时，必须以单季为准并在正文高亮，不能只报年度数

## Pitfalls

- ❌ 不要用模型记忆中的数据替代实时查询——PE/市值等数字必须来自westock-data
- ❌ 不要把不同上市主体混用（如港股腾讯vs美股ADR）
- ❌ 不要给单一结论——路线图要给不同路径让用户选择
- ❌ 不要使用专家个人名字——只用流派名称
- ❌ 不要忘记5年股价中位数计算和展示
- ❌ 不要忘记5年净利润变化计算和展示（从finance年度数据提取）
- ❌ 不要忘记5年总股本、总市值变化计算和展示（从finance/profile/kline数据提取）
- ❌ 总股本变动时（增发/回购/拆股/合股）不要忽略注释说明原因
- ❌ 港股/美股货币单位不能写成¥
- ❌ 机构覆盖<3家时，一致预期必须标注可信度打折
- ❌ 不要跳过宏观新闻获取（第零步）
- ❌ 增速<15%时不要用PEG，换PE Bands
- ❌ **不要假设 CLI 路径和命令集**——内置路径通常不存在、版本差异会删掉 `quote`/`news`/`risk`。先跑 `--help` 再动手
- ❌ **A股 K线不要指望一次拿全**——250 根上限会静默截断，必须按年分段抓后 `sort -u` 合并，并核对根数（应 >1200）
- ❌ **A股财报字段名不能照抄港股**——无 PeriodMark / GrossIncomeRatio / RoeWeighted / DebtAssetsRatio，需自行计算
- ❌ **PE 不要用期末总股本算**——必须用加权平均股本，否则与数据源 PE(TTM) 差 5% 对不上
- ❌ **不要用年度数据掩盖拐点**——必须单独挑出最新报告期的单季/半年同比，这是最容易漏也最有价值的信号
- ❌ **上传到股票精灵不要用 Canvas**——可能被过滤 `<script>` 导致图表全空，必须改用内联 SVG，且图表数量不能缩水
- ❌ **Canvas绘制必须包裹在 `window.addEventListener('load', ...)` 中**——直接同步执行会导致 `offsetWidth=0` 画布空白。必须：①用函数包裹所有绑定+绘制代码 ②`Math.max(offsetWidth, 800)` 尺寸兜底 ③监听resize自动重绘 ④**必须添加 `roundRect` polyfill**（部分WebView环境不支持此API，会静默中断绘制）⑤用 `var` 替代 `const/let` 提升兼容性 ⑥加 `setTimeout(drawCharts, 500)` 作为load事件的fallback
- ❌ 改动 Canvas 版 JS 后不要只靠肉眼——抽出 `<script>` 内容跑 `node --check`，SVG 版用 `xml.etree.ElementTree.fromstring` 校验每个 `<svg>` 片段

## Verification

- [ ] 所有数字都有对应查询动作
- [ ] CLI 路径已实际探测、`--help` 已确认命令集
- [ ] 5年K线根数已核对（>1200，未被 250 上限截断）
- [ ] 5年股价中位数已计算并展示
- [ ] 5年净利润变化已提取并展示（表格+图表）
- [ ] 净利润图表包含柱状图（净利润）+折线图（同比增速）双轴
- [ ] **最新报告期的单季/半年同比已单独挑出并高亮**
- [ ] 5年总股本、总市值变化已提取并展示（表格+图表）
- [ ] 总股本/总市值图表包含柱状图（总股本）+折线图（总市值）双轴
- [ ] **5年 PE/PB/PS 三曲线已计算并展示（三面板图 + 极值分位表）**
- [ ] **PE 用加权平均股本、PB/PS 用期末总股本，当前 PE 与数据源 PE(TTM) 偏差 <5%**
- [ ] 若总股本有显著变动（增发/回购/拆股等），已在图表下方注释说明
- [ ] 四派分析均包含【分析框架】+【关键数字】
- [ ] 总结区域每个派别均有一句话核心观点总结
- [ ] 无专家个人名字出现
- [ ] HTML文件可正常打开且视觉美观
- [ ] 免责声明已包含
- [ ] 数据截止时间已标注
- [ ] （若上传平台）SVG 版：0 script / 0 iframe / 图表齐全 / 含五节必备 / 不确定数字已标"估算"或"需核对"
