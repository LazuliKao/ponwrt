# AN7581 FTTR / PPE 接入诊断

## 当前已验证的边界

现有 HM2004-DU 与两个 HL202 ONU 足以执行无上联的中转测试。
通过在两端的源地址策略路由表中临时增加对端 `/32` 路由，强制流量经主设备路由，
已验证接收 16 MiB TCP 数据，并观察到：

- `ppe/config` 的 `npu_attached` 从 0 变为 1，PPE flow 配置寄存器启用。
- conntrack 显示软件 `OFFLOAD`；抓取时没有 `HW_OFFLOAD`。
- PPE 学到了对应五元组，但仍为 `UNB`，`ppe/bind` 为空。

因此已证实物理接口的 flower 回调可以触发 NPU 初始化，尚未证实硬件转发。
单凭初始化成功、收包计数增长或软件 `OFFLOAD` 不能判断硬件加速成功。

## 诊断补丁

`patches-6.18/931-net-airoha-add-opt-in-PPE-binding-diagnostics.patch`
为 Ethernet 驱动增加 `ppe_debug` 参数，默认关闭、仅 root 可读写。
开启后各日志点分别限频，不修改规则接受、hash 比较或 FOE 提交策略。
日志前缀为 `ppe-diag:`，包括：

| 日志 | 说明 |
| --- | --- |
| `init ret=...` | 首次 flower 回调的 NPU/PPE 初始化结果；已初始化时不重复输出 |
| `replace ...` / `action ...` | cookie、ingress ifindex、协议、action ID，及 redirect 设备、push VID |
| `replace-result ...` | 模板提交返回值、失败阶段与可用的 extack；`ret=0` 仍不代表硬件已绑定 |
| `rx-check ...` | RX 到达驱动学习检查入口，并通过原有 hash 合法性与时间节流 |
| `learn ...` | 学到的硬件项状态、类型和原始 ib1 |
| `compare-miss ...` | 同一软件 hash 桶存在模板，但键比较不匹配 |
| `no-l4-template ...` | 该次检查没有匹配的 L3/L4 模板，接下来仍会尝试 L2 查找 |
| `commit ... ret=...` | FOE 写入尝试的结果，含 hash 与 SRAM/DRAM 路径；也可能来自删除操作 |

`replace-result` 的 `stage` 为：`match`、`actions`、`ether-address`、
`prepare-egress`、`ports`、`mangle`、`translated-tuple`、`allocate-template`、
`commit-template` 或 `insert-template`。阶段是定位提示；成功时停在最后一个阶段。
`action id` 对照本次编译内核的 `include/net/flow_offload.h`。

限频可能丢弃日志；日志缺失不能单独证明某条路径从未执行。
`commit ret=0` 表示驱动提交函数成功返回，仍需回读硬件项确认 `BIND`，
并验证真实中转流量命中及撤销后的软件回退。

## 编译检查与同步

本补丁已在开发服务器现有 **6.18.52** 打好前序补丁的源码副本上，
以 `patch --fuzz=0 -p1` 应用成功，并使用该树的 GCC 14.4.0、配置和
`lk_h3c_hm2004_du-build-container:latest` 完成 `airoha_ppe.o` 交叉编译
（`KCFLAGS=-Werror`）。检查产物位于 `/tmp/fmcs-ppe-diag-lk/`。

现场固件是 **6.18.55**。上述检查不是匹配该固件的模块构建，未向设备加载产物。
同步补丁后，应通过目标固件对应源码与配置重新构建、部署完整固件。
不要只在已经 prepare 的旧 build_dir 中增量编译而漏掉新补丁；
检查实际构建源码包含 `ppe-diag:` 和 `module_param(ppe_debug, bool, 0600)`。

`931` 已在现场部署。后续撤销诊断需要同步本说明、Airoha `932` 补丁，
以及 `target/linux/generic/pending-6.18/676-netfilter-add-opt-in-flowtable-lifecycle-diagnostics.patch`。
没有执行 git 提交或推送。

### 规则被快速撤销：第二轮诊断

现场 `931` 日志已证实：双向 `replace-result ret=0`，模板匹配，
SRAM BIND 提交返回 0；但同一 hash 随后在约 30 ms 至 1 s 内被写为 INVALID。
尚不能由提交返回值证明硬件实际命中过数据包。

`932` 继续复用默认关闭的 `ppe_debug`，增加：

- `destroy`：flower 删除的 cookie、hash、类型和软件 ib1。
- `egress-remove`：出口映射变化触发的删除及其调用者。
- `stats`：idle、当前硬件 timestamp、lastused，核查误判老化。

通用 `676` 增加默认关闭、0600 的
`/sys/module/nf_flow_table/parameters/flowtable_debug`，日志前缀 `flowtable-diag:`：

- `teardown`：首次 teardown 的调用者与 conntrack 状态。
- `gc-teardown`：过期、conntrack dying 和剩余超时。
- `gc-closing`：TCP closing 导致硬件规则撤销。
- `cleanup`：显式设备清理的设备名和 ifindex，或整表清理。

日志的 `orig` / `reply` 与驱动的 flower cookie 对应；按时间和 cookie 联查。
两个补丁都不改变老化、关闭、清理和撤销策略，不能通过禁止删除来掩盖问题。

两份补丁均已在临时源码副本以 `patch --fuzz=0` 应用成功；
使用 GCC 14.4.0、ARM64 Linux 6.18.54 准备树，完成
`nf_flow_table_core.o` 和带前序诊断的 `airoha_ppe.o` 的 `KCFLAGS=-Werror`
编译检查。检查目录 `/tmp/fmcs-lifecycle.yi1rR0/`，
这不是现场 6.18.55 固件的构建或可加载模块。

当前服务器 `/data/wrt_release/ponwrt-an7581` 尚无匹配现场的 prepared kernel；
旧的 `ponwrt-hm2004-du` 树已不存在。必须重新构建匹配固件后再执行第二轮诊断。

## 部署后的无上联验证

测试工具位于 `openwrt-fttr/h3c-fmcs/tests/routed-offload-probe.sh`，
使用方式及清理行为见同目录 README。先确认两端的 MQTT MAC、IP、源地址
策略路由表与当前 ONU 槽位，以下参数仅对应本次现场记录。
确认两端 TCP 49175 空闲、driver mask 为 0，然后将工具复制到主设备的临时目录。

在主设备执行以下子 shell，使诊断参数在测试结束时恢复原值：

```sh
(
    set -e
    p=/sys/module/airoha_eth/parameters/ppe_debug
    test -w "$p"
    q=/sys/module/nf_flow_table/parameters/flowtable_debug
    test -w "$q"
    old=$(cat "$p")
    oldq=$(cat "$q")
    trap 'printf "%s\n" "$old" > "$p"; printf "%s\n" "$oldq" > "$q"' EXIT
    trap 'exit 1' HUP INT TERM
    printf 'Y\n' > "$p"
    printf 'Y\n' > "$q"
    sh /tmp/routed-offload-probe.sh \
        74ADCB3BA21D 1C94685DF998 \
        192.168.1.120 192.168.1.244 192.168.1.1 24 300
)
dmesg | grep -E 'ppe-diag:|flowtable-diag:'
```

另一个 SSH 会话可在测试期间采集 `ppe/config`、`ppe/entries`、`ppe/bind`、
对应 conntrack 项与 nft counters。不要以测试结束后消失的短连接判断绑定失败。
保留本轮时间范围内日志，按 cookie、五元组、hash 关联两个方向。
结束后核实参数恢复、mask 为 0、测试 nft table 和两端临时路由清理完成。

以上 mask 24 仅对应当前两个 HL202 位于 fttr4/fttr5（ONU ID 3/4）。
每次重注册或重启后重新核查槽位，不能沿用历史 mask 6。

### 当前现场阻塞

一次子猫 tcpdump 后台抓包尝试后，`74ADCB3BA21D` 的 MQTT exec 不再回复，
但 ping、光链路仍正常。停止继续向该通道追加测试；测试脚本已移除抓包选项。
主设备 mask=0、ppe_debug=N、redirects=1，临时 nft table 已删除；
另一端的临时路由已删除。该子猫的临时路由删除尚未得到执行确认。
恢复命令通道后，先核查并仅删除本次测试的精确路由：

```sh
ip route show table 300 exact 192.168.1.244/32
# 仅在存在以下同配置项时删除：
ip route del table 300 192.168.1.244/32 via 192.168.1.1 dev br0 metric 42760
```

MQTT exec 超时不取消子猫命令，也不能据此声称清理成功。
目前没有通过重启子猫或切断光链路强行恢复。

## 带宽验收

先完成双向 BIND 持续存在、实际命中、正确 VID 和撤销回退，再测带宽。
现有 16 MiB / 每秒 1 MiB 的探测只用于观察生命周期，不是吞吐基准。
满带宽测试需持续流量、足够性能的两端和 CPU/吞吐对照；子猫自身生成数据也可能成为瓶颈。
多 ONU 共享 FTTR 光链路与主设备 2.5G SerDes，分别记录单对、双向并发和多对的总吞吐，
以实测光链路和端口可用容量为上限，不能把所有子设备端口速率相加作为共享链路保证。

定位顺序：

1. `replace-result ret<0`：先按 stage / action / extack 修复提交问题。
2. `ret=0` 但无匹配：检查 RX reason、hash 桶与模板键；限频日志不能替代计数或跟踪。
3. 模板匹配但 `commit ret<0`：检查 SRAM ACK、NPU 生命周期与提交失败处理。
4. 提交成功：结合硬件 `BIND`、出口 NBQ/VID 和真实中转验证，不只检查软件 shadow。

## bridge 接入仍需单独处理

当前通用 bridge path 补丁会从 FDB 选定逻辑出口，再沿其
`ndo_fill_forward_path` 走到物理出口。FTTR 提供 VLAN path 后，最终
`info.indev=fttr0`，而 bridge 的 `dst_dev=fttrN`；
`675-11-netfilter-nft_flow_offload-Add-bridgeflow-to-nft_flo.patch` 中的
`info.indev != dst_dev` 检查会拒绝该路径。

需要分别保留逻辑 FDB 出口校验、软件路径的入接口身份和硬件入口的物理接口/VID。
还需要处理 fw4 对自定义 lower device 的发现与花式规则回调生命周期。
不能简单删除检查或仅代理 `ndo_setup_tc`。这部分在 routed 绑定原因明确后推进。

物理 fttr0 可以加入 bridge-family flowtable；此前 `nft -c` 的
`Not supported` 来自更新现有 flowtable 时遗漏 `flags offload`，不是桥成员限制。
