# 输出模板与变动对比算法 (Output Templates)

> 本文档定义 Step 5 compare_periods 的对比算法和 Step 6 format_report 的 Markdown 报告模板。

---

## 一、报告设计原则

### 1.1 用户偏好(已确认)
- ✅ **中文界面**
- ✅ **表格化呈现**(Markdown 表格)
- ✅ **简洁明了**,避免冗长
- ✅ **符号可视化**(✚↑↓✖=)让变动一目了然
- ✅ **置信度标注**,让用户判断数据可信度
- ✅ **业绩展示**(短期+中期收益率 + 同类排名)为默认功能
- ✅ **双版本输出**(Markdown + HTML 可视化)为默认

### 1.2 报告结构层次
```
单经理报告:
├── 标题 + 元信息(查询日/数据源/报告期)
├── 一、经理档案(1个表格)
├── 二、标杆基金(1个表格,高亮标杆行)
├── 三、业绩表现(1个表格:短期+中期收益率+同类排名) ★新增
├── 四、最新持仓(1个表格,>2%重仓,末列带变动符号)
├── 五、持仓变动(1个表格,✚↑↓✖=符号,按幅度排序)
├── 六、观点对比(diff风格 + 态度箭头)
│   ├── 最新期观点原文(引用块)
│   └── 观点变化(列表:态度/关键词/摘要)
├── 七、扩展维度(按开关显示,默认隐藏)
└── 八、数据来源与置信度(来源+时间戳)

批量报告:
├── 标题 + 汇总统计
├── 汇总表(每经理一行速览)
└── 逐位详情(折叠区域)
```

---

## 二、单经理报告模板

```markdown
# {经理名} 持仓与观点跟踪报告

> **查询日期**: YYYY-MM-DD HH:MM | **数据源**: neodata-financial-search | **报告期**: 最新 {period} | **对比期数**: {N}

---

## 一、经理档案

| 项目 | 内容 |
|---|---|
| 姓名 | {name} |
| 在职状态 | ✅ 在职 / ⛔ 已离任(截至 {query_date}) |
| 风格分组 | 🟦🟩🟨🟧🟫🟪 {style_group} |
| 管理总规模 | {total_scale} (在管 {managed_funds_count} 只基金) |
| 业绩基准 | {benchmark} |
| 风格标签 | {style_tags (逗号分隔)} |
| 🔍 **关注理由** | {watch_reason} |

{若 is_active=false:}
> ⚠️ **提示**:该经理当前在管基金列表为空,已离任(最后活跃: {last_active_date})。以下数据为历史最后记录。

> 🔄 **下次复查**:技能将在下次运行时自动检查是否已入职新公募(→重新跟踪)或跳槽至私募(→从名单移除)。当前状态: **{resigned_status_desc}**。

---

## 二、标杆基金选择

| 基金代码 | 基金名称 | 类型 | 投资风格 | 成立日 | 经理任职日 | 最新规模 | 投资范围 | 选取理由 |
|---|---|---|---|---|---|---|---|---|
| {code} | {name} | {type} | {style} | {setup_date} | {manager_since} | {scale} | {invest_scope} | ⭐ **{benchmark_reason}** |
| {code} | {name} | {type} | {style} | ... | ... | ... | ... | (非标杆) |

> 📊 **筛选逻辑**:从 {total_funds} 只在管基金中按"任职最久 > 成立最久 > 范围广 > 规模大"四键排序,按投资风格分组后每组选 1 只标杆。

{若有用户覆盖:}
> 🔧 **注意**:本次使用用户指定基金(--fund {code}),跳过自动筛选。

---

## 三、业绩表现

> **基金**: {fund_name} ({fund_code}) | **数据截止**: {performance_date} | **业绩基准**: {benchmark_name}

### 短期 + 中期收益率

| 时间段 | 基金收益率 | 同类排名 | 同类百分位 | 业绩基准收益率 | 基准百分位 | 超额收益 | 评价 |
|---|---|---|---|---|---|---|---|
| 近1月 | {r_1m}% | {rank_1m}/{total_1m} | 前{pct_1m}% | {b_1m}% | {bp_1m}% | {e_1m}% | {eval_1m} |
| 近3月 | {r_3m}% | {rank_3m}/{total_3m} | 前{pct_3m}% | {b_3m}% | {bp_3m}% | {e_3m}% | {eval_3m} |
| 近6月 | {r_6m}% | {rank_6m}/{total_6m} | 前{pct_6m}% | {b_6m}% | {bp_6m}% | {e_6m}% | {eval_6m} |
| **近1年** | **{r_1y}%** | {rank_1y}/{total_1y} | **前{pct_1y}%** | **{b_1y}%** | {bp_1y}% | **{e_1y}%** | {eval_1y} |
| 近3年 | {r_3y}% | {rank_3y}/{total_3y} | 前{pct_3y}% | {b_3y}% | {bp_3y}% | {e_3y}% | {eval_3y} |
| 成立来 | {r_since}% | {rank_since}/{total_since} | 前{pct_since}% | {b_since}% | {bp_since}% | {e_since}% | {eval_since} |

### 业绩评价规则(双维度)

| 同类百分位 | 评价 | 超额收益 | 评价 |
|---|---|---|---|
| 前 10% | ⭐⭐⭐ **优秀** | > +5% | ✅ **大幅跑赢基准** |
| 前 10%-25% | ⭐⭐ **良好** | +2%~+5% | ✅ **跑赢基准** |
| 前 25%-50% | ⭐ **中等** | -2%~+2% | ➖ **接近基准** |
| 前 50%-75% | ⚠️ **偏弱** | -5%~-2% | ⚠️ **跑输基准** |
| 后 25% | ❌ **落后** | < -5% | ❌ **大幅跑输基准** |

{若超额收益为正:}
> ✅ **超额收益**: 近1年跑赢基准 {e_1y}%,策略有效性得到验证。

{若超额收益为负:}
> ⚠️ **注意**: 近1年跑输基准 {abs(e_1y)}%,需关注策略是否失效。

**数据来源**: neodata 基金业绩 + 基金各阶段同类排名 + 业绩比较基准 | **置信度**: 高(结构化数据)

---

## 四、最新持仓(>{2}% 重仓股)

> **基金**: {fund_name} ({fund_code}) | **报告期**: {report_period} | **披露日**: {disclose_date}

| 排名 | 股票代码 | 股票名称 | 仓位(%) | 持股市值(万) | 较上期变动 |
|---|---|---|---|---|---|
| 1 | {stock_code} | {stock_name} | {weight_pct}% | {market_value} | {change_symbol} {prev_weight%→curr_weight%} |
| 2 | ... | ... | ... | ... | ... |
| ... | ... | ... | ... | ... | ... |

**统计**:本期共 {holdings_count_filtered} 只股票仓位>2%(全部 {holdings_count_total} 只重仓股)

{若 holdings_count_filtered < 5:}
> ⚠️ 本期重仓股较少,可能仓位集中度低或部分持股未达2%披露门槛。

---

## 五、持仓变动明细(对比上 {N} 期)

> **对比范围**: {prev_period} → {curr_period}

### 变动汇总

| 指标 | 数量 |
|---|---|
| 新建仓(✚) | {new_positions} 只 |
| 加仓(↑) | {increased_positions} 只 |
| 减仓(↓) | {decreased_positions} 只 |
| 清仓(✖) | {exited_positions} 只 |
| 持平(=) | {unchanged_positions} 只 |
| **最大单项变动** | **{max_delta_stock} {max_delta_type} {max_delta_abs}%**

### 变动详情(按幅度降序,新建/清仓置顶)

| 变动类型 | 股票代码 | 股票名称 | 上期仓位 | 本期仓位 | 变动幅度 | 备注 |
|---|---|---|---|---|---|---|
| ✚ **新建仓** | {code} | {name} | - | {curr}% | +{delta}% | 新进前十大 |
| ✖ **清仓** | {code} | {name} | {prev}% | - | -{delta}% | 不再持有 |
| ↑ **加仓** | {code} | {name} | {prev}% | {curr}% | +{delta}% | 排名上升 |
| ↓ **减仓** | {code} | {name} | {prev}% | {curr}% | -{delta}% | 排名下降 |
| = **持平** | {code} | {name} | {prev}% | {curr}% | 0.00% | 核心仓位稳定 |

### 📝 一句话摘要
{llm_generated_summary}

示例:
> "本期新建仓 1只(海康威视),无清仓;前三大重仓中贵州茅台减仓1.37pct(仍居首位),五粮液加仓0.82pct;整体维持消费+医药核心配置,新增科技方向布局。"

---

## 六、观点与策略变化

### 最新期观点原文({period})

> {raw_text 或 "⚠️ 观点原文未获取"}

{若 raw_text 存在:}
> **来源**: {source} | **置信度**: {confidence} | **提取方式**: Level {extraction_level}

{若 raw_text 为 null:}
> ⚠️ **说明**:季报/年报原文非 neodata 直覆盖能力,三级降级(neodata docData→WebSearch→WebFetch)均未成功获取。建议后续手动查阅[天天基金网](https://funds.eastmoney.com/{fund_code}.html)或[巨潮资讯网](https://www.cninfo.com.cn)获取完整报告。

### 观点变化分析

{若两期观点都获取成功:}

#### 态度转变
| 上期态度 | 本期态度 | 转变方向 |
|---|---|---|
| {prev_attitude} | {curr_attitude} | {attitude_shift_icon} {attitude_shift} |

{若 attitude_shift != "无变化":}
> 💡 **信号**:检测到显著态度转变!可能意味着经理对市场看法发生了重要调整。

#### 关注焦点变化

**✚ 新增关注的方向:**
- {keyword1}
- {keyword2}
- ...

**❌ 不再强调的方向:**
- {keyword1}
- {keyword2}
- ...

#### LLM 生成的变化摘要

{llm_generated_outlook_summary}

示例:
> "本期新增关注 AI 与港股配置方向(提及'人工智能应用场景'和'港股估值修复机会'),不再强调新能源产业链(上期重点提及'碳中和长期趋势'和'光伏景气度')。整体态度由'乐观'(score=0.65)转为'中性偏谨慎'(score=-0.05),显示经理在保持核心持仓的同时开始增加防御性资产配置。"

{若仅一期有观点或缺双方:}

> ℹ️ 无法进行跨期观点对比({reason})。

---

## 七、扩展维度(可选)

{仅当用户启用 --include-scale 时显示:}

### 6.1 规模变动趋势

| 报告期 | 规模(亿元) | 环比变化 | 变化原因推测 |
|---|---|---|---|
| {period_n} | {scale} | {change}% | {推测} |
| ... | ... | ... | ... |
| {period_0}(最新) | {scale} | {change}% | {推测} |

{仅当用户启用 --include-sector 时显示:}

### 6.2 行业配置变化

| 行业(申万一级) | 上期占比 | 本期占比 | 变动 | 说明 |
|---|---|---|---|---|
| 食品饮料 | 35.2% | 33.8% | -1.4pct | 核心持仓微降 |
| 医药生物 | 22.1% | 24.5% | +2.4pct | 加仓明显 |
| ... | ... | ... | ... | ... |

{仅当用户启用 --include-ranking 时显示:}

### 6.3 同类排名表现

| 时间段 | 同类排名 | 百分位 | 跑赢/跑输基准 | 评价 |
|---|---|---|---|---|
| 近1月 | 123/456 | 前27% | +2.3% | ✅ 优秀 |
| 近3月 | 234/456 | 前51% | -0.8% | 中等 |
| 近6月 | 89/456 | 前20% | +5.6% | ✅ 优秀 |
| 近1年 | 156/456 | 前34% | +3.2% | 良好 |

---

## 八、数据来源与置信度

| 数据项 | 来源 | 更新时间 | 置信度 |
|---|---|---|---|
| 经理档案 | neodata(主) + 同花顺问财(辅) | {query_date} | 高(多源验证) |
| 基金元数据 | neodata 基金基础信息 | {query_date} | 高 |
| 业绩数据 | neodata 基金业绩(主) + 同花顺问财(辅) | {query_date} | 高(结构数据) |
| 持仓明细 | neodata 基金重仓资产({disclose_date}, T+1) | {query_date} | **高**(一手数据) |
| 观点原文 | {source} | {query_date} | {confidence} |
| 扩展维度 | neodata 对应接口 / 同花顺问财 | {query_date} | 高 |
| 条件筛选(发现新经理) | 同花顺问财 | {query_date} | 高(结构化) |

{若使用了问财数据:}
> 🔄 **多源验证**: 经理信息和业绩数据经 neodata + 同花顺问财双重校验，数据可靠性更高。

> ⏰ **时效性提示**:基金季报通常在季度结束后 15 个工作日内披露(T+15),年报在 90 天内(T+90)。当前数据截至 {latest_disclose_date},下一期预计 {next_disclose_date_est} 前后更新。

---

*报告由 fund-manager-tracker Skill 自动生成*
*数据来源: neodata-financial-search + 同花顺问财 + WebFetch*
*如有疑问请核对原始数据源: [天天基金网](https://funds.eastmoney.com) / [同花顺问财](https://www.iwencai.com)*
```

---

## 三、批量报告模板

```markdown
# 基金经理跟踪汇总报告({N} 位经理)

> **查询日期**: YYYY-MM-DD HH:MM | **数据源**: neodata | **处理耗时**: XX 分 XX 秒

---

## 一、执行概要

| 指标 | 数值 |
|---|---|
| 名单总数 | {N} 位 |
| 成功处理 | {success_count} 位 |
| 处理失败 | {fail_count} 位 ({failed_names}) |
| 已离任 | {inactive_count} 位 |
| 平均每位持仓数 | {avg_holdings} 只 (>2%) |
| 总计发现显著变动 | {significant_changes} 处 |

---

## 二、经理状态总览表（按风格分组）

> **分组原则**: 按 watchlist.json 中的 `style_group` 字段分组，每组内按近1年收益率降序排列。
> 每个分组子表包含：经理 | 公司 | 标杆基金 | **业绩基准** | 规模 | 近1年 | 同类排名 | 关注理由

### 风格分组模板

```markdown
### 🟦 {风格组名}（{N}位）

| 经理 | 公司 | 标杆基金 | 业绩基准 | 规模 | 近1年 | 排名 | 🔍 关注理由/评价 |
|------|------|----------|----------|------|-------|------|------|
| {name} | {company} | {fund_name} ({code}) | {benchmark} | {scale} | {ret1y} | {rank} | {watch_reason} |
```

**预定义风格分组**（watchlist.json `style_group` 字段）:

| style_group | 颜色 | 典型经理 |
|-------------|------|----------|
| AI产业链 | 🟦 | 金梓才、郑希、刘元海、林清源 |
| 科技成长 | 🟩 | 曹晋、陈韫中、杨锐文、莫海波、张序 |
| QDII海外 | 🟨 | 徐成、恽雷 |
| 消费价值 | 🟧 | 张坤(双标杆:110011+118001亚洲精选)、谭丽 |
| 深度价值 | 🟫 | 姜诚、冯汉杰 |
| 周期轮动/灵活配置 | 🟪 | 胡中原、王冠桥 |
| 固收+可转债 | ⬛ | 李波 — **看债市观点**（可转债+利率策略为主，非个股持仓） |
| FOF/已离任 | ⬜ | 唐军、吴培文、叶勇、吴远怡 |

**分组内部排序规则**: 近1年收益率从高到低（数据缺失排后）

> 💡 **关注理由说明**:每位的"关注理由"来自用户预设的 manager-watchlist.json,方便快速回顾当初为什么跟踪这位经理。

### 整体股票仓位变化（季度对比）

> **数据源**: 天天基金网 API 基金资产组合表（权益投资占比）
> **用途**: 展示基金经理面对市场波动时的仓位调整方向，与季报观点交叉验证

```markdown
| 经理 | 标杆基金 | Q{N-1}仓位 | Q{N}仓位 | 变动 | 方向 |
|------|----------|----------|----------|------|:--:|
| 张坤 | 优质精选QDII | 94.2% | 93.8% | -0.4pct | = |
| 冯汉杰 | 主题领先A | 82.3% | 78.5% | -3.8pct | ↓↓ |
```

**仓位变动分级**:
- `↑↑/↓↓` — 变动 >±3pct（显著）
- `↑/↓` — 变动 ±1~3pct
- `=` — 变动 <±1pct（持平）

> 💡 仓位变动应结合季报观点验证。如冯汉杰Q1"战略保守"→仓位降3.8pct ✓；胡中原"增配航空+化工"→仓位升2.7pct ✓。若仓位变动与观点矛盾，需标注"⚠️ 言行不一"。

### 季报观点分组展示模板

> **分组原则**: 批量报告中按经理的核心投资立场分组，同组横向对比一致性和分歧。

**预定义分类**:

| 观点分组 | 经理 | 特征 |
|----------|------|------|
| 🔥 AI信徒 | 金梓才、郑希、刘元海、林清源、曹晋 | AI全产业链+高仓位 |
| 🏭 AI延伸 | 陈韫中、莫海波、胡中原 | AI+周期/消费/航空 |
| 🛡️ 防守型 | 姜诚、冯汉杰、谭丽 | 安全边际+低估值+等待 |
| 🏠 内需消费 | 张坤、杨锐文 | 消费复苏+能源转型 |
| 🌏 跨市场 | 恽雷、张序、徐成、王冠桥 | QDII/量化/稳健 |

> 💡 批量报告优先展示完整观点原文段落（200-400字），方便深度阅读。

---

## 三、逐位详细报告

<details>
<summary><strong>📊 1. 张坤 — 易方达蓝筹精选混合</strong></summary>

{此处嵌入单经理完整报告}

</details>

<details>
<summary><strong>📊 2. 吴培文 — 信澳周期动力混合</strong></summary>

{此处嵌入单经理完整报告}

</details>

...

---

## 四、变动对比算法(compare_periods 详细实现)

### 4.1 持仓变动计算

```python
def compare_holdings(prev_snapshot: HoldingSnapshot,
                     curr_snapshot: HoldingSnapshot) -> List[HoldingChange]:
    """
    输入:相邻两期的持仓快照
    输出:HoldingChange[] (带类型/符号/幅度)
    """

    # 构建映射:股票代码 → 仓位%
    prev_map = {h.stock_code: h.weight_pct for h in prev_snapshot.holdings}
    curr_map = {h.stock_code: h.weight_pct for h in curr_snapshot.holdings}

    changes = []

    # 取所有出现过的股票并集
    all_stocks = set(prev_map.keys()) | set(curr_map.keys())

    for stock_code in sorted(all_stocks):
        p = prev_map.get(stock_code, 0)  # 上期仓位(不在则为0)
        c = curr_map.get(stock_code, 0)  # 本期仓位(不在则为0)
        stock_name = get_stock_name(stock_code, prev_snapshot, curr_snapshot)

        # 判定变动类型
        if p == 0 and c > 0:
            change_type = "新建仓"
            symbol = "✚"
            rank_change = f"新进→第{get_rank(curr_snapshot, stock_code)}"
        elif p > 0 and c == 0:
            change_type = "清仓"
            symbol = "✖"
            rank_change = f"第{get_rank(prev_snapshot, stock_code)}→退出"
        elif abs(c - p) <= 0.1:  # ±0.1% 视为持平(浮点误差容差)
            change_type = "持平"
            symbol = "="
            rank_change = f"第{get_rank(prev_snapshot, stock_code)}→第{get_rank(curr_snapshot, stock_code)}"
        elif c > p:
            change_type = "加仓"
            symbol = "↑"
            rank_change = f"第{get_rank(prev_snapshot, stock_code)}→第{get_rank(curr_snapshot, stock_code)}"
        else:  # c < p
            change_type = "减仓"
            symbol = "↓"
            rank_change = f"第{get_rank(prev_snapshot, stock_code)}→第{get_rank(curr_snapshot, stock_code)}

        delta = c - p
        delta_abs = abs(delta)

        changes.append(HoldingChange(
            stock_code=stock_code,
            stock_name=stock_name,
            prev_weight_pct=p,
            curr_weight_pct=c,
            change_type=change_type,
            symbol=symbol,
            delta_pct=round(delta, 2),
            delta_absolute=round(delta_abs, 2),
            rank_change=rank_change
        ))

    # 排序:新建仓/清仓置顶,其余按幅度降序
    def sort_key(change):
        # 新建仓/清仓优先级最高
        if change.change_type in ["新建仓", "清仓"]:
            priority = 0
        else:
            priority = 1
        return (priority, -change.delta_absolute)

    changes.sort(key=sort_key)

    return changes


def get_stock_name(code, prev_snap, curr_snap):
    """从快照中获取股票名称"""
    for h in curr_snap.holdings:
        if h.stock_code == code:
            return h.stock_name
    for h in prev_snap.holdings:
        if h.stock_code == code:
            return h.stock_name
    return "未知"


def get_rank(snapshot, stock_code):
    """获取股票在该期中的排名(按仓位降序)"""
    sorted_holdings = sorted(snapshot.holdings, key=lambda h: h.weight_pct, reverse=True)
    for i, h in enumerate(sorted_holdings, 1):
        if h.stock_code == stock_code:
            return i
    return "-"
```

### 4.2 统计汇总计算

```python
def calculate_statistics(changes: List[HoldingChange]) -> dict:
    """计算变动统计"""

    stats = {
        "new_positions": sum(1 for c in changes if c.change_type == "新建仓"),
        "exited_positions": sum(1 for c in changes if c.change_type == "清仓"),
        "increased_positions": sum(1 for c in changes if c.change_type == "加仓"),
        "decreased_positions": sum(1 for c in changes if c.change_type == "减仓"),
        "unchanged_positions": sum(1 for c in changes if c.change_type == "持平"),
        "total_changes": len(changes),
    }

    # 找最大单项变动
    if changes:
        max_change = max(changes, key=lambda c: c.delta_absolute)
        stats["max_single_delta"] = {
            "stock": f"{max_change.stock_name}({max_change.stock_code})",
            "delta": f"{max_change.delta_absolute:.2f}",
            "type": max_change.change_type
        }
    else:
        stats["max_single_delta"] = {"stock": "-", "delta": "0", "type": "-"}

    return stats
```

### 4.3 观点 diff 计算

```python
def compare_outlooks(prev_outlook: OutlookText,
                    curr_outlook: OutlookText) -> OutlookChange:
    """
    输入:相邻两期的观点文本
    输出:OutlookChange (关键词差集 + 态度转变 + LLM 摘要)
    """

    # 关键词集合差
    prev_keywords = set(prev_outlook.keywords_extracted or [])
    curr_keywords = set(curr_outlook.keywords_extracted or [])

    keywords_added = list(curr_keywords - prev_keywords)
    keywords_removed = list(prev_keywords - curr_keywords)

    # 态度转变
    prev_attitude = prev_outlook.attitude or "未知"
    curr_attitude = curr_outlook.attitude or "未知"

    if prev_attitude == curr_attitude:
        attitude_shift = "无变化"
        significant = False
    else:
        attitude_shift = f"{prev_attitude}→{curr_attitude}"
        # 仅跨类别转变才算显著(中性→中性不算)
        significant = (prev_attitude != curr_attitude and
                      not ({prev_attitude, curr_attitude} <= {"中性", "未知"}))

    # LLM 生成摘要(如果两期都有原文)
    if prev_outlook.raw_text and curr_outlook.raw_text:
        llm_summary = generate_llm_outlook_summary(
            prev_text=prev_outlook.raw_text[:500],
            curr_text=curr_outlook.raw_text[:500],
            keywords_added=keywords_added[:5],
            keywords_removed=keywords_removed[:5],
            attitude_shift=attitude_shift
        )
    else:
        llm_summary = None

    return OutlookChange(
        has_comparison=bool(prev_outlook.raw_text and curr_outlook.raw_text),
        prev_attitude=prev_attitude,
        curr_attitude=curr_attitude,
        attitude_shift=attitude_shift,
        keywords_added=keywords_added,
        keywords_removed=keywords_removed,
        llm_generated_summary=llm_summary,
        significant_shift_detected=significant
    )


def generate_llm_outlook_summary(prev_text, curr_text, added, removed, shift):
    """
    调用 LLM 生成一句话变化摘要
    prompt 示例:
    "以下是某基金经理相邻两期的季报观点片段,请用一句话总结主要变化(100字内):
     上期:{prev_text}
     本期:{curr_text}
     新增关注:{added}
     不再提及:{removed}
     态度转变:{shift}"
    """
    # 实际调用 LLM API
    summary = call_llm(prompt)
    return summary
```

---

## 五、HTML 可视化版本(默认生成,与 MD 并行)

**默认行为**: `--format both`(默认值)同时生成 Markdown + HTML 双版本。

### 5.1 包含的可视化组件

| 组件 | 图表库 | 用途 |
|---|---|---|
| **业绩走势图** | ECharts line | 近1月/3月/6月/1年/3年收益率走势 + 同类百分位 ★新增 |
| **持仓饼图** | ECharts pie | 最新期前10大重仓股分布(颜色区分行业) |
| **持仓变动柱状图** | ECharts bar | Top10 变动幅度(红涨绿跌,符合A股惯例) |
| **业绩排名雷达图** | ECharts radar | 各时间段同类百分位对比 ★新增 |
| **规模走势折线图** | ECharts line | 近4季度规模变化(如启用 --include-scale) |
| **行业配置堆叠面积** | ECharts stack area | 行业配置比例变化(如启用 --include-sector) |
| **态度仪表盘** | ECharts gauge | 态度分数(-1~1)可视化(谨慎←→乐观) |

### 5.2 交互功能

- 鼠标悬停显示详细数值
- 点击切换不同报告期
- 导出 PNG/PDF 按钮
- 响应式布局(适配移动端)
- 业绩对比模式:多基金叠加查看收益率曲线

### 5.3 HTML 结构

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>{经理名} 基金跟踪报告 - {YYYYMMDD}</title>
  <script src="https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js"></script>
  <style>
    /* 响应式布局,A股颜色规范:红涨绿跌 */
    body { font-family: "Microsoft YaHei", sans-serif; max-width: 1200px; margin: 0 auto; padding: 20px; }
    .card { background: #fff; border-radius: 8px; padding: 20px; margin-bottom: 20px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
    .chart { width: 100%; height: 400px; }
    .up { color: #FF0000; }   /* A股惯例:涨为红 */
    .down { color: #00AA00; } /* 跌为绿 */
  </style>
</head>
<body>
  <h1>{经理名} 基金跟踪报告</h1>
  <div class="card"><h2>📊 业绩表现</h2><div id="chart-performance" class="chart"></div></div>
  <div class="card"><h2>🥧 持仓分布</h2><div id="chart-holdings" class="chart"></div></div>
  <div class="card"><h2>📈 持仓变动</h2><div id="chart-changes" class="chart"></div></div>
  <!-- 更多图表... -->
  <script>
    // ECharts 初始化 + 数据渲染
  </script>
</body>
</html>
```

---

## 六、格式化最佳实践

### 6.1 数字格式化

| 数据类型 | 格式 | 示例 |
|---|---|---|
| 仓位百分比 | 保留2位小数 + % | `9.87%` |
| 变动百分点 | 保留2位小数,正负号 | `+1.37%` / `-0.85%` |
| 规模(亿元) | 保留1位小数 | `256.3 亿元` |
| 市值(万元) | 整数或保留1位 | `12,345 万` |
| 任职年限 | 保留1位小数 | `7.8 年` |
| 排名分数 | 整数 | `123 / 456` (前27%) |

### 6.2 特殊情况展示

| 场景 | 展示方式 |
|---|---|
| 数据缺失 | `⚠️ 缺失` 或 `—`(破折号) |
| 无变化 | `无变化` 或 `=` 符号 |
| 显著变动 | **粗体** + 颜色标识(Markdown中可用**加粗**) |
| 异常值 | 附注释(如"较上期大幅波动") |

### 6.3 颜色规范(A股惯例)

虽然 Markdown 不支持颜色,但在 HTML 版本中使用:

| 含义 | 颜色 | 使用场景 |
|---|---|---|
| 涨/多/买入 | **红色** (#FF0000) | 加仓/新建仓/乐观态度 |
| 跌/空/卖出 | **绿色** (#00AA00) | 减仓/清仓/谨慎态度 |
| 中性/持平 | **灰色/黑色** (#333333) | 持平/中性态度 |
| 警告 | **橙色** (#FF9500) | 缺失/异常/需关注 |

---

## 七、文件输出规范

### 7.1 单次查询输出(双版本)

| 版本 | 文件名 | 保存位置 | 编码 |
|---|---|---|---|
| **Markdown** | `{经理名}_跟踪报告_{YYYYMMDD}.md` | 工作区根目录 | UTF-8 |
| **HTML** | `{经理名}_跟踪报告_{YYYYMMDD}.html` | 工作区根目录 | UTF-8 |

**工作区根目录**: `D:\Personal-Data\AI自动化\跟踪A股基金经理-抄作业\`

### 7.2 批量查询输出(双版本)

| 版本 | 主报告 | 子报告目录 | 日志 |
|---|---|---|---|
| **Markdown** | `基金经理跟踪汇总_YYYYMMDD.md` | `reports/YYYYMMDD/` (每经理一个 .md) | `reports/YYYYMMDD/log.json` |
| **HTML** | `基金经理跟踪汇总_YYYYMMDD.html` | `reports/YYYYMMDD/` (每经理一个 .html) | 同上 |

### 7.3 输出流程

```
查询完成
    │
    ├── 生成 Markdown 报告 → 保存到工作区
    │       └── 在对话中展示主要内容(present_files)
    │
    └── 生成 HTML 报告 → 保存到工作区
            └── 通过 present_files 在内置浏览器预览
```

**注意**:
- 两个版本内容一致,仅呈现形式不同(MD=表格 / HTML=ECharts图表)
- HTML 版本可离线打开,所有图表数据内嵌(无需联网加载 ECharts 以外的数据)
- 默认同时输出两个版本;用户可用 `--format md` 或 `--format html` 指定单版本

### 7.4 输出文件内容约束

> **重要**: 报告文件（MD/HTML）中只包含结构化数据和核心发现。**数据质量与异常报告**（neodata查不到/基金匹配错/已离任提示/数据缺失等）只输出在对话中，不写入文件。

| 内容类型 | MD/HTML 文件 | 对话输出 |
|----------|:---:|:---:|
| 经理档案/持仓/业绩/变动对比 | ✅ | ✅ (摘要) |
| 观点原文/展望摘要 | ✅ | ✅ (摘要) |
| ❌ 未找到的经理列表 | — | ✅ |
| ⚠️ 数据错配/异常/警告 | — | ✅ |
| 🔧 修正说明（如"曹普→曹晋已修正"） | — | ✅ |
| 执行概要中的失败/异常计数 | 简化为"已处理/待处理" | ✅ (详细) |

---

## 八、全局一致性检查清单

> **适用时机**: 每次修改报告内容后（无论是通过Skill生成还是手动编辑），必须逐项执行

### 8.1 统计数字一致性

| 检查项 | 位置 | 方法 |
|--------|------|------|
| 名单总数 | MD执行概要 + HTML summary-grid | 与 watchlist.json managers 数组长度对比 |
| 在职/已离任数 | MD总览表 + HTML JS data | 统计 is_active=true/false 的各几行 |
| 异常/未找到数 | MD错误表 + HTML错误表 | 与明细行数一致 |
| 业绩值 | MD表格 + HTML JS data | 同一个经理的 ret1y/rank1y 数值完全相同 |

### 8.2 MD/HTML 双版本对齐

- [ ] 同一经理的收益率、排名、标杆基金代码在两版报告中一致
- [ ] 同一经理的状态标记(✅/⛔/⚠️/❌)在两版报告中一致
- [ ] 汇总数字(---项、---位)在两版报告中一致
- [ ] HTML `<script>` 中的 data 数组与 `<table>` 中的行数据一致

### 8.3 watchlist.json 权威性

- [ ] 报告中的 `is_active` 以 watchlist.json 为准
- [ ] 报告中的 `company` 以 watchlist.json 为准
- [ ] 报告中的 `benchmark_fund.code` 以 watchlist.json 为准
- [ ] `is_active` 字段类型为 boolean，不存在字符串

### 8.4 变更后关键词残留检查

- [ ] 旧经理名/旧公司名/旧数据值是否仍在 MD/HTML/JSON 中出现（除"已修正"说明外）
- [ ] Grep 命令确认：`grep -r "旧值" *.md *.html *.json`

### 8.5 检查命令参考

```bash
# 搜旧名残留
grep "曹普\|陈耀中" *.md *.html *.json

# 数字一致性（以未找到为例）
grep -c "❌ 未找到" summary.md
grep -c "notfound" summary.html

# watchlist is_active 类型检查
python3 -c "
import json; f=open('assets/manager-watchlist.json'); d=json.load(f)
bad = [m['name'] for m in d['managers'] if not isinstance(m.get('is_active'), bool)]
print(f'非boolean is_active: {bad}' if bad else 'OK')
"
```
