# ACCEPTANCE_TESTPLAN.md
# V7-Atlas USDC-M｜验收口径 + 测试计划（Codex 必须跑通）
# 目的：把“交付是否合格”变成可执行的红线与测试清单，避免只写代码不闭环。

---

## 0) 适用范围与交付定义
### 0.1 适用范围
- 交易系统：加密货币 USDC-M 永续合约实盘交易程序（以 Binance USDC-M 为目标实现）。
- 主周期：30m；辅助周期：2h（用于结构/趋势辅助判定）。
- 输出：可运行程序 + 自检 + 测试套件（至少覆盖红线与关键行为）。

### 0.2 交付定义（什么叫“可实盘交付”）
满足以下全部条件：
1) P0 红线全通过（见第 1 节）。
2) 关键策略行为通过（见第 2 节）。
3) tests/ 可一键运行并通过（见第 3 节）。
4) 无 mock/假数据参与 LIVE 决策；mock 仅允许在 tests/ 内使用。
5) 提供 selfcheck（上线前自检）与 dry-run/testnet/live 三模式。

---

## 1) P0 验收红线（任何一条失败 = 不交付）
> P0 是“生存线”，优先级最高；即使策略很强，只要执行层红线失败，也不允许上线。

### A1. 条件触发保护单必须统一 Algo Conditional
- 所有“条件触发类保护订单”（尤其止损 SL）必须走 Algo Conditional 通道/端点。
- 不允许使用普通 stop_market 端点替代。
- 任何触发类保护单，都必须具备可查询的 algo open orders 记录。

### A2. 1 秒可见性红线（Visibility Sentinel）
- 下达保护性止损（Algo）后：在 1 秒内必须在 open algo orders 可查到。
- 若 1 秒内不可见：必须立刻执行
  - emergency_flatten(symbol)：reduceOnly 市价平仓（或 closePosition reduceOnly）
  - 并对该 symbol 设置 30min 禁新开（计入互锁闸门）
- 必须记录日志事件：EMERGENCY_FLAT + 原因（algo not visible within 1s）+ 幂等键。

### A3. 幂等与重启恢复红线
- 普通单：clientOrderId；条件单：clientAlgoId（必须存在且格式统一）。
- 同 id 重发不得产生重复订单；撤单目标不存在必须视为成功（幂等成功）。
- 系统启动/断线恢复：
  - 必须扫描 open algo orders；
  - 若保护单缺失或偏离超过阈值（阈值可配置），必须撤并重建；
  - 恢复过程不得导致重复保护单或“无保护裸奔”。

### A4. LIVE 模式禁止使用 mock/假数据
- LIVE 的任何决策（开仓/加仓/平仓/调 SL/Runner）必须来源于真实交易所数据与真实仓位状态。
- mock/record-replay 仅允许在 tests/ 与 dry-run 回放场景中出现。
- 若检测到 LIVE 使用 mock：立即拒绝动作 + 写 FATAL 日志并退出。

---

## 2) 策略行为验收（必须）
> 这些决定“系统是否符合策略说明书行为”，不是“策略是否赚钱”。

### B1. 模板互斥路由（Regime -> Template）必须严格执行
- 输出模板只能是：Pullback / RangeEdge / 2B / Breakout / None
- Regime 路由要求：
  - TREND：默认 Pullback；只有突破硬条件成立才允许 Breakout；禁止 RangeEdge/2B
  - RANGE：默认 RangeEdge；满足完整 2B 形态才允许 2B；禁止 Breakout
  - CHOP：只允许 2B；其他全禁
  - SHOCK：必须 None；禁止新开（只允许 reduceOnly 风险维护）
- 任何违反必须拒单并写 reject_reason。

### B2. 抖动抑制与模板冷却必须生效
- REGIME_MIN_HOLD：Regime 连续满足至少 2 次检查才允许切换
- TEMPLATE_COOLDOWN：模板切换后 6 分钟内禁止再次切换
- 验收时必须能观察到：满足切换条件但被抑制时的 reject_reason/日志。

### B3. Loss Cooldown（亏损冷却）必须正确
- 任意“亏损平仓”（Realized PnL < 0）触发：
  - symbol 维度 30min 禁新开（含同向/反向）
- 冷却期间允许：
  - reduceOnly 风险动作（平仓、减仓、补挂保护单）
  - 不允许任何加仓/新开
- 互锁规则必须采用 max(...) 闸门（见 INTERFACES_CONTRACTS.md）。

### B4. Runner 必须具备三要素（时间上限 + 回吐阈值 + 环境禁入）
- 必须实现：
  1) 时间止盈：TP2 后 bars>=MAX_BARS → 平 Runner（限价优先，失败市价兜底）
  2) 回吐止盈：peak 达标后回吐到阈值 → 立即平 Runner
  3) 环境禁入：Regime 进入 SHOCK 或 CHOP → 立即平 Runner
- Runner 的任何平仓动作必须 reduceOnly，并写 RUNNER_EXIT 事件日志。

### B5. Regime Shift Playbook（环境突变处置）必须触发且顺序固定
- 触发：任意 Regime 从 A 切换到 B（含进入 SHOCK）必须触发一次持仓再定价处理。
- 处理顺序必须固定：
  1) ShockGuard / DeRisk（保证金风险优先）
  2) Runner 处理
  3) 剩余仓位止损/锁利/减仓参数更新
  4) 更新互锁状态（cooldown/freeze）+ 写 regime_shift_log
- 必须可配置动作矩阵；日志必须可追溯每一步动作与订单回执。

### B6. ShockGuard 分位数校准周期必须正确
- 默认：每 24h 重新计算一次（并记录 recalib_ts）
- 提前：满足“短时间内多次触发”或“波动比例超阈值”可提前重算
- 回退：无法计算分位数时必须进入保守模式：
  - 缩仓或限制开仓
  - 提高开仓门槛（MinScore 上调）
  - 禁用 Runner（或强制更保守 Runner）

### B7. B 档标的动态筛选（B_BLACKLIST）必须正确
- 滚动执行质量指标（例如 30 次样本或 24h）：
  - FillRate30、SlippageCost30_R、CancelRate30、SpreadP95
- 淘汰阈值触发即进入黑名单 7 天；并每日复评一次。
- 黑名单期间该 symbol 禁止新开（只允许 reduceOnly 风险动作）。

### B8. HealthScore 权重与校准周期必须存在且可配置
- HealthScore 必须可计算并用于日志/UI 展示（不强制作为开仓闸门，但必须能接入）。
- 必须支持周期性校准（例如满 200 笔或每月取更早）。
- 必须记录：权重版本、校准时间、输入统计样本数。

### B9. 终端 UI（curses）必须满足基本体验
- 上半滚动事件流；下半状态表（持仓/候选/风控状态/冷却/黑名单）。
- 不允许刷屏式重复输出；关键事件必须可追踪。

---

## 3) 测试计划（最小可交付集合）
> 允许用 MockExchange / RecordReplay 做“可验证逻辑闭环”，但必须严格隔离到 tests/。
> LIVE 路径不得依赖 mock。

### 3.1 测试分层
- Unit tests（纯逻辑）：路由、互锁、Runner、RegimeShift 顺序、冷却、黑名单判定、分位数校准触发条件
- Integration tests（接口行为）：幂等、恢复、1 秒可见性哨兵、撤单幂等、异常重试
- Scenario tests（回放/事件注入）：shock 序列、执行质量变差序列、亏损后尝试开仓序列

### 3.2 必须包含的单元测试（pytest）
文件建议：tests/unit/
1) test_router.py
- 输入 (regime, breakout_hard_ok, has_2b, …) -> 输出 template
- 覆盖全部分支：TREND/RANGE/CHOP/SHOCK/None

2) test_cooldowns.py
- realized_pnl<0 -> loss_cooldown_until = now+1800
- 模板冷却 6min 生效
- max(...) 互锁闸门：任一未到期都拒绝新开，并给出正确 reject_reason

3) test_runner.py
- TP2 后 bars>=MAX_BARS -> 触发平 Runner
- peak_pnl_r 达标且回吐到阈值 -> 平 Runner
- new_regime in {SHOCK, CHOP} -> 立即平 Runner

4) test_regime_shift.py
- 触发 regime 切换 -> 必须调用 playbook
- 验证执行顺序：DeRisk -> Runner -> StopsUpdate -> Cooldown/Log
- 验证日志事件结构包含动作明细与订单幂等键

5) test_blacklist.py
- FillRate30<阈值 / SlippageCost30_R>阈值 / CancelRate30>阈值 -> blacklisted 7 天
- 黑名单期间新开被拒绝，但 reduceOnly 允许

6) test_shockguard_recalib.py
- 24h 到期触发重算
- 提前触发条件满足时触发重算
- 计算失败进入保守模式（并影响开仓/Runner）

### 3.3 必须包含的集成测试
文件建议：tests/integration/
1) test_algo_visibility.py
- place_algo_stop 后在 1 秒内 get_open_algo_orders 可见 -> PASS
- 不可见 -> 必须触发 emergency_flatten + 30min 禁开 + 写事件

2) test_idempotency.py
- 相同 clientOrderId 重发不得重复下单
- 相同 clientAlgoId 重发不得重复 algo 单
- cancel 不存在不得抛致命异常

3) test_restart_recovery.py
- 启动扫描 open algo orders：
  - 缺失 -> 重建
  - 偏离过大 -> 撤并重建
- 恢复后保证“有且仅有一个有效保护单”

### 3.4 必须包含的场景回放测试
文件建议：tests/scenario/
1) test_shock_sequence.py
- 连续触发 shockguard（<60min 多次）-> 提前 recalib
- 进入保守模式 -> 禁 runner 或提高门槛

2) test_bad_execution_sequence.py
- 构造执行质量恶化（fill 低、滑点高、撤单高）-> 触发 B_BLACKLIST
- 黑名单期间拒绝新开；到期后允许恢复（复评逻辑）

3) test_loss_then_try_open.py
- 亏损平仓后 10min 尝试开仓 -> 必须拒绝并写明 loss cooldown 锁住

---

## 4) 必须提供的运行命令（用于 CI / 本地验收）
> 你可以按项目选择 ruff/mypy，但至少要有 pytest 与一个“自检命令”。

### 4.1 最小命令集合（必须）
- `pytest -q`
- `python -m main --selfcheck`
- `python -m main --mode dry-run --symbols BTCUSDC`（或你项目的等价命令）

### 4.2 推荐命令集合（强烈建议）
- `ruff check .`（或等价 lint）
- `mypy .`（或等价类型检查）
- `python -m main --mode testnet --selfcheck`（若实现 testnet）

---

## 5) 验收输出物检查清单（必须）
### 5.1 运行形态
- main/入口支持：
  - `--mode dry-run|testnet|live`
  - `--selfcheck`
  - `--symbols`（指定标的）
- dry-run 不下真实订单（必须可验证）。
- live 模式必须要求 API key/权限齐全，否则拒绝启动。

### 5.2 日志与事件（必须）
- 每次候选评估日志必须包含：
  symbol, regime, template, final_score, action(OPEN/REJECT), reject_reason
- 每次关键风控动作必须包含事件：
  LOSS_COOLDOWN / REGIME_SHIFT / RUNNER_EXIT / SL_RECOVER / EMERGENCY_FLAT / BLACKLIST_UPDATE
- 每条关键事件必须包含幂等键（clientOrderId/clientAlgoId）与订单回执字段。

---

## 6) 合格判定（最终）
- 若 1 节任一 P0 红线失败：不合格（禁止上线）。
- 若 2 节任一策略行为缺失：不合格（不符合说明书）。
- 若 3 节测试未覆盖红线：不合格（不可交付）。
- 若 tests 通过但 LIVE 使用 mock：不合格（立即下线）。
