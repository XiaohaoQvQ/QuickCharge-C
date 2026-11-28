# Quick Charge v3.1.18-c

> 从 3.1.16 或更早升上来的，3.1.17 和 3.1.18 的内容都在这一版里。业务逻辑与 shell 版
> 同源同测试，`lib/qc.sh` 两仓逐字节一致。

## 安全模式不再被兜底推翻

`post-fs-data.sh` 连续多次开机未完成时会写 `step=safe-mode` 跳过挂载；3.1.16 的
`service.sh` 只认 `done`，其余 step 一律当「没跑完」，于是把安全模式刚跳过的覆盖层
重建再挂回去。现在 `safe-mode` 单独识别为主动跳过：打一条 `W [boot]`，不重建、不补挂。

## 清理残留锁终于进日志

这段代码以前写了两遍（先 echo 到 stderr，再 `sd_info`），第一遍就把锁删了，第二遍的
条件永远不成立；而且它写在 `sd_state_load` 之前，`_sd_lv` 还是 0，`I` 级一条都落不了盘。
现在合并成一条并挪到 `sd_state_load` 之后。

## `[env]` 补齐 `nm=` 字段（C 侧专属修复）

3.1.16 的说明承诺了 `nm=bind|nomount`，C 侧实际只打了 `overrides` 和 `mounts`。元模块
激活时没有 bind mount，`mounts=0` 直接把 `alive` 推成 `no` —— 注入明明成功了，日志却说
没生效。现在按 `nm.list` 非空且 `$MODDIR/system` 存在判存活，与 shell 版同一套规则。

`qcmon` 按 `module.prop` 的 `3.1.18` 重编（版本号编进二进制），静态链接，Android 8~16 通用。

## 测试

`tests/c-test.sh` 新增三条（`nm=bind`、`nm=nomount`、不误报 `alive=no`），用例数 102 → 105，
全绿；`tests/sim/run.sh` 新增 T58 九条，用例数 643 → 652，全绿。

## 升级说明

覆盖安装即可。C 通道与 shell 通道共用 `state.conf`，`daemon.ver` 里记的是代码指纹，
从 shell 版切过来或反向切换时，旧守护会自己让位重启，不会出现「装新的跑旧的」。
