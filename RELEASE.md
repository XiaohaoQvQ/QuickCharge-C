# Quick Charge v3.1.12-c

这版 **C 守护代码零改动**（`src/qcmon.c` 一个字节没动），只修两处文本：**本仓的 `tests/preflight.sh` 副本分叉了**，以及 `TECH.md`「已知限制」还在教人调 `curr_max`。

## 一、本仓的 preflight 副本少了两版的守卫

主仓 `tests/preflight.sh` 的 `same` 清单覆盖了源码、`c-test`、`lib/qc.sh` 和四个脚本，**唯独没有它自己**，于是 3.1.11 加的两个东西没跟过来：

| 文件 | 行数 | 缺什么 |
|---|---|---|
| 主仓 `tests/preflight.sh` | 164 | — |
| 本仓 `tests/preflight.sh`（旧） | 148 | versionCode 反算、去重后的「电池偏热」检查 |

这是最该防的一类分叉：**本仓 `versionCode` 写错时没有任何检查会红**，而 3.1.11 整版的工作就是为了让这种错立刻红。旧副本单独跑也不可用 —— 它按主仓 README 的「当前版本」行取值，`A` 指到本仓时 9 项 FAIL。

现在清单补齐，本仓副本由主仓覆盖，两仓逐字节相同：

```sh
same "$A/tests/preflight.sh" "$B/tests/preflight.sh" "tests/preflight.sh 逐字节一致"
```

**负向验证**：往本仓副本尾部追加一行 `# drift`，`FAIL tests/preflight.sh 逐字节一致`；还原后全绿。

## 二、`TECH.md` 的「已知限制」还在教人调 `curr_max`

> **`curr_max` 只能限流，不能加速。** 充电功率由适配器、协议、电芯内阻决定，改上限无效。

`curr_max` 3.1.0 就删了，`customize.sh` 升级时还会把这个键从 `state.conf` 清掉。改成保留结论、去掉开关：

> **任何电流上限都只能限流，不能加速。** 充电功率由适配器、协议、电芯内阻决定，改上限无效；**本模块不提供这个开关**。真正能提速的是让温度反馈变低（壳温伪装）。

## 测试

| 项 | 结果 |
|---|---|
| 语法 `sh -n` | 8/8 |
| `preflight.sh --full` | 全绿（含 preflight 自比对） |
| 单元 | 92 |
| 仿真 | 613/613 ×2 |
| c-test（x86 现编） | 93 |
| c-test（arm64 bin + qemu） | 93 |

`bin/qcmon` 已重编、自报 3.1.12。用 `-DQC_VER="3.1.11"` 重编能与上一版发布的 bin 逐字节相同（`b1eda335…`），说明这一版的二进制除了内嵌版本串之外没有任何差异。

## 装机

覆盖安装，重启即可。只用 shell 版的用户装 v3.1.12 就行。

- 完整包：`QuickCharge-v3.1.12-c.zip`
- 更新清单：`update.json`
