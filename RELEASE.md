# Quick Charge v3.0.7-c

与 shell 版 3.0.7 同步。用户反馈「停用热管理并不能取消阶梯充电，好像这个开关
根本没有作用」——观察正确，`thermal off` 只做 `setprop persist.sys.horae.enable 0`
和 `stop horae`，从来没碰过充电。

## horae 管什么

`horae` 是欧加的性能温控服务：CPU/GPU 降频、亮度、部分场景限帧。
充电的「阶梯」来自另外两处，都不经过它：

| 来源 | 节点 |
|---|---|
| 系统 thermal HAL | `battery/charge_control_limit`、`thermal_charging` |
| 充电驱动自己分档 | `oplus_chg/battery/cool_down`、`constant_charge_current_max` |

所以停 horae 对充电电流零影响，用户的感受是对的。

## 3.0.7-c 改了什么

- `status` 的热管理行改为 `horae 已停（只管性能/亮度，不影响充电阶梯）`，不再叫「热管理 已停用」
- `thermal off` 输出追加提示，指向 `tiers`
- 新增 `action.sh tiers`：列出 `charge_control_limit`、`charge_control_limit_max`、
  `thermal_charging`、`step_charging`、`constant_charge_current_max`、`current_max`、
  `input_current_limit`、`usb/current_max`、`bcc_current`、`oplus cool_down`
  的当前值与可写性，并给出判读
- README / RELEASE 同步说明 horae 的边界

## 想真的不按温度降

先用 `batt 32` 伪装电池温区（默认关，真实 ≥46℃ 自动撤销），再 `diag 60` 看 `ibat`。
若 `tiers` 里 `thermal_charging` 仍为 1、`ibat` 不回升，说明限流在驱动/HAL 内部，
伪装温度碰不到，需要按实际节点做定向解锁——把 `tiers` 输出发出来。

## 版本号

`3.0.7-c` / `versionCode 300014`。

## 验证

C c-test 94 项全过；shell 仿真 670 项（同一份 lib/action，含 `tiers` 断言）。
