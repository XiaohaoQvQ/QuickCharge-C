# Quick Charge v3.0.8-c

与 shell 版 3.0.8 同步：删掉「停用系统热管理（horae）」，换成充电阶梯解锁。

## 为什么删

`thermal off` 只有 `setprop persist.sys.horae.enable 0` + `stop horae` 两行。
`horae` 是欧加的性能温控服务（CPU/GPU 降频、亮度、限帧），不控制充电电流，
所以停它不会取消阶梯降流。这个开关命名和位置都在误导用户。

## 换成了什么

- `unlock on|off`：写 0 到 `battery/charge_control_limit`、`thermal_charging`、
  `step_charging`，原值备份到 `run/unlock.orig`，关闭/超温/卸载时还原
- 不碰 `usb/current_max`（打断 PD/VOOC 协商）和 `cool_down`（触发重选档，
  PKB110 实测 2→0 后 18W 掉到 9.7W）
- 兜底：电池真实温度 ≥45℃ 或读不到温度，立刻撤销并还原
- 面板开关改为「取消充电阶梯限流」
- `bin/qcmon` 新增 unlock_tick，随主循环每轮执行
- 老用户开机时把 `horae=0` 恢复成系统默认并删掉该状态行

## 版本号

`3.0.8-c` / `versionCode 300016`。

## 验证

C c-test 101 项全过（新增 unlock 默认不写、写 0 备份、46℃ 拒绝、升温撤销）；
shell 仿真 681 项同步通过。
