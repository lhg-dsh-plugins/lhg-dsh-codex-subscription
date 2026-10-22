# Bugfix: ChatPass 窗口只有 reset_after_seconds 时 resetsAt 被算成 1970 年

## 标题
`parseCodexUsage` 中 ChatPass 窗口的相对重置时间未换算成绝对时间：`resetsAt` 返回 600 / 1201 等接近 0 的值，界面 `ResetTime` 会显示 1970 年日期。

## 证据链（失败 → 通过）
最小复现（离线，不调用任何真实模型/订阅 API）：以 `now = Date.parse('2026-10-10T10:00:00Z')` 调用
`parseCodexUsage({ chatpass: { windows: [
  { used_percent: 10, limit_window_seconds: 604800, reset_after_seconds: 600 },
  { used_percent: 20, limit_window_seconds: 604800, reset_after_seconds: 1200 } ] } }, now)`

- 修复前：`resetsAt` 分别为 `600`、`1201`，期望 `1791627000`（now/1000+600）、`1791627600` —— 复现脚本输出 `REPRO FAIL: resetsAt wrong`，退出码 1；`git stash` 回到基线后同样失败。
- 修复后：`resetsAt` 分别为 `1791627000`、`1791627600`，复现脚本输出 `PASS`。

现有测试只覆盖了 ChatPass 带 `reset_at` 的情形（tests/credit-expiry.test.mjs），以及 rate_limit 窗口带 `reset_after_seconds` 的情形，未覆盖 ChatPass + `reset_after_seconds` 组合，故全量 891 个测试在修复前也全部通过。

## 根因
`src/usage.js` 的 `chatPassOf(value)` 没有接收/传递 `nowSeconds`，且写成 `value.windows.map(windowOf)`：
`Array.prototype.map` 会把 `(element, index, array)` 传给回调，于是 `windowOf(window, nowSeconds)` 的
第二个参数被填成数组下标（0、1…），其相对时间分支 `nowSeconds + reset_after_seconds`
算出 `0+600=600`、`1+1200=1201`。对比 `limitOf` 是显式以 `windowOf(window, nowSeconds)` 调用的，正确。

## 改动
- `src/usage.js`：`chatPassOf(value, nowSeconds)`，内部改为 `value.windows.map(window => windowOf(window, nowSeconds))`；调用点 `chatPassOf(value.chatpass, nowSeconds)`。
- `lib/index.js`：由 `pnpm run build`（tsdown）从上述源码重新构建的产物，仅对应这处改动（git diff 共 3 行）。
- `package.json`：version `3.1.4` → `3.1.5`，`repository.url` 改为落点仓，其余不动。

## 复验
- 复现脚本：修复前 FAIL（600/1201），修复后 PASS（1791627000/1791627600）。
- 相关测试：`node --test tests/usage.test.mjs tests/credit-expiry.test.mjs` —— 16 pass / 0 fail。
- 全量测试：`npm test` —— 891 tests，865 pass / 26 skipped / 0 fail，与修复前基线完全一致，无回归。
- 构建：`pnpm run build` 成功。

## 边界
- 只修离线解析逻辑；未调用真实 ChatGPT/Codex/订阅 API，未验证真实后端是否对 ChatPass 只下发 `reset_after_seconds`——但 `windowOf` 自身注释已声明「Some payloads give only the time left」且 rate_limit 路径已按此处理，ChatPass 与其「share the rate-limit window shape」（源码注释），应走同一换算。
- 带绝对 `reset_at` 的 ChatPass 窗口行为不变（`reset_at` 优先于 `reset_after_seconds`，与既有逻辑一致）。
