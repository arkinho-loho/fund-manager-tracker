# 基金经理持仓与观点跟踪 (fund-manager-tracker)

> 📊 跟踪 A 股公募明星基金经理的持仓与观点变化，轻松"抄作业"。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![WorkBuddy Skill](https://img.shields.io/badge/WorkBuddy-Skill-blue)](https://www.codebuddy.cn/docs/workbuddy/Overview)

## 功能特性

- ✅ **自动验证基金经理在职状态** — 多数据源交叉确认（neodata + 同花顺问财）
- ✅ **智能筛选标杆基金** — 从经理管理的多只基金中自动选出最具代表性的 1-3 只
- ✅ **十大重仓股明细** — 含仓位比例、前十集中度
- ✅ **跨期持仓对比** — 相邻季度逐股比对（新建/加仓/减仓/清仓）
- ✅ **季报观点提取** — 五级降级链路获取"管理人展望"原文段落
- ✅ **跨期观点对比** — 态度变化追踪（↑更积极 /=不变 /↓转向谨慎）
- ✅ **整体股票仓位变化** — 追踪基金的权益仓位调整
- ✅ **风格分组展示** — AI 产业链/科技成长/消费价值/深度价值/周期/QDII 等
- ✅ **业绩基准标注** — 每只标杆基金的对标指数一目了然
- ✅ **Markdown 报告输出** — 可离线阅读、可版本控制的纯文本报告
- ✅ **首次使用引导** — 交互式问答帮你快速建立自己的跟踪名单

## 快速开始

### 前置依赖

本 Skill 依赖以下数据源（需提前在 WorkBuddy 中安装）：

| 依赖 | 用途 |
|------|------|
| `neodata-financial-search` | 基金经理/持仓/排名（主力数据源） |
| `hithink-fundmanager-selector` | 同花顺问财（交叉验证/补充业绩） |
| `announcement-search` | 公告搜索（季报原文直达） |

### 安装

```bash
# 在 WorkBuddy 中安装本 Skill（从 GitHub）
# 或手动复制到: ~/.workbuddy/skills/fund-manager-tracker/
```

### 首次使用

在 WorkBuddy 对话框中输入以下任一触发词：

```
"帮我设置基金经理跟踪名单"
"初始化基金经理跟踪"
```

首次运行时，Skill 会自动检测到你的 `manager-watchlist.json` 不存在，进入**引导模式**，依次询问：

1. 你想跟踪几位基金经理？
2. 输入他们的姓名和基金公司
3. 为什么关注他们？（可选，会显示在报告中）
4. 是否指定标杆基金？（留空让 Skill 自动选）
5. 报告输出路径
6. 是否需要定期自动跟踪

回答完成后，自动生成 `assets/manager-watchlist.json` 并运行第一次跟踪。

### 日常使用

```
"把我名单里的经理都过一遍"        → 批量扫描所有经理
"帮我看看张坤最新的持仓和观点"     → 单经理查询
"对比金梓才最近 2 期的变化"        → 跨期对比（N=2）
"帮我找近1年收益前20%的科技方向经理" → 问财条件筛选
```

### 输出示例

```
基金经理跟踪汇总_2026Q1_20260627.md

# 基金经理跟踪汇总报告(19 位经理)

## 一、执行概要（统计概览）
## 二、经理状态总览表（按风格分组 + 关注理由）
## 三、业绩表现排名
## 四、十大持仓明细（按风格分组，含前十合计集中度）
## 五、整体股票仓位变化（Q4→Q1）
## 六、季报观点摘要（按核心主题分组：AI信徒/防守型/消费坚守/跨市场）
## 七、观点变化对比（2025Q4 → 2026Q1，含态度变化统计）
```

## 目录结构

```
fund-manager-tracker/
├── README.md                          # 本文件
├── LICENSE                            # MIT
├── CHANGELOG.md                       # 版本记录
├── .gitignore                         # 排除用户私有文件和生成输出
├── SKILL.md                           # 核心 Skill 定义（六步工作流）
├── assets/
│   ├── manager-watchlist.example.json # 示例跟踪名单（3 位经理）
│   └── manager-watchlist.json         # 你的私有名单（gitignore 排除）
└── references/
    ├── benchmark-selection.md         # 标杆基金筛选算法
    ├── data-models.md                 # 6 个数据模型 JSON Schema
    ├── iwencai-integration.md         # 同花顺问财集成
    ├── outlook-extraction.md          # 五级观点降级策略
    ├── output-templates.md            # 报告模板与变动对比算法
    └── onboarding-guide.md            # 首次使用引导流程
```

## 兼容性

- **WorkBuddy**: v2.0+
- **Python**: 3.10+（需配置 `$PYTHON_PATH`）
- **操作系统**: Windows / macOS / Linux

## 数据源说明

| 数据维度 | 主力源 | 备选源 |
|----------|--------|--------|
| 经理基本信息 | neodata | 问财 |
| 最新期持仓 | neodata | 天天基金网 API |
| 历史期持仓（跨期对比） | 天天基金网 `FundArchivesDatas.aspx` | — |
| 业绩排名 | neodata | 问财 |
| 季报观点原文 | 问财公告搜索 → 天天基金网公告页 | WebSearch |
| 业绩基准 | neodata 基金基础信息 | — |

## License

MIT © 2026 He Yaqing
