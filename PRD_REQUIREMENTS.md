# PRD_REQUIREMENTS.md
# V7-Atlas（USDC-M）合约量化系统｜需求规格（Codex 版本）

## 0. 目的
将策略说明书固化为：可实现、可验收、可测试的工程需求，避免实现偏航与口径冲突。

## 1. 范围与非目标
### 1.1 必须实现（MUST）
- 交易所：Binance USDC-M 永续合约（USDC 计价/结算）。
- 主周期：30m；辅助周期：2h（趋势/结构约束）。
- 总流程：Regime识别 → 模板互斥路由 → 评分与闸门 → 执行（Entry/SL/TP/Runner） → 风控维护（互锁/黑名单/DeRisk） → 复盘日志。
- 必须补齐能力：Runner、Regime Shift Playbook、B档动态筛选（B_BLACKLIST含复归）、Health权重与校准周期、ShockGuard分位数校准周期、Loss Cooldown、curses 终端UI。
- P0 生存线：Algo Conditional + 1秒可见性 + 不可见立刻紧急平仓 + 禁新开；幂等与重启恢复。

### 1.2 非目标（NOT）
- 不做跨交易所；不做期权/现货；不做做市/网格；不做重型回测系统（仅需最小回放/record-replay用于测试）。

## 2. 优先级与冲突消解（强制）
- P0：安全/生存线（执行红线、紧急平仓、幂等恢复）最高优先级。
- P1：策略行为（互斥路由、冷却/冻结、Runner、RegimeShift 动作矩阵等）。
- P2：评分细节/权重与自适应（允许后续微调，但必须可配置可追溯）。

**关键冲突裁决（必须遵守）：**
- 仓位 sizing 同时存在“风险定额”与“10x+权益15%”口径。最终规则：
  1) 以风险定额公式为主（RISK_PER_TRADE / BO_RISK_CAP）。
  2) “权益15%”作为单笔**初始保证金占用上限**（MARGIN_CAP_PCT=0.15），用于裁剪风险定额计算出的 notional。
  3) A/B 档默认杠杆分别为 10x / 8x（可配置，但默认必须如此）。

## 3. 交易标的与分档（A/B 池）
### 3.1 A/B 池（默认写死，启动必须打印并校验）
- A：ETHUSDC、SOLUSDC
- B：XRPUSDC、DOGEUSDC、BNBUSDC、SUIUSDC、FILUSDC、LTCUSDC、AVAXUSDC、LINKUSDC、ADAUSDC、BCHUSDC、NEARUSDC、ARBUSDC、CRVUSDC、WLDUSDC、TIAUSDC、AAVEUSDC、ORDIUSDC

### 3.2 杠杆（默认）
- A：10x
- B：8x

## 4. 模块结构（文件数≤5，MUST）
建议拆分为 5 个 Python 模块（tests/ 与配置文件不计入）：
1) exchange_adapter：交易所适配（REST/WS、下单/撤单/查询、Algo Conditional）
2) engine_runner：主循环与调度（bar驱动、信号/闸门、下单、风控维护）
3) strategy_core：Regime→Template→Score→Action
4) risk_manager：仓位、SL/TP/Runner、RegimeShift、冷却/冻结/黑名单/DeRisk
5) state_store：持久化与事件日志（幂等键、状态机、滚动窗口指标）

## 5. 数据新鲜度闸门（MUST）
- 行情必须来自真实交易所数据（WS优先、REST兜底）。
- 新鲜度不足禁止新开，只允许 reduceOnly 风险维护。
- 默认阈值（可配置）：
  - 30m 最新K线 close_time 距 now > 75min：禁新开
  - 2h 最新K线 close_time 距 now > 5h：禁新开
- 触发时必须写 RejectReason=DATA_STALE，并输出延迟秒数。

## 6. Step2 模板互斥路由（MUST，严格）
输出模板只能是：Pullback / RangeEdge / 2B / Breakout / None

路由规则：
- TREND：默认 Pullback；只有满足“突破硬条件”才允许 Breakout 替代；禁止 RangeEdge/2B。
- RANGE：默认 RangeEdge；若出现完整 2B 形态可允许 2B；禁止 Breakout。
- CHOP：只允许 2B；其他全禁。
- SHOCK：None；禁止新开仓（只允许 reduceOnly）。

抖动抑制（MUST）：
- REGIME_MIN_HOLD：连续≥2次检查才允许切换
- TEMPLATE_COOLDOWN：模板切换后 6min 内禁止再次切换

## 7. 互锁闸门（MUST）
是否允许新开 = now >= max(
  template_cooldown_until,
  loss_cooldown_until,
  freeze_until,
  shock_cooldown_until,
  b_blacklist_until
)
任一未到期必须拒单并写 RejectReason（单次打印，避免刷屏）。

## 8. Loss Cooldown（MUST）
- 任意亏损平仓（realized_pnl < 0）触发：该 symbol 30min 禁新开（含同向/反向）。
- 冷却期间允许：reduceOnly 平仓/减仓/补挂保护单；禁止加仓/新开。
- 必须记录：symbol、realized_pnl、cooldown_until、当时 Regime/Template、入场分数组成、RejectReason。

## 9. P0 执行红线（MUST）
### 9.1 Algo Conditional 统一（生存线）
- 所有条件触发保护单（尤其止损）必须走 Algo Conditional 端点/通道。
- 必须支持 open algo orders 查询。

### 9.2 1 秒可见性（Visibility Sentinel）
- 保护性止损下达后 1 秒内必须在 open algo orders 可见；
- 若不可见：立刻 emergency_flatten(symbol)：reduceOnly 市价平仓；并对 symbol 设置 30min 禁新开；
- 必须写事件：EMERGENCY_FLAT + 原因 + 幂等键。

### 9.3 幂等与重启恢复（MUST）
- 普通单：clientOrderId；条件单：clientAlgoId；格式统一。
- 同 id 重发不得重复下单；撤单目标不存在视为成功。
- 启动时必须恢复保护单：扫描 open algo orders，缺失/偏离过大 → 撤并重建。

## 10. 仓位 sizing（MUST，已裁决口径）
默认参数：
- RISK_PER_TRADE = 0.008（0.8%）
- BO_RISK_CAP = 0.0045（0.45%）
- MARGIN_CAP_PCT = 0.15（单笔初始保证金占用上限：权益 15%）
- SIZE_BY_RISK = true（禁止固定张数）

计算口径（必须）：
1) Risk$ = Equity * min(RISK_PER_TRADE, is_breakout ? BO_RISK_CAP : RISK_PER_TRADE)
2) qty_by_risk = Risk$ / (SL_DIST * contract_value_per_unit)
   - SL_DIST 为入场到止损的价格距离（标记价口径，必须明确 working_type）
3) notional_cap = Equity * MARGIN_CAP_PCT * leverage
4) qty_cap = notional_cap / mark_price
5) 最终 qty = min(qty_by_risk, qty_cap)，并通过交易所精度/最小下单量校验
6) 日志必须打印：Risk$、SL_DIST、qty_by_risk、qty_cap、final_qty、被哪条上限裁剪

## 11. 尘埃仓位（Dust Position）清理（MUST）
- 若某持仓对应的初始保证金/保证金占用（initMargin 或估算值） < 0.1 USDC：
  - 立即 reduceOnly 市价平仓（避免占用仓位与策略互锁）
  - 记录事件：DUST_AUTOCLOSE
  - 不触发 loss cooldown（除非产生 realized_pnl<0 且你显式选择纳入；默认不纳入）

## 12. Entry 执行纪律（MUST）
目标：减少刷接口与劣化成交，避免高频 cancel/replace。

默认：
- ENTRY_TYPE = POST_ONLY_LIMIT
- TTL = 45s（每档入场挂单生命周期）
- REPRICE_MAX = 2（触发式改价最多2次）
- REPRICE_DEVIATION = 0.18 * ATR_1m（最新价与挂单价偏离阈值）
- CANCEL_REPLACE_LIMIT = 10 per 10min per symbol

规则（必须）：
- 入场限价必须 postOnly=true；timeInForce=GTC（或 GTD）；价格为对手价外一档。
- 仅当满足“偏离阈值且仍在边界带附近”才允许改价；总改价≤2。
- 超过 TTL 未成交：取消入场挂单，等待下一次信号；不得持续改价刷接口。
- cancel/replace 限频：单 symbol 10min 内≤10次；超限则冻结该 symbol 的 entry 动作到窗口结束（只允许 reduceOnly）。

## 13. TP / Runner（MUST）
Runner 目标：只放大利润，不允许盈利回吐变亏损尾巴。
默认参数（30m）：
- RUNNER_MAX_BARS_AFTER_TP2 = 6
- RUNNER_GIVEBACK = 0.50
- RUNNER_MIN_PEAK_R = 0.80
- RUNNER_TRAIL_ATR = 0.90 * ATR_30m
- RUNNER_HARD_TP = 2.50R

规则（必须）：
- TP2 后计时，bars>=MAX_BARS → 平 runner（限价优先，2秒未成交市价兜底）。
- peak 达标且回吐到阈值 → 立即平 runner。
- new_regime ∈ {SHOCK, CHOP}：立即平 runner。

## 14. Regime Shift Playbook（MUST）
触发：任意 Regime 切换（含进入 SHOCK）必须触发一次持仓再定价。
顺序必须固定：
1) ShockGuard / 保证金 DeRisk
2) Runner
3) 剩余仓位止损/锁利/减仓/追踪参数更新
4) 更新互锁与日志（RegimeShiftLog）

动作定义（默认，可配置）：
- 收紧止损：SL_DIST×0.80（或更近结构位）
- 锁利：BE+0.10R
- 减仓：reduceOnly 快速减 X%

## 15. ShockGuard（MUST）
### 15.1 SHOCK 触发阈值（默认）
- A：1m振幅>0.9% 或 3m振幅>1.6% → Regime=SHOCK → None
- B：1m振幅>1.5% 或 3m振幅>2.6% → Regime=SHOCK → None

### 15.2 分位数校准周期
- 默认：每 24h 重新计算 Q95_1m/Q95_3m，记录 recalib_ts
- 提前：连续触发≥3次且间隔<60min，或 vol_ratio>1.60 → 提前重算一次
- 回退：不可算则沿用最近有效Q95并进入保守模式（缩仓 + 抬MinScore + 禁Runner）

## 16. B_BLACKLIST 动态筛选（MUST）
在线指标（每币种滚动 30 次执行或 24h）：
- FillRate30、SlippageCost30_R、CancelRate30、SpreadP95

淘汰规则（默认）：
- FillRate30<0.60 或 SlippageCost30_R>0.20R 或 CancelRate30>0.45 → B_BLACKLIST 7天；每日复评
- 黑名单期间只允许 reduceOnly

复归规则（必须写全）：
- 连续 2 个窗口满足 FillRate30≥0.65 且 SlippageCost30_R≤0.15R 才允许回到 B 池

## 17. Health / FillScore（MUST，至少可计算可展示）
Health 建议权重（默认）：
- WinRate=25%，AvgR=30%，MaxDD=20%，ProfitFactor=15%，Fallback/FillQuality=10%
校准周期：
- 每满 200 笔交易或每月（取更早）校准一次
要求：
- 输出 0–100 的 Health；写入日志/UI；可用于上调 MinScore/缩仓（不强制作为硬闸门，但必须支持接入）

## 18. 终端 UI（MUST）
- 必须 curses 双窗口：
  - 上半：滚动日志（环形缓冲 N=500）
  - 下半：Top3信号表 + Positions表（含保证金字段：Equity/Available/UsedMargin/Maintenance/每仓InitMargin）
- 刷新 2–4Hz，diff 渲染只更新变化单元格，降低闪烁
- 日志去重：同一扣分原因 60s 内只打印一次（PEN_LOG_DEDUP_SEC=60）

## 19. 交付物（MUST）
- 入口：python -m main --mode dry-run|testnet|live
- 自检：python -m main --selfcheck（检查权限、Algo能力、WS连通、精度、单向模式等）
- tests/：覆盖验收红线与关键行为（见 ACCEPTANCE_TESTPLAN.md）
