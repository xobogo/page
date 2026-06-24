---
name: concrete-bid-hunter
description: 商砼猎标大脑 - AI驱动的商砼（混凝土）行业投标决策系统。为搅拌站老板提供区域商机雷达、竞对火力分析、利润精算、大客户画像、财务风险健康码五大模块的智能分析报告。当用户提到"商砼"、"混凝土"、"搅拌站"、"投标分析"、"商机雷达"、"标讯"、"猎标"等关键词，或需要分析招投标数据、评估项目利润、分析竞争对手时触发此技能。
agent_created: true
---

# 商砼猎标大脑 - AI驱动的商砼行业投标决策系统

> **一句话定位**：不只是告诉你"有什么标"，而是告诉你"能不能接、能赚多少、谁在和你抢、甲方靠不靠谱"。

---

## 一、首次运行：初始化引导

### 第1步：检测用户数据目录

```
检查 ~/.concrete-bid-hunter/ 是否存在
  ├── 不存在 → 创建目录结构，进入引导流程
  └── 已存在 → 检测依赖变更，跳到第2步
```

创建目录（如不存在）：
```bash
mkdir -p ~/.concrete-bid-hunter/{logs,output}
```

### 第2步：依赖检查（逐项，不中断）

#### P0 - 知了标讯（硬依赖，缺失则阻断）

```
检查 tender-search skill 是否已安装：
  检查 ~/.workbuddy/skills/tender-search/SKILL.md 是否存在
    ├── 不存在 → ❌ 阻断，提示用户安装 tender-search skill
    │   提示文案：「本技能依赖「知了标讯」skill，请先安装。安装后在当前对话重新运行即可。」
    └── 存在 → 检查 API Key
        检查 ~/.zlbx/config.json 中是否有 api_key
          ├── 有 → ✅ 记录为可用
          └── 没有 → 提示运行 tender-search skill 一次完成自动注册
              提示文案：「知了标讯 API Key 尚未配置，请先在对话中运行一次知了标讯查询，它会自动注册。完成后回来继续。」
```

#### P1 - 企查查 MCP（可选，降级运行）

```
检查 ~/.workbuddy/mcp.json 中是否包含 qcc-company 配置
  ├── 有 → ✅ 记录为可用
  └── 没有 → ⚠️ 标记不可用
      提示文案：「企查查未配置，企业风控分析功能将不可用。如需启用，请在 WorkBuddy 设置中添加企查查 MCP 连接器。当前可正常使用其他功能。」
```

#### P2 - 高德地图 API（可选，降级运行）

```
读取 ~/.concrete-bid-hunter/config.json 中的 amap_api_key
  ├── 有值 → ✅ 记录为可用
  └── 无值/null → 引导用户配置
      提示文案：「高德地图 API Key 用于精确定位项目位置和运输半径过滤。
      免费申请步骤：
      1. 访问 https://console.amap.com/dev/key/app
      2. 创建应用 → 添加 Key → 服务平台选「Web服务」
      3. 每天免费5万次调用
      请输入你的高德 API Key（输入「跳过」则使用省市粗匹配）：」
      用户输入后 → 写入 config.json
      用户跳过 → 标记为未配置
```

### 第3步：搅拌站建档

```
读取 ~/.concrete-bid-hunter/company.json
  ├── 不存在 → 进入建档引导
  └── 已存在 → 列出已有站点，选择分析目标

建档引导（一次性收集）：
  A. 公司信息（必填一次）：
     1. 公司/搅拌站品牌名称（必填，如「深圳市XX混凝土有限公司」）
     2. 所在省份（必填）
     3. 所在城市（必填）
     4. 已知竞争对手公司名（逗号分隔，可选）
     5. 已知合作客户公司名（逗号分隔，可选）

  B. 搅拌站信息（至少填1个，可后续新增）：
     1. 站点名称/别名（如「宝安站」、「1号站」）
     2. 站点详细地址（必填，如「广东省深圳市宝安区XX路XX号」）
     3. 运输半径（公里，默认60，可按站不同）
     4. 主营混凝土标号（逗号分隔，默认 C25,C30,C35）

  后续可随时：添加新站点 / 修改已有站点 / 添加竞对和客户
```

保存 company.json 示例：
```json
{
  "name": "深圳市XX混凝土有限公司",
  "province": "广东",
  "city": "深圳",
  "competitors": ["深圳市YY建材有限公司", "深圳市ZZ商砼有限公司"],
  "major_clients": ["中国建筑", "万科企业"],
  "default_transport_radius_km": 60,
  "default_grades": ["C25", "C30", "C35"],
  "created_at": "2026-05-15",
  "updated_at": "2026-05-15"
}
```

保存 ~/.concrete-bid-hunter/stations/宝安站.json 示例：
```json
{
  "name": "宝安站",
  "address": "广东省深圳市宝安区XX路XX号",
  "coordinates": { "lng": 113.88, "lat": 22.55 },
  "transport_radius_km": 60,
  "main_grades": ["C25", "C30", "C35"],
  "created_at": "2026-05-15"
}
```

### 第4步：保存全局配置

完成引导后，保存 `~/.concrete-bid-hunter/config.json`：
```json
{
  "version": "1.0",
  "dependencies": {
    "tender_search": { "available": true },
    "qcc": { "available": false, "detected_from_mcp": false },
    "amap": { "api_key": "用户的Key或null", "configured_at": "2026-05-15T14:00:00" }
  },
  "created_at": "2026-05-15T14:00:00"
}
```

---

## 二、日志记录（每次API调用必须记录）

### 日志目录结构

```
~/.concrete-bid-hunter/logs/
└── {YYYY-MM-DD}/
    └── {NNN}_{source}.log
```

- `{NNN}` = 当天第几次运行（001起始），每次会话开始时自动递增
- `{source}` = 数据源标识：`tender-search` / `qcc-company` / `qcc-risk` / `amap` / `websearch`

### 日志格式（追加写入）

```json
{
  "timestamp": "2026-05-15T14:30:25",
  "source": "tender-search",
  "api": "query_bids_advanced",
  "request_summary": {"keywords": ["混凝土"], "provinces": ["广东"]},
  "response_status": "success",
  "result_count": 23,
  "cost_units": 1,
  "duration_ms": 256
}
```

### 运行序号管理

每次会话开始分析时：
```bash
# 查找今天的最新序号
today=$(date +%Y-%m-%d)
log_dir=~/.concrete-bid-hunter/logs/$today
mkdir -p "$log_dir"
last_num=$(ls "$log_dir" 2>/dev/null | head -1 | grep -o '^[0-9]*')
if [ -z "$last_num" ]; then
  run_num=001
else
  run_num=$(printf "%03d" $((10#$last_num + 1)))
fi
echo "$run_num"  # 本次运行的序号
```

---

## 三、产物归档

### 输出目录结构

```
~/.concrete-bid-hunter/output/
└── {YYYY-MM-DD}/
    └── {NNN}_{模块名}.html
```

报告文件命名：`{序号}_{模块名}.html`
- 模块名：`区域商机雷达` / `竞对火力分析` / `利润精算` / `大客户画像` / `财务风险健康码` / `综合日报`
- 综合日报包含所有模块的合并报告

### 产物复制到 workspace

在 WorkBuddy 环境运行时，将生成的 HTML 同时复制到当前 workspace 目录供 `preview_url` 预览：
```bash
cp ~/.concrete-bid-hunter/output/2026-05-15/001_区域商机雷达.html ./区域商机雷达.html
```

---

## 四、五大分析模块

### 模块1：区域商机雷达

**目标**：找到搅拌站运输半径内、近期发布的混凝土相关标讯，按价值排序。

**执行流程**：

1. **读取配置**：加载 `company.json` 获取省份、城市，加载 `stations/` 下目标站点配置获取运输半径
2. **搜索标讯**：调用知了标讯 API
   ```
   POST https://mcp-server.zhiliaobiaoxun.com/api_v2/query_bids_advanced
   {
     "keywords": ["混凝土", "商砼", "砼"],
     "keyword_groups": [
       {
         "keywords": ["浇筑", "泵送"],
         "match_modes": ["sm", "title"]
       }
     ],
     "exclude_keywords": ["维修", "维保", "清洗", "检测", "试验"],
     "match_modes": ["sm", "title"],
     "provinces": ["{station.province}"],
     "begin_date": "{30天前}",
     "bid_process": [1, 2, 4],
     "page_size": 50
   }
   ```
3. **地理过滤**（如有高德 Key）：
   - 从标讯标题/详情中提取地名
   - 调用高德地理编码获取经纬度
   - 计算与搅拌站距离
   - 标记：可达 / 边缘 / 不可达
4. **AI分析**：对每条标讯进行分析
   - 项目类型识别（住宅/道路/桥梁/地铁...）
   - 估算方量（使用 `project_types.json` 参数）
   - 估算金额
   - 可行性评估
5. **排序输出**：按综合价值（方量 × 匹配度 × 可达性）降序
6. **生成报告**：使用 `report.html` 模板

**输出**：标注"高价值/中价值/一般"的商机列表，每条附带方量估算和AI建议。

---

### 模块2：竞对火力分析

**目标**：分析竞争对手的中标动态、实力评估和竞争策略。

**执行流程**：

1. **读取竞对列表**：从 `company.json` 的 `competitors` 字段获取
2. **自动发现竞对**（如果列表为空）：
   - 搜索同区域混凝土中标单位
   ```
   POST /api_v2/get_top_suppliers
   {
     "keywords": ["混凝土"],
     "provinces": ["{station.province}"],
     "begin_date": "{90天前}"
   }
   ```
   - 排除自身公司名
3. **逐个分析竞对**（每个竞对调用以下API）：
   ```
   # 中标记录
   POST /api_v2/search_bids
   {
     "keywords": ["{竞对公司名}"],
     "match_modes": ["winner"],
     "begin_date": "{180天前}",
     "page_size": 20
   }

   # 竞争对手分析
   POST /api_v2/find_competitors
   {"company": "{竞对公司名}"}

   # 合作伙伴
   POST /api_v2/get_company_partners
   {"company": "{竞对公司名}"}
   ```
4. **火力值评估**：
   - 近半年中标次数
   - 中标总金额
   - 活跃区域重叠度
   - 客户重叠度
5. **生成报告**：火力值排名 + 竞争策略建议

**输出**：竞对火力值排名表、近期中标动态、竞争策略建议（含围标串标预警）。

---

### 模块3：利润精算

**目标**：对特定标讯项目进行成本-利润测算。

**触发**：用户指定某个标讯/项目，或对商机雷达中的高价值项目自动测算。

**执行流程**：

1. **获取标讯详情**：
   ```
   POST /api_v2/get_bid_detail
   {"bid_url": "{标讯URL}"}
   ```
2. **运行利润计算脚本**：
   ```bash
   python3 {SKILL_DIR}/scripts/profit_calc.py
   ```
   - 加载 `concrete_grades.json`（配比表）
   - 加载 `material_prices.json`（原材料价格）
   - 加载 `station.json`（自定义配比，如有）
3. **AI提取关键参数**（从标讯详情中）：
   - 混凝土标号要求
   - 预估方量
   - 投标预算/控制价
   - 付款条件
   - 工期
4. **计算链路**：
   ```
   标号 → 配比 → 原材料用量 → 材料成本
   → + 运输费 + 泵送费 + 人工管理 + 设备折旧
   → 总成本/m³ → 投标价 - 总成本 = 利润
   → 利润率 → 利润等级
   ```
5. **生成报告**：成本结构图 + 利润率 + 盈亏平衡分析

**输出**：每m³成本明细（原材料/运输/泵送/人工/折旧）、利润率、总利润估算。

---

### 模块4：大客户画像

**目标**：分析潜在甲方/大客户的采购规律和信用状况。

**执行流程**：

1. **读取大客户列表**：从 `company.json` 的 `major_clients` 获取
2. **逐个分析客户**：
   ```
   # 招标历史
   POST /api_v2/search_bids
   {
     "keywords": ["{客户名}"],
     "match_modes": ["caller"],
     "begin_date": "{365天前}",
     "page_size": 30
   }

   # 中标供应商（了解客户常用谁）
   POST /api_v2/search_bids
   {
     "keywords": ["{客户名}"],
     "match_modes": ["caller"],
     "keyword_groups": [{"keywords": ["混凝土", "商砼"], "match_modes": ["sm"]}],
     "bid_process": [7, 8],
     "page_size": 20
   }

   # 企业画像（如企查查可用）
   通过 qcc-company MCP 获取企业工商信息
   ```
3. **客户画像维度**：
   - 年度混凝土采购量（从招标记录估算）
   - 采购频率和周期
   - 常用供应商（竞对渗透情况）
   - 企业性质和实力
   - 信用评估（企查查数据）
4. **生成报告**：客户价值评级 + 渗透策略

**输出**：大客户采购力排名、年度采购量估算、竞对渗透情况、信用评估。

---

### 模块5：财务风险健康码

**目标**：对甲方/潜在客户进行回款风险评估。

**执行流程**：

1. **数据采集**：
   - 如企查查可用：通过 `qcc-company` MCP 获取企业信息，通过 `qcc-risk` MCP 获取风险信息
   - 如不可用：从知了标讯的中标记录中提取线索（中标方变更频繁、金额异常等）
2. **运行风险评分脚本**：
   ```bash
   python3 {SKILL_DIR}/scripts/risk_scorer.py
   ```
3. **评分模型**：

| 维度 | 权重 | 数据来源 |
|------|------|---------|
| 企业性质（央企/国企/民企） | 25% | 企查查 / 标讯中标方 |
| 经营年限 | 15% | 企查查成立日期 |
| 被执行人记录 | 20% | 企查查风险信息 |
| 法律诉讼（被告） | 20% | 企查查风险信息 |
| 注册资本 | 10% | 企查查工商信息 |
| 行政处罚 | 10% | 企查查风险信息 |

4. **健康码等级**：
   - 🟢 绿码（80-100分）：回款风险低
   - 🟡 黄码（60-79分）：有一定风险
   - 🟠 橙码（40-59分）：风险较高
   - 🔴 红码（0-39分）：风险极高

5. **生成报告**：健康码 + 维度雷达图 + 风险标签 + 应对建议

---

## 五、运行时控制

### API调用顺序（成本优先）

每次分析按以下顺序执行，低成本调用在前：

1. 本地计算（利润测算、风险评分）→ 0成本
2. 知了标讯搜索类 API → 1单位/次
3. 知了标讯企业分析 API → 1单位/次
4. 高德地理编码 → 免费
5. 企查查 MCP → 免费

### 容错规则

- **单个API失败不中断流程**：记录错误日志，继续执行后续步骤，缺失数据用 AI 推理或标注"数据不足"
- **余额不足**：暂停API调用，使用已有数据生成报告，提示用户充值
- **网络超时**：重试1次，仍失败则跳过

### 消耗估算

单次完整分析（5模块）约消耗 15-20 单位知了标讯API额度。

---

## 六、HTML报告生成规则

### 报告结构

使用 `templates/report.html` 作为基础模板，AI根据分析结果动态填充内容区域。

### 移动端设计原则

- **宽度**：最大480px
- **首屏**：金句式摘要（3秒内看到核心结论）
- **信息密度**：一屏一个模块，滑动阅读
- **数据可视化**：用CSS实现的条形图、进度条，不依赖外部图表库

### 首屏金句示例

```
┌──────────────────────────────┐
│  深圳XX混凝土有限公司         │
│  2026年5月15日                │
│                              │
│  今日发现12条相关标讯         │
│  3条高价值商机等你抢           │
│  预估总方量8.5万m³           │
│                              │
│  ┌──────┐┌──────┐┌──────┐   │
│  │  12  ││   3  ││ 8.5万│   │
│  │相关标讯││高价值 ││总方量 │   │
│  └──────┘└──────┘└──────┘   │
└──────────────────────────────┘
```

### 报告文件命名

```
{序号}_{模块名}.html
例：001_区域商机雷达.html
```

---

## 七、配置与数据文件说明

### Skill 目录（可分享，不含密钥）

```
concrete-bid-hunter/
├── SKILL.md                          # 本文件
├── references/
│   ├── concrete_grades.json          # 混凝土标号配比表
│   ├── project_types.json            # 项目类型方量估算参数
│   └── material_prices.json          # 原材料参考价格
├── scripts/
│   ├── geo_filter.py                 # 地理过滤工具
│   ├── profit_calc.py                # 利润精算引擎
│   └── risk_scorer.py                # 风险评分引擎
└── templates/
    └── report.html                   # HTML报告模板
```

### 用户数据目录（不可分享，含密钥）

```
~/.concrete-bid-hunter/
├── config.json                       # 全局配置（API Keys + 依赖状态）
├── company.json                      # 公司信息（名称、竞对、大客户）
├── stations/                         # 搅拌站配置（每个站一个文件）
│   ├── 宝安站.json
│   ├── 龙岗站.json
│   └── 东莞站.json
├── logs/
│   └── {YYYY-MM-DD}/
│       └── {NNN}_{source}.log        # API调用日志
└── output/
    └── {YYYY-MM-DD}/
        └── {NNN}_{站点名}_{模块名}.html  # 分析报告
```

### 自定义配比（可选高级配置）

用户可在具体站点的 JSON 文件中添加 `custom_mixes` 字段覆盖默认配比：
```json
{
  "name": "宝安站",
  "address": "...",
  "custom_mixes": {
    "C30": {
      "cement_kg_per_m3": 370,
      "water_kg_per_m3": 170,
      "sand_kg_per_m3": 630,
      "gravel_kg_per_m3": 1180
    }
  }
}
```

---

## 八、用户指令处理

### 可用指令

| 用户说 | 执行 |
|--------|------|
| "分析一下"、"跑一次"、"今日商机" | 执行综合分析（全部5模块） |
| "看看附近有什么标"、"区域雷达" | 仅执行模块1 |
| "分析一下XX公司"、"竞对分析" | 仅执行模块2 |
| "这个项目能不能接"、"利润测算" | 仅执行模块3，需指定标讯 |
| "查一下XX公司"、"大客户画像" | 仅执行模块4 |
| "XX公司靠不靠谱"、"风险评估" | 仅执行模块5 |
| "修改站点信息"、"更新配置" | 进入配置编辑流程 |

### 对话流程

1. 用户发出指令
2. 执行初始化检查（如首次运行）
3. 确认分析范围和参数
4. 按模块顺序执行分析
5. 生成 HTML 报告
6. 归档到 output 目录 + 复制到 workspace
7. 展示报告 + 总结关键发现
8. 提供后续行动建议

---

## 九、安全与隐私

- API Key 仅存储在 `~/.concrete-bid-hunter/config.json`，不出现在日志、报告或任何输出文件中
- Skill 目录可自由拷贝分享，不含任何用户私有数据
- 每个用户各自独立的 `~/.concrete-bid-hunter/`，互不干扰
- 日志中记录 API 调用详情，但不记录请求体中的完整参数（仅记录摘要）
