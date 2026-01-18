# AGENTS.md - Guidelines for AI Coding Agents

This document provides guidelines for AI coding agents working on the Polymarket Copy Trading Bot repository.

## Project Overview

This repository contains two implementations of a Polymarket copy trading bot:
- **Python Bot** (`python/`): Feature-rich implementation with MongoDB persistence, extensive research tools, and simulation capabilities
- **Rust Bot** (`rust/`): High-performance, production-ready implementation optimized for speed and reliability

Both bots monitor Polymarket traders and automatically replicate their trades with scaled position sizing.

## Build, Lint, and Test Commands

### Python Bot

```bash
# Install dependencies
pip install -r python/requirements.txt

# Run the bot
python -m src.main

# Run setup wizard
python -m src.scripts.setup.setup

# Check system status
python -m src.scripts.setup.system_status

# Run specific wallet scripts
python -m src.scripts.wallet.check_my_stats
python -m src.scripts.wallet.check_positions_detailed

# Run research scripts
python -m src.scripts.research.find_best_traders
python -m src.scripts.research.scan_best_traders

# Run simulation scripts
python -m src.scripts.simulation.simulate_profitability --trader ADDRESS
python -m src.scripts.simulation.audit_copy_trading

# Run position management (requires CLOB client)
python -m src.scripts.position.manual_sell
python -m src.scripts.position.close_stale_positions
```

**Note**: There are no formal test files in this repository. When adding tests, use `pytest` and place them in a `tests/` directory.

### Rust Bot

```bash
# Build in debug mode
cd rust && cargo build

# Build in release mode (recommended for production)
cd rust && cargo build --release

# Run the bot
cargo run --release

# Validate setup
cargo run --release --bin validate_setup

# Run specific utility binaries
cargo run --release --bin test_order_types    # Test FAK order responses
cargo run --release --bin mempool_monitor     # Monitor mempool activity
cargo run --release --bin trade_monitor       # Monitor specific trades

# Run tests (if added)
cargo test
cargo test --release
cargo test --bin test_order_types  # Run specific test binary

# Check for warnings
cargo clippy

# Format code
cargo fmt
```

## Code Style Guidelines

### Python Guidelines

#### Imports
- Use absolute imports with `src.` prefix: `from src.config.env import ENV`
- Group imports: standard library first, then third-party, then local
- Alphabetize within groups
- Example:
  ```python
  import asyncio
  import signal
  import sys
  from typing import List, Dict, Optional

  import websockets
  from colorama import Fore, Style

  from src.config.db import connect_db
  from src.services.trade_monitor import trade_monitor
  ```

#### Type Hints
- Use type hints for all function signatures
- Use `Optional[T]` instead of `T | None` for compatibility
- Use `List[T]`, `Dict[K, V]` from `typing`
- Example:
  ```python
  def process_trade_activity(activity: Dict[str, Any], address: str) -> None:
      ...

  async def fetch_data_async(url: str, headers: Optional[Dict[str, str]] = None) -> Dict[str, Any]:
      ...
  ```

#### Naming Conventions
- **Classes**: `PascalCase` (e.g., `TradeExecutor`, `UserHistory`)
- **Functions/variables**: `snake_case` (e.g., `get_my_balance`, `stop_trade_monitor`)
- **Constants**: `UPPER_SNAKE_CASE` (e.g., `MAX_RECONNECT_ATTEMPTS`, `RTDS_URL`)
- **Private methods**: prefix with underscore (e.g., `_validate_env()`)
- **Async functions**: prefix with `async_` where appropriate (e.g., `async def fetch_data_async`)

#### Error Handling
- Use try/except blocks with specific exception types
- Log errors using the `error()` logger function
- Re-raise exceptions after logging when appropriate
- Validate configuration at module load time with clear error messages
- Example:
  ```python
  try:
      await connect_db()
  except Exception as e:
      error(f'Database connection failed: {e}')
      raise

  if not USER_ADDRESSES:
      raise ValueError('USER_ADDRESSES is not defined or empty')
  ```

#### Async/Await Patterns
- Use `asyncio.create_task()` for background tasks
- Use `asyncio.gather()` for parallel operations
- Implement graceful shutdown with `asyncio.Event()`
- Handle `KeyboardInterrupt` for clean exits
- Example:
  ```python
  shutdown_event = asyncio.Event()

  async def main():
      monitor_task = asyncio.create_task(trade_monitor())
      executor_task = asyncio.create_task(trade_executor(clob_client))

      await shutdown_event.wait()

      if shutdown_event.is_set():
          monitor_task.cancel()
          executor_task.cancel()
          await asyncio.gather(monitor_task, executor_task, return_exceptions=True)
  ```

#### Logging
- Use the `src/utils/logger.py` module for all output
- Use colored output: `info()`, `success()`, `warning()`, `error()`
- Log to file automatically via `write_to_file()`
- Use `waiting()` for ongoing operations with end='\r'
- Use `clear_line()` to update waiting messages

#### Docstrings
- Use triple double quotes for module/docstring
- Include description, args, and return type
- Example:
  ```python
  """
  Trade monitor service - monitors trader activity via WebSocket
  """
  ```

### Rust Guidelines

#### Imports
- Group by crate: std, external crates, local modules
- Use `use` statements for frequently used items
- Example:
  ```rust
  use anyhow::{Result, anyhow};
  use chrono::{DateTime, Utc};
  use tokio::{sync::{mpsc, oneshot}, time};

  use pm_whale_follower::{ApiCreds, OrderArgs, RustClobClient};
  use pm_whale_follower::settings::Config;
  use pm_whale_follower::risk_guard::{RiskGuard, RiskGuardConfig};
  ```

#### Error Handling
- Use `anyhow::Result<T>` for fallible operations
- Use `?` operator for early returns
- Use `anyhow!("message")` for error construction
- Example:
  ```rust
  fn main() -> Result<()> {
      let cfg = Config::from_env().await?;
      let (client, creds) = build_worker_state(...).await?;
      Ok(())
  }
  ```

#### Performance Patterns
- Use `thread_local!` buffers for hot paths to avoid allocations
- Use `#[inline]` on small frequently-called functions
- Pre-allocate `String` with capacity when size is known
- Use `RefCell` for thread-local mutable buffers
- Use `Arc<str>` for shared string constants

#### Naming Conventions
- **Structs/Enums**: `PascalCase`
- **Functions/variables**: `snake_case`
- **Constants**: `UPPER_SNAKE_CASE`
- **Type aliases**: `CamelCase` (e.g., `type HmacSha256 = Hmac<Sha256>;`)
- **Binaries**: descriptive names in `Cargo.toml` `[[bin]]` sections

#### Async Patterns
- Use `tokio` runtime with `#[tokio::main]`
- Use `mpsc` channels for work distribution
- Use `oneshot` channels for response handling
- Implement timeouts with `tokio::time::timeout`
- Example:
  ```rust
  let (resp_tx, resp_rx) = oneshot::channel();
  if let Err(e) = self.tx.try_send(WorkItem { event: evt, respond_to: resp_tx }) {
      return format!("QUEUE_ERR: {e}");
  }

  match tokio::time::timeout(ORDER_REPLY_TIMEOUT, resp_rx).await {
      Ok(Ok(msg)) => msg,
      Ok(Err(_)) => "WORKER_DROPPED".into(),
      Err(_) => "WORKER_TIMEOUT".into(),
  }
  ```

#### Performance Optimization
- Avoid allocations in hot paths
- Use `itoa` crate for integer-to-string conversion
- Use `ryu` for float-to-string conversion
- Use `memchr` for byte searching
- Use `rustc-hash` for faster hashing

#### Documentation Comments
- Use `///` for documentation above items
- Include example usage where helpful
- Document panic conditions and error returns
- Example:
  ```rust
  /// Pre-allocated common strings to avoid allocations
  const ZERO_STR: &str = "0";

  /// Build URL with query parameter
  ///
  /// # Arguments
  /// * `base` - Base URL
  /// * `path` - URL path
  /// * `param` - Query parameter name
  /// * `value` - Query parameter value
  #[inline]
  fn build_url_query_1(base: &str, path: &str, param: &str, value: &str) -> String {
  ```

## Configuration Files

### Python
- `pyproject.toml`: Project metadata and dependencies
- `requirements.txt`: Pin exact versions for reproducibility
- `.env.example`: Template for environment variables

### Rust
- `Cargo.toml`: Package and dependency configuration
- `[profile.dev]`: Optimized for fast compilation
- `[profile.release]`: Uses thin LTO for optimization
- `[[bin]]`: Binary entry points for different utilities

## Key Files to Understand

### Python
- `src/main.py`: Entry point with graceful shutdown handling
- `src/config/env.py`: Environment validation and configuration
- `src/services/trade_monitor.py`: WebSocket monitoring service
- `src/services/trade_executor.py`: Order execution service
- `src/utils/logger.py`: Colored logging utility

### Rust
- `src/main.rs`: Main entry point (~1000+ lines)
- `src/lib.rs`: Core library with CLOB client, order signing
- `src/settings.rs`: Configuration loading
- `src/risk_guard.rs`: Risk management logic
- `src/market_cache.rs`: Market data caching

## Important Notes

1. **Secrets**: Never commit `.env` files. They are gitignored.
2. **Wallet Security**: PRIVATE_KEY is sensitive. Use mock trading first.
3. **Python**: Requires MongoDB connection via `MONGO_URI`
4. **Rust**: Requires `.clob_creds.json` and `.clob_market_cache.json`
5. **Web3**: Both implementations interact with Polymarket CLOB API
6. **No Formal Tests**: Test binaries exist for Rust but no pytest/unittest suite
7. **Both Implementations Are Active**: Don't assume one is deprecated

## Common Development Tasks

### Adding a New Python Script
1. Create file in appropriate `src/scripts/` subdirectory
2. Add `if __name__ == '__main__':` block
3. Use `argparse` for CLI arguments if needed
4. Follow naming: `src/scripts/category/script_name.py`

### Adding a New Rust Binary
1. Create file in `src/bin/script_name.rs`
2. Add to `[[bin]]` section in `Cargo.toml`
3. Implement `fn main() -> Result<()>`
4. Add to `mod bin;` in `src/main.rs` if needed

### Modifying Configuration
- Python: Update `src/config/env.py` and add env var to `.env.example`
- Rust: Update `src/settings.rs` and `.env.example` in rust directory
