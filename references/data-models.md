# 数据模型定义 (Data Models)

> 本文档定义基金经理跟踪 Skill 的 6 个核心数据模型,所有 JSON 字段必须严格遵循此 schema。

---

## 1. ManagerProfile(基金经理档案)

**用途**: Step 1 validate_manager 的输出,记录经理基本状态

```json
{
  "name": "张坤",
  "name_normalized": "张坤",
  "is_active": true,
  "is_active_basis": "在管基金列表非空(截至 2026-06-24)",
  "career_summary": "经济学硕士,2008年加入易方达基金,现任权益投资总监...",
  "total_scale": "XXX 亿元",
  "managed_funds_count": 5,
  "managed_funds": [
    {
      "code": "005827",
      "name": "易方达蓝筹精选混合"
    }
  ],
  "style_tags_from_user": ["价值", "消费"],
  "query_date": "2026-06-24T17:01:44+08:00"
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| name | string | ✅ | 用户输入的原始姓名 |
| name_normalized | string | ✅ | 标准化后的姓名(去空格/全角转半角) |
| is_active | boolean | ✅ | 是否在职(true=在管基金非空) |
| is_active_basis | string | ✅ | 在职判断依据(可审计) |
| career_summary | string | ✅ | 履历摘要(neodata 召回) |
| total_scale | string | ⚠️ | 管理总规模(可能缺失) |
| managed_funds_count | number | ✅ | 在管基金数量 |
| managed_funds | FundInfo[] | ✅ | 在管基金列表(初始仅 code+name) |
| style_tags_from_user | string[] | ⚠️ | 用户提供的风格标签(watchlist.json 中) |
| query_date | ISO8601 | ✅ | 查询时间戳 |

---

## 2. FundInfo(基金信息)

**用途**: Step 2 select_benchmark_fund 的输入/输出,记录基金元数据

```json
{
  "code": "005827",
  "name": "易方达蓝筹精选混合",
  "type": "混合型",
  "type_full": "偏股混合型基金",
  "invest_scope": "A股+港股",
  "scope_breadth": 2,
  "setup_date": "2018-09-05",
  "manager_since": "2018-09-05",
  "manager_tenure_years": 7.8,
  "latest_scale": "XXX 亿元",
  "style": "价值",
  "style_source": "user_tag",  // user_tag | inferred_from_type
  "is_benchmark": true,
  "benchmark_reason": "任职自2018-09(管理7.8年),价值类成立最久,投资范围A股+港股",
  "data_quality": "complete",   // complete | partial | missing_metadata
  "warnings": []
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| code | string | ✅ | 基金代码(6位数字) |
| name | string | ✅ | 基金全称 |
| type | string | ✅ | 基金类型(股票型/混合型/债券型等) |
| type_full | string | ⚠️ | 完整类型(如"偏股混合型") |
| invest_scope | string | ⚠️ | 投资范围描述(如"A股+港股") |
| scope_breadth | number | ✅ | 投资范围广度评分(1=纯A股 / 2=A股+港股 / 3=全球QDII) |
| setup_date | ISO date | ✅ | 基金成立日期 |
| manager_since | ISO date | ⚠️ | 该经理任职日期(关键排序字段) |
| manager_tenure_years | number | ✅ | 任职年限(计算值,用于排序) |
| latest_scale | string | ⚠️ | 最新规模(亿元) |
| style | string | ✅ | 投资风格标签(用于分组) |
| style_source | string | ✅ | 风格来源(user_tag=用户提供 / inferred_from_type=推断) |
| is_benchmark | boolean | ✅ | 是否被选为标杆基金 |
| benchmark_reason | string | ⚠️ | 选取理由(仅标杆基金有值) |
| data_quality | enum | ✅ | 元数据完整度(complete/partial/missing_metadata) |
| warnings | string[] | ✅ | 警告信息列表(如"缺任职日,用成立日代替") |

### scope_breadth 评分规则

| 投资范围 | 分值 | 示例 |
|---|---|---|
| A股 + 港股 + 美股(全球/QDII) | 3 | 国富亚洲机会股票(QDII) |
| A股 + 港股(沪港深) | 2 | 易方达蓝筹精选混合 |
| 纯 A 股 | 1 | 易方达中小盘混合 |

**判断逻辑**(优先匹配):
1. 名称含"QDII"/"全球"/"海外" → 3
2. 名称含"沪港深"/"港股" 或 invest_scope 含"港股通" → 2
3. 其他 → 1(默认纯A股)

---

## 3. HoldingSnapshot(持仓快照)

**用途**: Step 3 extract_holdings 的输出,单只基金在单个报告期的持仓

```json
{
  "fund_code": "005827",
  "fund_name": "易方达蓝筹精选混合",
  "report_period": "2025Q4",
  "report_period_full": "2025年第四季度",
  "disclose_date": "2026-01-22",
  "is_latest": true,
  "holdings_count_total": 10,
  "holdings_count_filtered": 8,  // >2% 过滤后
  "holdings": [
    {
      "stock_code": "600519.SH",
      "stock_name": "贵州茅台",
      "weight_pct": 9.87,
      "shares": "XXX万股",
      "market_value": "XXX万元",
      "is_new": false,
      "change_symbol": "="
    }
  ],
  "source": "neodata 基金重仓资产",
  "confidence": "高",
  "warnings": []
}
```

### holdings[] 内部结构

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| stock_code | string | ✅ | 股票代码(含交易所后缀 .SH/.SZ) |
| stock_name | string | ✅ | 股票名称 |
| weight_pct | number | ✅ | 占净值比例(%),保留2位小数 |
| shares | string | ⚠️ | 持股数量(单位:万股) |
| market_value | string | ⚠️ | 持仓市值(单位:万元) |
| is_new | boolean | ✅ | 是否为新建仓(compare_periods 后填充) |
| change_symbol | string | ✅ | 变动符号("✚↑↓✖=",compare_periods 后填充) |

### 过滤规则

- **默认**: 仅保留 `weight_pct > 2` 的持仓
- **不足5只时**: 提示"⚠️ 本期重仓股较少(仅N只>2%),可能仓位集中度低或数据待披露"
- **全部过滤掉**: 提示"❌ 本期无>2%重仓股数据"

---

## 4. OutlookText(观点文本)

**用途**: Step 4a extract_outlook 的输出,季报/年报中的管理人展望段落

```json
{
  "fund_code": "005827",
  "fund_name": "易方达蓝筹精选混合",
  "report_period": "2025Q4",
  "section_type": "管理人展望",
  "section_alternative_names": ["投资策略", "运作分析", "后市展望", "报告期内基金投资策略"],
  "raw_text": "本基金在2025年第四季度保持高仓位运行,重点配置了消费、医药、互联网等行业...",
  "raw_text_length": 256,
  "source": "WebFetch:https://fundf10.eastmoney.com/FundArchivesDatas.aspx?type=jjbg&code=005827&year=2025&quarter=4",
  "source_url": "https://...",
  "confidence": "高",
  "extraction_level": 3,
  "attitude": "乐观",           // 乐观/中性/谨慎
  "attitude_score": 0.72,
  "keywords_extracted": ["消费", "医药", "互联网", "长期看好", "优质企业"]
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| fund_code | string | ✅ | 基金代码 |
| fund_name | string | ✅ | 基金名称 |
| report_period | string | ✅ | 报告期(YYYYQn 格式) |
| section_type | string | ✅ | 段落类型标识 |
| section_alternative_names | string[] | ✅ | 可能的同义名称列表(用于搜索匹配) |
| raw_text | string\|null | ✅ | 观点原文(null 表示未获取) |
| raw_text_length | number | ✅ | 原文字符数 |
| source | string | ✅ | 来源标识(见下方枚举) |
| source_url | string\|null | ⚠️ | 原始 URL(如有) |
| confidence | enum | ✅ | 置信度("高"/"中"/"无") |
| extraction_level | number | ✅ | 提取层级(1=neodata / 2=WebSearch / 3=WebFetch / 4=缺失) |
| attitude | enum\|null | ⚠️ | 态度判断(乐观/中性/谨慎,null=无法判断) |
| attitude_score | number\|null | ⚠️ | 态度分数(-1~1,-1=极度谨慎,0=中性,1=极度乐观) |
| keywords_extracted | string[] | ✅ | 提取的关键词(行业/板块/态度词) |

### source 枚举值

| source 值 | 对应提取层 | 置信度 | 示例 |
|---|---|---|---|
| `"WebFetch:{url}"` | L3 | **高** | 一手原文,最可信 |
| `"neodata docData"` | L1 | 中 | neodata 召回的资讯/公告 |
| `"WebSearch:{媒体名}"` | L2 | 中 | 从搜索引擎找到的媒体转载 |
| `"缺失"` | L4 | 无 | 全部降级失败 |

### attitude 态度判断词典

详见 `references/outlook-extraction.md` 第四节。

---

## 5. ChangeRecord(变动记录)

**用途**: Step 5 compare_periods 的输出,跨期对比结果

```json
{
  "fund_code": "005827",
  "comparison_range": "2025Q3 vs 2025Q4",
  "prev_period": "2025Q3",
  "curr_period": "2025Q4",
  "holding_changes": [
    {
      "stock_code": "600519.SH",
      "stock_name": "贵州茅台",
      "prev_weight_pct": 9.87,
      "curr_weight_pct": 8.50,
      "change_type": "减仓",
      "symbol": "↓",
      "delta_pct": -1.37,
      "delta_absolute": 1.37,
      "rank_change": "第1→第2"  // 排名变化(如有)
    },
    {
      "stock_code": "002415.SZ",
      "stock_name": "海康威视",
      "prev_weight_pct": 0,
      "curr_weight_pct": 6.32,
      "change_type": "新建仓",
      "symbol": "✚",
      "delta_pct": 6.32,
      "delta_absolute": 6.32,
      "rank_change": "新进→第5"
    }
  ],
  "summary_statistics": {
    "total_holdings_curr": 10,
    "total_holdings_prev": 9,
    "new_positions": 1,
    "exited_positions": 0,
    "increased_positions": 3,
    "decreased_positions": 4,
    "unchanged_positions": 2,
    "max_single_delta": {"stock": "海康威视", "delta": 6.32, "type": "新建仓"}
  },
  "outlook_change": {
    "has_comparison": true,
    "prev_attitude": "乐观",
    "curr_attitude": "中性",
    "attitude_shift": "乐观→中性",
    "keywords_added": ["AI", "港股配置", "政策支持"],
    "keywords_removed": ["新能源", "碳中和", "估值修复"],
    "prev_outlook_summary": "看好消费复苏和新能源产业链...",
    "curr_outlook_summary": "维持对核心资产的配置,新增关注AI应用...",
    "llm_generated_summary": "本期新增关注 AI 与港股配置方向,不再强调新能源主题,整体态度由乐观转为中性偏谨慎,显示经理在科技与防御性资产间寻求平衡。",
    "significant_shift_detected": true
  }
}
```

### holding_changes[] 内部字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| stock_code | string | ✅ | 股票代码 |
| stock_name | string | ✅ | 股票名称 |
| prev_weight_pct | number | ✅ | 上期仓位%(0表示不在上期) |
| curr_weight_pct | number | ✅ | 本期仓位%(0表示已清仓) |
| change_type | enum | ✅ | 变动类型(新建仓/加仓/减仓/清仓/持平) |
| symbol | char | ✅ | 可视化符号(✚↑↓✖=) |
| delta_pct | number | ✅ | 变动百分点(可为负) |
| delta_absolute | number | ✅ | 变动绝对值(用于排序) |
| rank_change | string | ⚠️ | 排名变化描述 |

### change_type 枚举与 symbol 映射

| 条件 | change_type | symbol | 排序权重 |
|---|---|---|---|
| prev=0, curr>0 | 新建仓 | ✚ | 最高(置顶) |
| prev>0, curr=0 | 清仓 | ✖ | 次高(置顶) |
| curr > prev (+0.1%) | 加仓 | ↑ | 按 delta_absolute 降序 |
| curr < prev (-0.1%) | 减仓 | ↓ | 按 delta_absolute 降序 |
| abs(curr-prev) ≤ 0.1% | 持平 | = | 最低 |

**注意**: ±0.1% 阈值用于判断"持平"(避免浮点误差),可根据实际数据精度调整。

### summary_statistics 统计汇总

| 字段 | 说明 |
|---|---|
| total_holdings_curr | 本期总持仓数(>2%过滤前或后,需标注) |
| total_holdings_prev | 上期总持仓数 |
| new_positions | 新建仓数量 |
| exited_positions | 清仓数量 |
| increased_positions | 加仓数量 |
| decreased_positions | 减仓数量 |
| unchanged_positions | 持平数量 |
| max_single_delta | 最大变动(股票名+幅度+类型) |

### outlook_change 观点变动

| 字段 | 类型 | 说明 |
|---|---|---|
| has_comparison | boolean | 两期观点是否都获取成功 |
| prev/curr_attitude | enum | 上期/本期态度 |
| attitude_shift | string | 态度转变描述(如无转变则"无变化") |
| keywords_added/removed | string[] | 新增/消失的关键词集合差 |
| prev/curr_outlook_summary | string | LLM 生成的上期/本期观点摘要(100字内) |
| llm_generated_summary | string | LLM 生成的一句话变化摘要 |
| significant_shift_detected | boolean | 是否检测到显著态度/焦点转变 |

---

## 6. PerformanceMetrics(业绩表现) ★新增

**用途**: Step 4b extract_performance 的输出,记录基金短期+中期业绩

```json
{
  "fund_code": "005827",
  "fund_name": "易方达蓝筹精选混合",
  "data_date": "2026-06-24",
  "benchmark_name": "沪深300指数收益率×50%+中证港股通综合指数收益率×30%+中债总财富指数收益率×20%",
  "benchmark_name_short": "沪深300*50%+港股*30%+中债*20%",
  "return_periods": {
    "1m": {
      "fund_return": 2.35,
      "benchmark_return": 1.82,
      "benchmark_percentile": 35.2,
      "excess_return": 0.53,
      "peer_rank": 156,
      "peer_total": 1234,
      "peer_percentile": 12.6,
      "evaluation": "⭐⭐⭐"
    },
    "3m": {
      "fund_return": 5.67,
      "benchmark_return": 4.21,
      "excess_return": 1.46,
      "peer_rank": 234,
      "peer_total": 1234,
      "peer_percentile": 19.0,
      "evaluation": "⭐⭐"
    },
    "6m": { "fund_return": 8.90, "benchmark_return": 7.50, "excess_return": 1.40, "peer_rank": 312, "peer_total": 1234, "peer_percentile": 25.3, "evaluation": "⭐⭐" },
    "1y": { "fund_return": 12.50, "benchmark_return": 10.20, "excess_return": 2.30, "peer_rank": 185, "peer_total": 1234, "peer_percentile": 15.0, "evaluation": "⭐⭐⭐" },
    "3y": { "fund_return": 28.90, "benchmark_return": 22.10, "excess_return": 6.80, "peer_rank": 98, "peer_total": 1100, "peer_percentile": 8.9, "evaluation": "⭐⭐⭐" },
    "since_inception": { "fund_return": 95.30, "benchmark_return": 65.40, "excess_return": 29.90, "peer_rank": 45, "peer_total": 856, "peer_percentile": 5.3, "evaluation": "⭐⭐⭐" }
  },
  "summary": {
    "best_period": "since_inception",
    "worst_period": "1m",
    "overall_evaluation": "⭐⭐⭐ 优秀",
    "has_consistent_outperformance": true,
    "max_drawdown_1y": -8.5,
    "sharpe_ratio_1y": 1.2
  },
  "source": "neodata 基金业绩 + 同类排名",
  "confidence": "高"
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| fund_code | string | ✅ | 基金代码 |
| fund_name | string | ✅ | 基金名称 |
| data_date | ISO date | ✅ | 业绩数据截止日 |
| **benchmark_name** | string | ⚠️ | 业绩比较基准全称 |
| **benchmark_name_short** | string | ⚠️ | 业绩基准简称(简化展示) |
| return_periods | object | ✅ | 各时间段业绩(见下方) |
| summary | object | ✅ | 业绩汇总评价 |
| source | string | ✅ | 数据来源 |
| confidence | enum | ✅ | 置信度(业绩为结构化数据,通常为"高") |

### return_periods 内部结构(每个时间段)

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| fund_return | number | ✅ | 基金收益率(%),保留2位小数 |
| benchmark_return | number | ⚠️ | 业绩基准收益率(%) |
| **benchmark_percentile** | number | ⚠️ | 基准同类百分位(基准收益率在同类中的位置) |
| excess_return | number | ⚠️ | 超额收益 = fund_return - benchmark_return |
| peer_rank | number | ✅ | 同类排名 |
| peer_total | number | ✅ | 同类总数 |
| peer_percentile | number | ✅ | 同类百分位(越小越好,前10%=10.0) |
| evaluation | string | ✅ | 评价标识(⭐⭐⭐/⭐⭐/⭐/⚠️/❌) |

### 时间段标识

| 标识 | 含义 |
|---|---|
| `1m` | 近1月 |
| `3m` | 近3月 |
| `6m` | 近6月 |
| `1y` | 近1年 |
| `3y` | 近3年 |
| `since_inception` | 成立来 |

### evaluation 评价规则

| peer_percentile 范围 | evaluation | 含义 |
|---|---|---|
| ≤ 10% | ⭐⭐⭐ | 优秀(显著跑赢同类) |
| 10% - 25% | ⭐⭐ | 良好(跑赢同类) |
| 25% - 50% | ⭐ | 中等(略好于均值) |
| 50% - 75% | ⚠️ | 偏弱(略低于均值) |
| > 75% | ❌ | 落后(显著跑输) |

### neodata query 示例

```
python scripts/query.py --query "基金 005827 的近1月、近3月、近6月、近1年、近3年、成立来收益率和同类排名"
```

---

## 数据模型关系图

```
ManagerProfile(1)
  ├── FundInfo(N) ←── select_benchmark_fund 筛选 ──→ FundInfo(is_benchmark=true)
  │     │
  │     ├── HoldingSnapshot(M期) ←── extract_holdings ──→ holdings[](K只)
  │     │                                          │
  │     │                                          └── compare_periods ──→ ChangeRecord(1)
  │     │                                                                ├── holding_changes[]
  │     │                                                                └── outlook_change{}
  │     │
  │     ├── PerformanceMetrics(1) ←── extract_performance ──→ return_periods{}
  │     │
  │     └── OutlookText(M期) ←── extract_outlook(三级降级) ──→ raw_text / null
```

## 使用注意事项

1. **所有时间相关字段统一使用 ISO 格式**: YYYY-MM-DD / YYYYQn / ISO8601
2. **百分比保留2位小数**: 如 9.87% 不写 9.871234%
3. **缺失字段必须显式标记为 null**,不要省略或留空字符串
4. **每个数据对象应包含 source 和 confidence 字段**,方便用户评估可信度
5. **批量场景**: ChangeRecord 数组长度 = 经理数 × 标杆基金数
6. **业绩数据为结构化数据**: 置信度通常为"高"(直接来自 neodata 接口,非爬取)
7. **收益率正负号**: 正收益为正数(如 +12.50),负收益为负数(如 -5.30),不要用括号表示
