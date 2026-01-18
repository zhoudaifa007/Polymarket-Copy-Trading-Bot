# main.rs 核心函数深度分析

> 分析日期: 2026-01-18
> 分支: learn
> 文件: rust/src/main.rs (933行)

## 目录

1. [函数调用关系概览](#函数调用关系概览)
2. [main() - 程序入口](#main---程序入口)
3. [run_ws_loop() - WebSocket循环](#run_ws_loop---websocket循环)
4. [parse_event() - 事件解析](#parse_event---事件解析)
5. [handle_event() - 事件处理](#handle_event---事件处理)
6. [OrderEngine::submit() - 订单提交](#orderengine-submit---订单提交)
7. [order_worker() - 订单处理](#order_worker---订单处理)
8. [calculate_safe_size() - 仓位计算](#calculate_safe_size---仓位计算)
9. [resubmit_worker() - 重试处理](#resubmit_worker---重试处理)
10. [辅助函数](#辅助函数)
11. [数据流总结](#数据流总结)

---

## 函数调用关系概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                     main.rs 函数调用关系                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  main()                                                              │
│    ├── market_cache::init_caches()                                  │
│    ├── market_cache::spawn_cache_refresh_task()                     │
│    ├── Config::from_env()                                           │
│    ├── build_worker_state()                                         │
│    ├── start_order_worker() ─────────────────────────────────────┐  │
│    │   └── order_worker()                                      │  │
│    │       ├── process_order()                                 │  │
│    │       │   ├── calculate_safe_size()                       │  │
│    │       │   ├── guard.check() (RiskGuard)                   │  │
│    │       │   └── fetch_book_depth_blocking()                 │  │
│    │       └── resubmit_tx.send()                              │  │
│    ├── tokio::spawn(resubmit_worker()) ────────────────────────┐  │
│    │   └── process_resubmit_chain()                            │  │
│    └── loop run_ws_loop() ─────────────────────────────────────┐  │
│        ├── connect_async()                                      │  │
│        ├── ws.send(subscription)                                │  │
│        ├── ws.next() ───────────────────────────────────────┐  │  │
│        │   └── parse_event()                                  │  │  │
│        └── handle_event()                                      │  │
│            ├── fetch_is_live()                                 │  │
│            ├── order_engine.submit()                           │  │
│            └── fetch_best_book()                               │  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## main() - 程序入口

**位置**: 第80-130行

### 核心逻辑

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // 1. 加载环境变量
    dotenv().ok();
    
    // 2. 初始化市场缓存
    market_cache::init_caches();
    let _cache_refresh_handle = market_cache::spawn_cache_refresh_task();
    
    // 3. 加载配置
    let cfg = Config::from_env().await?;
    
    // 4. 构建工作状态 (CLOB客户端 + API凭证)
    let (client, creds) = build_worker_state(
        cfg.private_key.clone(),
        cfg.funder_address.clone(),
        ".clob_market_cache.json",
        ".clob_creds.json",
    ).await?;
    
    let prepared_creds = PreparedCreds::from_api_creds(&creds)?;
    let risk_config = cfg.risk_guard_config();
    
    // 5. 创建消息队列
    let (order_tx, order_rx) = mpsc::channel(1024);           // 订单队列
    let (resubmit_tx, resubmit_rx) = mpsc::unbounded_channel(); // 重试队列
    
    let client_arc = Arc::new(client);
    let creds_arc = Arc::new(prepared_creds.clone());
    
    // 6. 启动订单处理Worker
    start_order_worker(
        order_rx, 
        client_arc.clone(), 
        prepared_creds, 
        cfg.enable_trading, 
        cfg.mock_trading, 
        risk_config, 
        resubmit_tx.clone()
    );
    
    // 7. 启动重试Worker
    tokio::spawn(resubmit_worker(resubmit_rx, client_arc, creds_arc));
    
    // 8. 创建订单引擎
    let order_engine = OrderEngine {
        tx: order_tx,
        resubmit_tx,
        enable_trading: cfg.enable_trading,
    };
    
    println!(
        "🚀 Starting trader. Trading: {}, Mock: {}",
        cfg.enable_trading, cfg.mock_trading
    );
    
    // 9. WebSocket主循环 (无限运行)
    loop {
        if let Err(e) = run_ws_loop(&cfg.wss_url, &order_engine).await {
            eprintln!("⚠️ WS error: {e}. Reconnecting...");
            tokio::time::sleep(WS_RECONNECT_DELAY).await;
        }
    }
}
```

### 职责说明

| 步骤 | 操作 | 说明 |
|-----|------|------|
| 1 | 加载.env | 环境变量配置 |
| 2 | 初始化缓存 | 市场数据本地缓存 |
| 3 | 加载配置 | 从环境变量读取配置 |
| 4 | 构建客户端 | CLOB API客户端和凭证 |
| 5 | 创建队列 | 订单队列1024缓冲 |
| 6 | 启动Worker | 异步处理订单 |
| 7 | 启动重试 | 处理失败订单 |
| 8 | WebSocket循环 | 监听区块链事件 |

### 线程本地缓冲区

```rust
thread_local! {
    static CSV_BUF: RefCell<String> = RefCell::new(String::with_capacity(512));
    static SANITIZE_BUF: RefCell<String> = RefCell::new(String::with_capacity(128));
    static TOKEN_ID_CACHE: RefCell<HashMap<[u8; 32], Arc<str>>> = RefCell::new(HashMap::with_capacity(256));
}
```

**用途**:
- `CSV_BUF`: CSV日志格式化,避免频繁分配
- `SANITIZE_BUF`: CSV转义处理
- `TOKEN_ID_CACHE`: Token ID字符串缓存,提升解析性能

---

## run_ws_loop() - WebSocket循环

**位置**: 第251-294行

### 核心逻辑

```rust
async fn run_ws_loop(wss_url: &str, order_engine: &OrderEngine) -> Result<()> {
    // 1. 连接WebSocket
    let (mut ws, _) = connect_async(wss_url).await?;
    
    // 2. 构建订阅消息
    let sub = serde_json::json!({
        "jsonrpc": "2.0", 
        "id": 1, 
        "method": "eth_subscribe",
        "params": ["logs", {
            "address": MONITORED_ADDRESSES,           // CLOB合约地址列表
            "topics": [
                [ORDERS_FILLED_EVENT_SIGNATURE],      // 事件签名过滤
                Value::Null,                          // 不过滤maker
                TARGET_TOPIC_HEX.as_str()              // 只监控目标whale地址
            ]
        }]
    }).to_string();
    
    println!("🔌 Connected. Subscribing...");
    ws.send(Message::Text(sub)).await?;
    
    let http_client = reqwest::Client::builder().no_proxy().build()?;
    
    // 3. 事件循环
    loop {
        let msg = tokio::time::timeout(WS_PING_TIMEOUT, ws.next()).await
            .map_err(|_| anyhow!("WS timeout"))?
            .ok_or_else(|| anyhow!("WS closed"))??;
        
        match msg {
            Message::Text(text) => {
                if let Some(evt) = parse_event(text) {
                    // 异步处理,避免阻塞
                    let engine = order_engine.clone();
                    let client = http_client.clone();
                    tokio::spawn(async move { 
                        handle_event(evt, &engine, &client).await 
                    });
                }
            }
            Message::Binary(bin) => {
                if let Ok(text) = String::from_utf8(bin) {
                    if let Some(evt) = parse_event(text) {
                        let engine = order_engine.clone();
                        let client = http_client.clone();
                        tokio::spawn(async move { 
                            handle_event(evt, &engine, &client).await 
                        });
                    }
                }
            }
            Message::Ping(d) => { ws.send(Message::Pong(d)).await?; }
            Message::Close(f) => return Err(anyhow!("WS closed: {:?}", f)),
            _ => {}
        }
    }
}
```

### 订阅参数详解

| 参数 | 值 | 说明 |
|-----|-----|------|
| `address` | `["0x4bFb41d5B35...", "0x4d97dcd9...", "0xC5d563A3..."]` | Polymarket CLOB合约地址 |
| `topics[0]` | `ORDERS_FILLED_EVENT_SIGNATURE` | 事件签名: `0xd0a08e8c...` |
| `topics[1]` | `null` | 不过滤maker地址 |
| `topics[2]` | `TARGET_TOPIC_HEX` | 只接收目标whale的交易 |

### 设计特点

1. **异步处理**: 每个事件通过 `tokio::spawn` 异步处理
2. **超时保护**: WS_PING_TIMEOUT (300秒) 防止长时间阻塞
3. **自动重连**: 主循环外层捕获错误后重连
4. **二进制兼容**: 同时处理Text和Binary消息

---

## parse_event() - 事件解析

**位置**: 第778-833行

### 核心逻辑

```rust
fn parse_event(message: String) -> Option<ParsedEvent> {
    // 1. 解析JSON
    let msg: WsMessage = serde_json::from_str(&message).ok()?;
    let result = msg.params?.result?;
    
    // 2. 验证topics长度
    if result.topics.len() < 3 { return None; }
    
    // 3. 验证whale地址 (关键过滤)
    let has_target = result.topics.get(2)
        .map(|t| t.eq_ignore_ascii_case(TARGET_TOPIC_HEX.as_str()))
        .unwrap_or(false);
    if !has_target { return None; }
    
    // 4. 验证data长度
    let hex_data = &result.data;
    if hex_data.len() < 2 + 64 * 4 { return None; }
    
    // 5. 解析maker/taker asset ID
    let (maker_id, maker_bytes) = parse_u256_hex_slice_with_bytes(hex_data, 2, 66)?;
    let (taker_id, taker_bytes) = parse_u256_hex_slice_with_bytes(hex_data, 66, 130)?;
    
    // 6. 判断交易方向
    let (clob_id, token_bytes, maker_amt, taker_amt, base_type) =
        if maker_id.is_zero() && !taker_id.is_zero() {
            // maker没有资产,买入taker资产
            let m = parse_u256_hex_slice(hex_data, 130, 194)?;
            let t = parse_u256_hex_slice(hex_data, 194, 258)?;
            (taker_id, taker_bytes, m, t, "BUY")
        } else if taker_id.is_zero() && !maker_id.is_zero() {
            // taker没有资产,卖出maker资产
            let m = parse_u256_hex_slice(hex_data, 130, 194)?;
            let t = parse_u256_hex_slice(hex_data, 194, 258)?;
            (maker_id, maker_bytes, m, t, "SELL")
        } else {
            return None;  // 不支持的交易类型
        };
    
    // 7. 计算份额和价格
    let shares = if base_type == "BUY" { 
        u256_to_f64(&taker_amt)? 
    } else { 
        u256_to_f64(&maker_amt)? 
    } / 1e6;  // USDC 6位小数
    
    if shares <= 0.0 { return None; }
    
    let usd = if base_type == "BUY" { 
        u256_to_f64(&maker_amt)? 
    } else { 
        u256_to_f64(&taker_amt)? 
    } / 1e6;
    
    let price = usd / shares;
    
    // 8. 构建结果
    let mut order_type = base_type.to_string();
    if result.topics[0].eq_ignore_ascii_case(ORDERS_FILLED_EVENT_SIGNATURE) {
        order_type.push_str("_FILL");
    }
    
    Some(ParsedEvent {
        block_number: result.block_number.as_deref()
            .and_then(|s| u64::from_str_radix(s.trim_start_matches("0x"), 16).ok())
            .unwrap_or_default(),
        tx_hash: result.transaction_hash.unwrap_or_default(),
        order: OrderInfo {
            order_type,
            clob_token_id: u256_to_dec_cached(&token_bytes, &clob_id),
            usd_value: usd,
            shares,
            price_per_share: price,
        },
    })
}
```

### 数据结构

```rust
pub struct OrderInfo {
    pub order_type: String,      // "BUY" 或 "SELL" 或 "BUY_FILL"
    pub clob_token_id: Arc<str>, // CLOB代币ID (订单簿中的token)
    pub usd_value: f64,          // 美元价值
    pub shares: f64,             // 份额数量
    pub price_per_share: f64,    // 每股价格
}

pub struct ParsedEvent {
    pub block_number: u64,       // 区块号
    pub tx_hash: String,         // 交易哈希
    pub order: OrderInfo,        // 订单信息
}
```

### RLP数据解析

```
data字段格式 (RLP编码的4个u256):
┌─────────────────────────────────────────────────────────────────┐
│ Offset │ Size  │ Field                              │ 说明     │
├────────┼────────┼────────────────────────────────────┼──────────┤
│ 2      │ 64     │ maker_asset_id (32 bytes)         │ Maker代币 │
│ 66     │ 64     │ taker_asset_id (32 bytes)         │ Taker代币 │
│ 130    │ 64     │ maker_amount (32 bytes)           │ Maker数量 │
│ 194    │ 64     │ taker_amount (32 bytes)           │ Taker数量 │
└─────────────────────────────────────────────────────────────────┘

交易方向判断:
• maker_asset_id == 0 && taker_asset_id != 0 → 买入taker_asset_id
• taker_asset_id == 0 && maker_asset_id != 0 → 卖出maker_asset_id
```

### 性能优化

```rust
// 1. Token ID缓存 (61% → 3.5% 解析时间下降)
fn u256_to_dec_cached(bytes: &[u8; 32], val: &U256) -> Arc<str> {
    TOKEN_ID_CACHE.with(|cache| {
        let mut cache = cache.borrow_mut();
        if let Some(s) = cache.get(bytes) { return Arc::clone(s); }
        let s: Arc<str> = val.to_string().into();
        cache.insert(*bytes, Arc::clone(&s));
        s
    })
}

// 2. 快速u256转f64 (6.9% → 0.1% 解析时间下降)
fn u256_to_f64(v: &U256) -> Option<f64> {
    if v.bit_len() <= 64 { 
        Some(v.as_limbs()[0] as f64)  // 快速路径
    } else { 
        v.to_string().parse().ok()     // 慢速路径
    }
}

// 3. 快速hex解析 (142倍加速)
const HEX_NIBBLE_LUT: [u8; 256] = { ... };
#[inline(always)]
fn hex_nibble(b: u8) -> Option<u8> {
    let val = HEX_NIBBLE_LUT[b as usize];
    if val == 255 { None } else { Some(val) }
}
```

---

## handle_event() - 事件处理

**位置**: 第296-360行

### 核心逻辑

```rust
async fn handle_event(
    evt: ParsedEvent, 
    order_engine: &OrderEngine, 
    http_client: &reqwest::Client
) {
    // 1. 检查市场是否直播 (先查缓存,没有则API查询)
    let is_live = match market_cache::get_is_live(&evt.order.clob_token_id) {
        Some(v) => Some(v),  // 缓存命中
        None => fetch_is_live(&evt.order.clob_token_id, http_client).await,  // API查询
    };
    
    // 2. 提交到订单引擎
    let status = order_engine.submit(evt.clone(), is_live).await;
    
    // 3. 延迟等待订单成交
    tokio::time::sleep(Duration::from_secs_f32(2.8)).await;
    
    // 4. 获取订单簿信息 (用于日志)
    let bests = fetch_best_book(
        &evt.order.clob_token_id, 
        &evt.order.order_type, 
        http_client
    ).await;
    let ((bp, bs), (sp, ss)) = bests.unwrap_or_else(|| {
        (("N/A".into(), "N/A".into()), ("N/A".into(), "N/A".into()))
    });
    let is_live = is_live.unwrap_or(false);
    
    // 5. 生成市场类型标识
    let tennis_display = if tennis_markets::get_tennis_token_buffer(&evt.order.clob_token_id) > 0.0 {
        "\x1b[32m(TENNIS)\x1b[0m "
    } else {
        ""
    };
    
    let soccer_display = if soccer_markets::get_soccer_token_buffer(&evt.order.clob_token_id) > 0.0 {
        "\x1b[36m(SOCCER)\x1b[0m "
    } else {
        ""
    };
    
    // 6. 打印交易日志 (带颜色)
    println!(
        "⚡ [B:{}] {}{}{} | ${:.0} | {} | best: {} @ {} | 2nd: {} @ {} | {}",
        evt.block_number, 
        tennis_display, 
        soccer_display, 
        evt.order.order_type, 
        evt.order.usd_value, 
        status, 
        format!("\x1b[38;5;199m{}\x1b[0m", bp),  // 亮粉色高亮
        bs, 
        sp, 
        ss, 
        if is_live { "\x1b[34mlive: true\x1b[0m" } else { "live: false" }
    );
    
    // 7. 写入CSV日志 (阻塞在后台任务)
    let ts: DateTime<Utc> = Utc::now();
    let row = CSV_BUF.with(|buf| {
        SANITIZE_BUF.with(|sbuf| {
            let mut b = buf.borrow_mut();
            let mut sb = sbuf.borrow_mut();
            sanitize_csv(&status, &mut sb);
            b.clear();
            let _ = write!(b,
                "{},{},{},{:.2},{:.6},{:.4},{},{},{},{},{},{},{},{}",
                ts.format("%Y-%m-%d %H:%M:%S%.3f"),
                evt.block_number, 
                evt.order.clob_token_id,
                evt.order.usd_value,
                evt.order.shares, 
                evt.order.price_per_share, 
                evt.order.order_type,
                sb, bp, bs, sp, ss, 
                evt.tx_hash, 
                is_live
            );
            b.clone()
        })
    });
    let _ = tokio::task::spawn_blocking(move || append_csv_row(row)).await;
}
```

### 延迟2.8秒的作用

```rust
tokio::time::sleep(Duration::from_secs_f32(2.8)).await;
```

**目的**: 等待订单成交完成后再查询订单簿状态

**原因**: 
- FAK订单是"立即成交或取消"
- 提交后需要时间确认是否成交
- 2.8秒足够大多数订单成交

---

## OrderEngine::submit() - 订单提交

**位置**: 第57-74行

### 核心逻辑

```rust
#[derive(Clone)]
struct OrderEngine {
    tx: mpsc::Sender<WorkItem>,
    resubmit_tx: mpsc::UnboundedSender<ResubmitRequest>,
    enable_trading: bool,
}

impl OrderEngine {
    async fn submit(&self, evt: ParsedEvent, is_live: Option<bool>) -> String {
        // 1. 检查是否启用交易
        if !self.enable_trading {
            return "SKIPPED_DISABLED".into();
        }
        
        // 2. 发送到异步队列 (非阻塞)
        let (resp_tx, resp_rx) = oneshot::channel();
        if let Err(e) = self.tx.try_send(WorkItem { 
            event: evt, 
            respond_to: resp_tx, 
            is_live 
        }) {
            return format!("QUEUE_ERR: {e}");
        }
        
        // 3. 等待处理结果 (超时10秒)
        match tokio::time::timeout(ORDER_REPLY_TIMEOUT, resp_rx).await {
            Ok(Ok(msg)) => msg,           // 处理成功
            Ok(Err(_)) => "WORKER_DROPPED".into(),  // worker已丢弃
            Err(_) => "WORKER_TIMEOUT".into(),       // 超时
        }
    }
}
```

### WorkItem结构

```rust
pub struct WorkItem {
    pub event: ParsedEvent,              // 解析后的事件
    pub respond_to: oneshot::Sender<String>,  // 结果通道
    pub is_live: Option<bool>,           // 市场是否直播
}
```

### 设计特点

1. **非阻塞提交**: `try_send` 立即返回,不等待
2. **超时保护**: ORDER_REPLY_TIMEOUT (10秒) 防止无限等待
3. **结果通道**: 使用oneshot接收处理结果
4. **状态检查**: 快速检查是否启用交易

---

## order_worker() - 订单处理

**位置**: 第157-167行 (函数体为空,需要查看完整实现)

根据代码结构和 `settings.rs` 配置,推断核心逻辑:

```rust
fn order_worker(
    mut rx: mpsc::Receiver<WorkItem>,
    client: Arc<RustClobClient>,
    creds: PreparedCreds,
    enable_trading: bool,
    mock_trading: bool,
    guard: &mut RiskGuard,
    resubmit_tx: &mpsc::UnboundedSender<ResubmitRequest>,
) {
    while let Some(work) = rx.recv().await {
        let whale_shares = work.event.order.shares;
        let whale_price = work.event.order.price_per_share;
        
        // 1. 风控检查
        let safety = guard.check(&work.event.order.clob_token_id, whale_shares);
        
        match safety.decision {
            SafetyDecision::Block => {
                // 被风控阻止
                let _ = work.respond_to.send(format!("BLOCKED_{}", safety.reason.as_str()));
                continue;
            }
            SafetyDecision::FetchBook => {
                // 需要验证订单簿深度
                let depth = fetch_book_depth_blocking(
                    &client,
                    &work.event.order.clob_token_id,
                    TradeSide::Buy,
                    200.0  // 阈值
                );
                if depth < 200.0 {
                    let _ = work.respond_to.send("BLOCKED_LOW_LIQUIDITY".into());
                    continue;
                }
            }
            SafetyDecision::Allow => { /* 继续处理 */ }
        }
        
        // 2. 计算安全仓位
        let (safe_size, size_type) = calculate_safe_size(
            whale_shares, 
            whale_price, 
            size_multiplier
        );
        
        // 3. 执行订单
        let result = process_order(
            &work.event.order,
            &mut client.clone(),
            &creds,
            enable_trading,
            mock_trading,
            guard,
            &resubmit_tx,
            work.is_live
        );
        
        // 4. 发送结果
        let _ = work.respond_to.send(result);
    }
}
```

---

## calculate_safe_size() - 仓位计算

**位置**: 第186-188行 (函数体为空)

根据 `settings.rs` 中的配置推断核心逻辑:

```rust
fn calculate_safe_size(
    whale_shares: f64, 
    price: f64, 
    size_multiplier: f64
) -> (f64, SizeType) {
    // 根据whale交易规模确定层级
    let (base_multiplier, price_buffer) = if whale_shares >= 4000.0 {
        // 大单层级
        (1.25, 0.01)  // 2.5%仓位, +1%价格缓冲
    } else if whale_shares >= 2000.0 {
        // 中单层级
        (1.0, 0.01)   // 2%仓位, +1%价格缓冲
    } else {
        // 小单层级
        (1.0, 0.00)   // 2%仓位, 无缓冲
    };
    
    // 计算最终仓位
    let my_shares = whale_shares * SCALING_RATIO * base_multiplier * size_multiplier;
    let my_price = price + price_buffer;
    
    (my_shares, SizeType::Scaled)
}
```

### 执行层级配置 (settings.rs)

```rust
pub const EXECUTION_TIERS: [ExecutionTier; 3] = [
    ExecutionTier {
        min_shares: 4000.0,
        price_buffer: 0.01,      // +1%
        order_action: "FAK",
        size_multiplier: 1.25,   // 2.5%
    },
    ExecutionTier {
        min_shares: 2000.0,
        price_buffer: 0.01,      // +1%
        order_action: "FAK",
        size_multiplier: 1.0,    // 2%
    },
    ExecutionTier {
        min_shares: 1000.0,
        price_buffer: 0.00,      // 无缓冲
        order_action: "FAK",
        size_multiplier: 1.0,    // 2%
    },
];
```

---

## resubmit_worker() - 重试处理

**位置**: 第366-514行

### 核心逻辑

```rust
async fn resubmit_worker(
    mut rx: mpsc::UnboundedReceiver<ResubmitRequest>,
    client: Arc<RustClobClient>,
    creds: Arc<PreparedCreds>,
) {
    println!("🔄 Resubmitter worker started");
    
    while let Some(req) = rx.recv().await {
        // 1. 确定重试策略
        let max_attempts = get_max_resubmit_attempts(req.whale_shares);
        let is_last_attempt = req.attempt >= max_attempts;
        
        // 2. 计算新价格
        let increment = if should_increment_price(req.whale_shares, req.attempt) {
            RESUBMIT_PRICE_INCREMENT  // +0.01
        } else {
            0.0
        };
        
        let new_price = if req.side_is_buy {
            (req.failed_price + increment).min(0.99)
        } else {
            (req.failed_price - increment).max(0.01)
        };
        
        // 3. 价格上限检查
        if !is_last_attempt && req.side_is_buy && new_price > req.max_price {
            println!(
                "🔄 Resubmit ABORT: attempt {} price {:.2} > max {:.2} | ...",
                req.attempt, new_price, req.max_price
            );
            continue;
        }
        
        // 4. 提交订单
        let result = tokio::task::spawn_blocking(move || {
            submit_resubmit_order_sync(
                &client, &creds, 
                &req.token_id, new_price, req.size, 
                req.is_live, is_last_attempt
            )
        }).await;
        
        match result {
            Ok(Ok((true, _, filled_this_attempt))) => {
                if is_last_attempt {
                    // GTD订单: 挂单成功,等待成交
                    println!(
                        "\x1b[32m🔄 Resubmit GTD SUBMITTED: attempt {} @ {:.2} ...\x1b[0m",
                        req.attempt, new_price
                    );
                } else {
                    // FAK订单: 完全成交
                    let total_filled = req.cumulative_filled + filled_this_attempt;
                    println!(
                        "\x1b[32m🔄 Resubmit SUCCESS: attempt {} @ {:.2} ...\x1b[0m",
                        req.attempt, new_price
                    );
                }
            }
            Ok(Ok((false, body, filled_this_attempt))) => {
                if req.attempt < max_attempts {
                    // 继续重试
                    let next_req = ResubmitRequest { ... };
                    let _ = process_resubmit_chain(&client, &creds, next_req).await;
                } else {
                    // 重试次数耗尽
                    println!("🔄 Resubmit FAILED: attempt {} ...", req.attempt);
                }
            }
            Ok(Err(e)) => {
                println!("🔄 Resubmit ERROR: {}", e);
            }
            Err(e) => {
                println!("🔄 Resubmit TASK ERROR: {}", e);
            }
        }
    }
}
```

### 重试策略配置

```rust
// 最大重试次数 (settings.rs)
#[inline]
pub fn get_max_resubmit_attempts(whale_shares: f64) -> u8 {
    if whale_shares >= 4000.0 { 5 } else { 4 }
}

// 是否应该加价 (settings.rs)
#[inline]
pub fn should_increment_price(whale_shares: f64, attempt: u8) -> bool {
    if whale_shares >= 4000.0 {
        attempt == 1  // 只有第一次尝试加价
    } else {
        false  // 小单不平价追单
    }
}

// 重试价格递增 (settings.rs)
pub const RESUBMIT_PRICE_INCREMENT: f64 = 0.01;
```

### ResubmitRequest结构

```rust
pub struct ResubmitRequest {
    pub token_id: String,           // 代币ID
    pub whale_price: f64,           // 原始whale价格
    pub failed_price: f64,          // 失败价格
    pub size: f64,                  // 订单大小
    pub whale_shares: f64,          // whale份额(用于分层)
    pub max_price: f64,             // 价格上限
    pub cumulative_filled: f64,     // 累计成交
    pub original_size: f64,         // 原始订单大小
    pub side_is_buy: bool,          // 方向
    pub attempt: u8,                // 当前尝试次数
    pub is_live: bool,              // 市场是否直播
}
```

---

## 辅助函数

### fetch_is_live() - 获取市场状态

**位置**: 第708-721行

```rust
async fn fetch_is_live(token_id: &str, client: &reqwest::Client) -> Option<bool> {
    // 1. 获取市场slug
    let market_url = format!(
        "{}/markets?clob_token_ids={}", 
        GAMMA_API_BASE, 
        token_id
    );
    let resp = client.get(&market_url)
        .timeout(Duration::from_secs(2))
        .send()
        .await.ok()?;
    let val: Value = resp.json().await.ok()?;
    let slug = val.get(0)?.get("slug")?.as_str()?.to_string();
    
    // 2. 获取直播状态
    let event_url = format!("{}/events/slug/{}", GAMMA_API_BASE, slug);
    let resp = client.get(&event_url)
        .timeout(Duration::from_secs(2))
        .send()
        .await.ok()?;
    let val: Value = resp.json().await.ok()?;
    
    Some(val["live"].as_bool().unwrap_or(false))
}
```

### fetch_best_book() - 获取最优档位

**位置**: 第723-772行

```rust
async fn fetch_best_book(
    token_id: &str, 
    order_type: &str, 
    client: &reqwest::Client
) -> Option<((String, String), (String, String))> {
    let url = format!("{}/book?token_id={}", CLOB_API_BASE, token_id);
    let resp = client.get(&url)
        .timeout(BOOK_REQ_TIMEOUT)
        .send()
        .await.ok()?;
    if !resp.status().is_success() { return None; }
    
    let val: Value = resp.json().await.ok()?;
    let key = if order_type.starts_with("BUY") { "asks" } else { "bids" };
    let entries = val.get(key)?.as_array()?;
    
    // 查找最优和次优档位
    let is_buy = order_type.starts_with("BUY");
    let (best, second) = entries.iter().fold((None, None), |(best, second), entry| {
        let price: f64 = entry.get("price").and_then(|v| v.as_str()).and_then(|s| s.parse().ok())?;
        let better = |candidate: f64, current: f64| {
            if is_buy { candidate < current } else { candidate > current }
        };
        // ... 逻辑处理
        Some(((best_price, best_size), (second_price, second_size)))
    });
    
    best.map(|(bp, bs)| ((bp, bs), (sp, ss)))
}
```

### fetch_book_depth_blocking() - 订单簿深度检查

**位置**: 第211-245行

```rust
fn fetch_book_depth_blocking(
    client: &RustClobClient,
    token_id: &str,
    side: TradeSide,
    threshold: f64,
) -> Result<f64, &'static str> {
    let url = format!("{}/book?token_id={}", CLOB_API_BASE, token_id);
    let resp = client.http_client()
        .get(&url)
        .timeout(Duration::from_millis(500))
        .send()
        .map_err(|_| "NETWORK")?;
    
    if !resp.status().is_success() { return Err("HTTP_ERROR"); }
    
    let book: Value = resp.json().map_err(|_| "PARSE")?;
    let key = if side == TradeSide::Buy { "asks" } else { "bids" };
    
    // 栈分配数组,避免Vec分配
    let mut levels: [(f64, f64); 10] = [(0.0, 0.0); 10];
    let mut count = 0;
    if let Some(arr) = book[key].as_array() {
        for lvl in arr.iter().take(10) {
            if let (Some(p), Some(s)) = (
                lvl["price"].as_str().and_then(|s| s.parse().ok()),
                lvl["size"].as_str().and_then(|s| s.parse().ok()),
            ) {
                levels[count] = (p, s);
                count += 1;
            }
        }
    }
    
    Ok(calc_liquidity_depth(side, &levels[..count], threshold))
}
```

### get_fill_color() - 填充率颜色

**位置**: 第191-198行

```rust
fn get_fill_color(filled: f64, requested: f64) -> &'static str {
    if requested <= 0.0 { return "\x1b[31m"; }  // 红色
    let pct = (filled / requested) * 100.0;
    if pct < 50.0 { "\x1b[31m" }                // 红色
    else if pct < 75.0 { "\x1b[38;5;208m" }     // 橙色
    else if pct < 90.0 { "\x1b[33m" }           // 黄色
    else { "\x1b[32m" }                          // 绿色
}
```

### get_whale_size_color() - Whale规模颜色

**位置**: 第201-209行

```rust
fn get_whale_size_color(shares: f64) -> &'static str {
    if shares < 500.0 { "\x1b[90m" }              // 灰色
    else if shares < 1000.0 { "\x1b[36m" }        // 青色
    else if shares < 2000.0 { "\x1b[34m" }        // 蓝色
    else if shares < 5000.0 { "\x1b[32m" }        // 绿色
    else if shares < 8000.0 { "\x1b[33m" }        // 黄色
    else if shares < 15000.0 { "\x1b[38;5;208m" } // 橙色
    else { "\x1b[35m" }                           // 洋红色
}
```

---

## 数据流总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                        核心数据流                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 区块链事件监听                                                    │
│     ┌──────────────────────────────────────────────────────────┐   │
│     │ WebSocket消息                                            │   │
│     │ {                                                         │   │
│     │   "topics": ["0xd0a0...", null, "0x00000...{addr}"],    │   │
│     │   "data": "0x{maker_asset_id}{taker_asset_id}...",      │   │
│     │   "blockNumber": "0x12345",                              │   │
│     │   "transactionHash": "0xabc..."                          │   │
│     │ }                                                         │   │
│     └──────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  2. 事件解析                                                         │
│     ┌──────────────────────────────────────────────────────────┐   │
│     │ parse_event()                                            │   │
│     │ • 验证topics[2]匹配whale地址                             │   │
│     │ • 解析data字段 (RLP编码的4个u256)                         │   │
│     │ • 判断交易方向 (BUY/SELL)                                 │   │
│     │ • 计算shares和price                                      │   │
│     │ • 返回ParsedEvent                                        │   │
│     └──────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  3. 事件处理                                                         │
│     ┌──────────────────────────────────────────────────────────┐   │
│     │ handle_event()                                           │   │
│     │ • 检查is_live (缓存/API)                                 │   │
│     │ • 提交到order_engine                                     │   │
│     │ • 延迟2.8秒                                              │   │
│     │ • 获取订单簿信息                                          │   │
│     │ • 打印日志+写入CSV                                        │   │
│     └──────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  4. 订单入队                                                         │
│     ┌──────────────────────────────────────────────────────────┐   │
│     │ OrderEngine::submit()                                    │   │
│     │ • 检查enable_trading                                     │   │
│     │ • 发送到mpsc队列                                          │   │
│     │ • 等待oneshot结果                                        │   │
│     └──────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  5. 订单处理 (order_worker)                                          │
│     ┌──────────────────────────────────────────────────────────┐   │
│     │ • RiskGuard.check()                                      │   │
│     │   - 熔断检查                                              │   │
│     │   - 序列检查                                              │   │
│     │   - 深度检查 (如需要)                                     │   │
│     │ • calculate_safe_size()                                  │   │
│     │ • process_order()                                        │   │
│     │   - 创建订单参数                                          │   │
│     │   - EIP-712签名                                          │   │
│     │   - POST /order                                          │   │
│     │ • 失败则发送到resubmit队列                                │   │
│     └──────────────────────────────────────────────────────────┘   │
│                              │                                       │
│              ┌───────────────┴───────────────┐                       │
│              ▼                               ▼                       │
│      成功                               失败                         │
│              │                               │                       │
│              ▼                               ▼                       │
│  6. 记录日志                           7. 重试队列                    │
│     CSV文件                          resubmit_worker()              │
│     • timestamp                      • 最多4-5次重试                  │
│     • block_number                   • 价格递增策略                  │
│     • clob_token_id                  • FAK → GTD                    │
│     • usd_value                                                       │
│     • shares                                                          │
│     • price_per_share                                                │
│     • status                                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键配置参数

| 参数 | 值 | 说明 |
|-----|-----|------|
| `ORDER_REPLY_TIMEOUT` | 10秒 | 订单处理超时 |
| `WS_PING_TIMEOUT` | 300秒 | WebSocket超时 |
| `WS_RECONNECT_DELAY` | 3秒 | 重连延迟 |
| `RESUBMIT_PRICE_INCREMENT` | 0.01 | 重试价格递增 |
| `SCALING_RATIO` | 0.02 | 默认仓位比例(2%) |
| `BOOK_REQ_TIMEOUT` | 2500ms | 订单簿请求超时 |
| `GET_PING_TIMEOUT` | 2秒 | API超时 |
| `RESUBMIT_DELAY` | 50ms | 小单重试延迟 |

---

## 总结

### 函数职责矩阵

| 函数 | 行号 | 职责 | 关键操作 |
|-----|------|------|---------|
| `main()` | 80-130 | 程序入口 | 初始化、启动Worker |
| `run_ws_loop()` | 251-294 | WebSocket循环 | 连接、订阅、事件分发 |
| `parse_event()` | 778-833 | 事件解析 | JSON解析、RLP解码 |
| `handle_event()` | 296-360 | 事件处理 | 提交订单、日志记录 |
| `OrderEngine::submit()` | 57-74 | 订单提交 | 队列发送、超时等待 |
| `order_worker()` | 157-167 | 订单处理 | 风控、仓位计算 |
| `calculate_safe_size()` | 186-188 | 仓位计算 | 分层策略 |
| `resubmit_worker()` | 366-514 | 重试处理 | 价格递增、FAK→GTD |
| `fetch_is_live()` | 708-721 | 市场状态 | API查询 |
| `fetch_best_book()` | 723-772 | 订单簿 | 最优档位 |

### 核心设计模式

1. **异步队列**: 使用mpsc通道解耦监听和处理
2. **超时保护**: 所有I/O操作都有超时
3. **重试机制**: 失败订单自动重试,最多4-5次
4. **缓存优化**: Token ID缓存、hex解析优化
5. **颜色输出**: 终端日志带颜色区分
