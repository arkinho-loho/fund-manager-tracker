# 问财工具集成指南 (iWencai Integration)

> 本文档定义 `hithink-fundmanager-selector`(同花顺问财基金经理筛选) 和 `announcement-search`(同花顺问财公告搜索) 与 `fund-manager-tracker` 的集成方式。
> 问财两大工具分别提供基金经理画像/条件筛选能力与金融公告搜索能力,作为 neodata 的辅助数据源。

---

## 一、工具定位与合作关系

### 分工矩阵

| 功能维度 | neodata(主力) | 问财-基金经理筛选 | 问财-公告搜索 | 说明 |
|---|---|---|---|---|
| 经理基本信息(履历/在管/规模) | ✅ 主数据源 | ✅ 辅助验证 | ❌ 不覆盖 | neodata+基金经理筛选双源校验 |
| 基金持仓明细(重仓股) | ✅ **唯一来源** | ❌ 不覆盖 | ❌ 不覆盖 | 问财无此能力 |
| 季报/年报观点原文 | ✅ **唯一来源** | ❌ 不覆盖 | ✅ L2直达 | **观点提取链路关键升级** |
| 各阶段收益率 | ✅ 主数据源 | ✅ 补充 | ❌ 不覆盖 | neodata 缺失时 fallback |
| 同类排名 | ✅ 主数据源 | ✅ 补充 | ❌ 不覆盖 | neodata 缺失时 fallback |
| 最大回撤/夏普/波动率 | ⚠️ 部分覆盖 | ✅ 更丰富 | ❌ 不覆盖 | 问财基金经理筛选在此维度更优 |
| 投资风格判断 | ⚠️ 需推断 | ✅ 直接提供 | ❌ 不覆盖 | 问财有风格标签 |
| 条件筛选(找新经理) | ❌ 不支持 | ✅ **唯一来源** | ❌ 不覆盖 | 问财筛选的核心特色能力 |
| 基金/股票公告搜索 | ❌ 不支持 | ❌ 不覆盖 | ✅ **唯一来源** | 季报/年报/分红/回购等公告直达 |
| 重仓股动态查询 | ❌ 不支持 | ❌ 不覆盖 | ✅ **唯一来源** | 搜索重仓股近期公告(可选功能) |

### 调用优先级规则

```
路由判断:
  1. 持仓相关 → 只走 neodata
  2. 观点原文 → neodata docData(L1) → 问财公告搜索(L2) → WebSearch(L3) → WebFetch(L4) → 标记缺失(L5)
  3. 经理基本信息 → neodata 优先,查不到时 fallback 问财基金经理筛选
  4. 业绩数据 → neodata 优先,缺失/不全时问财基金经理筛选补充
  5. 条件筛选(找新经理) → 只走问财基金经理筛选
  6. 风控指标(回撤/夏普) → 问财基金经理筛选优先(数据更丰富)
  7. 公告搜索(季报原文/重仓股动态) → 只走问财公告搜索
```

---

## 二、问财调用方式

### 2.1 环境要求

- `IWENCAI_BASE_URL` 和 `IWENCAI_API_KEY` 环境变量已配置(已在 `.bash_profile` 中)
- `hithink-fundmanager-selector` 技能已安装在工作区

### 2.2 CLI 调用

```bash
# 技能安装路径
SKILL_PATH="$SKILL_DIR/../hithink-fundmanager-selector"
PYTHON="$PYTHON_PATH"

# 查询基金经理
cd "$SKILL_PATH" && $PYTHON scripts/cli.py --query "查询条件" --limit 20
```

### 2.3 常用查询模板

#### 经理画像(获取详细信息)
```python
# 单经理详细查询
query = "基金经理张坤的管理规模、近一年收益率、同类排名、最大回撤、投资风格"
```

#### 业绩数据补充
```python
# 指定基金的业绩
query = "易方达蓝筹精选混合(005827)的近一月、近三月、近六月、近一年、近三年收益率和最大回撤"
```

#### 条件筛选(特色能力)
```python
# 按多条件筛选经理
query = "管理规模超100亿、近一年收益排名前20%、近三年最大回撤小于-20%的基金经理"

# 按风格筛选
query = "价值型、管理规模50亿以上的基金经理"

# 按经验筛选
query = "从业年限10年以上、年化收益超10%的基金经理"

# 按规模筛选
query = "管理规模超200亿的明星基金经理"
```

---

## 三、Step 1: 经理验证增强(双源校验)

### 3.1 标准流程

```python
def validate_manager_enhanced(name: str) -> ManagerProfile:
    """增强版在职验证: neodata(主) + 问财(辅)"""

    # ---- Phase 1: neodata 查询 ----
    result = call_neodata(f"基金经理 {name} 的履历和在管基金及规模")
    neodata_profile = parse_neodata_profile(result)

    if neodata_profile and neodata_profile.is_active:
        # neodata 查询成功，标记为主来源
        neodata_profile.primary_source = "neodata"

        # ---- Phase 2: 问财辅助验证(可选) ----
        try:
            iwencai_result = call_iwencai(f"基金经理{name}的管理规模、从业年限、投资风格")
            if iwencai_result:
                neodata_profile.iwencai_cross_check = {
                    "found": True,
                    "scale_match": compare_scale(neodata_profile.total_scale, iwencai_result),
                    "style_from_iwencai": iwencai_result.get("投资风格"),
                    "data_discrepancies": []
                }
        except:
            neodata_profile.iwencai_cross_check = {"found": False}

        return neodata_profile

    # ---- Fallback: neodata 查不到，切问财 ----
    iwencai_result = call_iwencai(f"基金经理{name}的在管基金和管理规模")
    if iwencai_result and iwencai_result.get("在管基金数", 0) > 0:
        return ManagerProfile(
            name=name,
            is_active=True,
            primary_source="iwencai(fallback)",
            total_scale=iwencai_result.get("管理规模"),
            managed_funds_count=iwencai_result.get("在管基金数"),
            warnings=["数据来源: 同花顺问财(fallback from neodata)"]
        )

    # 两数据源都查不到
    return ManagerProfile(name=name, is_active=False,
                          warnings=["两数据源(neodata+问财)均未查到该经理"])
```

### 3.2 交叉验证输出

当两数据源都有数据时，输出交叉验证表：

```markdown
## 经理信息交叉验证

| 对比项 | neodata | 问财 | 一致性 |
|---|---|---|---|
| 管理总规模 | 416.7 亿 | 420.5 亿 | ✅ 基本一致(差异<5%) |
| 在管基金数 | 4 只 | 4 只 | ✅ 完全一致 |
| 投资风格 | value(推断) | 价值型 | ✅ 一致 |
| 近1年收益率 | -14.1% | -13.8% | ✅ 基本一致 |
```

---

## 四、Step 4b: 业绩获取增强(双源补充)

### 4.1 标准流程

```python
def extract_performance_enhanced(fund_code: str, fund_name: str) -> PerformanceMetrics:
    """增强版业绩获取: neodata(主) + 问财(辅)"""

    # ---- Phase 1: neodata ----
    neodata_result = call_neodata(f"基金{fund_code}的各阶段收益率和同类排名")
    metrics = parse_performance(neodata_result)

    # ---- Phase 2: 补充缺失项 ----
    missing_periods = detect_missing_periods(metrics)
    if missing_periods:
        try:
            iwencai_result = call_iwencai(
                f"{fund_name}({fund_code})的" +
                "、".join([f"{p}收益率" for p in missing_periods]) +
                "和最大回撤、夏普比率"
            )
            if iwencai_result:
                for period in missing_periods:
                    fill_from_iwencai(metrics, period, iwencai_result)

                # 问财特有数据：最大回撤、夏普比率
                if "最大回撤" in iwencai_result:
                    metrics.max_drawdown_1y = iwencai_result["最大回撤"]
                if "夏普比率" in iwencai_result:
                    metrics.sharpe_ratio_1y = iwencai_result["夏普比率"]
        except:
            pass

    return metrics
```

### 4.2 问财特有数据字段

| 字段 | 问财描述 | 在报告中的位置 |
|---|---|---|
| 最大回撤 | `最大回撤` | 业绩表额外行或备注 |
| 夏普比率 | `夏普比率` | 同上 |
| 投资风格 | `投资风格`(如"价值型"/"成长型") | 经理档案 |
| 从业年限 | `从业年限`(如"13年") | 经理档案 |
| 年化收益 | `年化收益率` | 经理档案 |
| 同类排名 | `同类排名`(如"45/856") | 业绩表 |

> 问财数据标注为 `来源:同花顺问财`，置信度：**高**(结构化数据)

---

## 五、条件筛选(问财特色功能)

### 5.1 从筛选到跟踪

```python
def discover_and_track_managers(query_condition: str, limit: int = 10):
    """
    用问财筛选符合条件的基金经理，自动加入临时跟踪列表
    示例: "管理规模超100亿、近一年收益排名前20%、近三年最大回撤小于-20%的基金经理"
    """

    # Step 1: 问财筛选
    iwencai_result = call_iwencai(query_condition, limit=limit)
    if not iwencai_result or not iwencai_result.get("datas"):
        return {"found": 0, "managers": []}

    # Step 2: 解析经理列表
    managers = []
    for item in iwencai_result["datas"]:
        manager = {
            "name": item.get("基金经理姓名"),
            "scale": item.get("管理规模"),
            "return_1y": item.get("近一年收益率"),
            "peer_rank": item.get("同类排名"),
            "max_drawdown": item.get("最大回撤"),
            "style": item.get("投资风格"),
            "tenure": item.get("从业年限"),
            "source": "问财筛选"
        }
        managers.append(manager)

    # Step 3: 对每位经理执行全流程跟踪
    for m in managers:
        track_single_manager(m["name"])

    return {"found": len(managers), "managers": managers}
```

### 5.2 常用筛选条件模板

| 场景 | 问财 query |
|---|---|
| 大规模 + 高收益 | `管理规模超100亿、近一年收益排名前20%的基金经理` |
| 低回撤 + 稳定 | `近三年最大回撤小于-20%、夏普比率大于1的基金经理` |
| 长期绩优 | `从业年限10年以上、年化收益超15%的基金经理` |
| 风格筛选 | `价值型、管理规模50亿以上、近三年收益排名前30%的基金经理` |
| 择时能力 | `近一年换手率适中、行业轮动型的基金经理` |
| 新锐经理 | `从业年限3-5年、近一年收益排名前10%、管理规模10-50亿的基金经理` |

---

## 六、数据冲突处理

### 6.1 冲突分级

| 差异程度 | 定义 | 处理方式 |
|---|---|---|
| ✅ **一致** | 差异 < 5% 或等值 | 采信 neodata，标注"两数据源一致" |
| ⚠️ **轻微差异** | 差异 5-20% | 采信 neodata，标注"neodata:XXX / 问财:YYY" |
| ❌ **显著差异** | 差异 > 20% 或数据矛盾 | 标注冲突，建议用户手动核实 |

### 6.2 冲突处理示例

```python
def detect_conflict(neodata_val, iwencai_val, field_name, tolerance=0.2):
    """检测并报告两数据源间的数据冲突"""

    if neodata_val is None and iwencai_val is None:
        return None  # 两源均无数据
    if neodata_val is None:
        return {"source": "iwencai", "value": iwencai_val, "note": "仅问财有数据"}
    if iwencai_val is None:
        return {"source": "neodata", "value": neodata_val, "note": "仅 neodata 有数据"}

    diff = abs(neodata_val - iwencai_val) / max(abs(neodata_val), abs(iwencai_val), 0.01)
    if diff <= 0.05:
        return {"source": "neodata", "value": neodata_val,
                "consistency": "✅ 一致",
                "note": f"neodata:{neodata_val} / 问财:{iwencai_val}"}
    elif diff <= 0.20:
        return {"source": "neodata(优先)", "value": neodata_val,
                "consistency": "⚠️ 轻微差异",
                "note": f"neodata:{neodata_val} / 问财:{iwencai_val}"}
    else:
        return {"source": "neodata(仲裁)", "value": neodata_val,
                "consistency": "❌ 显著差异",
                "note": f"neodata:{neodata_val} / 问财:{iwencai_val}，建议核实"}
```

---

## 七、公告搜索集成(announcement-search) ★新增

### 7.1 能力说明

`announcement-search` 是同花顺问财的**公告搜索**技能,可直接搜索基金的季报、年报、分红、公告等原始文档。在 `fund-manager-tracker` 中的主要用途:

1. **观点提取链路升级**:作为 L2 降级层,替代 WebSearch 搜索季报原文(更快、更准)
2. **重仓股动态**:搜索十大重仓股的近期公告(分红、业绩预告、回购等)
3. **经理相关公告**:搜索基金经理的任职/离任/变动公告

### 7.2 调用方式

```bash
# 技能路径
ANNOUNCE_PATH="$SKILL_DIR/../announcement-search/scripts"
PYTHON="$PYTHON_PATH"

# 搜索基金季报公告(观点提取 L2)
cd "$ANNOUNCE_PATH" && $PYTHON __main__.py "005827 易方达蓝筹精选混合 2025年第四季度 季报" --limit 5

# 搜索基金年报
cd "$ANNOUNCE_PATH" && $PYTHON __main__.py "005827 易方达蓝筹精选混合 年报 管理人展望" --limit 5

# 搜索重仓股公告
cd "$ANNOUNCE_PATH" && $PYTHON __main__.py "贵州茅台 业绩预告 分红" --limit 5

# 搜索经理公告
cd "$ANNOUNCE_PATH" && $PYTHON __main__.py "张坤 易方达 基金经理 任职" --limit 5
```

### 7.3 在观点提取中的角色

announcement-search 被插入到观点提取的五级降级链中作为 **Level 2**:

```
L1: neodata docData 试搜
    ↓ 未命中
L2: announcement-search(问财公告直达) ★新加入
    ↓ 未命中
L3: WebSearch(通用搜索)
    ↓ 命中URL
L4: WebFetch(抓取原文)
    ↓ 全部失败
L5: 标记缺失,不阻断
```

相比原来的 L2 WebSearch,announcement-search 的优势:
- 直接针对金融公告数据源,命中率更高
- 返回结构化公告数据(标题+摘要+URL),不需要额外解析
- 速度和稳定性优于通用网页搜索

### 7.4 重仓股动态(可选功能)

当报告涉及重仓股分析时,可搜索重仓股的近期公告:

```python
def get_holding_stock_news(stock_name, stock_code, days=30):
    """搜索重仓股近期公告"""
    cmd = f'cd "{ANNOUNCE_PATH}" && $PYTHON __main__.py "{stock_name} 公告" --limit 3'
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

    if result.returncode != 0:
        return []

    try:
        data = json.loads(result.stdout)
        return data.get("data", []) if isinstance(data, dict) else data
    except:
        return []
```

返回结构:
```json
[
  {
    "title": "贵州茅台:2025年度业绩预告",
    "summary": "预计2025年度实现归属于上市公司股东的净利润约XXX亿元...",
    "url": "https://...",
    "publish_date": "2026-01-15"
  }
]
```

### 7.5 错误处理

| 故障场景 | 处理 | 是否阻断 |
|---|---|---|
| announcement-search 鉴权失败 | IWENCAI_API_KEY 不可用,降级到 L3 WebSearch | ❌ 不阻断 |
| 公告搜索无结果 | 降级到 L3 WebSearch | ❌ 不阻断 |
| 搜索结果质量低(摘要不完整) | 保留 URL 供 L4 WebFetch 抓取全文 | ❌ 不阻断 |
| API 超时 | 视为无结果,降级到 L3 | ❌ 不阻断 |

---

## 八、集成清单

### 每次执行时的检查点

- [ ] IWENCAI_API_KEY 环境变量是否存在（不存在则降级为仅 neodata）
- [ ] 需要查询的是持仓(仅neodata)还是经理信息/业绩(可多源)还是观点原文(五级降级)
- [ ] neodata 查询是否成功（失败则尝试问财 fallback）
- [ ] 问财基金经理筛选查询是否成功（失败则仅用 neodata 数据）
- [ ] 问财公告搜索查询是否成功（失败则降级到 WebSearch）
- [ ] 三数据源数据是否冲突（冲突则标注说明）
- [ ] 条件筛选新经理(场景4)是否触发（触发则用问财筛选并自动跟踪）

### 推荐场景

| 场景 | 建议走 | 原因 |
|---|---|---|
| 查张坤的持仓变化 | 仅 neodata | 问财不覆盖持仓 |
| 确认经理在职状态 | 多源验证 | 更高准确率 |
| 拉短期+中期业绩 | neodata 优先 + 问财基金经理筛选补充 | 互补覆盖 |
| 找符合条件的经理 | 仅问财基金经理筛选 | neodata 无此能力 |
| 风控分析(回撤/夏普) | 问财基金经理筛选优先 | 问财数据更丰富 |
| **季报观点获取** | **L1 neodata → L2 公告搜索 → L3 WebSearch** | **公告搜索显著提高命中率** |
| 重仓股近期动态 | 仅问财公告搜索 | 公告搜索专属能力 |
| 经理任职/离职公告 | 仅问财公告搜索 | 公告搜索专属能力 |
