# Quick Charge v3.1.14-c

C 守护版。业务逻辑与 shell 版同源同测试，这一版的改动集中在「日志能不能回答问题」和
「两个守护同时在跑」，两侧逐条对齐。

## 日志：1 级也要有功率

`chg_snapshot` 的门禁从 2 级降到 1 级。用户默认就是 1 级，锁在 2 级等于**每一份功率反馈都
缺同一块证据**——PLG110 那份「10–20W」的日志 `[chg]` 计 0 条，连「在不在充」都判不了。

## 通道闸：qcmon 不再被谁拉起来就跑

`main()` 读完 `module.prop` 后校验 `channel`，不是 `c` 就写一条 `E` 退出：

```
E [daemon] channel=shell 不是 c，qcmon 拒绝启动（避免与 shell 守护双写日志与备份）
```

以前守卫只在 `service.sh` 的 `sd_use_c` 分支里，二进制被顺手执行、被旧脚本拉起、被
C 包刷回 shell 包后残留，都会和 shell 守护双写同一份 `log.txt`，并互相覆盖 `run/` 下的
`scr.orig`、`fcc.norm`、`unlock.orig`。而 ≤3.1.12 的 qcmon 还会写 `cool_down`——PJD110 的
33.6W→2.2W 就是这么来的。

shell 侧配套：`service.sh` 收掉 cmdline 含 `bin/qcmon` 的进程，`customize.sh` 在非 C 通道
装完后删掉 `$MODPATH/bin`（Magisk 不清理新包里没有的旧文件）。

## 屏幕与降流日志不再刷屏

- 「亮屏快充：本机不支持」改成**状态变化记一条 + 每 30 分钟兜底一条**；
- `Full` 从 `Charging` 里分出来，满电不刷这条 W；
- `cool_down` 节点存在但读不到时记一条一次性 `W`，不再静默。

## env 行

- `charge nodes=存在/候选` 两个数都对了（shell 侧的 1/0 是子 shell 丢变量所致，C 侧本来就对）；
- 新增 `charge nodes 命中=<全路径>`，`battery/current_max` 与 `usb/current_max` 分得开；
- 新增 `cool_down=<值>（<路径>）`，只给数量时 `fcc 节点=none` 追不下去。

## 其他

- 启动日志记 `service.sh`/`lib/qc.sh` 的 `md5@mtime`，混合安装一眼可查；
- 「N 条未卸净」带上具体路径。

## 测试

- c-test 100 项（x86 现编 + arm64 包内二进制 + qemu 双跑）
- 单元 95
- 仿真 629（shell 侧，跑两遍全绿）
- `preflight --full` 全绿
