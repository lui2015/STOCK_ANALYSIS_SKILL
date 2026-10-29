# STOCK_ANALYSIS_SKILL

输入一只股票的名字或代码，自动生成一份**可视化深度分析 HTML 报告**——包含实时快照、近五年股价中位数、近五年净利润变化、近五年总股本与市值、近五年 PE/PB/PS 估值曲线、四派圆桌讨论、操作路线图；可另出 SVG 版上传「股票精灵」平台。

## 目录结构

```
.
├── README.md
└── stock-analysis/                          # Skill 本体，直接放到 ~/.workbuddy/skills/ 下即可使用
    ├── SKILL.md                             # Skill 定义（触发条件、11 个步骤、数据铁律、坑位、验收清单）
    ├── README.md                            # 使用说明
    └── templates/
        └── stock-analysis-template.html     # HTML 报告样式参考（深色主题）
```

## 安装

```bash
cp -r stock-analysis ~/.workbuddy/skills/
```

## 使用

```
@skill:stock-analysis 顺丰控股 002352
```

## 报告模块（10 个）

头部 → 实时快照 → 近五年股价中位数 → 近五年净利润变化 → 近五年总股本与市值 → 近五年 PE/PB/PS 曲线 → 宏观环境快扫 → 四派圆桌 → 总结（含操作路线图）→ 免责声明

## License

MIT
