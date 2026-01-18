# Rust Scripts 分析

> 分析日期: 2026-01-18
> 分支: learn

## 概述

`rust/scripts/` 目录包含 **7个脚本**,分为3类:

| 类型 | 脚本 | 用途 |
|-----|------|------|
| **监控分析** | `realtime_divergence.py`, `divergence_server.py` | 跟单效果追踪与分析 |
| **缓存构建** | `build_live_cache.py`, `build_sports_cache.py` | 市场数据本地缓存 |
| **数据采集** | `fetch_categorized_atp.py`, `fetch_ligue1.py` | 特定市场数据抓取 |
| **运维工具** | `start_divergence_monitor.sh` | 监控服务启动脚本 |

这些脚本是 **独立运行的Python程序**,与Rust核心bot **松散交互**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Scripts与Rust的交互关系                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Rust Bot (cargo run --release)                                     │
│       │                                                              │
│       │  读取缓存文件                                                 │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 文件交互 (JSON)                                               │   │
│  │ • .live_cache.json - 市场状态                                 │   │
│  │ • .atp_categorized.json - ATP市场分类                         │   │
│  │ • .ligue1_tokens.json - 法甲市场代币                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       │                                                              │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Python Scripts (独立运行)                                     │   │
│  │ • divergence_server.py - 监控面板                             │   │
│  │ • build_*.py - 缓存构建                                       │   │
│  │ • fetch_*.py - 数据采集                                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  特点:                                                               │
│  • 无进程间通信                                                     │
│  • 通过共享文件交互                                                 │
│  • 可独立启动/停止                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 脚本详解

### 1. 跟单效果监控 (最重要)

#### `realtime_divergence.py`

**功能**: 实时计算你(跟单者)与whale之间的收益/持仓偏离度

```
┌─────────────────────────────────────────────────────────────────────┐
│                     divergence计算逻辑                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  预期关系 (基于SCALING_RATIO):                                        │
│                                                                      │
│    我的收益 ≈ whale收益 × 8%                                         │
│    我的持仓 ≈ whale持仓 × 8%                                         │
│                                                                      │
│  实际计算:                                                           │
│                                                                      │
│    偏离度 = 实际值 - 预期值                                           │
│           = 我的PnL - (whale PnL × 0.08)                            │
│                                                                      │
│  数据来源:                                                           │
│  • https://data-api.polymarket.com/value?user={address}  # 持仓价值  │
│  • https://user-pnl-api.polymarket.com/user-pnl?user_address={addr}&interval=1d  # PnL │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**使用示例**:

```bash
# 默认每60秒更新
python realtime_divergence.py

# 每30秒更新
python realtime_divergence.py --interval 30

# 运行1小时
python realtime_divergence.py --duration 3600
```

**输出示例**:

```
2024-01-18 10:00:00 | INFO |
  You (0x123...):    Value: $500.00  |  Day PnL: +$25.00  |  Rank: 1500
  swisstony (0x456...): Value: $6,250.00  |  Day PnL: +$312.50  |  Rank: 5
  
  Expected PnL (8%): +$25.00
  Actual PnL:       +$25.00
  Divergence:       $0.00 (0.00%)
  Efficiency:       100.00%
```

#### `divergence_server.py`

**功能**: 将divergence监控包装成Web服务,提供可视化面板

```
┌─────────────────────────────────────────────────────────────────────┐
│                     divergence_server架构                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  数据流:                                                            │
│                                                                      │
│  Polymarket APIs                                                   │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────┐                                    │
│  │ divergence_server.py       │                                    │
│  │ • 定时抓取数据 (60秒)       │                                    │
│  │ • 存储到 SQLite            │                                    │
│  │ • 提供 Web Dashboard       │                                    │
│  └─────────────────────────────┘                                    │
│       │                                                             │
│       ▼                                                             │
│  http://localhost:8765                                              │
│                                                                      │
│  页面包含:                                                          │
│  • 实时偏离度仪表盘                                                  │
│  • 历史趋势图表                                                      │
│  • PnL效率分析                                                      │
│  • 持仓对比                                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**使用**:

```bash
# 前台运行
python divergence_server.py
# 访问 http://localhost:8765

# 后台运行 (使用screen)
./start_divergence_monitor.sh

# 检查状态
./start_divergence_monitor.sh --status

# 停止
./start_divergence_monitor.sh --stop
```

**数据库结构**:

```sql
CREATE TABLE snapshots (
    id INTEGER PRIMARY KEY,
    timestamp TEXT,
    user1_value REAL,      -- 我的持仓
    user1_pnl REAL,        -- 我的收益
    user2_value REAL,      -- whale持仓
    user2_pnl REAL,        -- whale收益
    expected_pnl REAL,     -- 预期收益
    pnl_vs_expected REAL,  -- 偏离度
    pnl_efficiency REAL,   -- 效率
    scaling_ratio REAL     -- 跟单比例
);
```

---

### 2. 市场缓存构建

#### `build_live_cache.py`

**功能**: 获取所有活跃体育市场的"直播状态",保存到 `.live_cache.json`

```python
# 从Gamma API获取所有活跃市场
# 提取每个token的 live=true/false 状态

# 输出格式:
# {
#   "token_id_1": true,   # 正在直播
#   "token_id_2": false,  # 未直播
#   ...
# }
```

**为什么需要这个?**

Rust bot在处理订单时需要知道市场是否"直播":

```rust
// lib.rs / settings.rs
pub fn get_gtd_expiry_secs(is_live: bool) -> u64 {
    if is_live { 61 }     // 直播市场: 短过期时间
    else { 1800 }         # 非直播: 长过期时间
}
```

**定时任务**:

```bash
# crontab -e
*/2 * * * * cd rust && python3 scripts/build_live_cache.py >> /tmp/live_cache.log 2>&1
```

#### `build_sports_cache.py`

**功能**: 类似 `build_live_cache.py`,但专门处理体育市场

---

### 3. 特定市场数据采集

#### `fetch_categorized_atp.py`

**功能**: 抓取ATP网球市场并按类型分类

```
分类类型:
• moneyline - 胜负投注
• set_handicap - 盘口让分
• game_totals - 总局数大小
• set_totals - 盘数大小
• tournament_winner - 赛事冠军
• other - 其他
```

**用途**: Rust bot的 `tennis_markets.rs` 模块使用此缓存进行特殊处理

```rust
// tennis_markets.rs
// 根据市场类型应用不同的风险参数
```

**定时任务**:

```bash
0 */4 * * * cd rust && python3 scripts/fetch_categorized_atp.py > /tmp/atp_cache_update.log 2>&1
```

#### `fetch_ligue1.py`

**功能**: 抓取法甲足球市场的代币列表

**用途**: 用于识别法甲相关市场,可能用于特殊策略

```json
// 输出 .ligue1_tokens.json
[
  "token_id_1",
  "token_id_2",
  ...
]
```

---

### 4. 运维脚本

#### `start_divergence_monitor.sh`

**功能**: 管理 `divergence_server.py` 的生命周期

```bash
# 启动 (后台screen)
./start_divergence_monitor.sh

# 前台运行
./start_divergence_monitor.sh --fg

# 查看状态
./start_divergence_monitor.sh --status

# 停止
./start_divergence_monitor.sh --stop
```

---

## 与Rust Bot的交互

### 交互方式: 共享文件

```
┌─────────────────────────────────────────────────────────────────────┐
│                     文件交互图                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Python Scripts                     Rust Bot                        │
│       │                                 │                            │
│       │  写入 .live_cache.json          │                            │
│       └────────────────────────────────►│  读取,用于:               │
│                                          │  • GTD过期时间计算        │
│                                          │  • 市场状态判断           │
│                                          │                           │
│       │  写入 .atp_categorized.json     │                            │
│       └────────────────────────────────►│  读取,用于:               │
│                                          │  • 网球市场特殊处理        │
│                                          │  • 风险参数调整           │
│                                          │                           │
│       │  读取 matches_optimized.csv     │                            │
│       │◄────────────────────────────────┘  写入,交易日志             │
│                                          │                           │
│       │  读取 divergence_data.db        │                            │
│       │◄────────────────────────────────┘  (divergence_server.py写入) │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 缓存文件列表

| 文件 | 来源 | 用途 |
|-----|------|------|
| `.live_cache.json` | build_live_cache.py | 市场直播状态 |
| `.atp_categorized.json` | fetch_categorized_atp.py | ATP市场分类 |
| `.ligue1_tokens.json` | fetch_ligue1.py | 法甲市场代币 |
| `matches_optimized.csv` | Rust bot | 交易记录 |
| `divergence_data.db` | divergence_server.py | 偏离度数据 |

---

## 数据流全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         完整数据流                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Polymarket APIs                          │   │
│  │  • gamma-api.polymarket.com (市场信息)                      │   │
│  │  • data-api.polymarket.com (持仓/PnL)                       │   │
│  │  • user-pnl-api.polymarket.com (收益)                       │   │
│  │  • clob.polymarket.com (订单执行)                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│              ┌───────────────┴───────────────┐                      │
│              ▼                               ▼                      │
│  ┌─────────────────────┐         ┌─────────────────────┐           │
│  │ 缓存构建脚本         │         │  Rust Bot           │           │
│  │ • build_live_cache  │         │  • 读取缓存          │           │
│  │ • fetch_categorized │         │  • 监听区块链        │           │
│  │ • fetch_ligue1      │         │  • 执行订单          │           │
│  └──────────┬──────────┘         └──────────┬──────────┘           │
│             │                               │                       │
│             │  JSON文件                      │ CSV日志               │
│             ▼                               ▼                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      共享文件层                               │   │
│  │  • .live_cache.json                                         │   │
│  │  • .atp_categorized.json                                    │   │
│  │  • .ligue1_tokens.json                                      │   │
│  │  • matches_optimized.csv                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    divergence_server.py                     │   │
│  │  • 读取Rust日志                                              │   │
│  │  • 抓取API数据                                               │   │
│  │  • 存储SQLite                                               │   │
│  │  • Web面板                                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 为什么这样设计?

### 1. 分离关注点

| 层 | 技术 | 职责 |
|---|-----|------|
| **核心交易** | Rust | 实时监控、订单执行、高性能 |
| **数据采集** | Python | API调用、数据处理、缓存构建 |
| **监控分析** | Python | 可视化、趋势分析 |

### 2. 各自优势

| 任务 | 推荐语言 | 原因 |
|-----|---------|------|
| 区块链监听 | Rust | 低延迟、内存安全 |
| HTTP API调用 | Python | 生态丰富、容易调试 |
| 数据分析 | Python | Pandas、NumPy |
| Web服务 | Python | aiohttp、简洁 |

### 3. 文件交互的优势

- **简单**: 无需进程通信
- **可靠**: 文件系统保证原子性
- **灵活**: 可独立更新任一组件
- **可观测**: 文件内容可直接查看

---

## 使用建议

### 最小配置运行

只需要启动Rust bot:

```bash
cargo run --release
```

### 添加监控 (推荐)

```bash
# 1. 启动divergence监控
./start_divergence_monitor.sh

# 2. 打开浏览器
# http://localhost:8765
```

### 完整运维

```bash
# 定时任务 (crontab)
# 每2分钟更新live缓存
*/2 * * * * cd rust && python3 scripts/build_live_cache.py >> /tmp/live_cache.log 2>&1

# 每4小时更新ATP缓存
0 */4 * * * cd rust && python3 scripts/fetch_categorized_atp.py > /tmp/atp_cache.log 2>&1

# divergence_server持续运行
./start_divergence_monitor.sh
```

---

## 总结

| 脚本类型 | 脚本 | 重要性 | 与Rust交互 |
|---------|------|-------|-----------|
| **监控分析** | divergence_server.py | ⭐⭐⭐ 核心 | 读取日志、共享数据库 |
| **监控分析** | realtime_divergence.py | ⭐⭐ 辅助 | 独立运行 |
| **缓存构建** | build_live_cache.py | ⭐⭐⭐ 核心 | 写入.json供Rust读取 |
| **缓存构建** | build_sports_cache.py | ⭐ 辅助 | 写入.json |
| **数据采集** | fetch_categorized_atp.py | ⭐ 辅助 | 写入.json |
| **数据采集** | fetch_ligue1.py | ⭐ 辅助 | 写入.json |
| **运维** | start_divergence_monitor.sh | ⭐⭐ 辅助 | 管理divergence_server |

**核心交互**: 通过 **JSON缓存文件** 和 **CSV日志文件** 进行数据交换。
