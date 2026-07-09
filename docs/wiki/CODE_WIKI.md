# Code Wiki — 股票智能分析系统（DSA）

> 本文档是项目的结构化技术 Wiki，面向开发者与维护者，覆盖整体架构、模块职责、关键类与函数、依赖关系与运行方式。
> 面向终端用户的功能与操作文档请参阅同目录下的 [USER_GUIDE.md](USER_GUIDE.md)。

## 目录

- [1. 项目定位与主流程](#1-项目定位与主流程)
- [2. 顶层架构总览](#2-顶层架构总览)
- [3. 目录结构与边界](#3-目录结构与边界)
- [4. 核心模块详解](#4-核心模块详解)
  - [4.1 主流程编排 `src/core/`](#41-主流程编排-src_core)
  - [4.2 分析引擎 `src/analyzer.py` / `src/stock_analyzer.py` / `src/market_analyzer.py`](#42-分析引擎)
  - [4.3 数据源适配层 `data_provider/`](#43-数据源适配层-data_provider)
  - [4.4 Agent 编排系统 `src/agent/`](#44-agent-编排系统-src_agent)
  - [4.5 LLM 后端抽象 `src/llm/`](#45-llm-后端抽象-src_llm)
  - [4.6 业务服务层 `src/services/`](#46-业务服务层-src_services)
  - [4.7 数据访问层 `src/repositories/`](#47-数据访问层-src_repositories)
  - [4.8 存储层 `src/storage.py`](#48-存储层-src_storagepy)
  - [4.9 通知系统 `src/notification*.py` + `src/notification_sender/`](#49-通知系统)
  - [4.10 搜索与情报 `src/search_service.py`](#410-搜索与情报-src_search_servicepy)
  - [4.11 API 层 `api/`](#411-api-层-api)
  - [4.12 机器人模块 `bot/`](#412-机器人模块-bot)
  - [4.13 调度与任务 `src/scheduler.py` + `src/services/task_*.py`](#413-调度与任务)
  - [4.14 配置体系 `src/config.py` + `src/core/config_manager.py`](#414-配置体系)
  - [4.15 认证模块 `src/auth.py`](#415-认证模块-src_authpy)
  - [4.16 报告渲染 `templates/` + `src/services/report_renderer.py`](#416-报告渲染)
  - [4.17 Web 前端 `apps/dsa-web/`](#417-web-前端-apps_dsa_web)
  - [4.18 桌面端 `apps/dsa-desktop/`](#418-桌面端-apps_dsa_desktop)
- [5. 依赖关系总览](#5-依赖关系总览)
- [6. 项目运行方式](#6-项目运行方式)
- [7. CI/CD 与部署](#7-cicd-与部署)
- [8. 测试体系](#8-测试体系)

---

## 1. 项目定位与主流程

**项目定位**：股票智能分析系统（DSA，Daily Stock Analysis），基于 AI 大模型对 A 股、港股、美股、日股、韩股、台股及 ETF 进行自动化分析，生成「决策仪表盘」报告并推送到多种通知渠道。

**主流程**：

```
抓取数据 → 技术分析 / 新闻检索 → LLM 综合分析 → 生成报告 → 通知推送 → (可选) 回测与决策信号沉淀
```

**关键入口**：

| 入口 | 作用 |
|------|------|
| [main.py](../../main.py) | CLI / 定时任务 / FastAPI 服务主入口 |
| [server.py](../../server.py) | FastAPI 后端独立服务入口（uvicorn） |
| [api/app.py](../../api/app.py) | FastAPI 应用工厂 |
| [apps/dsa-web/](../../apps/dsa-web/) | Web 前端（React + Vite） |
| [apps/dsa-desktop/](../../apps/dsa-desktop/) | Electron 桌面端 |
| [.github/workflows/](../../.github/workflows/) | CI、发布、每日定时任务 |

---

## 2. 顶层架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        触发层 (Triggers)                          │
│  CLI (main.py)  │  GitHub Actions  │  Docker/定时  │  Bot/Web    │
└──────────┬──────────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────┐
│                    编排层 (Orchestration)                         │
│   StockAnalysisPipeline  │  MarketAnalyzer  │  AgentOrchestrator │
│   run_market_review      │  RuntimeSchedulerService              │
└──────────┬──────────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────┐
│                      服务层 (Services)                            │
│  AnalysisService │ DecisionSignalService │ BacktestService       │
│  PortfolioService│ AlertService/Worker   │ IntelligenceService   │
│  HistoryService  │ SystemConfigService   │ TaskService/Queue     │
└──────────┬──────────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────┐
│            数据与计算层 (Data & Compute)                          │
│  DataFetcherManager (多源 fallback) │ StockTrendAnalyzer(技术)   │
│  GeminiAnalyzer(LLM) │ SearchService(新闻) │ SocialSentiment     │
└──────────┬──────────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────┐
│              基础设施层 (Infrastructure)                          │
│  DatabaseManager(SQLite) │ ConfigManager(.env) │ TradingCalendar │
│  NotificationService(14渠道) │ AuthMiddleware │ LLM Backend抽象   │
└─────────────────────────────────────────────────────────────────┘
```

**设计原则**：

- **多源 fallback**：单一数据源/通知渠道失败不拖垮整体流程。
- **低并发防封**：默认 `max_workers=3`，数据源内置随机休眠与熔断器。
- **fail-open 优先**：基本面、社交舆情等增强能力失败时降级而非中断。
- **配置即真源**：`.env` 为唯一配置真源，WebUI/CLI/Docker 共用。
- **断点续传**：按交易日判断是否已存在数据，避免重复抓取。

---

## 3. 目录结构与边界

```
/workspace
├── main.py                    # CLI/调度/服务主入口
├── server.py                  # FastAPI 独立入口
├── api/                       # FastAPI 层（端点、中间件、Schema）
├── apps/
│   ├── dsa-web/               # React Web 前端
│   └── dsa-desktop/           # Electron 桌面端
├── bot/                       # 机器人（命令、平台适配）
├── data_provider/             # 多数据源适配与 fallback
├── docker/                    # Docker 部署
├── scripts/                   # 本地脚本（CI、构建、索引生成）
├── src/
│   ├── agent/                 # Agent 编排系统
│   ├── core/                  # 主流程编排、配置管理、交易日历
│   ├── data/                  # 股票索引/映射
│   ├── llm/                   # LLM 后端抽象
│   ├── notification_sender/   # 14 种通知发送器
│   ├── patches/               # 第三方库补丁
│   ├── repositories/          # 数据访问层（薄封装）
│   ├── schemas/               # Schema / 数据结构
│   ├── services/              # 业务服务层
│   ├── utils/                 # 工具函数
│   ├── analyzer.py            # LLM 分析器（GeminiAnalyzer）
│   ├── config.py              # Config 配置类
│   ├── stock_analyzer.py      # 技术分析器（StockTrendAnalyzer）
│   ├── market_analyzer.py     # 市场复盘分析器
│   ├── storage.py             # 数据库存储层
│   ├── notification.py        # 通知服务核心
│   ├── scheduler.py           # 定时调度
│   └── ...
├── strategies/                # 15 个内置策略 YAML
├── templates/                 # Jinja2 报告模板
├── tests/                     # pytest 测试
└── docs/                      # 文档
```

**目录边界约定**（来自 [AGENTS.md](../../AGENTS.md)）：

- 后端逻辑：`src/`、`data_provider/`、`api/`、`bot/`
- Web 前端：`apps/dsa-web/`
- 桌面端：`apps/dsa-desktop/`
- 部署与流水线：`scripts/`、`.github/workflows/`、`docker/`

---

## 4. 核心模块详解

### 4.1 主流程编排 `src/core/`

#### `StockAnalysisPipeline`（[src/core/pipeline.py](../../src/core/pipeline.py)）

个股分析主流程调度器，约 161KB 单类大型调度器。

**构造参数**：

```python
StockAnalysisPipeline(
    config: Optional[Config] = None,
    max_workers: Optional[int] = None,
    source_message: Optional[BotMessage] = None,
    query_id: Optional[str] = None,
    query_source: Optional[str] = None,        # cli/bot/api
    save_context_snapshot: Optional[bool] = None,
    analysis_skills: Optional[List[str]] = None,
    analysis_phase: str = "auto",              # auto/premarket/intraday/postmarket
    portfolio_context: Optional[Dict] = None,
    daily_market_context_enabled: Optional[bool] = None,
)
```

**关键方法**：

| 方法 | 职责 |
|------|------|
| `run(stock_codes, dry_run, send_notification, merge_notification, current_time)` | 主流程编排：预取数据 → 线程池并发分析 → 单股/汇总推送 |
| `process_single_stock(code, ...)` | 线程池任务入口：获取数据 → AI 分析 → 单股推送 |
| `analyze_stock(code, report_type, query_id, current_time)` | 增强单股分析：实时行情、筹码、趋势、情报、AI 综合 |
| `fetch_and_save_stock_data(code, force_refresh, current_time)` | 断点续传获取并入库日线数据 |

**`run()` 编排逻辑**：

1. `config.refresh_stock_list()` 热读取自选股
2. 股票数 ≥ 5 时批量预取日 K 与实时行情（`prefetch_daily_klines`、`prefetch_realtime_quotes`）
3. `prefetch_stock_names` 预取股票名称
4. `ThreadPoolExecutor(max_workers)` 并发提交 `process_single_stock`
5. `as_completed` 收集结果，单股推送模式下串行通知
6. `_save_local_report` 保存汇总报告，按配置决定推送策略

**组装的子模块**：`DataFetcherManager`、`StockTrendAnalyzer`、`GeminiAnalyzer`、`NotificationService`、`SearchService`（可选）、`SocialSentimentService`（可选）、`DatabaseManager`。

#### 大盘复盘 `run_market_review`（[src/core/market_review.py](../../src/core/market_review.py)）

模块级函数，非类方法。

```python
def run_market_review(
    notifier, analyzer=None, search_service=None, config=None,
    send_notification=True, merge_notification=False,
    override_region=None, query_id=None, trigger_source="cli",
) -> Optional[str] | Optional[MarketReviewRunResult]
```

**职责**：

1. `_resolve_market_review_regions` 解析 `MARKET_REVIEW_REGION`（cn/hk/us/jp/kr/both 或逗号子集）
2. 多区域：循环为每个市场创建 `MarketAnalyzer`，调用 `run_daily_review_with_snapshot()`，合并报告
3. 渲染 Markdown，可选保存 `market_review_YYYYMMDD.md`
4. `_persist_market_review_history` 持久化到 `analysis_history`（`code=MARKET`，`report_type=market_review`）
5. 推送通知（`merge_notification=True` 时跳过独立推送）

#### 关键支撑模块

| 模块 | 核心类/函数 | 职责 |
|------|------------|------|
| [market_profile.py](../../src/core/market_profile.py) | `MarketProfile` + `get_profile(region)` | 区域能力位：`mood_index_code`、`news_queries`、`has_market_stats`、`has_sector_rankings`（仅 cn 有涨跌家数/板块榜） |
| [trading_calendar.py](../../src/core/trading_calendar.py) | `MarketPhase` 枚举、`get_market_for_stock(code)`、`is_market_open`、`get_effective_trading_date`、`infer_market_phase`、`compute_effective_region` | 交易日历与市场阶段判断；依赖 `exchange_calendars`（未装时 fail-open） |
| [market_strategy.py](../../src/core/market_strategy.py) | `MarketStrategyBlueprint` + `get_market_strategy_blueprint(region)` | 各区域策略蓝图，`to_prompt_block()` 渲染为 LLM Prompt |
| [market_review_lock.py](../../src/core/market_review_lock.py) | `try_acquire_market_review_lock` / `release_market_review_lock` | 大盘复盘并发互斥锁 |
| [config_manager.py](../../src/core/config_manager.py) | `ConfigManager` | `.env` 原子读写，支持乐观版本控制与 Docker Compose 转义 |

---

### 4.2 分析引擎

#### LLM 分析器 `GeminiAnalyzer`（[src/analyzer.py](../../src/analyzer.py)，约 199KB）

通过 LiteLLM Router 统一接入 Gemini/Anthropic/OpenAI/DeepSeek/Ollama/Hermes 等多渠道。

**关键方法**：

```python
GeminiAnalyzer(config=...).analyze(
    context: Dict[str, Any],
    news_context: Optional[str] = None,
    progress_callback=None,
    analysis_context_pack_summary=None,
) -> AnalysisResult

GeminiAnalyzer(config=...).generate_text(
    prompt: str, max_tokens=2048, temperature=0.7
) -> Optional[str]   # MarketAnalyzer 必须用此入口
```

**`AnalysisResult` 数据类核心字段**：

- `code`、`name`、`sentiment_score`（0-100 综合评分）、`trend_prediction`、`operation_advice`
- `decision_type`（buy/hold/sell）、`confidence_level`、`action`、`action_label`
- `dashboard: Optional[Dict]`（决策仪表盘完整数据）
- `analysis_summary`、`key_points`、`risk_warning`、`buy_reason`
- `current_price`、`change_pct`、`model_used`、`query_id`

**模块级辅助函数**：`stabilize_decision_with_structure`（结构化决策稳定化）、`populate_decision_action_fields`、`get_analyzer()`（全局访问器）。

#### 技术分析器 `StockTrendAnalyzer`（[src/stock_analyzer.py](../../src/stock_analyzer.py)）

注意：这是**技术分析器**，非 LLM 分析器。

**交易理念**（融入分析）：

1. 严进策略：不追高，乖离率 `(Close-MA5)/MA5 < 5%`
2. 趋势交易：MA5 > MA10 > MA20 多头排列
3. 偏好缩量回调买点

**关键枚举**：`TrendStatus`、`VolumeStatus`、`BuySignal`、`MACDStatus`、`RSIStatus`。

**综合评分系统**（`_generate_signal`，0-100 分）：

| 维度 | 权重 | 评分逻辑 |
|------|------|---------|
| 趋势 | 30 分 | STRONG_BULL=30 … STRONG_BEAR=0 |
| 乖离率 | 20 分 | 受 `config.bias_threshold` 控制，强势趋势阈值放宽 ×1.5 |
| 量能 | 15 分 | 缩量回调=15（最佳），放量下跌=0 |
| 支撑 | 10 分 | MA5/MA10 支撑各 5 分（容忍度 2%） |
| MACD | 15 分 | 零轴上金叉=15，死叉=0 |
| RSI | 10 分 | 超卖=10，超买=0 |

**关键方法**：`analyze(df, code) -> TrendAnalysisResult`、`_calculate_mas`、`_calculate_macd`（12/26/9）、`_calculate_rsi`（Wilder EMA，6/12/24）。

#### 市场复盘分析器 `MarketAnalyzer`（[src/market_analyzer.py](../../src/market_analyzer.py)）

**核心编排方法** `run_daily_review_with_snapshot()`：

```
get_market_overview()  →  search_market_news()  →  _merge_persisted_market_intelligence()
        ↓
generate_market_review()  →  build_market_light_snapshot()  →  build_market_review_payload()
```

**市场温度计**（`build_market_light_snapshot`）：基于 breadth(45%) + index(35%) + limit(20%) 加权计算 0-100 分数，映射 green/yellow/red 状态与「可进攻/需观察/偏防守」标签。

---

### 4.3 数据源适配层 `data_provider/`

#### 基类与策略管理器（[data_provider/base.py](../../data_provider/base.py)，约 155KB）

采用**策略模式**。

**`BaseFetcher(ABC)` 抽象基类**：

```python
class BaseFetcher(ABC):
    name: str = "BaseFetcher"
    priority: int = 99              # 数字越小越优先
    allow_empty_daily_data: bool = False

    @abstractmethod
    def _fetch_raw_data(self, stock_code, start_date, end_date) -> pd.DataFrame: ...
    @abstractmethod
    def _normalize_data(self, df, stock_code) -> pd.DataFrame: ...

    def get_daily_data(self, stock_code, start_date=None, end_date=None, days=30) -> pd.DataFrame: ...
    def get_main_indices(self, region="cn"): ...       # 默认返回 None
    def get_market_stats(self): ...                     # 涨跌统计
    def get_sector_rankings(self, n=5): ...             # 板块涨跌榜
    def get_concept_rankings(self, n=5): ...
    def get_hot_stocks(self, n=10): ...
    def get_limit_up_pool(self, date=None, n=20): ...
```

**`DataFetcherManager` 策略管理器**：

- `_init_default_fetchers()`：按配置初始化数据源链
- `get_daily_data(code, ...) -> Tuple[pd.DataFrame, str]`：日线获取主入口，按市场路由 + failover
- `get_realtime_quote(code)`：实时行情聚合
- `get_chip_distribution(code)`：筹码分布
- `get_stock_name(code)`：股票名称查询（带缓存）
- `get_fundamental_context(code, budget_seconds)`：基本面上下文（fail-open，分块：valuation/growth/earnings/institution/capital_flow/dragon_tiger/boards/coverage）
- 熔断器：`CircuitBreaker(failure_threshold=3, cooldown_seconds=300.0)`

**市场路由**：

| 市场 | 路由策略 |
|------|---------|
| 美股指数 | YfinanceFetcher 首选 |
| 美股 | 配置长桥时 `Longbridge→Finnhub→AlphaVantage→Yfinance` |
| 港股/日韩台/A股 | 通用 fetcher 循环（按 priority 排序 failover） |

#### 各数据源 Fetcher

| Fetcher | 优先级 | 市场 | 数据来源 | 特性 |
|---------|--------|------|---------|------|
| `EfinanceFetcher` | 0 | cn | 东方财富（efinance 库） | 免费无 Token，支持板块榜 |
| `TencentFetcher` | 0 | cn | 腾讯直连行情 | 前复权日线，最多 800 根 |
| `AkshareFetcher` | 1 | cn/hk | 东方财富 + 新浪 + 腾讯 | 数据最全面（含筹码/涨停池） |
| `TushareFetcher` | -1/2 | cn/hk | Tushare Pro API | 需 Token，80次/分流控 |
| `PytdxFetcher` | 2 | cn | 通达信服务器 | 免费无 Token，多服务器切换 |
| `BaostockFetcher` | 3 | cn | 证券宝 API | 免费无 Token，T+1 延迟 |
| `YfinanceFetcher` | 4 | 全市场 | Yahoo Finance | 兜底，覆盖最广 |
| `LongbridgeFetcher` | 5 | hk/us | 长桥 OpenAPI | 需 OAuth/Legacy 凭证 |
| `FinnhubFetcher` | 2 | us | Finnhub.io | 60 calls/min |
| `AlphaVantageFetcher` | 3 | us | AlphaVantage | 25 calls/day |
| `TickFlowFetcher` | 2 | cn | TickFlow API | 可选，支持批量日线/多复权 |
| `TwInstitutionalFetcher` | - | tw | 台湾政府开放资料 | 独立模块，三大法人买卖超 |

**工具函数**：`normalize_stock_code(code)`（统一代码格式）、`canonical_stock_code(code)`、`_market_tag(code)`、`is_bse_code(code)`。

---

### 4.4 Agent 编排系统 `src/agent/`

多 Agent ReAct 编排系统，支持 Web/Bot/API 三端。

#### 核心编排

| 文件 | 核心类 | 职责 |
|------|--------|------|
| [orchestrator.py](../../src/agent/orchestrator.py) | `AgentOrchestrator` | 多 Agent 编排：Technical → Intel → Risk → Specialist → Decision，`_apply_risk_override()` 风控降级 |
| [executor.py](../../src/agent/executor.py) | `AgentExecutor` | 单 Agent 执行器，对外暴露 `chat()` / `chat_stream()`（SSE） |
| [runner.py](../../src/agent/runner.py) | `run_agent_loop()` | 共享 ReAct 循环，含预算守卫（stage 15s / step 8s） |
| [factory.py](../../src/agent/factory.py) | `build_agent_executor()` / `build_orchestrator()` | 工厂与单例缓存 |
| [research.py](../../src/agent/research.py) | `ResearchAgent` | 深度研究：分解 → 逐子问题研究 → 综合报告 |

#### 共享协议

| 文件 | 核心类型 | 职责 |
|------|---------|------|
| [protocols.py](../../src/agent/protocols.py) | `Signal` 枚举、`AgentContext`、`AgentOpinion`、`StageResult` | 共享状态袋与结构化输出 |
| [chat_context.py](../../src/agent/chat_context.py) | `AgentChatContextBundle` | 可见对话历史构建 + 滚动摘要压缩 |
| [provider_trace.py](../../src/agent/provider_trace.py) | `ProviderTraceTurn` | Provider 协议 trace 隔离（thinking/tool-call） |
| [litellm_route_resolution.py](../../src/agent/litellm_route_resolution.py) | `resolve_agent_litellm_route()` | Agent 安全路由（排除 Hermes-only 部署） |
| [stock_scope.py](../../src/agent/stock_scope.py) | `resolve_stock_scope()` | 股票作用域：maintain/switch/compare |

#### Agents（按编排顺序）

| Agent | 职责 |
|-------|------|
| `TechnicalAgent` | 技术面分析（趋势/形态/指标） |
| `IntelAgent` | 情报面分析（新闻/公告/舆情） |
| `RiskAgent` | 风险评估，产出 risk_flags |
| `PortfolioAgent` | 组合视角叠加（多股对比时） |
| `DecisionAgent` | 决策汇总，综合各 opinion 输出最终信号 |

#### Tools（`@tool` 装饰器注册）

| 文件 | 工具 |
|------|------|
| [data_tools.py](../../src/agent/tools/data_tools.py) | get_stock_info / get_realtime_quote / get_daily_history |
| [analysis_tools.py](../../src/agent/tools/analysis_tools.py) | 技术分析（趋势/支撑压力/筹码） |
| [search_tools.py](../../src/agent/tools/search_tools.py) | search_stock_news / search_comprehensive_intel |
| [market_tools.py](../../src/agent/tools/market_tools.py) | get_market_indices / get_sector_rankings |
| [backtest_tools.py](../../src/agent/tools/backtest_tools.py) | 回测相关 |
| [registry.py](../../src/agent/tools/registry.py) | `ToolRegistry`，导出 OpenAI 格式工具声明 |

#### Skills / Strategies

- **skills/**：可插拔交易技能，`SkillManager` 加载 YAML / SKILL.md frontmatter，按 `default_priority` 排序
- **strategies/**：按维度分组（trend/pattern/reversal/framework），与 skills 类似
- 自定义目录通过 `AGENT_SKILL_DIR` 环境变量指定

---

### 4.5 LLM 后端抽象 `src/llm/`

| 文件 | 核心类型 | 职责 |
|------|---------|------|
| [generation_backend.py](../../src/llm/generation_backend.py) | `GenerationBackend`(Protocol)、`GenerationResult`、`GenerationError`、`GenerationCapabilities` | 后端统一契约 |
| [backend_registry.py](../../src/llm/backend_registry.py) | 后端 ID 常量 + resolver | 后端注册 |
| [backend_factory.py](../../src/llm/backend_factory.py) | `create_generation_backend(config, backend_id)` | 按 ID 实例化后端 |
| [litellm_backend.py](../../src/llm/litellm_backend.py) | `LiteLLMGenerationBackend` | LiteLLM 调用薄适配器 |
| [local_cli_backend.py](../../src/llm/local_cli_backend.py) | `LocalCliGenerationBackend` | Codex/Claude Code/OpenCode CLI 沙箱（read-only + 白名单 env + shell 元字符拦截） |
| [generation_params.py](../../src/llm/generation_params.py) | `TemperatureDirective`、`GenerationParamRecovery` | 温度/wire model 解析与参数恢复 |
| [errors.py](../../src/llm/errors.py) | `classify_litellm_generation_param_error()`、`call_litellm_with_param_recovery()` | 参数错误分类与一次性重试 |
| [provider_cache.py](../../src/llm/provider_cache.py) | `ProviderCacheCaps`、`PROVIDER_CACHE_REGISTRY`(14 provider) | Provider 缓存能力画像 + 安全注入 cache hint（HMAC 脱敏） |
| [hermes.py](../../src/llm/hermes.py) | Hermes 本地 OpenAI 兼容通道 | loopback HTTP，`build_route_provenance_map()` |
| [usage.py](../../src/llm/usage.py) | `normalize_litellm_usage()`、`persist_llm_usage()` | 用量标准化与 HMAC 遥测 |

**支持 15+ 渠道**：Gemini、OpenAI、DeepSeek、Anthropic、AIHubMix、Moonshot、DashScope、智谱、MiniMax、火山、SiliconFlow、OpenRouter、Ollama、Hermes、Anspire。

---

### 4.6 业务服务层 `src/services/`

| 服务 | 核心类 | 职责 |
|------|--------|------|
| [analysis_service.py](../../src/services/analysis_service.py) | `AnalysisService` | API 调用入口，封装 `analyze_stock()` |
| [task_service.py](../../src/services/task_service.py) | `TaskService` | 轻量异步任务（Bot 触发） |
| [task_queue.py](../../src/services/task_queue.py) | `AnalysisTaskQueue` | API/SSE 任务队列 + 防重 + SSE 广播 |
| [runtime_scheduler.py](../../src/services/runtime_scheduler.py) | `RuntimeSchedulerService` | API/Web 运行时调度器，共享 `_RUNTIME_ANALYSIS_LOCK` |
| [history_service.py](../../src/services/history_service.py) | `HistoryService` | 历史查询、Markdown 重建 |
| [history_loader.py](../../src/services/history_loader.py) | `load_history_df()` | DB-first K线加载（修复 #1066） |
| [decision_signal_service.py](../../src/services/decision_signal_service.py) | `DecisionSignalService` | 决策信号资产：创建/查询/过期/互斥失效 |
| [decision_signal_extractor.py](../../src/services/decision_signal_extractor.py) | 信号提取器 | 从分析报告提取决策信号 |
| [backtest_service.py](../../src/services/backtest_service.py) | `BacktestService` | 回测执行与汇总 |
| [portfolio_service.py](../../src/services/portfolio_service.py) | `PortfolioService` | 持仓管理（FIFO/均价、汇率、快照） |
| [portfolio_risk_service.py](../../src/services/portfolio_risk_service.py) | - | 持仓风险评估 |
| [alert_service.py](../../src/services/alert_service.py) | `AlertService` | 告警规则 CRUD + dry-run |
| [alert_worker.py](../../src/services/alert_worker.py) | `AlertWorker` | 后台轮询 worker（指纹去重 + DB 冷却） |
| [intelligence_service.py](../../src/services/intelligence_service.py) | `IntelligenceService` | 合规资讯源（RSS/Atom/NewsNow）抓取 |
| [stock_service.py](../../src/services/stock_service.py) | `StockService` | 股票数据 API 入口 |
| [system_config_service.py](../../src/services/system_config_service.py) | `SystemConfigService` | `.env` 配置读写/校验/导入导出 + LLM/通知测试 |
| [run_flow.py](../../src/services/run_flow.py) | `build_task_run_flow_snapshot()` | 运行流（RunFlow）快照构建 |
| [report_renderer.py](../../src/services/report_renderer.py) | - | Jinja2 报告渲染 |
| [daily_market_context.py](../../src/services/daily_market_context.py) | `DailyMarketContextService` | 大盘上下文复用 |
| [social_sentiment_service.py](../../src/services/social_sentiment_service.py) | `SocialSentimentService` | 美股社交舆情（Reddit/X/Polymarket） |
| [image_stock_extractor.py](../../src/services/image_stock_extractor.py) | - | 图片识别股票（Vision LLM） |
| [import_parser.py](../../src/services/import_parser.py) | - | CSV/Excel/剪贴板导入解析 |

**决策信号（核心资产）**：连接「分析报告 → 持仓 → 回测 → 告警」，有 horizon 过期、与 opposing action 互斥失效机制。

---

### 4.7 数据访问层 `src/repositories/`

薄封装数据访问层，无业务逻辑，均接受 `db_manager` 参数（默认单例）。

| Repo | 类 | 职责 |
|------|-----|------|
| [analysis_repo.py](../../src/repositories/analysis_repo.py) | `AnalysisRepository` | `AnalysisHistory` 表操作 |
| [stock_repo.py](../../src/repositories/stock_repo.py) | `StockRepository` | `StockDaily` 表操作 |
| [portfolio_repo.py](../../src/repositories/portfolio_repo.py) | `PortfolioRepository` | 持仓 P0 域 DB 访问（账户/事件/快照） |
| [alert_repo.py](../../src/repositories/alert_repo.py) | `AlertRepository` | 告警 P1 表访问 |
| [decision_signal_repo.py](../../src/repositories/decision_signal_repo.py) | `DecisionSignalRepository` | 决策信号 DB（`create_if_absent` 幂等） |
| [intelligence_repo.py](../../src/repositories/intelligence_repo.py) | `IntelligenceRepository` | 资讯源/条目 DB |
| [backtest_repo.py](../../src/repositories/backtest_repo.py) | `BacktestRepository` | 回测候选/结果 DB |

---

### 4.8 存储层 `src/storage.py`

`DatabaseManager`（单例，元类 `_DatabaseManagerMeta`）。

**SQLite 配置**：

- 自动启用 WAL 模式 + `busy_timeout`
- 写事务统一走 `_run_write_transaction()`：`BEGIN IMMEDIATE` 抢占写锁，遇 `database is locked` 指数退避重试
- `CURRENT_SCHEMA_VERSION = "2026-06-05-create-all-baseline"`

**关键方法**：

```python
has_today_data(code, target_date=None) -> bool                    # 断点续传检查
save_daily_data(df, code, data_source="Unknown") -> int           # UPSERT，分块 50
save_analysis_history(result, query_id, report_type, ...) -> int  # 返回 history.id
get_analysis_history(code=None, query_id=None, days=30, ...) -> List[AnalysisHistory]
save_news_intel(code, name, dimension, query, response, ...) -> int
save_conversation_message(...) / get_conversation_history(...)
record_llm_usage(call_type, model, prompt_tokens, ...)            # LLM 用量审计
```

**表结构（约 28 张表）**：

| 表名 | 职责 |
|------|------|
| `stock_daily` | 日线行情 + 技术指标，唯一约束 `(code, date)` |
| `analysis_history` | 分析结果历史（含狙击点位列） |
| `news_intel` | 新闻情报条目，唯一约束 `url` |
| `intelligence_sources` / `intelligence_items` | 资讯源 / 沉淀情报 |
| `fundamental_snapshot` | 基本面上下文快照（write-only） |
| `backtest_results` / `backtest_summaries` | 回测结果 / 汇总 |
| `portfolio_accounts` / `portfolio_trades` / `portfolio_cash_ledger` / `portfolio_corporate_actions` / `portfolio_positions` / `portfolio_position_lots` / `portfolio_daily_snapshots` / `portfolio_fx_rates` | 持仓体系 |
| `conversation_messages` / `conversation_summaries` / `agent_provider_turns` | Agent 对话 |
| `llm_usage` | LLM token 用量审计 |
| `alert_rules` / `alert_triggers` / `alert_notifications` / `alert_cooldowns` | 告警体系 |
| `decision_signals` / `decision_signal_outcomes` / `decision_signal_feedback` | 决策信号体系 |
| `schema_migrations` | schema 版本标记 |

---

### 4.9 通知系统

#### `NotificationService`（[src/notification.py](../../src/notification.py)，2800+ 行）

**多继承 14 个 Sender**：

```python
class NotificationService(WechatSender, FeishuSender, DingtalkSender, TelegramSender,
                          EmailSender, PushoverSender, NtfySender, GotifySender,
                          PushplusSender, Serverchan3Sender, CustomWebhookSender,
                          DiscordSender, SlackSender, AstrbotSender):
```

**14 渠道**：WECHAT / DINGTALK / FEISHU / TELEGRAM / EMAIL / PUSHOVER / NTFY / GOTIFY / PUSHPLUS / SERVERCHAN3 / CUSTOM / DISCORD / SLACK / ASTRBOT。

**关键能力**：

- `detect_configured_channels()`：静态检测已配置渠道
- `get_channels_for_route(route_type)`：按路由类型（report/alert/system_error）过滤
- 上下文渠道优先（钉钉 sessionWebhook / 飞书 chat_id / Telegram chat_id），无上下文时广播静态渠道
- `generate_dashboard_report()` / `generate_brief_report()` / `generate_single_stock_report()`：多种报告格式
- `send_with_results()` 返回 `NotificationDispatchResult`（per-channel 诊断）
- Markdown 转图片降级（`_should_use_image_for_channel`）

#### 路由与能力画像

| 文件 | 职责 |
|------|------|
| [notification_routing.py](../../src/notification_routing.py) | 三类路由配置（report/alert/system_error） |
| [notification_capabilities.py](../../src/notification_capabilities.py) | 14 渠道能力画像（`ChannelProfile`：max_text_chars/supports_card/supports_image 等） |
| [notification_noise.py](../../src/notification_noise.py) | 噪声控制 |
| [notification_contracts.py](../../src/notification_contracts.py) | 通知契约 |

#### 各 Sender（`src/notification_sender/`）

| Sender | 渠道 | 特性 |
|--------|------|------|
| `WechatSender` | 企业微信 | markdown 4096B / text 2000B 自动分批，图片 2MB |
| `FeishuSender` | 飞书 | Webhook + App Bot 双模式，卡片优先，20000B 分批 |
| `DingtalkSender` | 钉钉 | Webhook + secret 签名 |
| `TelegramSender` | Telegram | Bot API |
| `EmailSender` | 邮件 | SMTP |
| `DiscordSender` / `SlackSender` | Discord / Slack | Webhook |
| 其他 | Pushover/Ntfy/Gotify/Pushplus/Server酱3/Custom/Astrbot | 各自协议 |

---

### 4.10 搜索与情报 `src/search_service.py`

**Provider 体系**（继承 `BaseSearchProvider(ABC)`，多 key 轮询负载均衡）：

- `AnspireSearchProvider`（最优先）、`BochaSearchProvider`、`TavilySearchProvider`、`BraveSearchProvider`、`SerpAPISearchProvider`、`MiniMaxSearchProvider`、`SearXNGSearchProvider`（实例轮换 + 公共实例发现）

**`SearchService` 主类**：

- `search_stock_news(stock_code, ...)`：股票新闻搜索
- `search_stock_events(stock_code, ...)`：事件搜索
- `search_stock_with_enhanced_fallback(stock_code, ...)`：增强 fallback
- 内存缓存（TTL 600s）+ inflight 去重
- 维度常量：`ANALYTICAL_INTEL_LOOKBACK_DAYS=180`、`ANALYTICAL_INTEL_DIMENSIONS={"market_analysis","earnings"}`
- 与 `run_diagnostics` 集成：`record_provider_run`

---

### 4.11 API 层 `api/`

#### 应用入口（[api/app.py](../../api/app.py)）

FastAPI 工厂 + `lifespan` 上下文管理（启动 `RuntimeSchedulerService` + 加载 `SystemConfigService`）+ CORS + 中间件 + v1 router。

#### 中间件

| 文件 | 类 | 职责 |
|------|-----|------|
| [auth.py](../../api/middlewares/auth.py) | `AuthMiddleware` | Cookie 会话认证，`ADMIN_AUTH_ENABLED=true` 时校验 |
| [error_handler.py](../../api/middlewares/error_handler.py) | `ErrorHandlerMiddleware` | 统一异常处理，返回 `{error, message, detail}` |

#### 端点（`api/v1/endpoints/`，14 个）

| 端点 | 核心路由 | 职责 |
|------|---------|------|
| `analysis.py` | POST /analyze、POST /market-review、GET /tasks、GET /tasks/stream（SSE）、GET /tasks/{id}/flow | 股票分析主入口 + SSE 实时推送 |
| `agent.py` | GET /models、GET /skills、POST /chat、POST /chat/stream（SSE）、POST /research | Agent 对话/技能/深研 |
| `alerts.py` | /rules CRUD、/triggers、/notifications | 告警规则 + 触发历史 |
| `alphasift.py` | /status、/strategies、/hotspots、/screen | AlphaSift 选股（异步） |
| `auth.py` | /status、/settings、/login、/change-password、/logout | 管理员认证 |
| `backtest.py` | /run、/results、/performance | 回测 |
| `decision_signals.py` | / CRUD、/outcomes、/reassess、/feedback | 决策信号 |
| `health.py` | GET /health | 健康检查 |
| `history.py` | / 列表、/by-code、/{id} 详情、/{id}/markdown | 历史记录 |
| `intelligence.py` | /sources、/items | 情报源管理 |
| `portfolio.py` | /accounts、/trades、/cash-ledger、/corporate-actions、/snapshot、/imports、/fx、/risk | 投资组合 |
| `stocks.py` | /extract-from-image、/parse-import、/watchlist、/{code}/quote、/{code}/history | 股票数据 + 自选队列 |
| `system_config.py` | /config、/config/import、/llm/test-channel、/notification/test-channel、/scheduler | 系统配置 |
| `usage.py` | /summary、/dashboard | LLM 用量统计 |

**SSE 事件类型**（analysis tasks）：`connected` / `task_created` / `task_started` / `task_progress` / `task_completed` / `task_failed` / `heartbeat`（30s 心跳）。

**SSE 事件类型**（agent chat）：`thinking` / `stage_start` / `stage_done` / `tool_start` / `tool_done` / `generating` / `done` / `error`。

---

### 4.12 机器人模块 `bot/`

#### 核心

| 文件 | 核心类 | 职责 |
|------|--------|------|
| [dispatcher.py](../../bot/dispatcher.py) | `CommandDispatcher` | 命令注册/分发，限流，自然语言路由（LLM 意图解析） |
| [handler.py](../../bot/handler.py) | `handle_webhook()` / `handle_webhook_async()` | 平台适配器获取 + 延迟响应 |
| [models.py](../../bot/models.py) | `BotMessage` / `BotResponse` | 消息模型（中文命令映射：分析→analyze, 大盘→market） |

#### 命令（`bot/commands/`，10 个）

| 命令 | 别名 | 职责 |
|------|------|------|
| `AnalyzeCommand` | analyze / a / 分析 / 查 | 单股分析 |
| `AskCommand` | ask / 问股 | 策略问股（支持多股并行 + 技能） |
| `BatchCommand` | batch / b / 批量 | 后台批量分析 |
| `ChatCommand` | chat / c / 问 | Agent 对话 |
| `MarketCommand` | market / m / 大盘 / 复盘 | 大盘复盘（防重锁） |
| `ResearchCommand` | research / 深研 | 深度研究 |
| `HistoryCommand` | history / 历史 | 对话历史 |
| `StatusCommand` | status / s / 状态 | 系统状态 |
| `StrategiesCommand` | strategies / skills / 策略 | 策略列表 |
| `HelpCommand` | help / h / 帮助 | 帮助 |

#### 平台（`bot/platforms/`）

| 平台 | 类 | 模式 |
|------|-----|------|
| 钉钉 | `DingtalkPlatform` | Webhook（HMAC 签名） |
| 钉钉 Stream | `DingtalkStreamClient` | WebSocket 长连接（无需公网 IP） |
| Discord | `DiscordPlatform` | Ed25519 签名 + 延迟响应（type=5） |
| 飞书 Stream | `FeishuStreamClient` | lark-oapi SDK，per-conversation FIFO |

---

### 4.13 调度与任务

#### 定时调度（[src/scheduler.py](../../src/scheduler.py)）

- `normalize_schedule_times(schedule_times, fallback_time="18:00")`：解析多时间点
- `Scheduler` 类：基于 `schedule` 库，支持 `schedule_time_provider` 运行时热更新
- `run_with_schedule(task, ...)`：便捷入口
- 后台任务：`add_background_task(task, interval_seconds, ...)`，间隔 < 30s 自动 clamp

#### 运行时调度器（[src/services/runtime_scheduler.py](../../src/services/runtime_scheduler.py)）

`RuntimeSchedulerService`：面向长生命周期 API/Web/Desktop 进程。

- 共享 `_RUNTIME_ANALYSIS_LOCK`，避免并发分析
- `start(run_immediately)` / `stop()` / `run_now()` / `status()`
- 后台任务：`AlertWorker.run_once`（默认 5 分钟间隔）

#### 任务队列（[src/services/task_queue.py](../../src/services/task_queue.py)）

`AnalysisTaskQueue`（单例）：

- `TaskStatus`：PENDING / PROCESSING / COMPLETED / FAILED / CANCEL_REQUESTED / CANCELLED
- `submit_task()` / `submit_tasks_batch()`：去重键 `_dedupe_stock_code_key`，重复抛 `DuplicateTaskError`
- SSE 广播：`subscribe(queue)` / `unsubscribe(queue)` / `_broadcast_event()`
- `sync_max_workers()`：繁忙时延后调整

---

### 4.14 配置体系

#### `Config`（[src/config.py](../../src/config.py)，约 151KB）

`@dataclass` 单例模式，关键字段分组：

| 分组 | 关键字段 |
|------|---------|
| 自选股 | `stock_list` |
| 数据源 | `tushare_token`、`tickflow_api_key`、`finnhub_api_key`、`longbridge_*`、`alphasift_enabled` |
| LLM | `generation_backend`、`litellm_model`、`llm_channels`、`llm_model_list`、`llm_temperature` |
| 多 Key 池 | `gemini_api_keys`、`anthropic_api_keys`、`openai_api_keys`、`deepseek_api_keys` |
| 搜索 | `bocha_api_keys`、`tavily_api_keys`、`brave_api_keys`、`serpapi_keys`、`anspire_api_keys`、`searxng_base_urls` |
| 通知 | `wechat_webhook_url`、`feishu_*`、`dingtalk_*`、`telegram_*`、`email_*`、14 渠道配置 |
| Agent | `agent_mode`、`agent_max_steps`(10)、`agent_skills`、`agent_arch`(single/multi) |
| 报告 | `report_type`(simple/full/brief)、`report_language`(zh/en/ko)、`merge_email_notification` |
| 调度 | `schedule_enabled`、`schedule_times`、`market_review_enabled`、`market_review_region` |
| 实时行情 | `enable_realtime_quote`、`realtime_source_priority`、`realtime_cache_ttl` |
| 回测 | `backtest_enabled`、`backtest_eval_window_days`(10)、`backtest_min_age_days`(14) |
| 流控 | `max_workers`(3)、`akshare_sleep_min/max`、`max_retries` |

**关键方法**：`get_instance()`、`_load_from_env()`、`refresh_stock_list()`、`validate_structured()`、`reset_instance()`。

#### `ConfigManager`（[src/core/config_manager.py](../../src/core/config_manager.py)）

`.env` 原子读写：

- `read_config_map()` / `get_config_version()`（mtime_ns + sha256）
- `apply_updates(updates, sensitive_keys, mask_token)`：返回 (updated_keys, skipped_masked, version)
- `_atomic_upsert`：临时文件 + `os.replace`，遇 EBUSY/EXDEV 回退原地重写
- Docker Compose 转义：`escape_compose_sensitive_env_value` / `unescape_compose_sensitive_env_value`

---

### 4.15 认证模块 `src/auth.py`

模块级函数（无类），基于 Cookie 会话的可选管理员认证。

**常量**：`COOKIE_NAME = "dsa_session"`、`PBKDF2_ITERATIONS = 100_000`、`RATE_LIMIT_WINDOW_SEC = 300`、`RATE_LIMIT_MAX_FAILURES = 5`、`SESSION_MAX_AGE_HOURS_DEFAULT = 24`。

**关键函数**：

- `is_auth_enabled()` / `refresh_auth_state()`
- `set_initial_password(password)`（首次登录设置）
- `verify_password(password)` / `change_password(current, new)` / `reset_password_cli()`
- `create_session() -> str` / `verify_session(value) -> bool`
- `rotate_session_secret()`（轮换即吊销所有会话）
- `check_rate_limit(ip)` / `record_login_failure(ip)`（IP 限流，5 分钟 5 次）

密码用 PBKDF2-HMAC + salt 存储，会话密钥独立于密码。

---

### 4.16 报告渲染

#### 模板（`templates/`，Jinja2）

| 模板 | 用途 |
|------|------|
| `_macros.j2` | 宏定义（market_snapshot 等） |
| `report_markdown.j2` | 主 Markdown 报告（仪表盘标题、汇总、逐股详情） |
| `report_brief.j2` | 简报模板 |
| `report_wechat.j2` | 微信适配模板 |

#### 格式化（[src/formatters.py](../../src/formatters.py)）

模块级函数：

- `markdown_to_html_document(markdown_text)` / `markdown_to_plain_text(markdown_text)`
- 分块：`chunk_markdown_preserving_blocks`、`chunk_content_by_max_bytes`、`chunk_content_by_max_words`
- 表格转 KV：`markdown_tables_to_key_value_rows`（绕过不支持表格的渠道）
- 渠道专用：`format_feishu_markdown`、`format_telegram_markdown`、`format_wechat_markdown`、`format_slack_mrkdwn`

#### Markdown 转图片（[src/md2img.py](../../src/md2img.py)）

```python
markdown_to_image(markdown_text: str, max_chars: int = 15000) -> Optional[bytes]
```

引擎：`wkhtmltoimage`（默认）/ `markdown-to-file`（emoji 支持更好）。

---

### 4.17 Web 前端 `apps/dsa-web/`

**技术栈**：

- React 19.2 + React Router 7.13（懒加载路由）
- Vite 7.2 + TypeScript 5.9（启用 `babel-plugin-react-compiler`）
- Tailwind CSS 4.1（HSL CSS 变量设计令牌，darkMode class）
- Zustand 5.0（状态管理）
- Axios 1.13（401 自动跳转登录）
- recharts 3.3（图表）、motion 12.36（动画）
- react-markdown 10.1 + remark-gfm
- Vitest 4.1 + Playwright 1.58

**脚本**：`dev`（vite，端口 5173，代理 /api 到 8000）、`build`（输出到项目根 `../../static`）、`lint`、`test`、`test:smoke`。

**路由**（全部懒加载，外层 Shell + RouteBoundary）：

| 路径 | 页面 | 职责 |
|------|------|------|
| `/` | HomePage | 主仪表盘：分析触发、市场复盘、运行流、任务面板、自选股栏 |
| `/chat` | ChatPage | Agent 对话：技能选择、Markdown 渲染、会话导出 |
| `/portfolio` | PortfolioPage | 组合管理：持仓饼图、现金账本、公司行为 |
| `/decision-signals` | DecisionSignalsPage | 决策信号：卡片、时间线、反馈、结果统计 |
| `/screening` | StockScreeningPage | 选股：AlphaSift 策略/候选/热点 |
| `/backtest` | BacktestPage | 回测：运行、性能指标、阶段过滤 |
| `/alerts` | AlertsPage | 告警中心：规则、触发历史 |
| `/usage` | TokenUsagePage | 用量仪表盘 |
| `/settings` | SettingsPage | 系统设置：LLM 渠道、通知测试、智能导入 |
| `/login` | LoginPage | 登录 |

**状态管理**：`analysisStore`、`agentChatStore`、`stockPoolStore`（含 sessionStorage 持久化）。

**上下文**：`AuthContext`（认证状态）、`UiLanguageContext`（zh/en）。

**关键 Hooks**：`useTaskStream`（SSE 任务流）、`useDashboardLifecycle`、`useRunFlowSnapshot`、`useSystemConfig`、`useWatchlist`、`useAutocomplete`。

---

### 4.18 桌面端 `apps/dsa-desktop/`

Electron 桌面端（version 3.21.0）。

**main.js 主进程职责**：

- 窗口管理：BrowserWindow 1200x800，根据 nativeTheme 切换深浅色
- 后端进程托管：打包模式 spawn `stock_analysis.exe`（PyInstaller），开发模式 spawn `python main.py`，启动参数 `--serve-only --host <bind> --port <port>`
- 端口探测：8000-8100 范围 `findAvailablePort`
- 健康检查：轮询 `/api/health`，60s 超时
- `.env` 管理：从 `.env.example` 复制初始化
- 更新检查：Windows NSIS 用 electron-updater，其他用 GitHub Releases API 比对语义化版本
- 运行时状态备份/恢复（更新前备份 .env/db/alphasift 数据）

**preload.js**：通过 `contextBridge.exposeInMainWorld('dsaDesktop', ...)` 暴露安全 API（version、getUpdateState、checkForUpdates、installDownloadedUpdate、onUpdateStateChange）。

**构建配置**：

- Windows: NSIS 安装器（发布到 GitHub）
- macOS: DMG
- extraResources: 打包 `.env.example` 和后端 `dist/backend/stock_analysis`（PyInstaller 产物）

---

## 5. 依赖关系总览

### 5.1 个股分析主链路

```
main.py / server.py / bot / API
   │
   ▼
StockAnalysisPipeline.__init__  (src/core/pipeline.py)
   │  组装: DataFetcherManager, StockTrendAnalyzer, GeminiAnalyzer,
   │       NotificationService, SearchService, SocialSentimentService
   │
   ▼
StockAnalysisPipeline.run(stock_codes)
   │ 1. config.refresh_stock_list()
   │ 2. fetcher_manager.prefetch_daily_klines / prefetch_realtime_quotes
   │ 3. ThreadPoolExecutor → process_single_stock(code)
   │       │
   │       ▼
   │    fetch_and_save_stock_data(code)    ← DataFetcherManager
   │       │  断点续传: get_effective_trading_date (trading_calendar)
   │       ▼
   │    analyze_stock(code, report_type, query_id)
   │       │  build_market_phase_context → infer_market_phase
   │       │  trend_analyzer.analyze(df, code)   ← StockTrendAnalyzer
   │       │  search_service.search_stock_news   ← SearchService
   │       │  analyzer.analyze(context, news)    ← GeminiAnalyzer
   │       │       │
   │       │       ▼
   │       │    LiteLLM Router → Provider API → AnalysisResult
   │       │    stabilize_decision_with_structure
   │       ▼
   │    _send_single_stock_notification (单股推送模式)
   │
   ▼
_save_local_report + _send_notifications
   │  formatters (渠道格式化) → md2img (可选转图片)
   │  notification_sender/* (14 渠道发送)
   ▼
DatabaseManager.save_analysis_history
   │  → 自动提取 DecisionSignal
   │  → (可选) BacktestService.run_backtest
```

### 5.2 大盘复盘链路

```
main.py (--market-review) / API / schedule
   │
   ▼
run_market_review(notifier, analyzer, search_service, config)
   │ 1. _resolve_market_review_regions(config.market_review_region)
   │    可选: compute_effective_region 过滤当日开市市场
   │ 2. for region in run_markets:
   │       MarketAnalyzer(search_service, analyzer, region, config)
   │           │  get_profile(region)            ← market_profile.py
   │           │  get_market_strategy_blueprint   ← market_strategy.py
   │       run_daily_review_with_snapshot()
   │           │  get_market_overview()  ← DataFetcherManager
   │           │  search_market_news()  ← SearchService
   │           │  generate_market_review() → analyzer.generate_text()
   │           │  build_market_light_snapshot()  (温度计)
   │ 3. _render_market_review_payload_markdown
   │ 4. notifier.save_report_to_file + _persist_market_review_history
   │ 5. notifier.send (非 merge 模式)
```

### 5.3 Agent 链路

```
Web /chat / Bot ask / API /chat
   │
   ▼
AgentExecutor.chat()
   │  build_agent_chat_context_bundle  ← chat_context.py
   │  resolve_stock_scope              ← stock_scope.py
   │  resolve_agent_litellm_route      ← litellm_route_resolution.py
   │
   ▼
run_agent_loop()  ← runner.py (共享 ReAct 循环)
   │  LLMToolAdapter ← llm_adapter.py → src/llm/*
   │  ToolRegistry ← tools/registry.py → data/analysis/search/market/backtest tools
   │
   ▼ (多 Agent 模式)
AgentOrchestrator.run()
   │  Technical → Intel → Risk → Specialist → Decision
   │  AgentContext (共享状态袋) + AgentOpinion (结构化输出)
   │  _apply_risk_override (风控降级)
   ▼
AgentResult → stream_events (SSE) / 对话存储
```

### 5.4 任务与调度链路

```
CLI (main.py --schedule)
   │  → run_with_schedule → Scheduler (schedule 库)
   │      → scheduled_task → _reload_runtime_config → run_full_analysis
   │      → background_tasks (AlertWorker)
   │
API/Web (main.py --serve)
   │  → RuntimeSchedulerService (共享 _RUNTIME_ANALYSIS_LOCK)
   │      → reconcile_from_config / run_now
   │
API /analyze
   │  → AnalysisTaskQueue.submit_task → 线程池 _execute_task
   │      → AnalysisService.analyze_stock
   │      → SSE 广播 (connected/task_created/.../task_completed)
```

### 5.5 通知链路

```
报告生成
   │  NotificationService.generate_dashboard_report / generate_brief_report
   ▼
formatters.py (渠道格式化)
   │  format_feishu_markdown / format_telegram_markdown / ...
   │  markdown_tables_to_key_value_rows (绕过不支持表格的渠道)
   ▼
md2img.py (可选 Markdown 转图片)
   ▼
notification_sender/* (14 渠道发送)
   │  上下文渠道优先 (钉钉 sessionWebhook / 飞书 chat_id / Telegram chat_id)
   │  无上下文时广播静态渠道
   ▼
NotificationDispatchResult (per-channel 诊断)
```

---

## 6. 项目运行方式

### 6.1 安装

```bash
# 克隆项目
git clone https://github.com/ZhuLinsen/daily_stock_analysis.git
cd daily_stock_analysis

# 安装 Python 依赖（Python 3.10+）
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env，至少配置：AI 模型 Key、通知渠道、STOCK_LIST
```

### 6.2 CLI 运行

```bash
python main.py                          # 正常运行（个股 + 大盘复盘）
python main.py --debug                  # 调试模式（详细日志）
python main.py --dry-run                # 仅获取数据，不进行 AI 分析
python main.py --stocks 600519,hk00700,AAPL,2330.TW   # 指定股票
python main.py --market-review          # 仅运行大盘复盘
python main.py --no-market-review       # 跳过大盘复盘
python main.py --no-notify              # 不发送推送
python main.py --check-notify           # 检查通知配置（只读）
python main.py --single-notify          # 单股推送模式（每分析完一只立即推送）
python main.py --force-run              # 跳过交易日检查，强制执行
python main.py --schedule               # 定时任务模式
python main.py --no-run-immediately     # 定时模式启动时不立即执行
python main.py --workers 5              # 指定并发线程数
python main.py --backtest               # 运行回测
python main.py --backtest-code 600519   # 仅回测指定股票
```

### 6.3 Web 服务

```bash
python main.py --serve                  # API 服务 + 执行分析
python main.py --serve-only             # 仅 API 服务，不自动分析
python main.py --webui                  # 等价于 --serve
python main.py --webui-only             # 等价于 --serve-only
python main.py --serve --port 9000 --host 0.0.0.0

# 或直接用 uvicorn
uvicorn server:app --reload --host 0.0.0.0 --port 8000
```

访问 `http://127.0.0.1:8000`，API 文档 `http://127.0.0.1:8000/docs`。

### 6.4 Web 前端开发

```bash
cd apps/dsa-web
npm ci                    # 安装依赖（Node >=20.19.0）
npm run dev               # 开发服务器（端口 5173，代理 /api 到 8000）
npm run lint              # ESLint
npm run build             # 构建（输出到 ../../static）
npm run test              # Vitest 单元测试
npm run test:smoke        # Playwright E2E
```

### 6.5 桌面端

```bash
cd apps/dsa-desktop
npm install
npm run dev               # 开发模式
npm run build             # electron-builder 打包
```

### 6.6 运行模式选择

| 需求 | 推荐方式 |
|------|---------|
| 零成本每日自动分析 | GitHub Actions（Fork 仓库 + 配置 Secrets） |
| 本地手动分析 | `python main.py` |
| 长期运行 + Web 管理 | `python main.py --schedule` 或 Docker |
| 仅 Web 服务 | `python main.py --serve-only` |
| 桌面客户端 | 下载 Release 或 `npm run dev` |
| Bot 接入 | 配置 Webhook / Stream 模式 |

---

## 7. CI/CD 与部署

### 7.1 CI 检查（[.github/workflows/ci.yml](../../.github/workflows/ci.yml)）

| 检查项 | 阻断 | 说明 |
|--------|------|------|
| `ai-governance` | 是 | `python scripts/check_ai_assets.py` |
| `backend-gate` | 是 | `./scripts/ci_gate.sh`（syntax/flake8/deterministic/offline-tests） |
| `docker-build` | 是 | Docker 构建 + 关键模块导入 smoke |
| `web-gate` | 是（触发时） | `npm ci && npm run lint && npm run build` |

### 7.2 每日分析（[.github/workflows/00-daily-analysis.yml](../../.github/workflows/00-daily-analysis.yml)）

- 触发：cron `0 10 * * 1-5`（周一至周五 UTC 10:00 = 北京 18:00）+ 手动
- 模式：full / market-only / stocks-only，支持 force_run
- 产物：上传 reports/ + logs/，保留 30 天

### 7.3 发布流水线

```
commit 含 #patch/#minor/#major
   │
   ▼ auto-tag.yml
tag v*.*.*
   ├── create-release.yml    (GitHub Release + 自动发布说明)
   ├── desktop-release.yml   (Windows NSIS + macOS DMG)
   └── docker-publish.yml    (GHCR + Docker Hub 多架构)
```

### 7.4 Docker 部署（[docker/](../../docker/)）

**多阶段构建**：

- Stage 1 `web-builder`：node:20-slim，构建前端静态资源
- Stage 2：python:3.11-slim-bookworm，非 root 用户 dsa，wkhtmltopdf（md 转图），EXPOSE 8000，HEALTHCHECK

**docker-compose.yml**：

- `analyzer` 服务：定时模式（默认 CMD）
- `server` 服务：FastAPI 模式（`--serve-only`）

**entrypoint.sh**：root 启动时修复挂载目录权限，`gosu dsa:dsa` 降权执行。

### 7.5 其他工作流

- `network-smoke.yml`：每日网络冒烟（非阻断）
- `pr-review.yml`：PR 静态检查 + AI 审查 + 自动标签
- `stale.yml`：Issue 7d stale + 7d close；PR 21d stale + 14d close

---

## 8. 测试体系

**pytest 测试**（[tests/](../../tests/)）：

```bash
python -m pytest -m "not network"       # 离线测试（默认）
python -m pytest -m "network"           # 网络测试（非阻断）
./scripts/ci_gate.sh                    # CI 门禁（四阶段）
```

**Web 测试**：

```bash
cd apps/dsa-web
npm run test          # Vitest 单元测试
npm run test:smoke    # Playwright E2E
```

**桌面端测试**：

```bash
cd apps/dsa-desktop
npm run test          # node --test
```

**关键测试覆盖**：数据源 fetcher、API 契约、Agent 编排、决策信号、回测、持仓、告警、通知、配置、运行流等。

---

## 附录：技术栈速览

| 类别 | 技术 |
|------|------|
| 后端语言 | Python 3.10+ |
| Web 框架 | FastAPI + Uvicorn |
| 数据库 | SQLite（SQLAlchemy 2.0 ORM，WAL 模式） |
| LLM 网关 | LiteLLM（15+ 渠道） |
| 数据源 | AkShare / Tushare / TickFlow / YFinance / Longbridge / Finnhub / 腾讯 / 通达信 / Baostock / efinance / AlphaVantage |
| 搜索 | Anspire / SerpAPI / Tavily / Bocha / Brave / MiniMax / SearXNG |
| 通知 | 14 渠道（企业微信/飞书/钉钉/Telegram/Discord/Slack/邮件/Pushover/Ntfy/Gotify/Pushplus/Server酱3/Custom/Astrbot） |
| 报告 | Jinja2 模板 + imgkit（wkhtmltoimage） |
| 前端 | React 19 + Vite + TypeScript + Tailwind + Zustand |
| 桌面端 | Electron 31 + electron-builder |
| CI/CD | GitHub Actions |
| 容器 | Docker（多阶段构建） |
| 调度 | schedule 库 + 自研 RuntimeSchedulerService |
| 测试 | pytest + Vitest + Playwright |
