---
name: stock-analysis
description: 个股深度分析skill，基于westock-partner圆桌讨论框架，输出可视化HTML报告。包含：实时快照、近五年股价中位数、近五年总市值变化、PE Bands估值、产业趋势/估值/基本面/信号四派圆桌分析、操作路线图。触发条件：用户提到"分析XX股票"、"XX股票怎么看"、"帮我分析XX"等个股分析请求。输入：股票名称或代码。输出：完整可视化HTML文件。
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

**不触发**：纯行情查询（用westock-data quote）、纯新闻浏览、板块级别讨论。

## 输入

- 股票名称或代码（如"安克创新"或"sz300866"）
- 可选：用户特别关注的维度

## Steps

### 1. 搜索股票代码

```bash
cd /Users/lyman/.workbuddy/plugins/marketplaces/cb_teams_marketplace/plugins/finance-data/skills/westock-data && node scripts/index.js search <股票名称>
```

确认标的身份（公司名、代码、市场、是否为ETF/ADR等），避免混用不同上市主体。

### 2. 查询全量数据

按以下顺序一次性查询所有需要的数据：

```bash
# 宏观指数
node scripts/index.js quote sh000001,sh000300,hkHSI

# 个股实时行情
node scripts/index.js quote <代码>

# 近4期财报
node scripts/index.js finance <代码> --num 4

# 5年K线（用于股价中位数和PE Bands）
node scripts/index.js kline <代码> --period day --limit 1200

# A股资金流向 / 港股资金 / 美股卖空
node scripts/index.js asfund <代码>   # A股
node scripts/index.js hkfund <代码>   # 港股
node scripts/index.js usfund <代码>   # 美股

# 技术指标
node scripts/index.js technical <代码> --group all

# 公司简况
node scripts/index.js profile <代码>

# 一致预期
node scripts/index.js consensus <代码>

# 股东结构
node scripts/index.js shareholder <代码>

# 机构评级
node scripts/index.js rating <代码>

# 近期新闻
node scripts/index.js news <代码> --limit 10 --type 2

# 分红数据
node scripts/index.js dividend <代码> --years 3

# 风险事件
node scripts/index.js risk <代码>
```

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

### 5. 计算近五年总市值变化

从实时行情获取当前总市值和股价，计算总股本，再乘以每年的中位数股价估算各年市值：

```javascript
// 总股本 = 当前总市值 / 当前股价
const totalShares = currentMarketCap / currentPrice;

// 各年总市值 = 年中位数股价 × 总股本
const yearlyMarketCap = byYear.map(y => ({
  year: y.year,
  marketCap: y.median * totalShares,
  yoy: y.year > firstYear ? ((y.median * totalShares - prevCap) / prevCap * 100) : null
}));
```

输出：每年总市值(亿港元) + 同比变化百分比 + 柱状图（红涨绿跌）。

**注意**：此方法假设总股本不变，实际可能有增发/回购，但对5年尺度的趋势展示已足够。当前年份使用实时市值数据。

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
4. **近五年总市值变化**：表格（总市值+同比变化） + 柱状图（红涨绿跌，当前市值参考线，5年均值参考线）
5. **PE Bands估值带**：25%/50%/75%分位价格带
6. **宏观环境快扫**：主要指数 + 宏观要闻
7. **四派圆桌分析**：每个流派独立卡片（左侧色条区分）
8. **总结**：态度汇总表 + 给小白一句话 + 操作路线图 + 最值得验证的预测 + 最值得重视的风险
9. **免责声明**

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

### 10. 写入工作记忆

完成分析后，将执行记录追加到 `/Users/lyman/WorkBuddy/2026-05-19-task-11/.workbuddy/memory/YYYY-MM-DD.md`。

## 数据铁律

1. **先查后写**：所有数字来自本次对话的真实查询，严禁从模型训练知识中"补"数据
2. **决策节点交叉验证**：PE用quote直接取 + 股价÷EPS反算，偏差>5%必须标注
3. **时效标注**：盘后2小时以上数据必须标注"数据截止 YYYY-MM-DD HH:MM"
4. **异常主动提**：PE为负/极端、机构覆盖<3家、股东户数暴增等必须主动提及
5. **货币单位正确**：港股港元/美股美元，禁止混用人民币符号

## Pitfalls

- ❌ 不要用模型记忆中的数据替代实时查询——PE/市值等数字必须来自westock-data
- ❌ 不要把不同上市主体混用（如港股腾讯vs美股ADR）
- ❌ 不要给单一结论——路线图要给不同路径让用户选择
- ❌ 不要使用专家个人名字——只用流派名称
- ❌ 不要忘记5年股价中位数计算和展示
- ❌ 不要忘记5年总市值变化柱状图（红涨绿跌+同比百分比）
- ❌ 港股/美股货币单位不能写成¥
- ❌ 机构覆盖<3家时，一致预期必须标注可信度打折
- ❌ 不要跳过宏观新闻获取（第零步）
- ❌ 增速<15%时不要用PEG，换PE Bands

## Verification

- [ ] 所有数字都有对应查询动作
- [ ] 5年股价中位数已计算并展示
- [ ] 5年总市值变化柱状图已绘制（红涨绿跌+同比百分比）
- [ ] 四派分析均包含【分析框架】+【关键数字】
- [ ] 总结区域每个派别均有一句话核心观点总结
- [ ] 无专家个人名字出现
- [ ] HTML文件可正常打开且视觉美观
- [ ] 免责声明已包含
- [ ] 数据截止时间已标注
