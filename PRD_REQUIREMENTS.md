# PRD_REQUIREMENTS.md
# V7-Atlas（USDC-M）合约量化实盘系统｜需求规格（给 Codex 的“硬口径”）

## 0. 文档目的
把策略说明书转成可实现、可验收、可测试的工程需求；减少 Codex 自行脑补。

## 1. 范围与非目标
### 1.1 范围（必须实现）
- 交易所：Binance USDC-M 永续合约；结算资产 USDC（USDC-M）。（说明书抬头明确 USDC-M 与更新说明）:contentReference[oaicite:4]{index=4}
- 主周期：30m；辅助周期：2h（用于趋势/结构辅助判断）。
- 核心流程：Regime 识别 → 模板互斥路由 → 评分/闸门 → 执行 → 风控维护 → 冷却/冻结/复盘日志。
- 四模板互斥：Pullback / RangeEdge / 2B / Breakout / None。（模板选择是分类，不打分）:contentReference[oaicite:5]{index=5}
- 必须补齐：Runner、环境突变处置（Regime Shift Playbook）、B档动态筛选、Heal:contentReference[oaicite:6]{index=6}校准周期、Loss Cooldown。（升级点列表）:contentReference[oaicite:7]{index=7}

### 1.2 非目标（可以不做）
- 不做跨交易所；不做期权/现货；不做网格/做市。
- 不追求回测系统（只需提供最小回放/仿真接口用于测试）。

## 2. 需求优先级（用于消解冲突）
- P0（红:contentReference[oaicite:8]{index=8}幂等恢复、止损兜底、禁新开）优先级最高。:contentReference[oaicite:9]{index=9}
- P1：策略行为（模板互斥、冷却、Runner、RegimeShift 动作矩阵、Loss Cooldown 等）。:contentReference[oaicite:10]{index=10}P1/P2 与 P0 冲突时，以 P0 为准；当 P1 内部冲突时，以“RegimeShift 处理顺序必须固定”与“互锁规则 max(...)”为准。:contentReference[oaicite:11]{index=11} :contentReference[oaicite:12]{index=12}

## 3. 关键实体与术语:contentReference[oaicite:13]{index=13}P / SHOCK / None。
- Template：Pullback / RangeEdge / 2B / Breakout / None。
- Zone：结构区（支撑/阻力/回踩区），用于 Pullback/RangeEdge 的结构约束。
- R：风险单位（以 SL_DIST 为基准换算），用于 TP/Runner/S:contentReference[oaicite:14]{index=14}模:contentReference[oaicite:15]{index=15}tests/）：
1) exchange_adapter：交易所适配（REST/WS、下单、撤单、查询、Algo Conditional）
2) engine_runner：主循环与调度（30m/2h bar、信号计算、下单、风控维护）
3) strategy_core：Regime → Template → Score → 决策
4) risk_manager：仓位、止损止盈、Runner、RegimeShift、冷却/冻结
5) state_store：持久化（幂等键、仓位状态机、日志、指标滚动窗口）

## 5. 数据与新鲜度（必须）
- 行情与指标必须来自真实交易所数据（WS 为主，REST 兜底）。
- 必须实现数据新鲜度闸门：30m/2h 的 K 线若超时则禁止新开，只允许风险维护（reduceOnly）。  
（具体阈值允许配置；默认遵循工程常识：主周期更严格、辅助周期略松。）

## 6. Step2 模板互斥路由（必须严格按表）
- 模板选择是“分类”，输出只能：Pullback / RangeEdge / 2B / Breakout / None。:contentReference[oaicite:16]{index=16}
- 路由规则（硬约束）：:contentReference[oaicite:17]{index=17}
  - TREND：默认 Pullback；只有满足“突破硬条件”才允许 Breakout 替代；禁止 RangeEdge/2B。
  - RANGE：默认 RangeEdge；若出现完整 2B 形态可允许 2B；禁止 Breakout。
  - CHOP：只允许 2B；其他全禁。
  - SHOCK：None；禁止新开仓。
- 抖动抑制（必须）：REGIME_MIN_HOLD（连续 ≥2 次:contentReference[oaicite:18]{index=18}切换后 6min 内不允许:contentReference[oaicite:19]{index=19}:contentReference[oaicite:20]{index=20}

## 7. 冷却/冻结互锁（必须）
- Loss Cooldown：任意“亏损平仓”（Realized PnL < 0）触发后，该币种 30 分钟内禁止新开（含同向/反向）。:contentReference[oaicite:21]{index=21}
- 冷却期间允许：reduceOnly/closePosition 风险动作；允许补挂止盈止损保护单（不允许加仓）。:contentReference[oaicite:22]{index=22}
- 实现口径：symbol 维度维护 loss_cooldown:contentReference[oaicite:23]{index=23}l<0 则 now+1800s。:contentReference[oaicite:24]{index=24}
- 互锁规则：是否允许新开 = max(template_cooldown_until:contentReference[oaicite:25]{index=25}eze_until_ts, shock_cooldown_until)；任何未到期都拒单并写原因。:contentReference[oaicite:26]{index=26}:contentReference[oaicite:27]{index=27}:contentReference[oaicite:28]{index=28}

## 8. 执行层 P0 红线（必须）
### 8.1 条件单必须走 Algo Conditional（生存线）
-:contentReference[oaicite:29]{index=29}al：REST POST /fapi/v1/algoOrder 或 WS algoOrder.place。:contentReference[oaicite:30]{index=30}
- 验收红线：无论挂单成交/市价补充/部分成交，保护性止损必须 1 秒内在 “:contentReference[oaicite:31]{index=31}否:contentReference[oaicite:32]{index=32}ymbol 30 分钟禁开。:contentReference[oaicite:33]{index=33}

### 8.2 幂等与重启恢复（必须）
- 普通单用 clientOrderId；条件单用 clientAlgoId；统一格式，便于回放/恢复。:contentReference[oaicite:34]{index=34}:contentReference[oaicite:35]{index=35}pen Orders：止损不存在或偏离过大就撤销重建；撤单不存在视为成功（幂等）。:contentReference[oaicite:36]{index=36}
- 所有下单必须带 clientOrderId；同 id 重发不得产生重复订单；:contentReference[oaicite:37]{index=37}:contentReference[oaicite:38]{index=38}

## 9. TP / Runner（必须）
- Runner 的目标：只放大利润，不允许“盈利回吐变亏损尾巴”。必须具备：时:contentReference[oaicite:39]{index=39}:contentReference[oaicite:40]{index=40}
- 参数（默认，30m）：RUNNER_MAX_BARS_AFTER_TP2=6；RUNNER:contentReference[oaicite:41]{index=41}K_R=0.80；RUNNER_TRAIL_ATR=0.90×ATR_30m；RUNNER_HARD_:contentReference[oaicite:42]{index=42}:contentReference[oaicite:43]{index=43}
- 时间止盈（必须）：TP2 后计时，bars>=RUNNER_MAX_BARS_AFTER_TP2 → 无论盈亏全部平 Runner（redu:contentReference[oaicite:44]{index=44}:contentReference[oaicite:45]{index=45}
- 回吐止盈（必须）：peak_pnl≥RUNNER_MIN_PEAK_R 且 当前浮盈 ≤ peak_pnl×(1-RUNNER_GIVEBACK) → 立即平 Runner。:contentReference[oaicite:46]{index=46}:contentReference[oaicite:47]{index=47}CK 或 CHOP 时 Runner 立即平；剩余仓位按 RegimeShift 规则收紧。:contentReference[oaicite:48]{index=48}

## 10. 环境突变处置（Regi:contentReference[oaicite:49]{index=49}）
- 触发：任意 Regime 从 TREND/RANGE/CHOP/None 切换为另一个状态（含进入 SHOCK）即触发一次持仓再定价。:contentReference[oaicite:50]{index=50}:contentReference[oaicite:51]{index=51}处理 ShockGuard/保证金 DeRisk；②再处理 Runner；③再处理剩余仓位止损/止盈参数；④最后更新冷却/冻结与日志。:contentReference[oaicite:52]{index=52}:contentReference[oaicite:53]{index=53}作定义：收紧止损=SL_DIST×0.80（或移到更近结构位）；锁利=止损抬到 BE+0.10R；减仓=reduceOnly 快速减掉 X%。:contentReference[oaicite:54]{index=54}
- 动作矩阵（默认示例，必须可配置且可:contentReference[oaicite:55]{index=55}+ 其余减仓/锁利 + 止损收紧等。:contentReference[oaicite:56]{index=56}

## 11. ShockGuard 分位数校准周期（必须）
- 默认：每 24:contentReference[oaicite:57]{index=57}calib_ts。:contentReference[oaicite:58]{index=58}
- 提前：连续触发 ShockGuard ≥3 次且间隔<60min，或:contentReference[oaicite:59]{index=59}:contentReference[oaicite:60]{index=60}
- 回退：分位数不可算则沿用最近一次有效 Q95 并进入保守模式（缩仓 + 抬 Mi:contentReference[oaicite:61]{index=61}:contentReference[oaicite:62]{index=62}

## 12. B 档标的动态筛选（必须）
- 在线指标（每币种滚动 30 次执行或 24h）：FillRate3:contentReference[oaicite:63]{index=63}30、SpreadP95。:contentReference[oaicite:64]{index=64}
- 淘汰规则：FillRate30<:contentReference[oaicite:65]{index=65} 或 CancelRate30>0.45 → 进入 B_BLACKLIST 7 天；每日复评一次。:contentReference[oaicite:66]{index=66}:contentReference[oaicite:67]{index=67}thScore 权重校准（必须）
- 建议权重（默认）：WinRate=25%，AvgR=30%，MaxDD=20%，ProfitFactor=15%，Fallback/FillQuality=10%:contentReference[oaicite:68]{index=68}:contentReference[oaicite:69]{index=69}
- 校准周期：每满 200 笔交易或每月（取更早）校准一次；目标优先“回撤最小化+收益稳定性”。:contentReference[oaicite:70]{index=70}

## 14. 日志与复盘（必须）:contentReference[oaicite:71]{index=71}l、realized_pnl、cooldown_until、触发来源、当时 Regime/Template、最近一次入场总分与子分、拒单原因。:contentReference[oaicite:72]{index=72}
- RegimeShift:contentReference[oaicite:73]{index=73}作明细、参数变化、订单回执。:contentReference[oaicite:74]{index=74}
- 关键流转日:contentReference[oaicite:75]{index=75}:contentReference[oaicite:76]{index=76}

## 15. 终端 UI（必须）
- 必须提供 curses 终端：上半滚动事件流、下半表格（持仓/候选/风控状态）；减少刷屏与重复信息；能跑在延迟高环境。

## 16. 交付物
- 可运行：p:contentReference[oaicite:77]{index=77}er
- 自检：python main.py --selfcheck（检查接口、权限、可下 Algo Condit:contentReference[oaicite:78]{index=78}收红线场景（见 ACCEPTANCE_TESTPLAN:contentReference[oaicite:79]{index=79}”来自说明书这些关键段落**：  
- 模板互斥与冷却：:contentReference[oaicite:80]{index=80}  
- Loss Cooldown 与互锁：:contentReference[oaicite:81]{index=81}  
- Runner 与参数：:contentReference[oaicite:82]{index=82}  
- RegimeShift 流程/顺序/矩阵：:contentReference[oaicite:83]{index=83} :contentReference[oaicite:84]{index=84}  
- Algo Conditional 生存线 + 1秒可见性：:contentReference[oaicite:85]{index=85}:contentReference[oaicite:86]{index=86}重、ShockGuard 校准：:contentReference[oaicite:87]{index=87}:contentReference[oaicite:88]{index=88}

# 三件套文件 2：INTE:contentReference[oaicite:89]{index=89}
```text
# INTERFACES_CONT:contentReference[oaicite:90]{index=90}签:contentReference[oaicite:91]{index=91} 交易所适配层（ExchangeAdapter）｜必须实现
### :contentReference[oaicite:92]{index=92}onal（/fapi/v1/algoOrder 或 WS algoOrder.pl:contentReference[oaicite:93]{index=93}tOrderId / clientAlgoId；撤单不存在视为成功；同 id 重发不得重复下单。:contentReference[oaicite:94]{index=94}
- “1秒可见性”哨兵：挂保护 SL 后 1 秒内必须在 algo open orders 可查到，否则立刻 reduceOnly 市价平仓 + 30min 禁开。:contentReference[oaicite:95]{index=95}

### 1.2 Python 接口（签名必须一致）
class ExchangeAdapter:
  # --- Market data ---
  def ws_start(self) -> None: ...
  def get_kline(se:contentReference[oaicite:96]{index=96} limit:int) -> list[dict]: ...
  def get_mark_price(self, symbol:str) -> float: ...:contentReference[oaicite:97]{index=97} symbol:str) -> float: ...
  def get_open_interest(self, symbol:str) -> float: ...

  # --- Account/position ---
  def get_balances(self) -> dict: ...
  def get_positions(self) -> dict[str, dict]: ...
  def set_leverage(self, symbol:str, leverage:int) -> None: ...

  # --- Orders (normal) ---
  def place_limit_postonly(self, *, symbol:str, side:str, qty:float, price:float,
                           tif:str, reduce_only:bool, client_order_id:str) -> dict: ...
  def place_market(self, *, symbol:str, side:str, qty:float,
                   reduce_only:bool, client_order_id:str) -> dict: ...
  def cancel_order(self, *, symbol:str, order_id:str|None, client_order_id:str|None) -> dict: ...
  def get_open_orders(self, symbol:str) -> list[dict]: ...

  # --- Algo Conditional (MUST) ---
  def place_algo_stop(self, *, symbol:str, side:str, qty:float, stop_price:float,
                      reduce_only:bool, client_algo_id:str,
                      working_type:str, close_position:bool=False) -> dict: ...
  def cancel_algo(self, *, symbol:str, algo_id:str|None, client_algo_id:str|None) -> dict: ...
  def get_open_algo_orders(self, symbol:str) -> list[dict]: ...

  # --- Idempotent recovery ---
  def recover_protection(self, symbol:str, desired_sl:dict) -> dict: ...

## 2) 状态存储（StateStore）｜必须
- 目标：重启恢复、loss_cooldown_until_ts、template_cooldown_until、freeze_until_ts、shockguard recalib_ts、rolling 指标等。

class StateStore:
  def load(self) -> None: ...
  def flush(self) -> None: ...
  def get_symbol_state(self, symbol:str) -> dict: ...
  def set_symbol_state(self, symbol:str, state:dict) -> None: ...
  def append_event(self, event:dict) -> None: ...

## 3) 仓位状态机（Position State Machine）｜必须
- position_stage：PRE_TP1 / BETWEEN_TP1_TP2 / RUNNER_ONLY （说明书明确）:contentReference[oaicite:98]{index=98}
- runner 存在且 new_regime in {SHOCK, CHOP}：立即平 runner（reduceOnly）。:contentReference[oaicite:99]{index=99}

字段（每 symbol）：
- side, entry_price, qty_total, qty_runner, sl_price, tp1_done, tp2_done
- peak_pnl_r, bars_after_tp2
- last_template, last_regime
- :contentReference[oaicite:100]{index=100}o, tp_orders...}
- cooldown_until: {loss, template, freeze, shock:contentReference[oaicite:101]{index=101}须按顺序）
函数签名：
def on_regime_shift(symbol:str, old_regime:str, new_regime:str, pos_state:dict) -> dict

顺序（必须固定）：ShockGuard/保证金 DeRisk → Runner → 剩余仓位止损/止盈参数 → 更新互锁/日志。:contentReference[oaicite:102]{index=102}

## 5) 评分输出格式（用于日志与 UI）
- 每次候选必须输出：
  symbol, regime, template, gates(PASS/FAIL+原因), base_score, pen_common, pen_template, final_score, action(OPEN/REJECT), reject_reason
（日志模板示例存在）:contentReference[oaicite:103]{index=103}

## 6) Runner 执行契约（必须）
函数签名：
:contentReference[oaicite:104]{index=104} pos_state:dict, now_bar_index:int) -> dict

必须实现：
- 时间止盈：bars>=RUNNER_MAX_BARS_AFTER_TP2 → 平 runner（reduceOnly）。:contentReference[oaicite:105]{index=105}
- 回吐止盈：满足 peak 条件则平 runner。:contentReference[oaicite:106]{index=106}:contentReference[oaicite:107]{index=107}OCK/CHOP 立即平 runner。:contentReference[oaicite:108]{index=108}

## 7) Loss Cooldown 契约（必须）
函数签名：
def on_position_closed(symbol:str, realized_pnl:float, source:str) -> None

规则：realized_:contentReference[oaicite:109]{index=109}s = now+1800s；并参与 max(...) 互:contentReference[oaicite:110]{index=110}:contentReference[oaicite:111]{index=111}
