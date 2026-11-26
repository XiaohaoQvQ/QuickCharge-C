# Quick Charge v3.1.7-c

C 守护，功能与 shell 版 3.1.7 等价。

## 守护逻辑这版没改

`src/qcmon.c` 与 3.1.6-c 逐字节相同。`bin/qcmon` 重编了一次，差异只在注入的版本串（`-v` 报 `qcmon 3.1.7`，md5 `6d9b0e8e`）。也就是说：**3.1.6-c 的充电行为不变**，不必为了"修 bug"升级。

3.1.6-c 的 87 项 c-test 这版照样全过，x86 现编和官方 arm64 二进制（qemu）各跑一遍。

## 脚本层跟上主仓

C 包里的 `customize.sh`、`action.sh`、`lib/qc.sh` 是同一份，所以拿到了主仓这版的改动：

- `overrides on|off` 写值时落 `overrides_manual=1`，迁移不再拿 `defrev` 猜用户意图；
- `sd_state_get` 补 `overrides_manual` 分支（否则又是"写了读不到"）；
- 安装提示里 `overrides` 遗留校正只在无标记时触发。

## 文档

- README 的测试数字跟到 87 项 / 547 行，`src/qcmon.c` 2711 行、`lib/qc.sh` 4116 行。
- `TECH.md` 修正 `shell-temp` 关闭语义（不写，不是归零）与配置改写目录数。

## 测试

| 项 | 结果 |
|---|---|
| c-test（x86 现编） | 87 |
| c-test（官方 arm64 bin，qemu） | 87 |
| shell 侧仿真 / 单元 | 613 / 97 |

## 装机

覆盖安装，重启即可。与 shell 版随时互换，`state.conf` 是同一份。

- 完整包：`QuickCharge-v3.1.7-c.zip`
- 更新清单：`update.json`
