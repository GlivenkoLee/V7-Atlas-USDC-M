# INTERFACES_CONTRACTS.md
# V7-Atlas USDC-M｜接口签名 + 状态机 + 执行顺序“硬契约”
# 目的：把 Codex 的手绑住，避免实现偏航、字段乱、恢复断链、下单端点错用。

## 0) P0 红线（任何实现都不得违反）
0.1 保护性“条件触发”订单（尤其止损）必须走 Algo Conditional（交易所 Algo 端点/通道），不得用普通 stop_market 端点代替。
0.2 保护性止损下单后 1 秒内必须在 open algo orders 可见；否则立刻执行 reduceOnly 市价平仓，并对该 symbol 30min 禁新开。
0.3 幂等：
- 普通单：clientOrderId；条件单：clientAlgoId
- 同 id 重发不得重复下单
- cancel 不存在视为成功（幂等成功）
0.4 LIVE 决策不得使用 mock/假数据；mock 仅限 tests/。

---

## 1) ExchangeAdapter（交易所适配层）｜必须实现这些方法（签名保持一致）
> 说明：不要求你用某个 SDK，但要求这些能力与返回字段能满足上层状态机与风控闭环。

### 1.1 连接与行情
class ExchangeAdapter:
  def ws_start(self) -> None:
      """启动 WS，订阅 tick/kline/mark price 等；断线重连策略由实现保证。"""

  def ws_stop(self) -> None:
      """停止 WS。"""

  def get_kline(self, symbol: str, interval: str, limit: int) -> list[dict]:
      """返回 K 线数组，每个元素至少包含: open_time, open, high, low, close, volume, close_time"""

  def get_mark_price(self, symbol: str) -> float:
      """返回标记价格（MARK），用于 working_type=MARK_PRICE 等。"""

  def get_last_price(self, symbol: str) -> float:
      """返回最新成交价/指数价（实现自行选择，但必须清晰记录）。"""

  def get_funding_rate(self, symbol: str) -> float:
      """返回当前资金费率（若策略需要）。"""

  def get_open_interest(self, symbol: str) -> float:
      """返回当前 OI（若策略需要）。"""

### 1.2 账户与仓位
  def get_balances(self) -> dict:
      """返回余额/可用保证金等，至少包含: total_equity, available_balance, margin_used"""

  def get_positions(self) -> dict[str, dict]:
      """
      返回所有 symbol 的持仓字典。
      每个 symbol 至少包含: side(LONG/SHORT/NONE), qty, entry_price, unrealized_pnl, leverage
      """

  def set_leverage(self, symbol: str, leverage: int) -> None:
      """设置杠杆（A/B 档不同杠杆可在上层调用）。"""

  def set_position_mode_one_way(self) -> None:
      """可选：将账户切成单向模式（如需求要求单向）。若不支持必须在 selfcheck 里报错。"""

### 1.3 普通订单（入场/减仓/平仓）
  def place_limit_postonly(
      self, *, symbol: str, side: str, qty: float, price: float,
      tif: str, reduce_only: bool, client_order_id: str
  ) -> dict:
      """下限价单（尽量 post-only）；返回必须包含 order_id / client_order_id / status"""

  def place_market(
      self, *, symbol: str, side: str, qty: float,
      reduce_only: bool, client_order_id: str
  ) -> dict:
      """下市价单；用于 P0 兜底强平/紧急 reduceOnly"""

  def cancel_order(
      self, *, symbol: str, order_id: str | None, client_order_id: str | None
  ) -> dict:
      """
      撤普通单。
      约束：订单不存在/已成交/已撤 => 视为成功（幂等成功），不得抛出致命异常。
      """

  def get_open_orders(self, symbol: str) -> list[dict]:
      """查询未成交普通单。"""

### 1.4 Algo Conditional（保护单必须走这里）
  def place_algo_stop(
      self, *, symbol: str, side: str, qty: float, stop_price: float,
      reduce_only: bool, client_algo_id: str,
      working_type: str, close_position: bool = False
  ) -> dict:
      """
      下 Algo 条件单（止损/触发类）。
      返回必须包含 algo_id / client_algo_id / status。
      working_type 必须支持 MARK_PRICE（默认推荐）。
      """

  def cancel_algo(
      self, *, symbol: str, algo_id: str | None, client_algo_id: str | None
  ) -> dict:
      """
      撤 Algo 单。
      约束：不存在/已触发/已撤 => 视为成功（幂等成功）。
      """

  def get_open_algo_orders(self, symbol: str) -> list[dict]:
      """查询 open algo orders（用于 1 秒可见性与重启恢复）。"""

### 1.5 幂等恢复（启动/断线后必须执行）
  def recover_protection(self, symbol: str, desired_sl: dict) -> dict:
      """
      目标：确保 desired_sl 描述的保护单真实存在且参数正确。
      - 若不存在：创建
      - 若存在但偏离过大：撤并重建
      - 若存在且可用：返回确认信息
      """

---

## 2) StateStore（持久化）｜必须（重启恢复的根）
class StateStore:
  def load(self) -> None:
      """启动读取本地状态（json/sqlite 均可）。"""

  def flush(self) -> None:
      """落盘（建议写入原子文件/事务）。"""

  def get_symbol_state(self, symbol: str) -> dict:
      """读取 symbol 状态；若无返回默认结构。"""

  def set_symbol_state(self, symbol: str, state: dict) -> None:
      """写入 symbol 状态。"""

  def append_event(self, event: dict) -> None:
      """追加事件日志（用于复盘/验收/追责）。"""

### 2.1 SymbolState 结构（字段必须存在）
SymbolState（每 symbol）至少包含：
- last_regime: str
- last_template: str
- cooldown_until: dict{
    loss: int(epoch seconds),
    template: int,
    freeze: int,
    shock: int
  }
- b_blacklist_until: int
- shockguard: dict{
    q95_1m: float | None,
    q95_3m: float | None,
    recalib_ts: int,
    trigger_count_60m: int
  }
- exec_quality_roll: dict{
    fill_rate_30: float,
    slippage_cost30_r: float,
    cancel_rate_30: float,
    spread_p95: float,
    sample_n: int,
    last_update_ts: int
  }
- health_roll: dict{
    win_rate: float,
    avg_r: float,
    max_dd: float,
    profit_factor: float,
    fallback_quality: float,
    trade_n: int,
    last_calib_ts: int
  }
- position_state: dict（见第 3 节）

---

## 3) Position State Machine（仓位状态机）｜必须统一字段（否则恢复必断）
position_stage 枚举：
- PRE_TP1
- BETWEEN_TP1_TP2
- RUNNER_ONLY

PositionState（每 symbol）至少包含：
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

硬规则：
- new_regime ∈ {SHOCK, CHOP} 且存在 runner => 立即 reduceOnly 平 runner（不得等待）。
- 任何持仓都必须有保护性 SL（Algo）；如果 SL 不可见（open algo orders 找不到）=> P0 紧急 reduceOnly 平仓。

---

## 4) 互锁开仓闸门（cooldown/freeze）｜必须是 max(...) 规则
允许新开 = (now_ts >= max(
  cooldown_until.loss,
  cooldown_until.template,
  cooldown_until.freeze,
  cooldown_until.shock,
  b_blacklist_until
))

拒单必须输出 reject_reason（字符串，写清是哪一项锁住）。

---

## 5) Runner 管理契约｜必须
函数签名建议：
def manage_runner(symbol: str, pos_state: dict, now_bar_index: int, ctx: dict) -> dict

必须实现：
- 时间止盈：bars_after_tp2 >= RUNNER_MAX_BARS_AFTER_TP2 => 平 runner（reduceOnly，限价优先，失败市价兜底）
- 回吐止盈：peak_pnl_r >= RUNNER_MIN_PEAK_R 且 当前浮盈 <= peak_pnl_r * (1 - RUNNER_GIVEBACK) => 平 runner
- 环境禁入：进入 SHOCK/CHOP => 立即平 runner

---

## 6) RegimeShift Playbook（环境突变处置）｜必须按固定顺序
函数签名建议：
def on_regime_shift(symbol: str, old_regime: str, new_regime: str, st: dict, ctx: dict) -> dict

处理顺序必须固定：
1) ShockGuard / 保证金 DeRisk
2) Runner（必要时立即平）
3) 剩余仓位止损/锁利/减仓参数更新
4) 更新 cooldown/freeze 并写 regime_shift_log 事件

---

## 7) 1 秒可见性哨兵（Visibility Sentinel）｜必须
函数签名建议：
def ensure_algo_visible(symbol: str, client_algo_id: str, deadline_ms: int = 1000) -> bool

规则：
- place_algo_stop 后，最多 1 秒内轮询 get_open_algo_orders
- 若不可见：触发 emergency_flatten(symbol)（reduceOnly 市价平仓）+ set loss/freeze 30min（按 PRD/验收）

---

## 8) 日志输出契约（用于 UI 与验收）
每次候选评估必须输出：
- symbol, regime, template, gates(PASS/FAIL+原因), base_score, penalties, final_score, action(OPEN/REJECT), reject_reason

每次关键风控动作必须输出：
- event_type（LOSS_COOLDOWN / REGIME_SHIFT / RUNNER_EXIT / SL_RECOVER / EMERGENCY_FLAT）
- symbol, side, qty, price, order_ids, client_ids, reason

---

## 9) SelfCheck（上线前自检）｜必须提供
- API 权限/连通性（REST/WS）
- 是否支持 Algo Conditional 下单与查询 open algo orders
- position mode 是否符合要求（单向/双向）
- 最小下单量/精度检查（避免实盘报错）
- 关键参数是否加载成功（A/B 档、冷却、Runner 参数、ShockGuard 参数）
