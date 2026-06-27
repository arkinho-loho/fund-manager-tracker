# 观点提取五级降级策略 (Outlook Extraction)

> 本文档定义 Step 4a extract_outlook 的完整算法,用于从基金季报/年报中提取"管理人展望/投资策略"段落原文。

---

## 一、能力缺口与降级背景

### 问题
neodata-financial-search **不直接提供**基金季报/年报的原文内容:
- neodata 的 `docData` 是**财经资讯/研报/公告召回引擎**,不是季报数据库
- 无法保证能精确召回指定基金+指定报告期的原文
- 即使召回,也可能是媒体转载片段而非完整原文

### 升级方案:五级降级链(引入问财公告搜索)

```
Level 1: neodata docData 试搜(最快速)
    ↓ 未命中或置信度不足
Level 2: announcement-search(问财公告直达) ← 优先
         OR 天天基金网公告页(fundf10.eastmoney.com/jjgg_{code}_3.html) ← 备选
    ↓ 未命中或置信度不足
Level 3: WebSearch 搜索(不限站点)
    ↓ 命中 URL 但未抓取到正文
Level 4: WebFetch 抓取一手原文(最高置信度)
    ↓ 全部失败
Level 5: 标记缺失,继续执行持仓分析(不阻断)
```

---

## 二、Level 1 — neodata docData 试搜

### 2.1 调用方式

```bash
cd D:\workbuddy\resources\app.asar.unpacked\resources\builtin-skills\neodata-financial-search
python scripts/query.py --query "{fund.code} {fund.name} {period} 年报 季报 管理人展望 投资策略 运作分析"
```

**关键参数**:
- **不传 `data_type`**: 使用默认值(通常是 `all`),最大化召回范围
- query 中包含多个同义词("管理人展望"/"投资策略"/"运作分析"),提高匹配率

### 2.2 结果解析

```python
result = call_neodata(query)

if result.docData and result.docData.docRecall:
    for doc in result.docData.docRecall[0].docList:
        content = doc.content

        # 匹配目标段落
        if is_target_section(content):
            return OutlookText(
                raw_text=extract_section(content),
                source="neodata docData",
                confidence="中",  # 非保证是一手原文,可能为转载
                extraction_level=1
            )
```

### 2.3 目标段落识别(正则/关键词匹配)

**优先匹配的段落标题**(按优先级排序):

| 优先级 | 标题关键词 | 典型出现位置 |
|---|---|---|
| 1 | "管理人展望" / "后市展望" | 年报/季报末尾 |
| 2 | "投资策略" / "报告期内基金投资策略" | 季报中部 |
| 3 | "运作分析" / "报告期内基金的投资策略和业绩表现" | 季报前半部分 |
| 4 | "基金经理展望" / "后市操作策略" | 各类报告 |

**匹配逻辑**:
```python
SECTION_PATTERNS = [
    r'(?:管理人|基金经理)?(?:后市)?展望[：:\s]*(.{200,1000})',
    r'投资策略[：:\s]*(.{200,1000})',
    r'运作分析[：:\s]*(.{200,1000})',
    r'报告期内基金的投资策略[^。\n]{0,50}[：:\s]*(.{200,1000})',
]

def is_target_section(content: str) -> bool:
    return any(re.search(p, content) for p in SECTION_PATTERNS)


def extract_section(content: str) -> str:
    """提取第一个匹配段落的正文"""
    for pattern in SECTION_PATTERNS:
        match = re.search(pattern, content, re.DOTALL)
        if match:
            return clean_text(match.group(1))
    return None


def clean_text(text: str) -> str:
    """清洗提取文本:去多余空白/HTML标签/特殊字符"""
    text = re.sub(r'<[^>]+>', '', text)  # 去 HTML 标签
    text = re.sub(r'\s+', ' ', text)      # 合并空白
    return text.strip()
```

### 2.4 L1 结果判断

| 情况 | 处理 |
|---|---|
| 召回非空且命中目标段落 | ✅ 返回结果(confidence="中") |
| 召回非空但未命中目标段落 | ⚠️ 尝试全文返回前500字作为候选 → 仍需人工确认是否为观点段落 |
| **召回为空**(最常见) | ❌ 降级到 Level 2 |

---

## 三、Level 2 — announcement-search 公告搜索(问财直达) ★新增

### 3.1 能力说明

`announcement-search` 是同花顺问财的公告搜索技能,可直接搜索基金的**季报、年报、分红、公告**等原始文档。
相比 WebSearch 的通用搜索,`announcement-search` 的优势:
- 🎯 **精准直达**：专为金融公告设计,直达基金季报/年报原文
- 🚀 **速度快**：结构化 API,无需爬取页面(一般 2-5 秒返回)
- 📦 **内容完整**：返回公告标题、摘要、发布时间、原文链接

### 3.2 调用方式

```bash
# 技能路径
SKILL_PATH="$SKILL_DIR/../announcement-search"
PYTHON="$PYTHON_PATH"

# 搜索基金季报/年报公告
cd "$SKILL_PATH/scripts" && $PYTHON __main__.py "{fund.code} {fund.name} {period} 季报" --limit 5
cd "$SKILL_PATH/scripts" && $PYTHON __main__.py "{fund.code} {fund.name} 年报 管理人展望" --limit 5
```

### 3.3 搜索词设计

```python
# 搜索基金季报
query = f"{fund.code} {fund.name} {period} 季报"

# 搜索年报(含管理人展望)
query = f"{fund.code} {fund.name} 年报 管理人展望"

# 搜索具体标题(更精确)
query = f"{fund.code} {fund.name} 年度报告 投资策略 运作分析"

# 搜索公告(经理相关)
query = f"{fund.name} 基金经理 任职 离职"

# 搜索重仓股公告(持仓分析补充)
query = f"{stock_name} 业绩预告 分红"
```

### 3.4 结果解析

```python
import json, subprocess

def call_announcement_search(fund_code, fund_name, period, limit=5):
    """调用 announcement-search 搜索基金公告"""

    skill_path = "$SKILL_DIR/../announcement-search"
    python = "$PYTHON_PATH"

    query = f"{fund_code} {fund_name} {period} 季报 管理人展望 投资策略 运作分析"

    cmd = f'cd "{skill_path}/scripts" && {python} __main__.py "{query}" --limit {limit} --format json'
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=30)

    if result.returncode != 0:
        return None

    try:
        data = json.loads(result.stdout)
    except json.JSONDecodeError:
        return None

    # 提取公告条目
    announcements = data.get("data", []) if isinstance(data, dict) else data
    return announcements


def extract_outlook_from_announcements(announcements, fund_code, fund_name, period):
    """从公告搜索结果中提取观点原文"""

    for ann in announcements:
        title = ann.get("title", "")
        summary = ann.get("summary", "")
        content = title + " " + summary

        # 检查是否为目标季报/年报
        if not any(kw in title for kw in ["季报", "年报", "中期报告", "年度报告"]):
            continue

        # 从摘要中提取观点段落
        outlook_text = extract_section(content)
        if outlook_text:
            return OutlookText(
                fund_code=fund_code,
                fund_name=fund_name,
                report_period=period,
                section_type="管理人展望/投资策略",
                raw_text=outlook_text,
                source=f"announcement-search:{ann.get('url', '')}",
                source_url=ann.get("url", ""),
                confidence="高",
                extraction_level=2
            )

    return None
```

### 3.5 L2 结果判断

| 情况 | 处理 |
|---|---|
| 公告命中且摘要含目标段落 | ✅ 返回结果(confidence="高",结构化公告数据) |
| 公告命中但摘要不含目标段落 | ⚠️ 返回标题+URL,供 WebFetch 抓取全文 |
| **公告搜索无结果** | ⚠️ 先尝试天天基金网公告页(fundf10.eastmoney.com/jjgg_{code}_3.html),仍无则降级到 Level 3 |

### 3.6 L2 备选:天天基金网公告页

> **适用场景**: `announcement-search` 未安装或查询失败时,直接访问天天基金网的基金公告列表页。

**URL模板**: `https://fundf10.eastmoney.com/jjgg_{基金代码}_3.html`

**调用方式**:
```python
import urllib.request

url = f"https://fundf10.eastmoney.com/jjgg_{fund_code}_3.html"
web_fetch(url, prompt="提取最新的基金季报/年报链接,标题包含'季度报告'或'年度报告'。列出标题和URL。")
```

**结果判断**:
| 情况 | 处理 |
|---|---|
| 页面返回季报链接 | ✅ 记录 URL,跳转 Level 4(WebFetch)抓取全文 |
| 页面无季报链接 | ❌ 降级到 Level 3(WebSearch) |

---

## 四、Level 3 — WebSearch 搜索

### 4.1 调用方式

```python
WebSearch(
    query=f"{fund.code} {fund.name} {period} 季报 管理人展望 投资策略",
    # 不限制 allowed_domains (用户确认不限制)
)
```

**搜索词设计原则**:
- 包含**基金代码**(精确匹配)
- 包含**基金名称**(辅助匹配)
- 包含**报告期**(如"2025年第四季度"或"2025Q4")
- 包含**多个同义词**(提高召回率)

### 3.2 结果筛选与优先级排序

**优先访问的网站**(经验证通常有季报原文):

| 优先级 | 网站 | URL 特征 | 说明 |
|---|---|---|---|
| ⭐⭐⭐ | 天天基金网 | `funds.eastmoney.com` | 最权威,结构化最好 |
| ⭐⭐⭐ | 巨潮资讯网 | `cninfo.com.cn` | 监管披露源头 |
| ⭐⭐ | 雪球 | `xueqiu.com` | 用户讨论多,可能有转载 |
| ⭐⭐ | 和讯网 | `hexun.com` | 老牌财经媒体 |
| ⭐ | 其他财经媒体 | — | 作为补充 |

**排序逻辑**:
```python
def rank_search_results(results: List[ SearchResult]) -> List[SearchResult]:
    PRIORITY_DOMAINS = {
        'funds.eastmoney.com': 5,
        'cninfo.com.cn': 5,
        'xueqiu.com': 3,
        'hexun.com': 3,
    }

    def score(result):
        domain = extract_domain(result.url)
        return PRIORITY_DOMAINS.get(domain, 1)  # 默认优先级 1

    return sorted(results, key=score, reverse=True)
```

### 3.3 L2 结果判断

| 情况 | 处理 |
|---|---|
| 搜索结果非空且有高优先级域名 | ✅ 取 Top 1-3 个 URL 进入 Level 3 |
| 搜索结果非空但都是低质量来源 | ⚠️ 仍然取 Top 1 尝试 Level 3 |
| **搜索结果为空** | ❌ 降级到 Level 4(跳过 L3) |

---

## 五、Level 4 — WebFetch 抓取一手原文

### 5.1 调用方式

```python
for url in top_urls[:3]:  # 最多尝试 3 个 URL
    try:
        response = WebFetch(
            url=url,
            prompt="""请从以下基金季报/年报页面中,提取"投资策略"、"管理人展望"、"运作分析"或"后市展望"部分的原文段落。
                   如果找到多个相关段落,请全部提取并标注段落标题。
                   请保留原文,不要总结或改写。"""
        )

        if response and len(response.strip()) > 50:  # 至少 50 字才视为有效
            return OutlookText(
                raw_text=response,
                source=f"WebFetch:{url}",
                source_url=url,
                confidence="高",  # 一手原文,最可信
                extraction_level=3
            )
    except Exception as e:
        logger.warning(f"WebFetch {url} 失败: {e}")
        continue
```

### 4.2 成功标准

| 条件 | 标准 |
|---|---|
| 最小长度 | ≥ 50 字符(避免抓到无关碎片) |
| 最大长度 | ≤ 3000 字符(避免抓到整页) |
| 关键词命中 | 文本中含至少 1 个目标词汇(展望/策略/分析/看好/配置) |
| 格式检查 | 不含大量 HTML 标签(< 占比 < 10%) |

### 4.3 L3 结果判断

| 情况 | 处理 |
|---|---|
| 至少 1 个 URL 成功抓取到有效段落 | ✅ 返回最长/最完整的那个 |
| 所有 URL 都失败或内容太短 | ❌ 降级到 Level 4 |

---

## 六、Level 5 — 标记缺失(最终兜底)

### 5.1 输出

```python
return OutlookText(
    fund_code=fund.code,
    fund_name=fund.name,
    report_period=period,
    raw_text=None,
    source="缺失",
    source_url=None,
    confidence="无",
    extraction_level=4,
    attitude=None,
    keywords_extracted=[]
)
```

### 5.2 不阻断原则

**核心规则**:L4 失败**绝不阻断**后续流程!

```python
outlook = extract_outlook(fund, period)  # 可能返回 L4 缺失

# 继续执行持仓对比和报告生成,不受影响
changes = compare_periods(holdings_snapshots, [outlook])
report = format_report(manager_profile, benchmark, holdings, changes, outlook)

# 在报告中显式标注缺失
# report 将包含: "⚠️ 观点原文未获取(季报原文非 neodata 直覆盖)"
```

---

## 七、态度判断词典(Attitude Dictionary)

### 6.1 三分类体系

将经理观点态度分为三类:**乐观(Optimistic)** / **中性(Neutral)** / **谨慎(Cautious)**

### 6.2 乐观关键词(权重 +1)

| 类别 | 关键词 | 权重 |
|---|---|---|
| 正面情绪 | 看好、积极、乐观、信心、期待 | +1.0 |
| 行动导向 | 布局、加仓、增持、重点配置、超配 | +0.8 |
| 机会描述 | 机会、景气、成长性、空间、潜力 | +0.7 |
| 时间维度 | 长期看好、持续关注、逢低吸纳 | +0.6 |

**示例句子与得分**:
- "我们**长期看好**消费升级带来的投资机会" → +1.0(long 看好) +0.7(机会) = **+1.7**
- "当前是**积极布局**科技板块的良好时机" → +0.8(布局) +0.8(积极) = **+1.6**

### 6.3 谨慎关键词(权重 -1)

| 类别 | 关键词 | 权重 |
|---|---|---|
| 负面情绪 | 谨慎、观望、担忧、风险、不确定性 | -1.0 |
| 行动导向 | 减仓、减持、规避、控制仓位、防御 | -0.8 |
| 风险描述 | 高估、泡沫、回调、波动、压力 | -0.7 |
| 时间维度 | 短期承压、等待时机、保持耐心 | -0.6 |

**示例句子与得分**:
- "对市场保持**谨慎**,**规避**高估值品种" → -1.0(谨慎) -0.8(规避) = **-1.8**
- "短期面临一定**回调压力**,**控制仓位**应对" → -0.7(回调) -0.8(控制仓位) = **-1.5**

### 6.4 中性关键词(权重 0)

| 关键词 | 示例 |
|---|---|
| 平衡表述 | 稳健、平衡、维持、中性、震荡 |
| 观望表述 | 密切关注、跟踪观察、动态调整 |
| 结构性机会 | 结构性机会、分化、精选个股 |

### 6.5 态度计算算法

```python
def judge_attitude(text: str) -> Tuple[str, float]:
    """
    返回: (attitude_label, score)
    score 范围: -1.0 ~ 1.0
    """
    if not text:
        return (None, 0.0)

    score = 0.0
    matched_keywords = []

    # 计算乐观分
    for word, weight in OPTIMISTIC_WORDS.items():
        count = text.count(word)
        if count > 0:
            score += weight * min(count, 3)  # 单词最多计 3 次,防重复堆砌
            matched_keywords.append((word, weight))

    # 计算谨慎分
    for word, weight in CAUTIOUS_WORDS.items():
        count = text.count(word)
        if count > 0:
            score += weight * min(count, 3)
            matched_keywords.append((word, weight))

    # 归一化到 [-1, 1]
    score = max(-1.0, min(1.0, score / 3.0))  # 除以 3 使分布合理

    # 判定类别
    if score > 0.2:
        attitude = "乐观"
    elif score < -0.2:
        attitude = "谨慎"
    else:
        attitude = "中性"

    return (attitude, score)
```

### 6.6 态度阈值说明

| score 范围 | 态度 | 含义 |
|---|---|---|
| > 0.2 | **乐观** | 明确看多,积极配置 |
| -0.2 ~ 0.2 | **中性** | 平衡看待,无明显倾向 |
| < -0.2 | **谨慎** | 控制风险,防御为主 |

**±0.2 阈值的理由**:避免因个别中性词导致频繁切换,要求明确信号才判定转变。

---

## 八、关键词提取(Keyword Extraction)

### 7.1 提取目标

从观点原文中提取**行业/板块/主题**相关的关键词,用于跨期 diff 对比:

**提取类别**:

| 类别 | 示例关键词 | 用途 |
|---|---|---|
| 行业 | 消费、医药、新能源、半导体、军工、银行... | 关注方向变化 |
| 板块 | 港股、科创板、创业板、北交所... | 配置区域变化 |
| 主题 | AI、碳中和、ESG、国产替代、出海... | 投资主题变化 |
| 因子 | 价值、成长、质量、动量、红利... | 投资风格变化 |

### 7.2 提取方法(基于词典匹配)

```python
KEYWORD_DICT = {
    "行业": ["消费", "医药", "新能源", "半导体", "军工", "银行", "地产", "传媒",
             "农业", "化工", "机械", "汽车", "家电", "食品饮料", "纺织服装"],
    "板块": ["港股", "A股", "科创板", "创业板", "北交所", "中概股"],
    "主题": ["AI", "人工智能", "碳中和", "ESG", "国产替代", "出海", "数字经济",
             "新基建", "元宇宙", "量子科技"],
    "因子": ["价值", "成长", "质量", "动量", "红利", "低波", "小盘"]
}

def extract_keywords(text: str) -> Set[str]:
    """提取所有命中的关键词"""
    keywords = set()
    for category, words in KEYWORD_DICT.items():
        for word in words:
            if word in text:
                keywords.add(word)
    return keywords
```

---

## 九、完整调用示例

### 场景:获取易方达蓝筹精选混合(005827)2025Q4 的观点

```python
fund = FundInfo(code="005827", name="易方达蓝筹精选混合")
period = "2025Q4"

# === Level 1: neodata docData ===
outlook_l1 = try_neodata_docdata(fund, period)
if outlook_l1:
    print(f"L1 成功: source={outlook_l1.source}, length={len(outlook_l1.raw_text)}")
    final_outlook = outlook_l1
else:
    print("L1 未命中,降级到 L2")

    # === Level 2: WebSearch ===
    search_results = websearch_outlook(fund, period)
    if search_results:
        top_urls = [r.url for r in search_results[:3]]
        print(f"L2 找到 {len(top_urls)} 个候选URL")

        # === Level 3: WebFetch ===
        for url in top_urls:
            outlook_l3 = try_webfetch(url)
            if outlook_l3:
                print(f"L3 成功: source={outlook_l3.source}")
                final_outlook = outlook_l3
                break
        else:
            print("L3 全部失败")
            final_outlook = create_missing_outlook(fund, period)  # L4
    else:
        print("L2 也未找到")
        final_outlook = create_missing_outlook(fund, period)  # L4

# 最终输出
print(f"\n=== 最终结果 ===")
print(f"source: {final_outlook.source}")
print(f"confidence: {final_outlook.confidence}")
print(f"attitude: {final_outlook.attitude}")
print(f"keywords: {final_outlook.keywords_extracted}")
if final_outlook.raw_text:
    print(f"text preview: {final_outlook.raw_text[:100]}...")
else:
    print("text: ⚠️ 缺失")
```

---

## 十、性能考量

| Level | 预计耗时 | 并发安全 | 使用频率 |
|---|---|---|---|
| L1 neodata | 2-5 秒 | ✅ 安全(内部 API) | **每次必试** |
| L2 announcement-search | 2-5 秒 | ✅ 安全(结构化 API) | 仅在 L1 失败时,优于旧 WebSearch |
| L3 WebSearch | 3-8 秒 | ⚠️ 有频率限制 | 仅在 L1+L2 失败时 |
| L4 WebFetch | 5-15 秒/URL | ⚠️ 有频率限制 | 仅在 L3 命中时,最多 3 次 |
| L5 标记缺失 | <0.1 秒 | ✅ 安全 | 兜底 |

**批量场景优化建议**:
- 30 位经理 × 平均每经理 2 期观点 = 60 次提取
- 若 L1 命中率 20%, L2 命中率 40%, 则需约 15 次 L3+L4
- 预计总耗时: 60×3s + 24×4s + 15×10s ≈ **7-12 分钟**(相比旧方案减少约 30%)
- **建议**:可在报告中先输出持仓部分(已就绪),观点异步加载后补充

---

## 十一、最佳实践提示

1. **不要过度依赖观点原文**:持仓数据才是硬指标,观点只是辅助参考
2. **置信度分层使用**:
   - 高(L3):可直接采信用于决策
   - 中(L1/L2):需结合持仓变动交叉验证
   - 无(L4):仅参考持仓,忽略观点
3. **态度判断仅供参考**:词典匹配可能误判(如"谨慎看好"这种矛盾表达),最终以人工阅读为准
4. **季度间对比最有价值**:单期观点意义有限,连续几期的**态度转变**才值得关注
5. **保存原始 URL**:即使本次未成功,下次可复用(缓存思路)
