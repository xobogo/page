---
name: wx-emailsend-pro
description: |-
  传统企业邮件营销助手。根据客户数据（Excel/CSV/乐享知识库/MCP 服务）和邮件模板，
  批量生成个性化邮件内容，并通过 SMTP 或 MCP 服务自动群发。支持模板学习（占位符
  智能匹配与中性替换）、国籍语言自动翻译、发送频率控制、发送结果统计与报告生成。
  触发词：客户邮件发送、老客户邮件发送、邮件营销、批量邮件、群发邮件。
agent_created: true
---

# WX EmailSend Pro — 企业邮件营销助手

传统企业批量邮件发送工具。从数据源读取客户信息 → 匹配模板渲染个性化邮件 →
按频率限制逐封发送 → 生成发送报告。

## 触发条件

当用户提到以下关键词时激活本 skill：
- 客户邮件发送、老客户邮件发送、邮件营销
- 批量邮件、群发邮件、给客户发邮件
- 任何涉及"读取 Excel + 模板 + 批量发邮件"的场景

## 目录结构约定

Skill 运行过程严格遵循三目录分离原则，**禁止跨目录写入**：

```
                     三目录分离模型

  Skill 目录                          配置目录                    中间产物目录
  (只读，仅供引用)                   (仅配置，永不清空)             (运行时产物)
  
  ~/.workbuddy/skills/             ~/.wx-emailsend-pro/         ${workspace}/wx-emailsend-pro/
  wx-emailsend-pro/                └── config.json              {任务标识}/
  ├── SKILL.md                                                  ├── source_customers.json
  ├── scripts/              ← 脚本从此处执行                    ├── email_queue_preview.md
  ├── references/           ← 配置时参考                        ├── template_learning_report.md
  └── assets/               ← 默认配置模板来源                  ├── email_content_queue.md
                                                                 ├── emails/           ← 每封邮件独立 .md
  运行时不在该目录下                                              │   ├── email_001_xx@ex.com.md
  生成任何文件。                                                  │   └── email_002_yy@ex.com.md
                                                                 └── send_results.json

                    最终交付目录
                    (发送清单 + 统计报告)

                    ${workspace}/发送计划/
                    {任务标识}/
                    ├── email_send_report_*.xlsx
                    └── email_send_statistics_*.html
```

| 目录 | 路径 | 用途 | 规则 |
|------|------|------|------|
| **Skill 目录** | `~/.workbuddy/skills/wx-emailsend-pro/` | 存放 SKILL.md + 脚本 + 参考文档 + 默认配置模板 | **只读，运行时不产生任何文件** |
| **配置目录** | `~/.wx-emailsend-pro/` | 存放用户配置文件 | **仅放 config.json，不存放任何运行时产物** |
| **中间产物目录** | `${workspace}/wx-emailsend-pro/{任务标识}/` | 存放单次任务所有中间文件（原始数据、模板学习、邮件队列、发送结果） | **任务结束后保留，供用户复查；用户可手动清理历史任务** |
| **最终交付目录** | `${workspace}/发送计划/{任务标识}/` | 存放最终发送清单（Excel）与统计报告（HTML） | **任务结束后保留，是交付给用户的可读产出** |

> `${workspace}` 指当前 WorkBuddy 工作空间根目录。
>
> `{任务标识}` 格式: `YYYYMMDD_HHMMSS_` + 6 位随机 ID，如 `20250610_143022_a3f2d1`。
>
> 所有脚本通过 `{skill_dir}/scripts/xxx.py` 引用，**绝不**将脚本复制到其他目录执行。

## 配置文件

配置文件位于 `~/.wx-emailsend-pro/config.json`。

### 初始化流程（首次运行或用户显式要求时执行）

初始化是交互式过程，**逐项询问并保存**，不跳过任何一步。

#### 前置检查

1. 检查 `~/.wx-emailsend-pro/config.json` 是否存在
2. 若不存在，创建 `~/.wx-emailsend-pro/` 目录，进入交互式配置流程
3. 若已存在，告知用户"已有配置文件"，询问是"从头重新配置"还是"只修改特定部分"

#### 交互式配置（逐项询问）

**步骤 0: 发件人信息 (sender_info)**

先读取 `references/smtp_configs.md` 了解可用的邮箱服务商信息，然后逐个询问用户：

| 字段 | 询问内容 | 示例 |
|------|----------|------|
| `company_name` | 发件公司名称（显示在邮件底部签名区） | "XX 科技有限公司" |
| `contact_person` | 联系人姓名 | "张三" |
| `contact_phone` | 联系电话 | "138xxxx" |
| `website` | 公司网址（选填） | "www.example.com" |
| `address` | 公司地址（选填） | "深圳市南山区..." |

> 每次收集完 sender_info 后，展示汇总让用户确认。

**步骤 1: 数据源配置 (data_source)**

询问并解释选项：

| 字段 | 询问内容 | 选项说明 |
|------|----------|----------|
| `type` | 数据源类型？ | `local_excel`（本地 Excel/CSV 文件）/ `lekai_kb`（乐享知识库）/ `mcp_service`（MCP 服务） |
| `source_name` | 若选 `lekai_kb`，知识库名称？若选 `mcp_service`，服务标识？若选 `local_excel`，可留空 | — |
| `runtime_prompt` | 每次发送时是否手动指定数据文件？ | `true`（运行时再传文件路径）/ `false`（固定源） |

> 默认推荐：`local_excel` + `runtime_prompt: true`（最灵活）。

**步骤 2: 模板数据源配置 (template_source)**

| 字段 | 询问内容 | 选项说明 |
|------|----------|----------|
| `type` | 模板来源？ | `local_md`（本地 Markdown/HTML 文件）/ `lekai_kb`（乐享知识库）/ `mcp_service`（MCP 服务） |
| `source_name` | 若选 `lekai_kb` 或 `mcp_service`，填写标识 | — |
| `runtime_prompt` | 每次发送时是否手动选择模板？ | `true`（推荐，运行时再选模板）/ `false`（固定模板） |

> 默认推荐：`local_md` + `runtime_prompt: true`（最灵活）。

**步骤 3: 发送渠道配置 (send_channel)**

| 字段 | 询问内容 | 选项说明 |
|------|----------|----------|
| `type` | 发送渠道？ | `smtp`（传统 SMTP）/ `mcp_service`（MCP 邮件服务） |

若选择 **SMTP**，继续询问：

| 字段 | 询问内容 | 可选值 |
|------|----------|--------|
| `service` | 邮箱服务商？ | `qq` / `163` / `126` / `tencent_ex`（腾讯企业邮） / `gmail` / `outlook` / `aliyun`（阿里企业邮） / `custom`（自定义） |
| `username` | 邮箱账号（完整地址） | `yourname@qq.com` |
| `password` | SMTP 密码或授权码 | QQ 邮箱为 16 位授权码；163/126 为授权码 |
| `host` | 仅 custom 时填写 SMTP 服务器地址 | 如 `smtp.exmail.qq.com` |
| `port` | 仅 custom 时填写 SMTP 端口 | 通常 465(SSL) 或 587(TLS) |

> 若选 `custom`，还需填写 `host` 和 `port`。
>
> ⚠️ 提示：QQ 邮箱需开启 POP3/SMTP 服务并获取授权码。

若选择 **MCP 服务**：

| 字段 | 询问内容 |
|------|----------|
| `service_name` | MCP 服务标识 |
| `api_key` | MCP 服务的 API Key（直接写入配置） |
| `endpoint` | 服务端点 URL |

**步骤 4: 发送规则确认 (send_rules)**

向用户展示默认值并逐一确认，可修改：

| 字段 | 默认值 | 说明 |
|------|:---:|------|
| `emails_per_minute` | 10 | 每分钟最多发送封数（QQ 邮箱建议不超过 10） |
| `max_per_task` | 500 | 单次任务最多发送条数 |
| `delay_seconds_between_emails` | 0 | 每封之间额外延迟（秒），设为 0 则仅按每分钟限制控制 |
| `retry_times_on_failure` | 3 | 失败重试次数 |

> 询问用户："以上默认值是否合适？如需修改请告知。" 用户确认或修改后写入。

**步骤 5: 完成初始化**

全部配置完成后：
1. 将完整配置写入 `~/.wx-emailsend-pro/config.json`
2. 展示配置摘要供用户最终确认
3. ⚠️ 告知用户：配置文件 `~/.wx-emailsend-pro/config.json` 含明文密码，请确保文件权限安全（建议 `chmod 600 config.json`）
4. **到此为止，停止。** 不扫描工作空间、不查找数据文件、不搜索模板、不预加载任何内容

### 配置文件结构

```json
{
  "sender_info": {
    "company_name": "",         // 发件公司名称，显示在邮件底部签名区
    "contact_person": "",       // 联系人姓名
    "contact_phone": "",        // 联系电话
    "website": "",              // 公司网址
    "address": ""               // 公司地址
  },
  "data_source": {
    "type": "local_excel",      // local_excel | lekai_kb | mcp_service
    "source_name": "",          // 乐享知识库名称 或 MCP 服务标识
    "runtime_prompt": false     // 是否运行时由用户传入路径
  },
  "template_source": {
    "type": "local_md",         // local_md | lekai_kb | mcp_service
    "source_name": "",
    "runtime_prompt": true      // 是否运行时由用户选择模板
  },
  "send_channel": {
    "type": "smtp",             // smtp | mcp_service
    "smtp_config": {
      "service": "qq",          // qq|163|126|tencent_ex|gmail|outlook|aliyun|custom
      "host": "",               // custom 时必填
      "port": 465,
      "username": "",
      "password": "",             // SMTP 密码/授权码（直接配置）
      "use_ssl": true
    },
    "mcp_service_config": {
      "service_name": "",
      "api_key": "",              // MCP 服务 API Key（直接配置）
      "endpoint": ""
    }
  },
  "send_rules": {
    "emails_per_minute": 10,
    "max_per_task": 500,
    "delay_seconds_between_emails": 0,
    "retry_times_on_failure": 3
  }
}
```

### 部分更新配置

用户可通过自然语言更新配置，例如：
- "设置发件公司为 XX 有限公司，联系人张三，电话 138xxxx" → 修改 `sender_info`
- "更新邮件发送频率为每分钟20封" → 修改 `send_rules.emails_per_minute`
- "切换 SMTP 为 163 邮箱" → 修改 `send_channel.smtp_config.service`
- "重新初始化邮件配置" → 重新生成 config.json

## 核心纪律 — ⚠️ 必须遵守

**本 skill 严格按步骤执行，禁止跳跃、预判、"顺手帮用户多做一点"。**

| 纪律 | 说明 |
|------|------|
| **初始化只做初始化** | 交互式收集完 sender_info → data_source → template_source → send_channel → send_rules 并写入 config.json 后立即停止。**禁止**扫描工作空间、寻找潜在数据文件、猜测联系人列表、预加载模板。后续步骤到哪一步再说哪一步的事 |
| **步骤不可跳跃** | 必须按 执行前检查 → 步骤 1 → 步骤 2 → 步骤 3 → 步骤 4 的顺序执行，不得跳步 |
| **配置怎么说就怎么做** | 如果 `data_source.type = "lekai_kb"`，就在步骤 1 去知识库取数据，不要在初始化阶段提前拉取；如果 `template_source.runtime_prompt = true`，就在步骤 2 询问用户，不要在步骤 1 猜测模板 |
| **不自动发现文件** | 即使工作空间下有 `customers.xlsx` 或 `template.md` 之类看起来像数据源/模板的文件，也**不主动使用**。严格按照 config 配置或用户明确指定的路径执行 |
| **不提前渲染** | 步骤 1 只做数据获取和预览，不渲染邮件内容；步骤 2 只做模板学习、翻译和内容生成，不发邮件 |

## 执行流程（邮件营销任务）

### 执行前检查

1. 验证 `~/.wx-emailsend-pro/config.json` 存在且格式正确
2. 验证发送通道可用性：
   - SMTP: 运行 `python3 ~/.workbuddy/skills/wx-emailsend-pro/scripts/send_email_smtp.py --config ~/.wx-emailsend-pro/config.json --queue /dev/null --test`
   - MCP: 调用对应 MCP 服务的健康检查
3. 验证数据源可访问：**仅验证连接可用性，不拉取实际数据**（实际数据拉取在步骤 1 执行）

**任务标识**: 由当前时间戳 + 随机 ID 生成，格式 `YYYYMMDD_HHMMSS_XXXXXX`。

**中间产物目录**: `${workspace}/wx-emailsend-pro/{任务标识}/`。下文所有 `<task_dir>` 均指此路径。

> `${workspace}` 指当前 WorkBuddy 工作空间根目录。

### 步骤 1：获取客户数据

**严格按 config 执行，不要尝试"顺便帮用户找数据"。**

根据 `data_source.type` 获取数据：

**local_excel**: 
- 如果 `runtime_prompt = true`：**直接询问用户**提供文件路径，不猜测、不扫描工作空间
- 如果 `runtime_prompt = false` 且 `source_name` 有值：使用 `source_name` 指定的路径
- 运行:
```bash
python3 ~/.workbuddy/skills/wx-emailsend-pro/scripts/read_data_source.py <filepath> -o <task_dir>/source_customers.json
```

**lekai_kb**: 使用乐享知识库 MCP（`mcp__lexiang__*` 工具）读取指定知识库中的表格/文档。
- 先通过 `lexiang_search` 搜索对应知识库和文件名
- 使用 `lexiang_fetch` 或文件下载获取数据
- 解析后保存为 `<task_dir>/source_customers.json`

**mcp_service**: 通过对应 MCP 服务获取数据。

完成后生成预览文件 `<task_dir>/email_queue_preview.md`：

```markdown
# 邮件发送队列预览

- 总记录数: 200
- 缺失邮箱字段: 3 条
- 缺失姓名字段: 1 条

## 前5行数据示例

| 序号 | 姓名 | 邮箱 | 公司 | ...
|------|------|------|------|-----
| 1 | 张三 | zhang@ex.com | XX公司 | ...
...
```

**确认**: 展示预览摘要并请求用户确认："是否继续使用此数据源进行发送？"

### 步骤 2：模板选择、学习与确认

**核心原则**: 一次任务只允许一个模板。不管模板来自知识库、用户传入还是 MCP 服务，都必须先进行【模板学习】，学习结果将决定后续渲染和发送的质量。

---

#### 2.1 获取模板

**严格按 config 执行，不猜测、不扫描、不预判。**

根据 `template_source.type` 获取模板：

| 来源类型 | 获取方式 |
|----------|----------|
| `local_md` | 如果 `runtime_prompt = true`：**直接询问用户**提供模板文件路径或模板内容；如果 `runtime_prompt = false` 且 `source_name` 有值：使用指定路径 |
| `lekai_kb` | 使用乐享知识库 MCP（`lexiang-search` → `lexiang_fetch`）读取模板文档 |
| `mcp_service` | 通过对应 MCP 服务获取模板 |

> **禁止**：扫描工作空间寻找 `.md` 或 `.html` 文件、猜测模板位置、列出文件让用户选（除非 `runtime_prompt = true` 且用户要求列出）。

---

#### 2.2 模板学习（Template Learning）— ⚠️ 关键步骤，不可跳过

模板学习是本次任务质量的**决定性环节**。你必须逐字分析模板内容，产出精确到每个占位符的学习报告。

**学习流程**：

(A) **提取占位符清单**
   识别模板中所有 `{{字段名}}` 占位符，逐一列出。

(B) **逐字段交叉比对**
   将每个占位符与以下两个数据源交叉比对，确定可替换性：

   | 比对源 | 说明 |
   |--------|------|
   | **客户数据表头** | 来自步骤 1 获取的 `source_customers.json` 中的字段名列表 |
   | **发件人信息** | 来自 `config.json` 的 `sender_info` 字段（`sender_company_name`、`sender_contact_person`、`sender_contact_phone`、`sender_website`、`sender_address`） |

(C) **分类标记**
   每个占位符分为三类：

   | 分类 | 含义 | 处理方式 |
   |------|------|----------|
   | ✅ **精确匹配** | 占位符名与客户表头或 sender_info 完全一致 | 正常替换为对应字段值 |
   | ⚠️ **模糊可匹配** | 占位符名与某个数据字段语义相近但文字不完全一致 | 自动建立映射关系，在报告中说明映射逻辑（如模板写 `{{客户姓名}}` → 数据表头为 `姓名`，自动映射） |
   | ❌ **无法匹配** | 占位符在客户数据和 sender_info 中均找不到对应字段 | **必须给出中性友好替换方案**：根据占位符的语义，用一个通用中性词语替代（如 `{{产品优势}}` → "我们的优质产品"；`{{促销折扣}}` → "专属优惠"） |

(D) **输出学习报告**
   生成 `<task_dir>/template_learning_report.md`，格式如下：

   ```markdown
   # 模板学习报告

   **模板来源**: /path/to/template.md（或 乐享知识库 / MCP 服务）
   **模板语言**: 中文 / English / ...
   **客户数据字段数**: 15 个
   **模板占位符总数**: 8 个

   ## 占位符匹配详情

   | # | 占位符 | 状态 | 映射字段 | 备注 |
   |---|--------|------|----------|------|
   | 1 | {{姓名}} | ✅ 精确匹配 | 姓名 | 客户表头 `姓名` 完全一致 |
   | 2 | {{邮箱地址}} | ✅ 精确匹配 | 邮箱地址 | 客户表头 `邮箱地址` 完全一致 |
   | 3 | {{公司名称}} | ⚠️ 模糊匹配 | 公司 | 模板写 `公司名称`，数据表头为 `公司`，自动映射 |
   | 4 | {{产品推荐}} | ❌ 无法匹配 | — | 替换为中性短语: "为您精选的优质产品" |
   | 5 | {{sender_company_name}} | ✅ sender_info | sender_info.company_name | 发件人信息字段 |
   | ... | ... | ... | ... | ... |

   ## 需要用户确认的替换

   | # | 占位符 | 建议替换为 | 原因 |
   |---|--------|-----------|------|
   | 1 | {{产品推荐}} | "为您精选的优质产品" | 数据源无此字段，使用中性替换 |
   | 2 | {{专属优惠}} | "限时专享优惠" | 数据源无此字段，使用中性替换 |

   ## 结论

   - ✅ 精确匹配: 5 个占位符
   - ⚠️ 模糊匹配（已自动映射）: 1 个占位符
   - ❌ 无法匹配（使用中性替换）: 2 个占位符
   - **整体可替换率: 75% (6/8)**
   ```

(E) **告知用户学习完毕**
   模板学习完成后，**必须**向用户输出以下信息：
   - "模板学习完毕。共识别 X 个占位符，Y 个精确匹配，Z 个需要中性替换。"
   - 如果有 ❌ 无法匹配的占位符，列出每个占位符及其建议的替换词，请用户确认
   - 如果所有占位符均可匹配，直接告知"所有占位符均可正常替换，无需人工干预"

---

#### 2.3 国籍/语言自动翻译（Internationalization）

**核心策略**: 全自动，不询问用户。

| 条件 | 行为 |
|------|------|
| 客户数据中**有** `国籍` / `国家` 字段 | **自动**按国籍翻译邮件正文为目标语言 |
| 客户数据中**没有** `国籍` / `国家` 字段 | **跳过此步骤**，全部使用模板原始语言 |

**自动执行逻辑**（有国籍字段时）:

1. 提取客户数据中所有不重复的国籍值（如 "中国"、"美国"、"日本"、"德国"...）
2. 统计各国籍的客户数量
3. 输出国籍分布作为信息告知（不等待用户确认）：

   ```markdown
   ## 客户国籍分布（已启用自动翻译）

   | 国籍 | 人数 | 最终发送语言 |
   |------|:----:|-------------|
   | 中国 | 120 | 中文（模板原语言，不翻译） |
   | 美国 | 45 | English（自动翻译） |
   | 日本 | 20 | 日本語（自動翻訳） |
   | 德国 | 15 | Deutsch（automatische Übersetzung） |
   ```

4. **直接进入翻译**，无需用户确认。翻译规则：
   - 与模板语言相同的国籍 → **不翻译**，保留模板原文
   - 与模板语言不同的国籍 → AI 自动翻译邮件正文为目标语言
   - **主题行不翻译**（始终保持模板原始语言）
   - 翻译由 AI 直接完成，无需外部翻译 API

**没有国籍字段时**:

直接告知用户："客户数据中未检测到国籍字段，将统一使用模板语言发送。" 然后跳过翻译步骤。

---

#### 2.4 生成邮件内容队列

在用户确认模板学习结果和语言选项后，生成邮件内容队列。

**生成方式**: AI 直接逐条渲染（不使用外部脚本），将每封邮件渲染为完整内容。

**保存格式**: **每一封邮件保存为独立的 `.md` 文件**，放在 `<task_dir>/emails/` 目录下：

```
<task_dir>/emails/
├── email_001_zhang@example.com.md
├── email_002_li@example.com.md
├── email_003_wang@example.com.md
├── ...
└── email_200_zhao@example.com.md
```

**每封邮件 MD 文件内容格式**：
```markdown
# 邮件 [1/200]

| 字段 | 值 |
|------|----|
| 收件人 | zhang@example.com |
| 姓名 | 张三 |
| 国籍 | 中国 |
| 主题 | 感谢您的支持 |
| 模板语言 | 中文 |
| 实际发送语言 | 中文 |

---

## 邮件正文

尊敬的张三，您好！

感谢您选择我们的服务...

---
[发件人签名]
XX有限公司 | 联系人: 李四 | 电话: 138xxxx
```

> 如果有国籍字段且已自动翻译，`实际发送语言` 字段记录翻译后的语言。

**同时生成汇总文件** `<task_dir>/email_content_queue.md`：

```markdown
# 邮件内容队列总览

- **总发送数**: 200 封
- **模板**: thanks_template.md
- **模板语言**: 中文
- **语言翻译**: 自动（有国籍字段，目标语言: English x45, 日本語 x20, Deutsch x15）

## 首封邮件预览

收件人：zhang@example.com
姓名：张三
主题：感谢您的支持

（完整正文首 200 字...）

## 队列摘要

| 语言 | 封数 |
|------|:---:|
| 中文 | 120 |
| English | 45 |
| 日本語 | 20 |
| Deutsch | 15 |
```

---

#### 2.5 最终确认

向用户输出以下摘要并请求确认：

1. **模板学习结果摘要**: 占位符匹配情况、中性替换清单（用户已确认则跳过）
2. **国籍翻译概览**: 各国语言分布（如有国籍字段；无则显示"模板原语言"）
3. **队列总览**: 总发送数、首封预览
4. **确认问句**: "以上内容已准备就绪，是否立即开始发送？"

用户确认后进入步骤 3。

### 步骤 3：执行邮件发送

邮件队列位于 `<task_dir>/emails/` 目录下（每个收件人一个 `.md` 文件），按文件名序号逐封发送。

运行发送脚本：
```bash
python3 ~/.workbuddy/skills/wx-emailsend-pro/scripts/send_email_smtp.py <task_dir>/emails/ \
  --config ~/.wx-emailsend-pro/config.json \
  -o <task_dir>/send_results.json
```

关键行为：
- 严格遵守 `emails_per_minute` 速率限制
- 单次上限 `max_per_task`，超出部分截断
- 失败自动重试（`retry_times_on_failure` 次）
- 中间结果实时写入 `send_results.json`（支持中断续传）
- 控制台实时输出进度

### 步骤 4：生成报告

报告生成到最终交付目录 `${workspace}/发送计划/{任务标识}/`（需先创建该目录）：

```bash
mkdir -p ${workspace}/发送计划/<任务标识>/
python3 ~/.workbuddy/skills/wx-emailsend-pro/scripts/generate_report.py <task_dir>/send_results.json \
  --task-id <任务标识> \
  --output-dir ${workspace}/发送计划/<任务标识>/
```

生成两个文件（位于 `发送计划/<任务标识>/` 下）：
- `email_send_report_<任务标识>.xlsx` — 每封邮件的详细发送清单
- `email_send_statistics_<任务标识>.html` — 可视化统计报告（含饼图、速率曲线等）

使用 `preview_url` 或 `open_result_view` 展示 HTML 报告。

## 乐享知识库数据源集成

当 `data_source.type` 或 `template_source.type` 为 `lekai_kb` 时：

1. 通过 `lexiang-search` skill 或 `mcp__lexiang__*` 工具搜索/读取知识库内容
2. 客户数据通常以 Excel 附件形式存储在知识库中：
   - 先用 `lexiang_search` 找到文档
   - 用 `lexiang_fetch` 读取页面内容获取附件 ID
   - 用 `file_download_file` 下载附件
3. 模板文档通常为 Markdown 页面，直接用 `lexiang_fetch` 读取内容即可

## 异常处理

- SMTP 连接中断: 自动重试连接并续传，最多 3 次；仍失败则暂停并保存已发送记录
- 用户中途终止: 保留所有中间文件（`<task_dir>/` = `${workspace}/wx-emailsend-pro/{任务标识}/`），提示已发送/未发送数量
- 确认超时: 5 分钟无响应自动取消任务
- 敏感信息: SMTP 密码、MCP API Key 直接存储在 `~/.wx-emailsend-pro/config.json` 中，请确保文件权限安全（`chmod 600`），不泄露给他人

## 脚本使用参考

| 脚本 | 用途 |
|------|------|
| `~/.workbuddy/skills/wx-emailsend-pro/scripts/read_data_source.py` | 读取 Excel/CSV/JSON 客户数据 → JSON |
| `~/.workbuddy/skills/wx-emailsend-pro/scripts/render_template.py` | (可选备用) 模板 + 客户数据 → 渲染邮件队列 JSON；主流程由 AI 直接渲染为 `emails/*.md` |
| `~/.workbuddy/skills/wx-emailsend-pro/scripts/send_email_smtp.py` | 读取 `emails/` 目录中的 `.md` 文件 + SMTP 配置 → 逐封发送 + 结果 JSON |
| `~/.workbuddy/skills/wx-emailsend-pro/scripts/generate_report.py` | 发送结果 JSON → Excel 清单 + HTML 统计报告 |

所有脚本支持 `--help` 查看完整参数说明。

## 依赖

- **发送脚本**: 依赖 `openpyxl`（用于生成 Excel 报告）。首次运行时执行：

  ```bash
  pip install openpyxl
  ```

- **模板渲染**: 由 AI 直接执行，无需额外依赖。
  
- **多语言翻译**: 有国籍字段时自动按国籍翻译邮件正文为目标语言，由 AI 直接完成，无需外部翻译 API。

## 资源文件

- `references/smtp_configs.md` — 各邮箱服务商 SMTP 配置参考（**先读此文件再配置 SMTP**）
- `references/template_guide.md` — 邮件模板编写指南与最佳实践
- `assets/default_config.json` — 默认配置文件模板