# 用户指南 — 股票智能分析系统（DSA）

> 本文档面向终端用户，介绍系统的核心功能、操作步骤、计算逻辑与使用注意事项。
> 面向开发者的技术架构文档请参阅同目录下的 [CODE_WIKI.md](CODE_WIKI.md)。

## 目录

- [1. 系统能做什么](#1-系统能做什么)
- [2. 核心功能](#2-核心功能)
- [3. 快速开始](#3-快速开始)
- [4. 操作步骤详解](#4-操作步骤详解)
- [5. 计算逻辑说明](#5-计算逻辑说明)
- [6. 配置说明](#6-配置说明)
- [7. 通知与推送](#7-通知与推送)
- [8. 使用注意事项](#8-使用注意事项)
- [9. 常见问题](#9-常见问题)

---

## 1. 系统能做什么

股票智能分析系统（DSA）是一款基于 AI 大模型的自动化股票分析工具，主要能力包括：

| 能力 | 说明 |
|------|------|
| **AI 决策报告** | 自动生成核心结论、综合评分（0-100）、趋势判断、买卖点位、风险警报、催化因素、操作检查清单 |
| **多市场覆盖** | A 股、港股、美股、日股、韩股、台股和 ETF |
| **大盘复盘** | 每日主要指数、涨跌统计、板块表现、市场温度计（红黄绿灯） |
| **Agent 策略问股** | 多轮对话，支持 15 种内置策略（均线、缠论、波浪、热点、事件等） |
| **自动推送** | 每日定时分析并推送到企业微信/飞书/Telegram/Discord/Slack/邮箱等 14 种渠道 |
| **Web 工作台** | 手动分析、任务监控、历史报告、回测、持仓管理、智能导入 |
| **桌面客户端** | Windows/macOS 桌面应用，自动更新 |
| **智能导入** | 图片识别、CSV/Excel、剪贴板导入自选股 |

> ⚠️ **免责声明**：本项目仅供学习和研究使用，不构成任何投资建议。股市有风险，投资需谨慎。

---

## 2. 核心功能

### 2.1 个股决策仪表盘

每只股票分析完成后，生成「决策仪表盘」报告，包含：

- **核心结论**：买入/观望/卖出建议 + 综合评分（0-100）
- **趋势预测**：看多/震荡/看空
- **狙击点位**：理想买点、次级买点、止损价、目标价
- **风险警报**：识别主力资金流出、筹码集中度、历史违规等风险
- **利好催化**：AI 属性、业绩增长、政策利好等
- **操作检查清单**：具体操作建议

**推送效果示例**：

```
🎯 2026-02-08 决策仪表盘
共分析3只股票 | 🟢买入:0 🟡观望:2 🔴卖出:1

📊 分析结果摘要
⚪ 中钨高新(000657): 观望 | 评分 65 | 看多
⚪ 永鼎股份(600105): 观望 | 评分 48 | 震荡
🟡 新莱应材(300260): 卖出 | 评分 35 | 看空
```

### 2.2 大盘复盘

每日市场总览，包含：

- **主要指数**：上证、深证、创业板等涨跌幅
- **市场概况**：上涨/下跌家数、涨停/跌停数
- **板块表现**：领涨/领跌板块与概念
- **市场温度计**：基于 breadth(45%) + index(35%) + limit(20%) 加权计算的红黄绿灯
  - 🟢 绿灯：可进攻
  - 🟡 黄灯：需观察
  - 🔴 红灯：偏防守

### 2.3 Agent 策略问股

通过 Web `/chat` 页面或 Bot 命令进行多轮对话分析：

- 支持 15 种内置策略，每次最多选 3 个
- 支持实时行情、K 线、技术指标、新闻调用
- 支持多轮追问、会话导出、发送到通知渠道
- 支持自定义策略文件（通过 `AGENT_SKILL_DIR`）

**15 种内置策略**：

| 分类 | 策略 |
|------|------|
| 趋势类 | 多头趋势、均线金叉、波浪理论 |
| 形态类 | 箱体震荡、缠论、一阳吃三阴、缩量回踩、放量突破、底部放量 |
| 主题/情绪 | 龙头股、热点主题、情绪周期 |
| 基本面/事件 | 成长质量、事件驱动、预期重定价 |

### 2.4 决策信号

每次分析会自动提取「决策信号」作为资产管理：

- **信号类型**：buy/add/hold/reduce/sell/avoid 等
- **有效期（horizon）**：intraday/1d/3d/5d/10d/swing/long
- **互斥失效**：同股票出现反向信号时，旧信号自动失效
- **持仓联动**：可按持仓过滤查看信号
- **后验评估**：信号到期后自动评估结果（命中/未命中/中性）
- **用户反馈**：可对信号标记反馈

### 2.5 回测

对历史分析结果进行回测评估：

- 评估窗口：默认 10 个交易日
- 最小年龄：分析记录需存在至少 14 天
- 评估结果：processed/saved/completed/insufficient/errors 统计
- 支持按股票、技能、策略维度汇总

### 2.6 持仓管理

- **账户**：多账户、多市场（cn/hk/us/jp/kr/tw）、多币种
- **成本方法**：FIFO（先进先出）或均价法
- **事件**：交易、现金账本、公司行为（分红/拆股）
- **快照**：每日账户快照、持仓饼图
- **汇率**：跨币种汇率自动刷新（YFinance）
- **风险**：集中度告警、回撤告警

### 2.7 告警中心

- **规则类型**：价格穿越、价格变化、量能突增、技术指标、持仓风险、市场灯
- **目标范围**：单只股票、自选股、持仓、账户、全市场
- **严重级别**：info / warning / critical
- **冷却机制**：24 小时 DB 冷却 + 指纹去重，避免重复告警
- **通知路由**：告警可独立配置通知渠道

### 2.8 情报源

合规资讯源管理（RSS/Atom/NewsNow）：

- 内置模板：SEC、HKEX、MarketWatch RSS
- NewsNow 默认源：财联社、雪球、华尔街见闻、金十、格隆汇
- 支持 DNS Guard 防 SSRF
- 自动去重、按 scope（symbol/market/sector）分类

---

## 3. 快速开始

### 方式一：GitHub Actions（推荐，零成本）

> 5 分钟完成部署，无需服务器。

1. **Fork 仓库**：点击 GitHub 右上角 `Fork` 按钮
2. **配置 Secrets**：`Settings` → `Secrets and variables` → `Actions` → `New repository secret`
   - **AI 模型**（至少一个）：`ANSPIRE_API_KEYS` / `AIHUBMIX_KEY` / `GEMINI_API_KEY` / `OPENAI_API_KEY` 等
   - **通知渠道**（至少一个）：`WECHAT_WEBHOOK_URL` / `FEISHU_WEBHOOK_URL` / `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` 等
   - **自选股**（必填）：`STOCK_LIST`，如 `600519,hk00700,AAPL,7203.T,005930.KS,2330.TW`
   - **新闻源**（推荐）：`ANSPIRE_API_KEYS` / `SERPAPI_API_KEYS` / `TAVILY_API_KEYS` 等
3. **启用 Actions**：`Actions` 标签 → `I understand my workflows, go ahead and enable them`
4. **手动测试**：`Actions` → `每日股票分析` → `Run workflow`
5. **完成**：默认每个工作日 18:00（北京时间）自动执行

### 方式二：本地运行

```bash
# 克隆项目
git clone https://github.com/ZhuLinsen/daily_stock_analysis.git
cd daily_stock_analysis

# 安装依赖（Python 3.10+）
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env，至少配置：AI 模型 Key、通知渠道、STOCK_LIST

# 运行分析
python main.py
```

### 方式三：Docker 部署

```bash
# 使用 docker-compose
docker-compose up -d
```

### 方式四：桌面客户端

从 [GitHub Releases](https://github.com/ZhuLinsen/daily_stock_analysis/releases) 下载对应平台安装包：

- Windows：NSIS 安装器（`.exe`）
- macOS：DMG 镜像

---

## 4. 操作步骤详解

### 4.1 配置 AI 模型

系统支持多种 AI 模型渠道，**至少配置一个**：

| 配置项 | 说明 | 推荐度 |
|--------|------|--------|
| `ANSPIRE_API_KEYS` | Anspire，一 Key 启用大模型 + 联网搜索，无需科学上网，含免费额度 | ⭐⭐⭐ 推荐 |
| `AIHUBMIX_KEY` | AIHubMix，一 Key 切换全系模型，无需科学上网 | ⭐⭐⭐ 推荐 |
| `GEMINI_API_KEY` | Google Gemini | 可选 |
| `OPENAI_API_KEY` + `OPENAI_BASE_URL` | OpenAI 兼容（支持 DeepSeek、通义千问等） | 可选 |
| `ANTHROPIC_API_KEY` | Anthropic Claude | 可选 |
| `DEEPSEEK_API_KEYS` | DeepSeek | 可选 |

**多模型/高级路由**：参考 [LLM 配置指南](../LLM_CONFIG_GUIDE.md)。

### 4.2 配置自选股

在 `.env` 中设置 `STOCK_LIST`：

```bash
# 代码格式
# A 股：6 位代码，如 600519
# 港股：hk + 5 位代码，如 hk00700
# 美股：字母代码，如 AAPL
# 日股：代码.T，如 7203.T
# 韩股：代码.KS/.KQ，如 005930.KS
# 台股：代码.TW/.TWO，如 2330.TW

STOCK_LIST=600519,hk00700,AAPL,7203.T,005930.KS,2330.TW
```

**代码规范**（统一大写）：

| 市场 | 格式 | 示例 |
|------|------|------|
| A 股 | 6 位数字 | `600519`、`000001`、`300260` |
| 港股 | `hk` + 5 位数字 | `hk00700`、`hk09988` |
| 美股 | 字母代码 | `AAPL`、`TSLA` |
| 日股 | 数字 + `.T` | `7203.T` |
| 韩股 | 数字 + `.KS`/`.KQ` | `005930.KS` |
| 台股 | 数字 + `.TW`/`.TWO` | `2330.TW` |

### 4.3 运行分析

#### CLI 方式

```bash
# 正常运行（个股 + 大盘复盘）
python main.py

# 指定股票（覆盖配置）
python main.py --stocks 600519,hk00700,AAPL

# 仅大盘复盘
python main.py --market-review

# 调试模式
python main.py --debug

# 仅获取数据，不分析
python main.py --dry-run

# 单股推送模式（每分析完一只立即推送，不汇总）
python main.py --single-notify

# 跳过交易日检查，强制执行
python main.py --force-run

# 定时任务模式
python main.py --schedule
```

#### Web 方式

```bash
# 启动 Web 服务（同时执行分析）
python main.py --serve

# 仅 Web 服务（不自动分析，通过界面手动触发）
python main.py --serve-only
```

访问 `http://127.0.0.1:8000`：

1. **首页**：输入股票代码 → 点击分析 → 实时查看任务进度（SSE）→ 查看报告
2. **聊天页**：选择策略 → 输入问题 → 多轮对话
3. **历史页**：查看历史报告、Markdown 全文、运行流
4. **设置页**：配置 LLM 渠道、通知渠道、定时任务、智能导入

#### Bot 方式

在配置好 Bot 平台（飞书/钉钉/Discord/Telegram）后，在群内或私聊发送命令：

| 命令 | 别名 | 说明 |
|------|------|------|
| `/analyze 600519` | `/a` / `分析` | 分析单只股票 |
| `/ask 600519 均线金叉` | `/问股` | 策略问股 |
| `/batch` | `/b` / `批量` | 批量分析自选股 |
| `/chat 这个股票能买吗` | `/c` / `问` | Agent 对话 |
| `/market` | `/m` / `大盘` / `复盘` | 大盘复盘 |
| `/research 半导体板块机会` | `/深研` | 深度研究 |
| `/history` | `/历史` | 对话历史 |
| `/status` | `/s` / `状态` | 系统状态 |
| `/strategies` | `/策略` | 策略列表 |
| `/help` | `/h` / `帮助` | 帮助 |

### 4.4 配置定时任务

```bash
# 启用定时任务
SCHEDULE_ENABLED=true

# 执行时间（默认 18:00，支持多时间点逗号分隔）
SCHEDULE_TIME=18:00
SCHEDULE_TIMES=09:30,15:00,18:00

# 启动时是否立即执行
SCHEDULE_RUN_IMMEDIATELY=true

# 大盘复盘区域（cn/hk/us/jp/kr/both 或逗号子集）
MARKET_REVIEW_REGION=cn
MARKET_REVIEW_ENABLED=true

# 交易日检查（默认开启，非交易日跳过）
TRADING_DAY_CHECK_ENABLED=true
```

### 4.5 智能导入

Web 设置页支持多种导入方式：

- **图片识别**：上传持仓截图，Vision LLM 自动识别股票代码
- **CSV/Excel**：上传券商导出文件，自动解析
- **剪贴板**：粘贴文本，自动提取代码

---

## 5. 计算逻辑说明

### 5.1 技术分析评分系统

技术分析器（`StockTrendAnalyzer`）采用 0-100 综合评分系统，融合交易理念：

**交易理念**：

1. 严进策略：不追高，乖离率 `(Close-MA5)/MA5 < 5%`
2. 趋势交易：只做 MA5 > MA10 > MA20 多头排列
3. 效率优先：关注筹码集中度好的股票
4. 买点偏好：缩量回踩 MA5/MA10 支撑

**评分维度**：

| 维度 | 权重 | 评分逻辑 |
|------|------|---------|
| 趋势 | 30 分 | STRONG_BULL=30 / BULL=25 / WEAK_BULL=20 / CONSOLIDATION=15 / WEAK_BEAR=10 / BEAR=5 / STRONG_BEAR=0 |
| 乖离率 | 20 分 | 受 `BIAS_THRESHOLD`（默认 5.0）控制，强势趋势阈值放宽 ×1.5 |
| 量能 | 15 分 | 缩量回调=15（最佳）/ 缩量上涨=12 / 放量上涨=10 / 正常=8 / 放量下跌=0 |
| 支撑 | 10 分 | MA5 支撑 5 分 + MA10 支撑 5 分（容忍度 2%） |
| MACD | 15 分 | 零轴上金叉=15 / 金叉=12 / 多头=10 / 即将金叉=8 / 即将死叉=5 / 空头=3 / 死叉=0 |
| RSI | 10 分 | 超卖(>70)=0 / 强买(50-70)=8 / 中性(30-50)=6 / 弱(20-30)=4 / 超卖(<20)=10 |

**技术指标参数**：

- 均线：MA5 / MA10 / MA20 / MA60
- MACD：12 / 26 / 9（标准参数）
- RSI：Wilder EMA 口径，6 / 12 / 24 周期
- 量比：当日成交量 / 前 5 日平均成交量
- 乖离率：`(Close - MA5) / MA5 × 100%`

**买入信号映射**：

| 评分区间 | 信号 |
|---------|------|
| ≥ 80 | STRONG_BUY（强烈买入） |
| 65-79 | BUY（买入） |
| 50-64 | HOLD（持有） |
| 35-49 | WAIT（观望） |
| 20-34 | SELL（卖出） |
| < 20 | STRONG_SELL（强烈卖出） |

### 5.2 市场温度计（大盘复盘）

基于三维加权计算 0-100 分数：

| 维度 | 权重 | 说明 |
|------|------|------|
| 市场广度（breadth） | 45% | 上涨家数占比 |
| 指数表现（index） | 35% | 主要指数涨跌幅 |
| 涨跌停（limit） | 20% | 涨停 vs 跌停比 |

**状态映射**：

- 🟢 绿灯（高分）：可进攻
- 🟡 黄灯（中分）：需观察
- 🔴 红灯（低分）：偏防守

> 注：仅 A 股市场有完整的涨跌家数与板块涨跌榜数据，其他市场（港股/美股/日韩）仅基于指数表现。

### 5.3 LLM 综合分析

技术分析结果作为输入之一，与以下数据一起送入 LLM 生成最终报告：

- **技术面**：趋势、均线、MACD、RSI、量能、支撑压力
- **基本面**：估值（PE/PB）、成长、盈利、机构持仓、资金流向、龙虎榜、关联板块（fail-open，失败降级）
- **情报面**：新闻搜索（多源）、公告、舆情、社交情绪（美股可选）
- **市场阶段**：盘前/盘中/盘后/休市（影响分析口径）
- **大盘上下文**：当日大盘复盘摘要（可选注入）

LLM 输出结构化 JSON，经 `stabilize_decision_with_structure` 稳定化处理后生成 `AnalysisResult`。

### 5.4 决策信号生命周期

```
分析完成 → 自动提取信号（active 状态）
   │
   ├─ horizon 到期 → expired
   ├─ 出现反向信号 → invalidated（互斥失效）
   ├─ 用户手动关闭 → closed
   └─ 归档 → archived
   │
   ▼
后验评估（到期后）
   ├─ 命中（hit）
   ├─ 未命中（miss）
   └─ 中性（neutral）
```

**horizon 默认值**（按市场）：

| 市场 | intraday 默认 TTL |
|------|------------------|
| cn | 4.0 小时 |
| hk | 5.5 小时 |
| us | 6.5 小时 |

### 5.5 回测评估逻辑

- **评估窗口**：默认 10 个交易日（`BACKTEST_EVAL_WINDOW_DAYS`）
- **最小年龄**：分析记录需存在至少 14 天（`BACKTEST_MIN_AGE_DAYS`）
- **中性带**：默认 ±2%（`BACKTEST_NEUTRAL_BAND_PCT`），价格变化在此范围内算中性
- **评估对象**：仅评估个股分析，排除 `market_review` 类型
- **结果维度**：支持 overall / stock / skill / strategy 汇总

### 5.6 持仓成本计算

支持两种成本方法：

- **FIFO（先进先出）**：卖出时优先消耗最早买入的批次，按批次计算已实现盈亏
- **均价法**：每次买入后重新计算平均成本

支持公司行为调整：

- **现金分红**：调整现金余额
- **拆股调整**：调整持股数量与成本

### 5.7 告警触发逻辑

```
AlertWorker 周期轮询（默认 5 分钟）
   │
   ├─ 加载规则（持久化 + legacy 运行时）
   ├─ 逐条评估（dry-run 复用同一逻辑）
   │   ├─ 价格穿越：实时价 vs 阈值
   │   ├─ 价格变化：N 日涨跌幅 vs 阈值
   │   ├─ 量能突增：量比 vs 阈值
   │   ├─ 技术指标：RSI/MACD 状态
   │   ├─ 持仓风险：集中度/回撤
   │   └─ 市场灯：红黄绿灯状态
   ├─ 指纹去重（24 小时 TTL）
   ├─ DB 冷却决策（24 小时默认）
   ├─ 触发写入（alert_triggers）
   └─ 通知分发（alert 路由渠道）
```

---

## 6. 配置说明

### 6.1 核心配置项

在 `.env` 文件中配置（参考 `.env.example`）：

```bash
# === 必填 ===
STOCK_LIST=600519,hk00700,AAPL    # 自选股

# === AI 模型（至少一个）===
ANSPIRE_API_KEYS=xxx              # 推荐
# GEMINI_API_KEY=xxx
# OPENAI_API_KEY=xxx
# OPENAI_BASE_URL=https://api.openai.com/v1
# OPENAI_MODEL=gpt-4o

# === 通知渠道（至少一个）===
WECHAT_WEBHOOK_URL=https://xxx    # 企业微信
# FEISHU_WEBHOOK_URL=https://xxx  # 飞书
# TELEGRAM_BOT_TOKEN=xxx
# TELEGRAM_CHAT_ID=xxx
# EMAIL_SENDER=xxx
# EMAIL_PASSWORD=xxx

# === 新闻源（推荐）===
ANSPIRE_API_KEYS=xxx              # 复用 AI 模型 Key
# SERPAPI_API_KEYS=xxx
# TAVILY_API_KEYS=xxx
```

### 6.2 报告配置

```bash
REPORT_TYPE=simple                # simple / full / brief
REPORT_LANGUAGE=zh                # zh / en / ko
MERGE_EMAIL_NOTIFICATION=false    # 合并推送（个股+大盘）
SINGLE_STOCK_NOTIFY=false         # 单股推送模式
MARKDOWN_TO_IMAGE_CHANNELS=wechat # 需要转图片的渠道
```

### 6.3 数据源配置

```bash
# Tushare（A 股增强）
TUSHARE_TOKEN=xxx

# TickFlow（A 股高级功能）
TICKFLOW_API_KEY=xxx
TICKFLOW_KLINE_ADJUST=forward     # none/forward/backward/...
TICKFLOW_PRIORITY=2

# Longbridge（港股/美股）
LONGBRIDGE_OAUTH_CLIENT_ID=xxx
# 或 Legacy: LONGBRIDGE_APP_KEY/APP_SECRET/ACCESS_TOKEN

# 实时行情
ENABLE_REALTIME_QUOTE=true
ENABLE_REALTIME_TECHNICAL_INDICATORS=true
ENABLE_CHIP_DISTRIBUTION=true
REALTIME_SOURCE_PRIORITY=tencent,akshare_sina,efinance,akshare_em

# 基本面
ENABLE_FUNDAMENTAL_PIPELINE=true
FUNDAMENTAL_STAGE_TIMEOUT_SECONDS=8.0
```

### 6.4 调度配置

```bash
SCHEDULE_ENABLED=true
SCHEDULE_TIME=18:00
SCHEDULE_TIMES=09:30,15:00,18:00
SCHEDULE_RUN_IMMEDIATELY=true

MARKET_REVIEW_ENABLED=true
MARKET_REVIEW_REGION=cn           # cn/hk/us/jp/kr/both
TRADING_DAY_CHECK_ENABLED=true

MAX_WORKERS=3                     # 并发数（防封禁）
```

### 6.5 Agent 配置

```bash
AGENT_MODE=true                   # 启用 Agent（默认 true）
AGENT_MAX_STEPS=10                # 最大步数
AGENT_ARCH=single                 # single / multi
AGENT_SKILLS=均线金叉,缠论         # 默认技能
# AGENT_SKILL_DIR=/path/to/skills # 自定义策略目录
```

### 6.6 认证配置

```bash
ADMIN_AUTH_ENABLED=false          # 启用 Web 管理员认证
# 首次登录时设置密码，或用 CLI 重置：
# python -m src.auth reset-password
```

---

## 7. 通知与推送

### 7.1 支持的 14 种渠道

| 渠道 | 配置项 | 特性 |
|------|--------|------|
| 企业微信 | `WECHAT_WEBHOOK_URL` | markdown 4096B 分批，支持图片（2MB） |
| 飞书 | `FEISHU_WEBHOOK_URL` 或 `FEISHU_APP_ID`+`FEISHU_APP_SECRET`+`FEISHU_CHAT_ID` | Webhook + App Bot 双模式，卡片优先 |
| 钉钉 | `DINGTALK_WEBHOOK_URL`+`DINGTALK_SECRET` | Webhook + secret 签名 |
| Telegram | `TELEGRAM_BOT_TOKEN`+`TELEGRAM_CHAT_ID` | Bot API |
| Discord | `DISCORD_WEBHOOK_URL` | Webhook |
| Slack | `SLACK_BOT_TOKEN`+`SLACK_CHANNEL_ID` | Bot |
| 邮件 | `EMAIL_SENDER`+`EMAIL_PASSWORD`+`EMAIL_RECEIVER` | SMTP，支持分组 |
| Pushover | `PUSHOVER_USER_KEY`+`PUSHOVER_API_TOKEN` | |
| Ntfy | `NTFY_TOPIC`+`NTFY_SERVER` | 自建推送 |
| Gotify | `GOTIFY_URL`+`GOTIFY_TOKEN` | 自建推送 |
| Pushplus | `PUSHPLUS_TOKEN` | |
| Server酱3 | `SERVERCHAN3_SENDKEY` | |
| Custom Webhook | `CUSTOM_WEBHOOK_URL` | 通用 HTTP |
| Astrbot | `ASTRBOT_BASE_URL`+`ASTRBOT_TOKEN` | |

### 7.2 通知路由

通知按路由类型分发，可独立配置渠道：

| 路由类型 | 配置项 | 默认 |
|---------|--------|------|
| report（报告） | `NOTIFICATION_CHANNELS_REPORT` | 所有已配置渠道 |
| alert（告警） | `NOTIFICATION_CHANNELS_ALERT` | 所有已配置渠道 |
| system_error（系统错误） | `NOTIFICATION_CHANNELS_SYSTEM_ERROR` | 所有已配置渠道 |

### 7.3 上下文渠道优先

当通过 Bot 触发分析时，回复会优先发送到触发来源的上下文渠道：

- 钉钉：`sessionWebhook`
- 飞书 Stream：`chat_id`
- Telegram：`chat_id`

无上下文时（如定时任务、CLI）才广播静态渠道。

### 7.4 Markdown 转图片

部分渠道（如企业微信）对 Markdown 支持有限，可配置自动转图片：

```bash
MARKDOWN_TO_IMAGE_CHANNELS=wechat  # 需要转图片的渠道
MARKDOWN_TO_IMAGE_MAX_CHARS=15000  # 最大字符数
MD2IMG_ENGINE=wkhtmltoimage        # 或 markdown-to-file
```

---

## 8. 使用注意事项

### 8.1 数据源与 fallback

- **多源 fallback**：系统按优先级自动切换数据源，单一数据源失败不会中断分析
- **A 股数据最全**：AkShare/Efinance 提供完整的实时行情、筹码、板块、涨停池
- **港股/美股**：YFinance 作为兜底，Longbridge/Finnhub 作为增强
- **日韩台股**：主要依赖 YFinance
- **熔断器**：数据源连续失败 3 次后冷却 5 分钟

### 8.2 交易日与市场阶段

- **交易日检查**：默认开启，非交易日（含 A/H/US 节假日）自动跳过
- **强制运行**：`--force-run` 跳过交易日检查
- **市场阶段感知**：分析时会判断盘前/盘中/盘后/休市，影响数据口径
- **断点续传**：按交易日判断是否已存在数据，避免重复抓取

### 8.3 并发与限流

- **默认低并发**：`MAX_WORKERS=3`，防止数据源封禁
- **随机休眠**：AkShare 等数据源内置 2-5 秒随机休眠
- **API 限流**：Tushare 80 次/分、Finnhub 60 次/分、AlphaVantage 25 次/天
- **LLM 重试**：失败自动重试与模型切换

### 8.4 fail-open 语义

以下能力失败时**降级而非中断**：

- 基本面数据（PE/PB/机构持仓等）
- 社交舆情（仅美股）
- 部分通知渠道（单一渠道失败不影响其他渠道）
- 大盘上下文注入

### 8.5 报告语言

- 支持 `zh`（中文）、`en`（英文）、`ko`（韩文）
- 韩文复用英文脚手架
- 通过 `REPORT_LANGUAGE` 全局配置，API 调用可单次覆盖

### 8.6 飞书云文档（可选）

配置飞书 App 凭证后，每日分析可自动创建飞书云文档：

```bash
FEISHU_APP_ID=xxx
FEISHU_APP_SECRET=xxx
FEISHU_DOC_FOLDER_TOKEN=xxx
```

### 8.7 桌面端注意事项

- **Windows**：NSIS 安装版支持自动更新；便携版需手动更新
- **macOS**：DMG 安装，支持 x64 和 arm64
- **后端进程**：桌面端会自动启动后端（打包模式用 PyInstaller exe，开发模式用 python）
- **端口**：自动探测 8000-8100 可用端口
- **更新前备份**：更新前自动备份 .env、数据库、alphasift 数据

### 8.8 GitHub Actions 注意事项

- **免费额度**：公开仓库 Actions 免费，私有仓库有额度限制
- **超时**：默认 30 分钟（`ANALYSIS_TIMEOUT_MINUTES` 可调）
- **交易日检查**：默认非交易日不执行
- **断点续传**：GitHub Actions 每次都是全新环境，无本地缓存
- **并发**：同一仓库同时只允许一个分析任务运行

### 8.9 安全注意事项

- **不要提交 `.env`**：`.gitignore` 已排除，切勿泄露密钥
- **Web 认证**：公网暴露 Web 服务时务必启用 `ADMIN_AUTH_ENABLED=true`
- **代理配置**：`USE_PROXY=true` 时使用本地代理，GitHub Actions 自动跳过
- **敏感 env 过滤**：本地 CLI 后端会过滤敏感环境变量

---

## 9. 常见问题

### Q1: 分析失败怎么办？

1. 检查 AI 模型 Key 是否有效（`python main.py --debug` 查看详细日志）
2. 检查网络连接（特别是访问境外 API）
3. 检查数据源是否可用（`python main.py --dry-run` 仅获取数据）
4. 查看日志：默认在 `logs/` 目录

### Q2: 没有收到推送？

1. `python main.py --check-notify` 检查通知配置（只读）
2. 确认通知渠道已正确配置
3. 检查 Webhook URL 是否有效
4. 查看通知诊断日志

### Q3: 数据源报错/限流？

1. 多数据源会自动 fallback，通常不影响整体
2. Tushare 配额用尽时切换到其他源
3. 调整 `MAX_WORKERS` 降低并发
4. 参考 [数据源稳定性](../data-source-stability.md)

### Q4: 非交易日为什么没有执行？

- 默认开启交易日检查（`TRADING_DAY_CHECK_ENABLED=true`）
- 使用 `--force-run` 强制执行
- 不同市场节假日不同（A/H/US）

### Q5: 如何添加自定义策略？

1. 创建 YAML 文件，参考 `strategies/` 目录下的格式
2. 设置 `AGENT_SKILL_DIR` 指向自定义目录
3. 在 Web `/chat` 或 Bot `/strategies` 中查看

### Q6: 桌面端无法启动？

1. 检查端口 8000-8100 是否被占用
2. 查看桌面端日志：`<appDir>/logs/desktop.log`
3. 杀毒软件可能拦截后端进程
4. macOS 首次打开需在「系统偏好设置 → 安全性与隐私」允许

### Q7: 如何重置 Web 管理员密码？

```bash
python -m src.auth reset-password
```

### Q8: 如何查看历史报告？

- **Web**：访问 `/history` 页面，支持按股票、日期筛选
- **API**：`GET /api/v1/history`
- **文件**：报告默认保存在 `reports/` 目录
- **数据库**：存储在 `stock_analysis.db` 的 `analysis_history` 表

### Q9: 回测结果不准确？

- 回测基于历史分析记录，需先有足够分析数据
- 默认最小年龄 14 天（`BACKTEST_MIN_AGE_DAYS`）
- 评估窗口默认 10 个交易日（`BACKTEST_EVAL_WINDOW_DAYS`）
- 中性带默认 ±2%（`BACKTEST_NEUTRAL_BAND_PCT`）

### Q10: 如何停止正在运行的分析？

- **CLI**：`Ctrl+C`
- **Web**：任务面板支持取消请求（`CANCEL_REQUESTED` 状态）
- **API**：调用任务取消接口

---

## 附录：相关文档

| 文档 | 内容 |
|------|------|
| [完整配置与部署指南](../full-guide.md) | 环境准备、运行方式、配置说明、部署路径 |
| [LLM 配置指南](../LLM_CONFIG_GUIDE.md) | 大模型渠道、三层配置、Web 设置页 |
| [通知能力基线](../notifications.md) | 通知渠道配置 |
| [Bot 命令与接入](../bot-command.md) | Bot 命令、Webhook、平台接入 |
| [FAQ](../FAQ.md) | 常见问题 |
| [数据源稳定性](../data-source-stability.md) | 数据源 fallback 与推荐配置 |
| [DecisionSignal 决策信号专题](../decision-signals.md) | 决策信号字段语义与生命周期 |
| [实时告警中心](../alerts.md) | 告警规则与触发 |
| [Code Wiki](CODE_WIKI.md) | 技术架构文档（面向开发者） |
