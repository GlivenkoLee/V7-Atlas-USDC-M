# ACCEPTANCE_TESTPLAN.md
# V7-Atlas USDC-M｜验收口径 + 测试计划（最终可交付标准）

## 0) 交付定义
合格交付必须同时满足：
1) P0 红线全通过（第 1 节）
2) 策略行为验收项全通过（第 2 节）
3) tests/ 一键运行通过（第 3 节）
4) LIVE 决策无 mock/假数据（mock 仅限 tests/）
5) 提供 selfcheck + dry-run/testnet/live 三模式

---

## 1) P0 验收红线（任何一条失败 = 不交付）
### A1. 条件触发保护单必须统一 Algo Conditional
- 所有条件触发保护单（尤其止损）必须走 Algo Conditional；
- 不允许普通 stop_market 端点替代。

### A2. 1 秒可见性红线（Visibility Sentinel）
- 保护性止损下达后 1 秒内必须在 open algo orders 可见；
- 不可见 → 必须立刻：
  1) emergency_flatten(symbol)：reduceOnly 市价平仓
  2) symbol 30min 禁新开
- 必须写事件：EMERGENCY_FLAT + 原因 + 幂等键

### A3. 幂等与重启恢复红线
- clientOrderId / clientAlgoId 必须存在且统一格式
- 同 id 重发不得重复下单；cancel 不存在视为成功
- 启动/恢复必须：
  - 扫描 open algo orders
  - 缺失或偏离过大 → 撤并重建
  - 不得出现“无保护裸奔”或重复保护单

### A4. LIVE 禁止 mock/假数据
- LIVE 任何决策必须来自真实交易所数据与真实仓位状态；
- 若检测到 LIVE 使用 mock：拒绝动作 + FATAL 日志并退出。

---

## 2) 策略行为验收（必须）
### B1. 模板互斥路由必须严格执行
- 输出模板只能 Pullback / RangeEdge / 2B / Breakout / None
- SHOCK 必须 None 且禁新开（只允许 reduceOnly）

### B2. 抖动抑制与模板冷却必须生效
- REGIME_MIN_HOLD≥2
- TEMPLATE_COOLDOWN=6min

### B3. Loss Cooldown 必须正确
- realized_pnl<0 → 30min 禁新开（含反向）
- 冷却期间仅允许 reduceOnly
- 互锁采用 max(...) 闸门

### B4. Runner 三要素必须齐全
- 时间止盈 / 回吐止盈 / 环境禁入（SHOCK/CHOP立即平）
- Runner 平仓必须 reduceOnly，并写 RUNNER_EXIT 事件

### B5. RegimeShift Playbook 必须触发且顺序固定
- 任意 regime 切换触发一次持仓再定价
- 固定顺序：DeRisk → Runner → StopsUpdate → Cooldown/Log
- 必须可追溯日志（动作明细+订单回执）

### B6. ShockGuard 校准周期必须正确
- 24h 重算 Q95；满足条件提前重算；不可算回退保守模式（缩仓 + 抬MinScore + 禁Runner）

### B7. B_BLACKLIST 必须正确（含复归）
- 触发阈值进入 7 天黑名单；每日复评
- 黑名单期间禁止新开（仅reduceOnly）
- 复归：连续 2 个窗口满足 FillRate30≥0.65 且 SlippageCost30_R≤0.15R 才允许回到 B

### B8. HealthScore 必须存在且可配置
- 权重与校准周期可配置
- 日志/UI 可看到 0–100 的 Health 与校准信息

### B9. 终端 UI（curses）基本体验
- 上半滚动日志 + 下半表格（含保证金字段）
- 不刷屏重复输出（去重节流生效）

### B10. Entry 执行纪律必须正确（新增重点）
- entry 必须 postOnly limit
- TTL=45s 到期必须取消并等待下一次信号（禁止无限改价）
- reprice<=2 且必须满足偏离阈值（默认 >0.18*ATR_1m 且仍在边界带附近）
- cancel/replace 限频：每 symbol 10min≤10，超限冻结 entry 到窗口结束

### B11. Dust Position 自动清理必须生效（新增重点）
- initMargin/估算保证金占用 < 0.1 USDC → 立即 reduceOnly 市价平仓
- 写事件：DUST_AUTOCLOSE

---

## 3) 测试计划（最小可交付集合）
> mock/record-replay 允许，但只能在 tests/；LIVE 路径不得依赖 mock。

### 3.1 单元测试（tests/unit/）
1) test_router.py
- 覆盖 TREND/RANGE/CHOP/SHOCK/None 全分支，验证模板互斥

2) test_cooldowns.py
- realized_pnl<0 → loss cooldown=1800s
- max(...) 闸门：任一未到期拒绝新开并给出正确 reject_reason

3) test_runner.py
- bars>=MAX_BARS → 平 runner
- peak 达标且回吐到阈值 → 平 runner
- new_regime in {SHOCK, CHOP} → 立即平 runner

4) test_regime_shift.py
- 触发切换必调用 playbook
- 校验顺序：DeRisk→Runner→StopsUpdate→Cooldown/Log
- 校验日志事件结构含动作明细与幂等键

5) test_blacklist.py
- 触发阈值进入 7 天黑名单
- 复归规则：连续 2 窗口达标才恢复

6) test_shockguard_recalib.py
- 24h 到期重算
- 提前触发重算
- 失败回退进入保守模式（影响开仓/Runner）

7) test_entry_discipline.py（新增）
- postOnly entry
- TTL=45s 到期取消
- reprice<=2 + 偏离阈值门槛
- 超限行为必须产生正确事件/拒绝原因

8) test_dust_autoclose.py（新增）
- init_margin_est <0.1 → 触发 DUST_AUTOCLOSE + reduceOnly 平仓

### 3.2 集成测试（tests/integration/）
1) test_algo_visibility.py
- 1 秒内可见 -> PASS
- 不可见 -> emergency_flatten + 30min 禁新开 + 写 EMERGENCY_FLAT

2) test_idempotency.py
- 同 clientOrderId / clientAlgoId 重发不重复下单
- cancel 不存在不抛致命异常

3) test_restart_recovery.py
- 启动扫描 open algo orders：缺失/偏离 → 撤并重建
- 恢复后“有且仅有一个有效保护单”

4) test_cancel_replace_rate_limit.py（新增）
- 在 10min 内触发 >10 次 cancel/replace → 必须冻结 entry 到窗口结束
- 冻结期间仍允许 reduceOnly 风险动作

### 3.3 场景回放测试（tests/scenario/）
1) test_shock_sequence.py
- 60min 内连续触发 ≥3 次 → 提前 recalib
- 保守模式禁 runner 或提高门槛

2) test_bad_execution_sequence.py
- 执行质量恶化 → 进入 B_BLACKLIST
- 黑名单期间拒绝新开；复归需连续 2 窗口达标

3) test_loss_then_try_open.py
- 亏损平仓后 10min 尝试开仓 → 必须拒绝并说明 loss cooldown 锁住

---

## 4) 必须提供的运行命令（验收用）
- pytest -q
- python -m main --selfcheck
- python -m main --mode dry-run --symbols ETHUSDC,SOLUSDC

（可选但推荐：ruff check . / mypy .）

---

## 5) 输出物检查清单（必须）
- main/入口支持：
  - --mode dry-run|testnet|live
  - --selfcheck
  - --symbols
- 每次候选评估日志必须包含：
  symbol, regime, template, final_score, action(OPEN/REJECT), reject_reason
- 关键事件必须覆盖：
  LOSS_COOLDOWN / REGIME_SHIFT / RUNNER_EXIT / SL_RECOVER / EMERGENCY_FLAT /
  BLACKLIST_UPDATE / DUST_AUTOCLOSE / ENTRY_EXPIRE / ENTRY_REPRICE_LIMIT
- 所有事件必须携带幂等键与订单回执字段

---

## 6) 合格判定
- 任一 P0 红线失败：不合格（禁止上线）
- 任一策略行为缺失：不合格
- tests 未覆盖红线：不合格
- LIVE 使用 mock：不合格（立即下线）
