# Quick Charge v3.0.0-test-c

C 守护与 shell 版同步的 3.0 测试版。

## 默认极限

- `curr_target` 只分 100%/75%，到墙才降，不再 lim-3 提前掉；
- `scr_target` 取消 38/41℃ 两档折损，熔断前 1℃ 才放手；
- `scr_tick` 的 42℃ 硬闸抬到熔断前 1℃；
- `cool_tick` 不再 45℃ 写 `2`，熔断前 1℃ 才交还；
- `wall_of("60")` 与 `off` 都按无限制处理；默认 `wall=60`、`curr_edge=0`；
- `max_mode` 字段、`max on/off` 与 `run/max.bak` 一并删除。

## 面板

与 shell 版同一份 `webroot/index.html`：极限按钮、关闭温度墙、限流提前量滑块都已移除，
熔断阈值并入核心参数。

## 版本号

`3.0.0-test-c` / `versioncode 300000`。3.0 起 `versioncode = (主×100 + 次)×1000 + 修订×2 + 通道`，
C 用偶数通道。

## 验证

c-test 88 项、`chg-unit` 44、`cool-unit` 20、`state-unit` 12 全过；
`bin/qcmon` 已重编为 arm64 静态版，版本串 `3.0.0-test`。
