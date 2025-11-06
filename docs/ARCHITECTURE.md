# NOFX AI Trading System - Architecture Documentation

## 系统概述 (System Overview)

NOFX 是一个基于 AI 的自动化加密货币交易系统,采用 3 分钟周期的决策循环,支持多交易所、多模型并行运行。系统通过 AI 大模型分析市场数据并生成交易决策,自动执行开仓、平仓、止损止盈等操作。

### 核心特性
- **AI 驱动决策**: 支持 DeepSeek、Qwen、Custom 等多种 AI 模型
- **多交易所支持**: Binance、Hyperliquid、Aster 等主流交易所
- **并行交易**: 多个独立 Trader 实例并发运行
- **智能风控**: 动态回撤监控、移动止损、账户级风险控制
- **完整日志**: 决策过程、执行结果、绩效分析全记录

---

## 系统架构 (System Architecture)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     NOFX AI Trading System                          │
│                  交易系统总体架构 (3分钟周期)                          │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Trader A   │    │   Trader B   │    │   Trader C   │
│  DeepSeek +  │    │   Qwen +     │    │  Custom +    │
│   Binance    │    │ Hyperliquid  │    │    Aster     │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                ┌──────────▼───────────┐
                │   TraderManager      │
                │   (启动与配置管理)     │
                └──────────┬───────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼────────┐  ┌──────▼───────┐  ┌──────▼──────┐
│  Market Data   │  │   Decision   │  │   Risk      │
│  (市场数据)     │  │   Engine     │  │   Control   │
│                │  │  (AI决策)     │  │  (风险控制)  │
└────────────────┘  └──────────────┘  └─────────────┘
```

---

## Phase 1: 启动与初始化 (Startup & Initialization)

### 1.1 TraderManager 启动流程

```go
// manager/trader_manager.go

TraderManager.Start()
    │
    ├──> LoadConfiguration()
    │    │
    │    ├─> 从数据库加载用户配置
    │    │   • User preferences
    │    │   • API credentials
    │    │
    │    ├─> 加载 AI 模型配置
    │    │   • DeepSeek API key & endpoint
    │    │   • Qwen API key & endpoint
    │    │   • Custom model settings
    │    │
    │    ├─> 加载交易所配置
    │    │   • Binance API credentials
    │    │   • Hyperliquid wallet & proxy
    │    │   • Aster exchange settings
    │    │
    │    └─> 加载风险参数
    │        • btc_eth_leverage (5-50x)
    │        • altcoin_leverage (5-20x)
    │        • max_daily_loss
    │        • max_drawdown
    │
    ├──> CreateAutoTraderInstances()
    │    │
    │    └─> For each configured trader:
    │        │
    │        ├─> InitializeMCPClient()
    │        │   • 连接 AI API 服务
    │        │   • 验证模型可用性
    │        │
    │        ├─> InitializeTrader()
    │        │   • 创建交易所客户端
    │        │   • 初始化 WebSocket 连接
    │        │   • 验证账户权限
    │        │
    │        └─> InitializeDecisionLogger()
    │            • 创建日志目录
    │            • 初始化 JSONL 写入器
    │
    └──> LaunchAllTraders()
         │
         └─> 使用 goroutine 并行启动所有 Trader
             go autoTrader.Run()  // Trader A
             go autoTrader.Run()  // Trader B
             go autoTrader.Run()  // Trader C
```

### 1.2 关键组件初始化

| 组件 | 文件路径 | 职责 |
|------|---------|------|
| TraderManager | `manager/trader_manager.go` | 管理所有 Trader 生命周期 |
| AutoTrader | `trader/auto_trader.go` | 核心交易执行引擎 |
| DecisionEngine | `decision/engine.go` | AI 决策生成器 |
| PromptManager | `decision/prompt_manager.go` | Prompt 模板管理 |
| BinanceTrader | `trader/binance_futures.go` | Binance 交易执行器 |
| HyperliquidTrader | `trader/hyperliquid_trader.go` | Hyperliquid 交易执行器 |
| AsterTrader | `trader/aster_trader.go` | Aster 交易执行器 |
| MarketData | `market/data.go` | 市场数据聚合器 |

---

## Phase 2: 交易主循环 (Main Trading Loop)

### 2.1 3分钟决策周期

```go
// trader/auto_trader.go

func (at *AutoTrader) Run() {
    ticker := time.NewTicker(3 * time.Minute)
    defer ticker.Stop()

    // 启动后台监控
    go at.startDrawdownMonitor()

    for {
        select {
        case <-ticker.C:
            at.runCycle()  // 核心决策执行周期
        case <-at.stopChan:
            return
        }
    }
}
```

### 2.2 runCycle() 执行流程

```
runCycle()
    │
    ├──> [Step 1] 风险控制检查
    │    │
    │    ├─> checkPauseStatus()
    │    │   • 检查是否在暂停期 (日亏损超限)
    │    │   • 检查是否回撤超限
    │    │
    │    ├─> resetDailyPnL()
    │    │   • 每24小时重置日盈亏计数器
    │    │
    │    └─> autoSyncBalance()
    │        • 每10分钟同步账户余额
    │        • 避免频繁 API 调用
    │
    ├──> [Step 2] 构建交易上下文
    │    │
    │    ├─> getAccountInfo()
    │    │   │
    │    │   └─> Returns:
    │    │       • totalEquity (总资产净值)
    │    │       • availableBalance (可用余额)
    │    │       • marginUsageRate (保证金使用率)
    │    │       • totalUnrealizedPnL (未实现盈亏)
    │    │
    │    ├─> getCurrentPositions()
    │    │   │
    │    │   └─> Returns (for each position):
    │    │       • symbol (币种)
    │    │       • side (long/short)
    │    │       • quantity (持仓数量)
    │    │       • entryPrice (入场价)
    │    │       • markPrice (标记价)
    │    │       • unrealizedPnL (未实现盈亏)
    │    │       • unrealizedPnLPercent (盈亏百分比)
    │    │       • leverage (杠杆倍数)
    │    │       • liquidationPrice (强平价)
    │    │       • holdDuration (持仓时长)
    │    │
    │    ├─> getCandidateCoins()
    │    │   │
    │    │   ├─> [Source 1] Coin Pool API
    │    │   │   • AI500评分前50币种
    │    │   │   • 高潜力新币 (新上线 < 30天)
    │    │   │
    │    │   ├─> [Source 2] OI Top API (Optional)
    │    │   │   • 持仓量增长 Top 30
    │    │   │   • 净多空比率异常币种
    │    │   │
    │    │   └─> [Source 3] Default Coins
    │    │       • BTC, ETH, SOL, BNB, DOGE
    │    │       • ADA, AVAX, MATIC (8个主流币种)
    │    │
    │    └─> getMarketData() [并发执行]
    │        │
    │        ├─> For 持仓币种 (必须获取):
    │        │   │
    │        │   ├─> 3分钟K线序列 (最近200根)
    │        │   │   • Open, High, Low, Close, Volume
    │        │   │
    │        │   ├─> 4小时K线序列 (最近100根)
    │        │   │   • 用于中长期趋势判断
    │        │   │
    │        │   ├─> 技术指标序列
    │        │   │   • EMA20 (指数移动平均)
    │        │   │   • MACD (DIF, DEA, Histogram)
    │        │   │   • RSI7, RSI14 (相对强弱指标)
    │        │   │
    │        │   ├─> 成交量 & 持仓量(OI)序列
    │        │   │   • Volume trend analysis
    │        │   │   • OI change rate
    │        │   │
    │        │   └─> 资金费率 (Funding Rate)
    │        │       • Current funding rate
    │        │       • Next funding time
    │        │
    │        └─> For 候选币种 (流动性过滤):
    │            │
    │            ├─> 流动性检查
    │            │   • OI Value ≥ 15M USD (可配置)
    │            │   • 24h Volume ≥ 10M USD
    │            │
    │            ├─> 动态候选数量
    │            │   • 无持仓时: 最多8个候选
    │            │   • 有持仓时: 减少候选数量
    │            │   • 持仓接近上限: 仅1-2个候选
    │            │
    │            └─> 获取相同市场数据
    │                (与持仓币种相同的数据结构)
    │
    ├──> [Step 3] 调用 AI 决策引擎
    │    │
    │    └─> DecisionEngine.GenerateDecisions()
    │        │
    │        ├─> buildSystemPrompt()
    │        │   │
    │        │   ├─> 加载 Prompt 模板
    │        │   │   │
    │        │   │   ├─> default.txt (保守策略)
    │        │   │   │   • 风险回报比 ≥ 1:3
    │        │   │   │   • 严格止损执行
    │        │   │   │   • 趋势确认后入场
    │        │   │   │
    │        │   │   ├─> adaptive.txt (自适应策略)
    │        │   │   │   • 根据市场状态调整
    │        │   │   │   • 震荡市降低仓位
    │        │   │   │   • 趋势市加大仓位
    │        │   │   │
    │        │   │   ├─> nof1.txt (激进策略)
    │        │   │   │   • 更高杠杆
    │        │   │   │   • 更大仓位
    │        │   │   │   • 快进快出
    │        │   │   │
    │        │   │   ├─> Hansen.txt (趋势策略)
    │        │   │   │   • 专注趋势跟踪
    │        │   │   │   • 移动止损优化
    │        │   │   │   • 金字塔加仓
    │        │   │   │
    │        │   │   └─> adaptive_relaxed.txt (宽松策略)
    │        │   │       • 更灵活的风控
    │        │   │       • 允许更大回撤
    │        │   │       • 更多持仓数量
    │        │   │
    │        │   ├─> 添加硬约束 (Hard Constraints)
    │        │   │   • 风险回报比 ≥ 1:3
    │        │   │   • 最多持仓 3 个币种
    │        │   │   • BTC/ETH 杠杆: 5-50x
    │        │   │   • 山寨币杠杆: 5-20x
    │        │   │   • 保证金使用率 ≤ 90%
    │        │   │   • 最小开仓金额 ≥ 12 USDT
    │        │   │   • 单币种最大仓位 ≤ 账户净值 40%
    │        │   │
    │        │   └─> 添加自定义 Prompt (可选)
    │        │       • 用户自定义策略偏好
    │        │       • 特殊市场条件指令
    │        │
    │        ├─> buildUserPrompt()
    │        │   │
    │        │   ├─> 系统状态
    │        │   │   • 当前时间: 2025-11-06 14:32:00
    │        │   │   • 周期数: #245
    │        │   │   • 运行时长: 12h 15m
    │        │   │
    │        │   ├─> BTC 市场状态 (市场基准)
    │        │   │   • 价格: $67,432
    │        │   │   • 24h涨跌幅: +2.3%
    │        │   │   • MACD: DIF=120, DEA=95 (金叉)
    │        │   │   • RSI14: 58 (中性偏多)
    │        │   │
    │        │   ├─> 账户状态
    │        │   │   • 净值: $10,234.56
    │        │   │   • 可用余额: $3,456.78
    │        │   │   • 持仓数: 2/3
    │        │   │   • 总盈亏百分比: +15.7%
    │        │   │   • 日盈亏百分比: +2.1%
    │        │   │
    │        │   ├─> 当前持仓详情
    │        │   │   │
    │        │   │   ├─> Position 1:
    │        │   │   │   • 币种: ETHUSDT
    │        │   │   │   • 方向: LONG
    │        │   │   │   • 数量: 2.5 ETH
    │        │   │   │   • 入场价: $2,450
    │        │   │   │   • 当前价: $2,520
    │        │   │   │   • 盈亏: +$175 (+2.86%)
    │        │   │   │   • 杠杆: 10x
    │        │   │   │   • 强平价: $2,205
    │        │   │   │   • 持仓时长: 2h 35m
    │        │   │   │   • 峰值盈亏: +3.5% (当前回撤 0.64%)
    │        │   │   │   • 止损: $2,400 (-2.04%)
    │        │   │   │   • 止盈: $2,600 (+6.12%)
    │        │   │   │
    │        │   │   └─> Position 2:
    │        │   │       • [Similar structure...]
    │        │   │
    │        │   ├─> 历史绩效分析
    │        │   │   • 夏普比率: 1.85
    │        │   │   • 胜率: 67.5%
    │        │   │   • 盈亏比: 2.1:1
    │        │   │   • 平均持仓时长: 4.5小时
    │        │   │   • 最大回撤: -8.3%
    │        │   │   • 最大连续亏损: 3次
    │        │   │   • 最大连续盈利: 7次
    │        │   │
    │        │   └─> 候选币种市场数据
    │        │       │
    │        │       └─> For each candidate:
    │        │           • 币种名称 & 排名信息
    │        │           • 完整 K线序列 (3m, 4h)
    │        │           • 技术指标序列 (EMA, MACD, RSI)
    │        │           • 成交量 & OI 序列
    │        │           • OI 排名 & 变化率
    │        │           • 资金费率 & 趋势
    │        │           • 价格变化百分比 (1h, 4h, 24h)
    │        │
    │        ├─> [AI API Call]
    │        │   │
    │        │   ├─> Request Parameters:
    │        │   │   • Model: DeepSeek-V3 / Qwen-Max / Custom
    │        │   │   • System Prompt: ~1,500 tokens
    │        │   │   • User Prompt: ~5,000 tokens
    │        │   │   • Temperature: 0.7
    │        │   │   • Max Tokens: 4096
    │        │   │   • Top P: 0.9
    │        │   │
    │        │   └─> AI Response (~1,800 tokens):
    │        │       │
    │        │       ├─> 思维链分析 (CoT Trace)
    │        │       │   ```
    │        │       │   # 市场分析
    │        │       │   BTC 处于上升趋势,MACD 金叉,RSI 58 显示健康...
    │        │       │
    │        │       │   # 持仓评估
    │        │       │   ETH多单盈利 2.86%,但回撤 0.64%,建议移动止损...
    │        │       │
    │        │       │   # 候选币分析
    │        │       │   SOL 突破关键阻力位,OI 增长 15%,考虑开仓...
    │        │       │   ```
    │        │       │
    │        │       └─> JSON 决策数组
    │        │           ```json
    │        │           [
    │        │             {
    │        │               "action": "update_stop_loss",
    │        │               "symbol": "ETHUSDT",
    │        │               "reason": "盈利2.86%,移动止损保护利润",
    │        │               "stop_loss_price": 2470,
    │        │               "confidence": 0.85
    │        │             },
    │        │             {
    │        │               "action": "open_long",
    │        │               "symbol": "SOLUSDT",
    │        │               "reason": "突破阻力位,OI增长显著",
    │        │               "position_size_usd": 2000,
    │        │               "leverage": 15,
    │        │               "stop_loss_price": 145.20,
    │        │               "take_profit_price": 162.80,
    │        │               "confidence": 0.78
    │        │             }
    │        │           ]
    │        │           ```
    │        │
    │        └─> parseFullDecisionResponse()
    │            │
    │            ├─> 提取思维链 (CoT Trace)
    │            │   • 使用正则提取 Markdown 内容
    │            │   • 保存完整分析过程
    │            │
    │            ├─> 提取 JSON 决策
    │            │   • 查找 ```json...``` 代码块
    │            │   • 修复全角字符 ({「 → {, }」 → })
    │            │   • 修复引号问题 (" → ")
    │            │   • 验证 JSON 格式
    │            │   • 解析为 Decision 数组
    │            │
    │            └─> 验证决策合规性
    │                │
    │                ├─> 杠杆检查
    │                │   • BTC/ETH: 5 ≤ leverage ≤ 50
    │                │   • 山寨币: 5 ≤ leverage ≤ 20
    │                │
    │                ├─> 仓位大小检查
    │                │   • position_size ≥ 12 USDT
    │                │   • position_size ≤ 账户净值 × 40%
    │                │
    │                ├─> 止损止盈检查
    │                │   • stop_loss 必须设置
    │                │   • 风险回报比 ≥ 1:3
    │                │   • 止损价格合理性验证
    │                │
    │                └─> 持仓数量检查
    │                    • 开仓后总持仓数 ≤ 3
    │
    ├──> [Step 4] 决策排序与执行
    │    │
    │    ├─> sortDecisionsByPriority()
    │    │   │
    │    │   └─> 优先级排序:
    │    │       1. close_long, close_short (平仓)
    │    │       2. partial_close (部分平仓)
    │    │       3. update_stop_loss, update_take_profit (调整)
    │    │       4. open_long, open_short (开仓)
    │    │       5. wait, hold (观望)
    │    │
    │    │   理由: 先平仓释放资金,避免仓位叠加超限
    │    │
    │    └─> 顺序执行所有决策
    │        │
    │        └─> For each decision:
    │            executeDecisionWithRecord(decision)
    │
    └──> [Step 5] 保存决策日志
         │
         └─> DecisionLogger.Save()
             • 文件路径: decisions/{trader_id}/2025-11-06.jsonl
             • 每行一条完整决策记录 (JSONL 格式)
             • 包含完整上下文、AI 分析、执行结果
```

---

## Phase 3: 决策执行详细流程

### 3.1 executeDecisionWithRecord()

```go
func (at *AutoTrader) executeDecisionWithRecord(decision Decision) error {

    switch decision.Action {

    case "wait", "hold":
        // 观望,不执行任何操作
        log.Info("Action: wait/hold, skipping execution")
        return nil

    case "open_long":
        return at.executeOpenLong(decision)

    case "open_short":
        return at.executeOpenShort(decision)

    case "close_long":
        return at.executeCloseLong(decision)

    case "close_short":
        return at.executeCloseShort(decision)

    case "update_stop_loss":
        return at.executeUpdateStopLoss(decision)

    case "update_take_profit":
        return at.executeUpdateTakeProfit(decision)

    case "partial_close":
        return at.executePartialClose(decision)

    default:
        return fmt.Errorf("unknown action: %s", decision.Action)
    }
}
```

### 3.2 开仓流程 (Open Long)

```
executeOpenLong(decision)
    │
    ├──> [Step 1] 确定杠杆
    │    │
    │    ├─> 根据币种类型
    │    │   • BTC/ETH: 使用 btc_eth_leverage (max 50x)
    │    │   • 山寨币: 使用 altcoin_leverage (max 20x)
    │    │
    │    └─> 不超过决策中的杠杆值
    │        leverage = min(decision.Leverage, maxLeverage)
    │
    ├──> [Step 2] 设置杠杆
    │    │
    │    └─> trader.SetLeverage(symbol, leverage)
    │        │
    │        ├─> [Binance]
    │        │   futures.SetLeverage(symbol, leverage)
    │        │
    │        ├─> [Hyperliquid]
    │        │   exchange.UpdateLeverage(symbol, leverage, isCross)
    │        │
    │        └─> [Aster]
    │            (Similar to Hyperliquid)
    │
    ├──> [Step 3] 设置仓位模式
    │    │
    │    └─> trader.SetMarginMode(symbol, isCrossMargin)
    │        • true: 全仓模式 (Cross Margin)
    │        • false: 逐仓模式 (Isolated Margin)
    │
    ├──> [Step 4] 计算开仓数量
    │    │
    │    ├─> 获取当前价格
    │    │   currentPrice = getMarkPrice(symbol)
    │    │
    │    ├─> 计算数量
    │    │   quantity = decision.PositionSizeUSD / currentPrice
    │    │
    │    └─> 处理精度要求
    │        • 向下取整到交易所允许的最小精度
    │        • 确保不小于最小交易量
    │
    ├──> [Step 5] 执行开仓
    │    │
    │    └─> trader.OpenLong(symbol, quantity, leverage)
    │        │
    │        ├─> [Binance Futures]
    │        │   │
    │        │   ├─> 1. 取消旧委托单
    │        │   │   futures.CancelAllOpenOrders(symbol)
    │        │   │
    │        │   ├─> 2. 设置杠杆
    │        │   │   futures.SetLeverage(symbol, leverage)
    │        │   │
    │        │   ├─> 3. 市价买入
    │        │   │   order := futures.NewCreateOrderService()
    │        │   │       .Symbol(symbol)
    │        │   │       .Side(SIDE_BUY)
    │        │   │       .Type(MARKET)
    │        │   │       .Quantity(quantity)
    │        │   │       .Do(ctx)
    │        │   │
    │        │   └─> 4. 等待成交确认
    │        │       • 轮询订单状态
    │        │       • 获取成交均价
    │        │
    │        ├─> [Hyperliquid]
    │        │   │
    │        │   ├─> 1. 取消旧委托单
    │        │   │   exchange.CancelAllOrders(symbol)
    │        │   │
    │        │   ├─> 2. 更新杠杆
    │        │   │   exchange.UpdateLeverage(symbol, leverage)
    │        │   │
    │        │   ├─> 3. IOC 限价单 (激进价格)
    │        │   │   │
    │        │   │   ├─> 获取当前盘口
    │        │   │   │   askPrice = getBestAsk(symbol)
    │        │   │   │
    │        │   │   ├─> 设置激进价格
    │        │   │   │   orderPrice = askPrice * 1.002 (买单)
    │        │   │   │
    │        │   │   ├─> 处理精度
    │        │   │   │   • 价格向上取整到 tick size
    │        │   │   │   • 数量向下取整到 step size
    │        │   │   │
    │        │   │   └─> 下单
    │        │   │       order := exchange.Order{
    │        │   │           Symbol: symbol,
    │        │   │           IsBuy: true,
    │        │   │           Sz: quantity,
    │        │   │           LimitPx: orderPrice,
    │        │   │           OrderType: Limit,
    │        │   │           Tif: TifIoc,  // Immediate or Cancel
    │        │   │           ReduceOnly: false,
    │        │   │       }
    │        │   │
    │        │   └─> 4. 验证成交
    │        │       • 检查订单状态
    │        │       • 获取成交数量和均价
    │        │
    │        └─> [Aster]
    │            • 流程类似 Hyperliquid
    │            • 使用 Aster 特定 API
    │
    ├──> [Step 6] 设置止损
    │    │
    │    └─> trader.SetStopLoss(symbol, "long", quantity, decision.StopLossPrice)
    │        │
    │        └─> 创建 Trigger Order:
    │            {
    │                "symbol": "ETHUSDT",
    │                "side": "SELL",  // 平多仓
    │                "quantity": 2.5,
    │                "trigger_price": 2400,  // 止损触发价
    │                "order_type": "MARKET",
    │                "tpsl": "sl",  // 标记为止损单
    │                "reduce_only": true
    │            }
    │
    └──> [Step 7] 设置止盈
         │
         └─> trader.SetTakeProfit(symbol, "long", quantity, decision.TakeProfitPrice)
             │
             └─> 创建 Trigger Order:
                 {
                     "symbol": "ETHUSDT",
                     "side": "SELL",  // 平多仓
                     "quantity": 2.5,
                     "trigger_price": 2600,  // 止盈触发价
                     "order_type": "MARKET",
                     "tpsl": "tp",  // 标记为止盈单
                     "reduce_only": true
                 }
```

### 3.3 平仓流程 (Close Long)

```
executeCloseLong(decision)
    │
    ├──> [Step 1] 获取持仓数量
    │    position = getPosition(symbol)
    │    quantity = position.Quantity
    │
    ├──> [Step 2] 取消止损止盈单
    │    │
    │    ├─> CancelStopLoss(symbol)
    │    └─> CancelTakeProfit(symbol)
    │
    └──> [Step 3] 执行平仓
         │
         └─> trader.CloseLong(symbol, quantity)
             │
             ├─> [Binance]
             │   • 市价卖出 (SIDE_SELL, TYPE_MARKET)
             │   • ReduceOnly: true (仅平仓)
             │
             ├─> [Hyperliquid]
             │   • IOC 限价单 (卖单价格 = bidPrice * 0.998)
             │   • ReduceOnly: true
             │
             └─> [Aster]
                 • 类似 Hyperliquid 流程
```

### 3.4 调整止损/止盈

```
executeUpdateStopLoss(decision)
    │
    ├──> [Step 1] 取消旧止损单
    │    CancelStopLoss(symbol)
    │
    └──> [Step 2] 设置新止损
         SetStopLoss(symbol, side, quantity, decision.StopLossPrice)

executeUpdateTakeProfit(decision)
    │
    ├──> [Step 1] 取消旧止盈单
    │    CancelTakeProfit(symbol)
    │
    └──> [Step 2] 设置新止盈
         SetTakeProfit(symbol, side, quantity, decision.TakeProfitPrice)
```

### 3.5 部分平仓

```
executePartialClose(decision)
    │
    ├──> [Step 1] 计算平仓数量
    │    │
    │    ├─> 获取当前持仓
    │    │   position = getPosition(symbol)
    │    │
    │    └─> 计算平仓数量
    │        closeQty = position.Quantity × (decision.ClosePercentage / 100)
    │        // 例如: 持仓 2.5 ETH, ClosePercentage=50, closeQty=1.25
    │
    └──> [Step 2] 执行部分平仓
         │
         ├─> trader.CloseLong(symbol, closeQty)
         │   或 trader.CloseShort(symbol, closeQty)
         │
         └─> 保留剩余持仓的止损止盈单
             (交易所会自动调整 trigger order 数量)
```

---

## Phase 4: 风险控制与监控

### 4.1 回撤监控 (Drawdown Monitor)

```go
// trader/auto_trader.go

func (at *AutoTrader) startDrawdownMonitor() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    peakPnL := make(map[string]float64)  // 每个持仓的峰值盈亏

    for {
        select {
        case <-ticker.C:
            at.checkDrawdown(peakPnL)
        case <-at.stopChan:
            return
        }
    }
}
```

### 4.2 回撤检测逻辑

```
checkDrawdown(peakPnL)
    │
    └─> For each position:
        │
        ├──> 计算当前盈亏百分比
        │    │
        │    └─> unrealizedPnLPct = (markPrice - entryPrice) / entryPrice
        │        例如: (2520 - 2450) / 2450 = 2.86%
        │
        ├──> 更新峰值盈亏
        │    │
        │    └─> peakPnL[symbol] = max(peakPnL[symbol], unrealizedPnLPct)
        │        例如: 持仓曾达到 +3.5%, 记录峰值
        │
        └──> 检测回撤
             │
             ├─> 触发条件:
             │   • peakPnL > 8% (盈利达到8%以上)
             │   • unrealizedPnLPct < peakPnL - 3% (回撤超过3%)
             │
             │   例如: 峰值 +10%, 当前 +6%, 回撤 4% → 触发
             │
             └─> 执行移动止损:
                 │
                 ├─> 计算新止损价
                 │   newStopLoss = entryPrice × (1 + 5%)
                 │   // 保护 5% 利润
                 │
                 ├─> 取消旧止损单
                 │   CancelStopLoss(symbol)
                 │
                 ├─> 设置新止损单
                 │   SetStopLoss(symbol, side, quantity, newStopLoss)
                 │
                 └─> 记录日志
                     log.Info("Trailing stop triggered",
                         "symbol", symbol,
                         "peak_pnl", peakPnL,
                         "current_pnl", unrealizedPnLPct,
                         "new_stop_loss", newStopLoss)
```

### 4.3 账户级风险控制

```
checkAccountRisk()
    │
    ├──> 日亏损检查
    │    │
    │    ├─> 计算日盈亏
    │    │   dailyPnL = 今日总盈亏
    │    │
    │    └─> 超限检查
    │        if dailyPnL < -MaxDailyLoss:
    │            • 暂停所有交易
    │            • 记录风控事件
    │            • 发送告警通知
    │            • 暂停时长: 24小时
    │
    ├──> 最大回撤检查
    │    │
    │    ├─> 计算当前回撤
    │    │   drawdown = (peak_equity - current_equity) / peak_equity
    │    │
    │    └─> 超限检查
    │        if drawdown > MaxDrawdown:
    │            • 立即平掉所有持仓
    │            • 暂停所有交易
    │            • 记录风控事件
    │            • 发送告警通知
    │            • 暂停时长: 直到手动恢复
    │
    └──> 保证金使用率检查
         │
         ├─> 计算保证金使用率
         │   marginUsageRate = usedMargin / totalEquity
         │
         └─> 超限检查
             if marginUsageRate > 90%:
                 • 禁止新开仓
                 • 仅允许平仓和减仓
                 • 记录告警日志
```

---

## Phase 5: 日志与记录

### 5.1 决策日志格式 (JSONL)

```jsonl
{
  "timestamp": "2025-11-06T14:32:00Z",
  "trader_id": "trader_001",
  "cycle_number": 245,
  "runtime_hours": 12.25,

  "account_snapshot": {
    "total_equity": 10234.56,
    "available_balance": 3456.78,
    "margin_usage_rate": 0.66,
    "total_unrealized_pnl": 175.30,
    "total_pnl_percent": 15.7,
    "daily_pnl_percent": 2.1
  },

  "positions_snapshot": [
    {
      "symbol": "ETHUSDT",
      "side": "long",
      "quantity": 2.5,
      "entry_price": 2450.00,
      "mark_price": 2520.00,
      "unrealized_pnl": 175.00,
      "unrealized_pnl_percent": 2.86,
      "leverage": 10,
      "liquidation_price": 2205.00,
      "hold_duration_minutes": 155,
      "peak_pnl_percent": 3.5,
      "stop_loss": 2400.00,
      "take_profit": 2600.00
    }
  ],

  "candidate_coins": [
    "BTCUSDT", "ETHUSDT", "SOLUSDT", "BNBUSDT",
    "ADAUSDT", "DOGEUSDT", "AVAXUSDT", "MATICUSDT"
  ],

  "system_prompt": "You are a professional crypto futures trader...",

  "user_prompt": "## System Status\n...",

  "ai_analysis": {
    "cot_trace": "# 市场分析\nBTC 处于上升趋势...",
    "raw_response": "完整 AI 响应内容"
  },

  "decisions": [
    {
      "action": "update_stop_loss",
      "symbol": "ETHUSDT",
      "reason": "盈利2.86%,移动止损保护利润",
      "stop_loss_price": 2470.00,
      "confidence": 0.85,
      "execution_result": {
        "success": true,
        "executed_at": "2025-11-06T14:32:15Z",
        "order_id": "12345678",
        "error": null
      }
    },
    {
      "action": "open_long",
      "symbol": "SOLUSDT",
      "reason": "突破阻力位,OI增长显著",
      "position_size_usd": 2000.00,
      "leverage": 15,
      "stop_loss_price": 145.20,
      "take_profit_price": 162.80,
      "confidence": 0.78,
      "execution_result": {
        "success": true,
        "executed_at": "2025-11-06T14:32:30Z",
        "order_id": "23456789",
        "filled_quantity": 13.5,
        "avg_fill_price": 148.15,
        "error": null
      }
    }
  ],

  "execution_summary": {
    "total_decisions": 2,
    "successful_executions": 2,
    "failed_executions": 0,
    "execution_time_ms": 1250
  }
}
```

### 5.2 绩效分析 (Performance Analysis)

```go
// decision/engine.go

type PerformanceMetrics struct {
    // 收益指标
    TotalReturnPercent    float64  // 总收益率
    DailyReturnPercent    float64  // 日收益率
    MonthlyReturnPercent  float64  // 月收益率

    // 风险指标
    SharpeRatio          float64  // 夏普比率
    MaxDrawdownPercent   float64  // 最大回撤
    Volatility           float64  // 波动率

    // 交易统计
    TotalTrades          int      // 总交易次数
    WinningTrades        int      // 盈利交易次数
    LosingTrades         int      // 亏损交易次数
    WinRate              float64  // 胜率
    ProfitLossRatio      float64  // 盈亏比

    // 持仓统计
    AvgHoldDuration      float64  // 平均持仓时长(分钟)
    MaxHoldDuration      float64  // 最大持仓时长
    MinHoldDuration      float64  // 最小持仓时长

    // 连续性统计
    MaxConsecutiveWins   int      // 最大连续盈利
    MaxConsecutiveLosses int      // 最大连续亏损
    CurrentStreak        int      // 当前连续状态

    // 币种统计
    MostProfitableCoin   string   // 最赚钱币种
    LeastProfitableCoin  string   // 最亏钱币种
    TradingFrequency     float64  // 交易频率(次/天)
}

func CalculatePerformance(decisions []DecisionLog) PerformanceMetrics {
    // 实现绩效计算逻辑
    // ...
}
```

### 5.3 夏普比率计算

```
Sharpe Ratio = (Portfolio Return - Risk Free Rate) / Portfolio Volatility

例如:
• 月收益率: 15%
• 无风险利率: 0.5% (假设)
• 月波动率: 8%

Sharpe Ratio = (15% - 0.5%) / 8% = 1.81

解读:
• > 2.0: 优秀
• 1.5 - 2.0: 良好
• 1.0 - 1.5: 一般
• < 1.0: 较差
```

---

## 关键数据流 (Data Flow)

```
┌──────────────────────────────────────────────────────────────────┐
│                        Data Flow Diagram                         │
└──────────────────────────────────────────────────────────────────┘

[币种来源]
    │
    ├─> Coin Pool API (AI500评分)
    ├─> OI Top API (持仓量排名)
    └─> Default Coins (主流币种)
    │
    └─────────────────> [候选币种池]
                              │
                              ↓
[交易所 WebSocket]        [Market Data]
    │                         │
    ├─> K线数据 ──────────────┤
    ├─> 盘口数据 ──────────────┤
    ├─> 成交数据 ──────────────┤
    └─> 资金费率 ──────────────┘
                              │
                              ↓
                        [技术指标计算]
                              │
                              ├─> EMA20
                              ├─> MACD
                              ├─> RSI7/14
                              └─> OI Analysis
                              │
                              ↓
[账户状态] ─────────────> [交易上下文构建]
    │                         │
    ├─> 净值                  │
    ├─> 可用余额              │
    ├─> 持仓信息              │
    └─> 历史绩效              │
                              │
                              ↓
[Prompt 模板] ───────────> [AI 决策引擎]
    │                         │
    ├─> System Prompt         │
    └─> 硬约束规则            │
                              │
                [交易上下文] ──┘
                              │
                              ↓
                        [AI API 调用]
                              │
                              ├─> DeepSeek-V3
                              ├─> Qwen-Max
                              └─> Custom Model
                              │
                              ↓
                        [AI 响应解析]
                              │
                              ├─> 思维链分析
                              └─> JSON 决策数组
                              │
                              ↓
                        [决策排序]
                              │
                              ├─> 1. 平仓
                              ├─> 2. 调整
                              └─> 3. 开仓
                              │
                              ↓
                        [执行决策]
                              │
                              ├─> Binance API
                              ├─> Hyperliquid API
                              └─> Aster API
                              │
                              ↓
                        [记录日志]
                              │
                              ├─> JSONL 决策日志
                              ├─> 执行结果
                              └─> 绩效统计
                              │
                              ↓
[风险监控] <─────────────> [回撤检测]
    │                         │
    ├─> 移动止损              │
    ├─> 账户风控              │
    └─> 告警通知              │
                              │
                              ↓
                        [下一周期]
```

---

## 技术栈 (Tech Stack)

### 后端核心
- **语言**: Go 1.21+
- **框架**: Gin (Web framework)
- **数据库**: PostgreSQL (用户配置、交易记录)
- **缓存**: Redis (市场数据缓存)
- **消息队列**: (可选) RabbitMQ

### AI 模型
- **DeepSeek-V3**: 高性能推理模型
- **Qwen-Max**: 通用大模型
- **Custom Models**: 支持自定义 API 接入

### 交易所 SDK
- **Binance**: `github.com/adshao/go-binance`
- **Hyperliquid**: 自研 SDK (基于官方 Python SDK 移植)
- **Aster**: 自研 SDK

### 前端 (Web Dashboard)
- **框架**: React + TypeScript
- **UI 库**: Ant Design / Material-UI
- **状态管理**: Redux / Zustand
- **图表**: TradingView Lightweight Charts
- **实时通信**: WebSocket

### DevOps
- **容器化**: Docker + Docker Compose
- **进程管理**: PM2
- **反向代理**: Nginx
- **CI/CD**: GitHub Actions
- **监控**: Prometheus + Grafana (可选)

---

## 配置参数 (Configuration)

### Trader 配置

```go
type TraderConfig struct {
    // 基础配置
    TraderID          string  `json:"trader_id"`
    Nickname          string  `json:"nickname"`
    IsEnabled         bool    `json:"is_enabled"`

    // AI 模型配置
    AIModel           string  `json:"ai_model"`        // "deepseek", "qwen", "custom"
    AIEndpoint        string  `json:"ai_endpoint"`
    AIApiKey          string  `json:"ai_api_key"`
    PromptTemplate    string  `json:"prompt_template"` // "default", "adaptive", etc.
    CustomPrompt      string  `json:"custom_prompt"`

    // 交易所配置
    Exchange          string  `json:"exchange"`        // "binance", "hyperliquid", "aster"
    ExchangeApiKey    string  `json:"exchange_api_key"`
    ExchangeSecretKey string  `json:"exchange_secret_key"`

    // 杠杆配置
    BtcEthLeverage    int     `json:"btc_eth_leverage"`    // 5-50
    AltcoinLeverage   int     `json:"altcoin_leverage"`    // 5-20

    // 仓位配置
    MaxPositions      int     `json:"max_positions"`       // 1-3
    MaxPositionSizePercent float64 `json:"max_position_size_percent"` // 20-40%
    MinPositionSizeUSD float64 `json:"min_position_size_usd"` // 12 USDT

    // 风控配置
    MaxDailyLossPercent  float64 `json:"max_daily_loss_percent"`  // 5-10%
    MaxDrawdownPercent   float64 `json:"max_drawdown_percent"`    // 15-25%
    DrawdownCheckInterval int    `json:"drawdown_check_interval"` // 30秒
    TrailingStopTrigger  float64 `json:"trailing_stop_trigger"`   // 8%
    TrailingStopDrawdown float64 `json:"trailing_stop_drawdown"`  // 3%
    TrailingStopProtect  float64 `json:"trailing_stop_protect"`   // 5%

    // 币种池配置
    UseCoinPoolAPI    bool    `json:"use_coin_pool_api"`
    UseOITopAPI       bool    `json:"use_oi_top_api"`
    MinOIValueUSD     float64 `json:"min_oi_value_usd"`    // 15M
    MaxCandidates     int     `json:"max_candidates"`      // 8

    // 循环周期
    CycleIntervalMinutes int  `json:"cycle_interval_minutes"` // 3
}
```

---

## 部署架构 (Deployment)

### Docker Compose 部署

```yaml
version: '3.8'

services:
  # PostgreSQL 数据库
  postgres:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: nofx
      POSTGRES_USER: nofx
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"

  # Redis 缓存
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

  # NOFX 后端
  nofx-backend:
    build: .
    depends_on:
      - postgres
      - redis
    environment:
      - DB_HOST=postgres
      - REDIS_HOST=redis
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY}
      - QWEN_API_KEY=${QWEN_API_KEY}
      - BINANCE_API_KEY=${BINANCE_API_KEY}
      - BINANCE_SECRET_KEY=${BINANCE_SECRET_KEY}
    volumes:
      - ./decisions:/app/decisions
      - ./logs:/app/logs
    ports:
      - "8080:8080"

  # Nginx 反向代理
  nginx:
    image: nginx:alpine
    depends_on:
      - nofx-backend
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./web/dist:/usr/share/nginx/html
    ports:
      - "80:80"
      - "443:443"

volumes:
  postgres_data:
  redis_data:
```

---

## 监控与告警 (Monitoring)

### 关键监控指标

1. **系统健康**
   - Trader 运行状态
   - API 连接状态
   - WebSocket 连接稳定性
   - 内存/CPU 使用率

2. **交易指标**
   - 决策周期执行时间
   - AI API 响应时间
   - 订单执行成功率
   - 滑点统计

3. **风控指标**
   - 当前净值
   - 日盈亏百分比
   - 最大回撤
   - 保证金使用率
   - 持仓数量

4. **绩效指标**
   - 夏普比率
   - 胜率
   - 盈亏比
   - 平均持仓时长

### 告警规则

```yaml
alerts:
  - name: high_drawdown
    condition: drawdown > 15%
    action: pause_trading + notify

  - name: daily_loss_limit
    condition: daily_loss > 10%
    action: pause_trading + notify

  - name: api_connection_lost
    condition: api_offline > 5min
    action: notify

  - name: execution_failure_rate
    condition: failure_rate > 20%
    action: notify

  - name: low_balance
    condition: available_balance < 100 USDT
    action: notify
```

---

## 安全考虑 (Security)

### API 密钥管理
- 使用环境变量存储敏感信息
- 支持 API Key 加密存储
- 定期轮换 API Key
- IP 白名单限制

### 交易安全
- 仅使用 ReduceOnly 平仓
- 强制设置止损
- 杠杆上限硬编码
- 仓位大小限制

### 数据安全
- 决策日志加密存储
- 数据库访问控制
- 定期备份
- 敏感信息脱敏

---

## 未来优化方向 (Future Improvements)

1. **多策略组合**
   - 支持策略权重分配
   - 策略绩效对比
   - 自动策略切换

2. **强化学习**
   - 基于历史数据训练
   - 在线学习优化
   - 自适应参数调整

3. **高频交易**
   - 缩短决策周期 (1分钟/30秒)
   - 网格交易策略
   - 套利机会捕捉

4. **社区功能**
   - 策略分享
   - 跟单功能
   - 绩效排行榜

5. **移动端**
   - iOS/Android App
   - 实时推送通知
   - 远程控制

---

## 附录 (Appendix)

### A. 决策 Action 类型

| Action | 说明 | 必需参数 |
|--------|------|---------|
| wait | 观望,不执行操作 | - |
| hold | 保持当前持仓 | - |
| open_long | 开多仓 | symbol, position_size_usd, leverage, stop_loss_price, take_profit_price |
| open_short | 开空仓 | symbol, position_size_usd, leverage, stop_loss_price, take_profit_price |
| close_long | 平多仓 | symbol |
| close_short | 平空仓 | symbol |
| update_stop_loss | 更新止损 | symbol, stop_loss_price |
| update_take_profit | 更新止盈 | symbol, take_profit_price |
| partial_close | 部分平仓 | symbol, close_percentage |

### B. 技术指标说明

- **EMA20**: 20周期指数移动平均线,趋势判断
- **MACD**: 移动平均收敛发散指标,动量判断
  - DIF: 快线 (12-26)
  - DEA: 慢线 (9日DIF均值)
  - Histogram: DIF - DEA
- **RSI7/14**: 7日和14日相对强弱指标,超买超卖判断
  - > 70: 超买
  - < 30: 超卖
  - 50: 中性

### C. 交易所对比

| 特性 | Binance | Hyperliquid | Aster |
|------|---------|-------------|-------|
| 手续费 | Maker 0.02% / Taker 0.04% | 0% / 0.025% | 自定义 |
| 最大杠杆 | 125x | 50x | 100x |
| 订单类型 | 限价/市价/止损 | 限价/IOC/Trigger | 全支持 |
| API 限制 | 1200次/分钟 | 无明确限制 | 自定义 |
| 结算币种 | USDT | USDC | 多币种 |

---

## 联系与贡献 (Contact & Contribution)

- **GitHub**: https://github.com/NOFX/nofx
- **文档**: https://docs.nofx.ai
- **Discord**: https://discord.gg/nofx
- **Email**: support@nofx.ai

欢迎提交 Issue 和 Pull Request!

---

**版本**: v1.0.0
**最后更新**: 2025-11-06
**作者**: NOFX Team
