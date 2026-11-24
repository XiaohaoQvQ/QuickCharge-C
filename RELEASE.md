# Quick Charge v3.0.4-c

与 shell 版 3.0.4 同步，修「功率锁在 30W」：屏幕快充的回写目标选错了节点。

## 原因

`usb/current_max` 是 USB 端口能力上限（PD 协商用，插电前常见 2000mA），不是电池
充电电流。3.0.3-c 及更早把它当 FCC 的兜底候选，写入后触发充电器重新协商，实测
OPPO 机型从 60W 掉到 37W、一加机型卡在 33W。有些机器没有电池侧节点，旧代码就
一路兜底到它，「接管」实际变成「限流」。

## 修复

- `scr_fcc_node` 只认电池侧节点（`constant_charge_current_max`、`battery/current_max`、
  `chg_current_max`），排除 `usb/`、`charger/`、`pc_port/`、`wireless/`、`input_current_limit`；
- 找不到时返回「不支持」，不写任何节点；
- 新增 `run/fcc.node` 记录基线对应的节点，节点变了就作废旧基线；
- `bin/qcmon` 重编为 `3.0.4`。

只有 `usb/current_max` 的机型会从「假装接管」变成「明确不接管」，功率不再被模块
打断，交回系统控制。

## 版本号

`3.0.4-c` / `versionCode 300008`。`updateJson` 仍指向 C 仓，不会把 shell 包推给 C 版用户。

## 验证

C c-test 90 项、`chg` 44、`cool` 20、`state` 12 全过；节点选择实测：battery 与 usb
同时存在时选 battery，只剩 usb 时返回不支持。
