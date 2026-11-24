# Quick Charge v3.0.3-c

与 shell 版 3.0.3 同步，修掉「装到最新版仍提示更新」的配置错误。守护逻辑没有改动，`bin/qcmon`
重编为 `3.0.3`。

## 原因

`module.prop` 的版本号字段写成了全小写 `versioncode`，管理器按驼峰 `versionCode` 取值，
取不到就当成 `0`。于是 `update.json` 里的 `versionCode`（300004）永远大于本地值 `0`，
模块卡片一直显示「更新」。

`ksud module list` 把 `module.prop` 的键原样输出，管理器直接 `optInt("versionCode", 0)`，
中间不做大小写归一化。

## 修复

- `module.prop` 改为 `versionCode=`（驼峰），版本 `3.0.3-c` / `300006`；
- `update.json` 同步 `version` / `versionCode` / `zipUrl`；
- 模块内读自身版本的代码兼容两种写法，升级瞬间不会读到空值。

## 这一版需要手动刷一次

设备上的 3.0.2-c 本地版本号被读成 `0`，刷入 3.0.3-c 后才能正确读出 `300006`，按钮才会消失。
直接点「更新」按钮装也可以，装完同样是这一版。

## 验证

C c-test 90 项、`chg` 44、`cool` 20、`state` 12 全过；`module.prop` 与 `update.json` 的
`versionCode` 逐字一致。C 版 `updateJson` 仍指向 C 仓，不会把 shell 包推给 C 版用户。
