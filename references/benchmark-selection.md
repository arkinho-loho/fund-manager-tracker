# 标杆基金筛选规则 (Benchmark Selection)

> 本文档定义 Step 2 select_benchmark_fund 的完整算法。目标:从经理管理的多只基金中,智能筛选出最具代表性的"标杆产品"用于后续持仓与观点分析。

---

## 一、筛选目标与设计原则

### 目标
当基金经理管理 **N 只基金**(通常 3-15 只)时,选出 **1-3 只** 标杆基金:
- 同一投资风格 → 选 **1-2 只**(优先取独立管理且范围最广的;若前2只差异化明显则都保留)
- 不同风格(如同时管价值+成长) → 每种风格各选 **1-2 只**
- **跨风格/跨市场差异化明显** → 可扩大到 **2-3 只**(如同时管QDII全球+QDII亚洲+港股通的经理,选2-3只覆盖不同维度)

### 设计原则

1. **可审计性**:每只标杆必须附明确的选取理由(为什么选它而不是其他)
2. **代表性优先**:选最能反映该经理投资理念的产品
3. **数据可得性**:确保 neodata 能查到该基金的完整持仓/规模/排名数据
4. **用户可控**:允许用户通过 `--fund` 参数覆盖自动选择
5. **容错性**:元数据不全时降级处理并标注警告

---

## 二、四键排序算法(核心)

### 2.1 排序优先级(由高到低)

```
sort_key = (
  ① manager_since   ASC,     # 经理任职最久(唯一硬标准)
  ② scope_breadth   DESC     # 投资范围越广越好(用户明确要求)
)
```

**前置检查**:在排序之前,先进行 **共管检查**——每只基金是否为该经理独立管理。
共管基金直接降权,不作为标杆首选。如果所有基金均非独立管理,选共管人数最少+任职最久的。

### 2.2 各键详细说明

#### 键 ①:经理任职时间(manager_since) — **最重要**

**理由**:任职时间越长,该基金越能反映经理的完整投资理念和调仓历史。

**数据来源**:neodata "基金 {code} 基金经理任职日期"

**降级策略**(如果缺任职日):
- 用基金成立日(setup_date)代替 → 标注警告:"⚠️ 缺任职日,用成立日代替"
- 如果成立日也缺 → 跳过此键,仅用剩余键排序 → 标注:"⚠️ 部分元数据缺失,排序可能受影响"

#### 键 ②:投资范围广度(scope_breadth) — **用户明确要求**

**理由**:用户要求"优选选择投资范围广的标杆基金",因为:
- A股+港股+美股 > 纯A股(覆盖更多机会)
- 经理在更广范围内的配置更能体现真实能力

**评分表**(详见 data-models.md FundInfo.scope_breadth):

| 投资范围 | 分值 | 判断规则 |
|---|---|---|
| A股+港股+美股(全球/QDII) | **3** | 名称含"QDII"/"全球"/"海外" |
| A股+港股(沪港深) | **2** | 名称含"沪港深"/"港股"/invest_scope 含"港股通" |
| 纯 A 股 | **1** | 默认 |

**实现逻辑**:
```python
def score_scope_breadth(fund_name, invest_scope):
    name_lower = fund_name.lower()
    scope_lower = (invest_scope or "").lower()

    if any(kw in name_lower + scope_lower for kw in ["qdii", "全球", "海外", "港美"]):
        return 3
    elif any(kw in name_lower + scope_lower for kw in ["沪港深", "港股通", "港股"]):
        return 2
    else:
        return 1  # 默认纯A股
```

#### 键 ③:共管降权(Co-Management Penalty)

**背景**:标杆基金应当最能反映**该经理独立**的投资能力。如果有其他基金经理共同管理,则持仓和观点可能受他人影响,不代表标的经理本人。

**规则**:
- **独立管理** → 权重为 1.0(无惩罚,推荐选择)
- **2人共管** → 权重 0.7(降权,非首选)
- **3人及以上共管** → 权重 0.5(显著降权,尽量避免)
- **团队制管理(无明确个人)** → 权重 0.3(不推荐,应该寻找其他基金)

**优先级等级**:此键的行为比较特殊——它不直接参与排序数值,而是在排序前**过滤**并将共管基金标记"⚠️ 共管",提示用户和管理者注意。

**实际影响**:
1. 如果经理的存在一支独立管理且任职时间足够长的基金,优先选它作为标杆
2. 如果经理的**所有基金都有共管** → 选共管人数最少 + 该经理任职最久的那支,并标注"⚠️ 该经理全部基金均有共管"
3. 如果经理的长期管理基金近期增聘了共管 → 考虑切换到另一支该经理独立管理的基金(即使任职时间短一点)

**数据来源**:neodata "基金 {code} 基金经理任职情况" 或 WebSearch 搜索"基金名称 基金经理 任职"了解是否有增聘

**示例**:
- 张坤 vs 易方达蓝筹精选:2026年5月增聘何一铖、杨思亮 → **3人共管 → 权重0.5,强烈不推荐**
- 张坤 vs 易方达优质精选(QDII):截至最新数据仍为张坤独立管理 → **权重1.0,推荐**

---

## 三、分组逻辑(按投资风格)

### 3.1 风格来源优先级

1. **第一优先**:用户提供标签(`assets/manager-watchlist.json` 的 `style_tags` 字段)
2. **第二优先**:从基金类型推断:
   - 基金名称含"价值"/"蓝筹" → `value`
   - 基金名称含"成长"/"新兴"/"科技" → `growth`
   - 基金名称含"中小盘" → `mid_small_cap`
   - 基金名称含"量化" → `quant`
   - 其他 → `balanced`(默认)
3. **第三优先**:如果无法判断 → 所有基金归为同一组 `unknown`,只选 1 只标杆

### 3.2 分组示例

**场景**:杨锐文管理以下基金:

| 基金代码 | 基金名称 | 用户标签 | 推断风格 | 最终分组 |
|---|---|---|---|---|
| 260112 | 景顺长城新能源产业股票A | ["行业轮动","新兴成长"] | growth | **成长组** |
| 260109 | 景顺长城鼎益混合A(LOF) | ["行业轮动"] | balanced | **均衡组** |
| 260111 | 景顺长城优选混合 | 无标签 | balanced | **均衡组** |

**结果**:
- 成长组(1只):260112 直接当选标杆
- 均衡组(2只):按四键排序,选 260109 或 26011 中得分高的

### 3.3 特殊情况处理

**场景**:某经理所有基金都无法确定风格(无标签且名称无明显特征)

→ 归入单一 `unknown` 组,按四键排序选第 1 名作为唯一标杆
→ 标注警告:"⚠️ 未检测到明显风格差异,已将所有基金视为同类处理"

---

## 四、完整算法伪代码

```python
def select_benchmark_fund(manager_profile: ManagerProfile) -> List[FundInfo]:
    """
    输入:经理档案(含 managed_funds 列表和 style_tags)
    输出:带 is_benchmark 标记的基金列表(每组1只标杆)
    """

    # Step 1: 补全每只基金的元数据
    for fund in manager_profile.managed_funds:
        fund = enrich_fund_metadata(fund)  # neodata 查询
        fund.style = determine_style(fund, manager_profile.style_tags_from_user)
        fund.scope_breadth = calculate_scope_breadth(fund)

    # Step 2: 按风格分组
    groups = group_by(manager_profile.managed_funds, key=lambda f: f.style)

    benchmarks = []
    for style, funds_in_group in groups.items():
        # Step 3: 先分两池:独立管理 vs 共管
        sole_pool = [f for f in funds_in_group if f.is_solely_managed]
        co_pool = [f for f in funds_in_group if not f.is_solely_managed]

        # 优先从独立池选,按两键排序
        pool = sole_pool if sole_pool else co_pool
        if not sole_pool:
            warnings.append(f"⚠️ {style}组全部基金均有共管,从共管中选最优")

        sorted_funds = sorted(pool, key=lambda f: (
            f.manager_since or datetime.max,          # ① 任职最早(ASC)
            -(f.scope_breadth or 1),                  # ② 范围最广(DESC,取负变ASC)
        ))

        # Step 4: 取前 1-3 只为标杆(按差异化程度)
        max_benchmarks = 1
        total_in_pool = len(sorted_funds)

        if total_in_pool >= 3:
            # 检查前3只的投资方向和范围是否有明显差异
            top3 = sorted_funds[:3]
            scopes = set(f.scope_breadth for f in top3)
            styles = set(f.style for f in top3)
            # 如果有 2 种以上投资范围或风格,就扩大到 2-3 只
            if len(scopes) >= 2 or len(styles) >= 2:
                # 至少选2只;如果第3只与第1/2只范围都不同,选3只
                max_benchmarks = 2 if len(scopes) >= 2 else 3
            elif any("QDII" in (f.name or "") or "全球" in (f.name or "") for f in top3):
                # 有QDII/全球产品的,选2只(全球 + 本土)
                max_benchmarks = 2

        benchmarks_in_group = sorted_funds[:max_benchmarks]
        for i, bm in enumerate(benchmarks_in_group):
            bm.is_benchmark = True
            bm.benchmark_reason = generate_reason(bm, total_in_pool, rank=i+1, total_bm=len(benchmarks_in_group))
            benchmarks.append(bm)

        # 非标杆基金标记
        for f in sorted_funds[max_benchmarks:]:
            f.is_benchmark = False

    return benchmarks


def generate_reason(benchmark: FundInfo, total_in_group: int, rank: int = 1, total_bm: int = 1) -> str:
    """生成可审计的选取理由"""

    reasons = []
    if total_bm > 1:
        reasons.append(f"[标杆{rank}/{total_bm}]")

    # 任职时间
    if benchmark.manager_since:
        tenure = years_since(benchmark.manager_since)
        reasons.append(f"任职自{benchmark.manager_since.strftime('%Y-%m')}(管理{tenure:.1f}年)")

    # 独立管理
    if benchmark.is_solely_managed:
        reasons.append("独立管理")
    else:
        reasons.append(f"共管{benchmark.co_managers}人")

    # 投资范围(取代规模)
    scope_desc = {3: "A股+港股+美股(全球)", 2: "A股+港股(沪港深)", 1: "纯A股"}
    reasons.append(f"投资范围{scope_desc.get(benchmark.scope_breadth, '未知')}")
    reasons.append(f"投资范围{scope_desc.get(benchmark.scope_breadth, '未知')}")

    # 共管状态
    if not benchmark.is_solely_managed:
        reasons.append(f"共管{benchmark.co_managers}人)

    # 组内竞争情况
    if total_in_group > 1:
        reasons.append(f"(从{total_in_group}只{benchmark.style}类基金中胜出)")

    return ", ".join(reasons)


def enrich_fund_metadata(fund: FundInfo) -> FundInfo:
    """调用 neodata 补全基金元数据"""

    query = f"基金 {fund.code} 的类型、投资范围、成立日期、现任经理{manager_name}任职日期、最新规模"

    result = call_neodata(query)
    # 解析 result → 更新 fund.type / invest_scope / setup_date / manager_since / latest_scale

    # 数据质量检查
    missing_fields = []
    if not fund.manager_since: missing_fields.append("任职日期")
    if not fund.invest_scope: missing_fields.append("投资范围")
    if not fund.latest_scale: missing_fields.append("最新规模")

    if missing_fields:
        fund.data_quality = "partial"
        fund.warnings.append(f"⚠️ 缺少字段:{', '.join(missing_fields)},部分排序依据可能不精确")
    else:
        fund.data_quality = "complete"

    return fund
```

---

## 五、输出示例

### 示例 1:单风格单标杆

**输入**:张坤管理 5 只基金(全部偏价值消费风格)

**输出**:
```json
[
  {
    "code": "005827",
    "name": "易方达蓝筹精选混合",
    "type": "混合型",
    "style": "value",
    "is_benchmark": false,
    "co_managers": 2,
    "warnings": ["⚠️ 共管:除张坤外还有2位经理"],
    ...
  },
  {
    "code": "110011",
    "name": "易方达优质精选混合(QDII)",
    "type": "混合型",
    "style": "value",
    "is_benchmark": true,
    "benchmark_reason": "任职自2012-09(管理13.3年),独立管理,投资范围A股+港股+美股(全球)(从5只基金中胜出)",
    ...
  },
  {"code": "110011", "name": "易方达中小盘混合", "is_benchmark": false, ...},
  {"code": "009484", "name": "易方达优质精选混合", "is_benchmark": false, ...},
  ...
]
```

### 示例 2:双风格双标杆

**输入**:某经理同时管价值和成长两类产品

**输出**:
```json
[
  {
    "code": "XXX001",
    "name": "XX价值精选混合",
    "style": "value",
    "is_benchmark": true,
    "benchmark_reason": "任职自2015-03(管理10.3年),价值类成立最久,投资范围纯A股(从3只价值类基金中胜出)"
  },
  {
    "code": "XXX002",
    "name": "XX成长先锋股票",
    "style": "growth",
    "is_benchmark": true,
    "benchmark_reason": "任职自2019-06(管理6.1年),成长类成立最久,投资范围A股+港股(从2只成长类基金中胜出)"
  },
  // ... 其余非标杆基金
]
```

### 示例 3:跨市场多标杆(2-3只)

**输入**:徐成(国海富兰克林)管理16只基金,包含全球科技QDII、亚洲机会QDII、港股通、A股混合等多个方向

**输出**:
```json
[
  {
    "code": "457001",
    "name": "国富亚洲机会股票(QDII)A",
    "style": "qdii_asia",
    "is_benchmark": true,
    "benchmark_reason": "[标杆1/2] 任职自2015-12(管理10.5年),独立管理,投资范围A股+港股+美股(全球),聚焦亚洲市场"
  },
  {
    "code": "006373",
    "name": "国富全球科技互联混合(QDII)人民币A",
    "style": "qdii_global",
    "is_benchmark": true,
    "benchmark_reason": "[标杆2/2] 任职自2018-06(管理8年),与狄星华共管(2人),投资范围A股+港股+美股(全球),聚焦全球科技,与亚洲机会形成互补"
  },
  // ... 其余非标杆基金
]
```

**适用场景**:经理同时管理 **投资范围显著不同** 的多只基金时,扩大标杆数量:
- QDII全球 + QDII亚洲 → 2只(全球覆盖+区域聚焦)
- 纯A股价值 + A股+港股成长 → 2只(本土+互联互通)
- QDII + 沪港深 + 纯A股 → 2-3只(覆盖完整投资版图)

---

## 六、用户覆盖机制

### 场景:用户指定具体基金

**用户输入**: "用易方达中小盘混合(110011)来跟踪张坤"

**处理**:
```python
if user_specified_fund_code:
    # 跳过整个筛选流程
    specified_fund = find_fund_by_code(managed_funds, user_specified_fund_code)
    if specified_fund:
        specified_fund.is_benchmark = True
        specified_found.benchmark_reason = "用户指定(--fund 参数)"
        return [specified_fund]
    else:
        raise ValueError(f"未找到基金 {user_specified_fund_code},请确认是否在该经理名下")
```

**注意**:即使用户指定,仍需验证该基金确实在经理的在管列表中。

---

## 七、边界情况与错误处理

| 场景 | 处理方式 | 输出 |
|---|---|---|
| 经理仅管 1 只基金 | 直接当选,无需排序 | 1 只标杆 |
| 多只基金完全同分 | 按代码字典序决胜 | 1 只标杆 + 警告"存在并列" |
| **跨市场/跨方向明显**(含QDII+港股+纯A股等不同投资范围) | 扩大到 2-3 只 | 2-3 只标杆(覆盖不同投资方向) |
| 所有基金都缺关键元数据(如任职日) | 用已有字段降级排序 | 可能不准确 + 强烈警告 |
| 风格分组后某组只有 1 只 | 该组直接当选 | 正常 |
| 用户指定的基金不在在管列表 | 报错提示 | 异常终止该经理流程 |
| 某只基金的元数据查询失败 | 从排序中排除或用默认值填充 | 标注"数据不全" |

---

## 八、可审计性检查清单

每次筛选完成后,标杆基金应包含以下可审计信息(用于报告展示):

- [ ] 候选池总数:N 只
- [ ] 分组情况:X 个风格组(组名+数量)
- [ ] 每组排序明细(前 3 名的关键字段值)
- [ ] 最终选中理由(自然语言描述)
- [ ] 是否有用户覆盖(是/否)
- [ ] 数据质量警告列表
