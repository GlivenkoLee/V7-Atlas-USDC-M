# INTERFACES_CONTRACTS.md
# V7-Atlas USDC-M｜接口签名 + 状态机 + 执行顺序契约（Codex 必须严格遵守）

## 0) P0 红线（任何实现不得违反）
- 条件触发保护单（尤其止损）必须走 Algo Conditional。
- 保护性止损下达后 1 秒内必须在 open algo orders 可见；否则 emergency_flatten + 30min 禁新开。
- 幂等：clientOrderId / clientAlgoId 必须存在且统一格式；重复发送不得产生重复订单；cancel 不存在视为成功。
- LIVE 决策路径禁止 mock/假数据（mock 仅限 tests/）。

---

## 1) ExchangeAdapter（交易所适配层）｜方法签名必须一致
class ExchangeAdapter:
  # --- market data ---
  def ws_start(self) -> None: ...
  def ws_stop(self) -> None: ...
  def get_kline(self, symbol: str, interval: str, limit: int) -> list[dict]: ...
  def get_mark_price(self, symbol: str) -> float: ...
  def get_last_price(self, symbol: str) -> float: ...
  def get_funding_rate(self, symbol: str) -> float: ...
  def get_open_interest(self, symbol: str) -> float: ...

  # --- account / position ---
  def get_balances(self) -> dict: ...
  def get_positions(self) -> dict[str, dict]: ...
  def set_leverage(self, symbol: str, leverage: int) -> None: ...
  def set_position_mode_one_way(self) -> None: ...

  # --- normal orders (entry/exit) ---
  def place_limit_postonly(
      self, *, symbol: str, side: str, qty: float, price: float,
      tif: str, reduce_only: bool, client_order_id: str
  ) -> dict: ...

  def place_limit_ioc(
      self, *, symbol: str, side: str, qty: float, price: float,
      reduce_only: bool, client_order_id: str
  ) -> dict: ...

  def place_market(
      self, *, symbol: str, side: str, qty: float,
      reduce_only: bool, client_order_id: str
  ) -> dict: ...

  def cancel_order(
      self, *, symbol: str, order_id: str | None, client_order_id: str | None
  ) -> dict: ...

  def get_open_orders(self, symbol: str) -> list[dict]: ...

  # --- Algo Conditional (MUST for protection) ---
  def place_algo_stop(
      self, *, symbol: str, side: str, qty: float, stop_price: float,
      reduce_only: bool, client_algo_id: str,
      working_type: str, close_position: bool = False
  ) -> dict: ...

  def cancel_algo(
      self, *, symbol: str, algo_id: str | None, client_algo_id: str | None
  ) -> dict: ...

  def get_open_algo_orders(self, symbol: str) -> list[dict]: ...

  # --- Recovery / selfcheck helpers ---
  def recover_protection(self, symbol: str, desired_sl: dict) -> dict: ...
  def exchange_selfcheck(self) -> dict: ...

---

## 2) StateStore（持久化）｜必须
class StateStore:
  def load(self) -> None: ...
  def flush(self) -> None: ...
  def get_symbol_state(self, symbol: str) -> dict: ...
  def set_symbol_state(self, symbol: str, state: dict) -> None: ...
  def append_event(self, event: dict) -> None: ...

---

## 3) SymbolState（每 symbol 必须字段齐全）
SymbolState 至少包含：

### 3.1 互锁与风控状态
- last_regime: str
- last_template: str
- cooldown_until: dict{
    loss: int,
    template: int,
    freeze: int,
    shock: int
  }
- b_blacklist_until: int

### 3.2 ShockGuard
- shockguard: dict{
    q95_1m: float | None,
    q95_3m: float | None,
    recalib_ts: int,
    trigger_count_60m: int
  }

### 3.3 执行质量滚动（用于 B_BLACKLIST）
- exec_quality_roll: dict{
    fill_rate_30: float,
    slippage_cost30_r: float,
    cancel_rate_30: float,
    spread_p95: float,
    sample_n: int,
    last_update_ts: int,
    pass_windows_streak: int
  }

### 3.4 Health 滚动
- health_roll: dict{
    win_rate: float,
    avg_r: float,
    max_dd: float,
    profit_factor: float,
    fallback_quality: float,
    trade_n: int,
    last_calib_ts: int,
    weights_version: str
  }

### 3.5 Entry 执行纪律状态（新增，MUST）
- entry_state: dict{
    working: bool,
    order_id: str | None,
    client_order_id: str | None,
    placed_ts: int,
    ttl_sec: int,              # default 45
    reprice_count: int,        # max 2
    last_reprice_ts: int,
    cr_count_10m: int,         # cancel/replace count in rolling window
    cr_window_start_ts: int
  }

### 3.6 PositionState（见第 4 节）
- position_state: dict

---

## 4) Position State Machine（仓位状态机）｜必须
position_stage 枚举：
- PRE_TP1
- BETWEEN_TP1_TP2
- RUNNER_ONLY

PositionState 至少包含：
- side: "LONG" | "SHORT" | "NONE"
- entry_price: float
- qty_total: float
- qty_runner: float
- sl_price: float
- sl_algo: dict{ algo_id, client_algo_id, working_type }
- tp1_done: bool
- tp2_done: bool
- peak_pnl_r: float
- bars_after_tp2: int
- last_orders: dict{
    entry: {order_id, client_order_id},
    tp: list[{order_id, client_order_id}],
    sl: {algo_id, client_algo_id}
  }
- init_margin_est: float   # 用于 dust 判断（可由交易所返回或估算）

硬规则：
- new_regime ∈ {SHOCK, CHOP} 且存在 runner => 立即 reduceOnly 平 runner
- 任意持仓必须具备保护性 SL（Algo）；SL 不可见 => P0 emergency_flatten

---

## 5) 互锁闸门契约（MUST：max(...)）
allow_open(symbol) = now_ts >= max(
  cooldown_until.loss,
  cooldown_until.template,
  cooldown_until.freeze,
  cooldown_until.shock,
  b_blacklist_until
)
拒单必须输出 reject_reason（字符串）。

---

## 6) 执行顺序契约（RegimeShift / Runner / 风控维护）
### 6.1 RegimeShift
def on_regime_shift(symbol: str, old_regime: str, new_regime: str, st: dict, ctx: dict) -> dict

顺序必须固定：
1) ShockGuard / DeRisk
2) Runner
3) Stops/锁利/减仓/追踪参数更新
4) 更新互锁与写 RegimeShiftLog

### 6.2 Runner
def manage_runner(symbol: str, pos_state: dict, now_bar_index: int, ctx: dict) -> dict

必须实现：
- 时间止盈：bars_after_tp2>=MAX_BARS -> 平 runner（IOC限价优先，2秒未成交市价兜底）
- 回吐止盈：peak达标且回吐到阈值 -> 平 runner
- 环境禁入：SHOCK/CHOP -> 立即平 runner

### 6.3 Entry 执行纪律（新增，MUST）
def manage_entry(symbol: str, sig: dict, sym_state: dict, ctx: dict) -> dict

必须实现：
- postOnly entry；TTL=45s；reprice<=2；reprice 需满足偏离阈值（默认 0.18*ATR_1m）且仍在边界带附近
- TTL 到期未成交：cancel entry（计入 C/R 次数）
- cancel/replace 限频：每 symbol 10min ≤10 次；超限则冻结 entry 到窗口结束（只允许 reduceOnly）

---

## 7) 1秒可见性哨兵（MUST）
def ensure_algo_visible(symbol: str, client_algo_id: str, deadline_ms: int = 1000) -> bool

规则：
- place_algo_stop 后最多 1 秒内轮询 get_open_algo_orders
- 不可见：触发 emergency_flatten(symbol) + 设置 30min 禁新开 + 写事件

---

## 8) Dust Position 清理（新增，MUST）
def dust_autoclose_if_needed(symbol: str, pos_state: dict, ctx: dict) -> bool

规则：
- init_margin_est < 0.1 USDC => reduceOnly 市价平仓，写事件 DUST_AUTOCLOSE
- 默认不触发 loss cooldown（除非你显式配置）

---

## 9) 日志输出契约（用于 UI 与验收）
每次候选评估必须输出：
- symbol, regime, template, final_score, action(OPEN/REJECT), reject_reason

关键事件必须输出：
- LOSS_COOLDOWN / REGIME_SHIFT / RUNNER_EXIT / SL_RECOVER / EMERGENCY_FLAT / BLACKLIST_UPDATE / DUST_AUTOCLOSE / ENTRY_EXPIRE / ENTRY_REPRICE_LIMIT

所有事件必须携带幂等键与订单回执字段（order_id/algo_id/client_id）。

---

## 10) SelfCheck（上线前自检，必须提供）
- REST/WS 连通与权限
- Algo Conditional 下单与 open algo orders 查询能力
- position mode 单向/双向是否符合要求
- 精度/最小下单量校验
- 参数加载成功（A/B池、杠杆、TTL/限频、Runner、ShockGuard、黑名单阈值）
