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

本轮需要提交同步的 PonWrt 文件为本说明和 `931` 补丁；没有执行 git 提交或推送。

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
    old=$(cat "$p")
    trap 'printf "%s\n" "$old" > "$p"' EXIT
    trap 'exit 1' HUP INT TERM
    printf 'Y\n' > "$p"
    sh /tmp/routed-offload-probe.sh \
        74ADCB3BA21D 1C94685DF998 \
        192.168.1.120 192.168.1.244 192.168.1.1 6 300
)
dmesg | grep 'ppe-diag:'
```

另一个 SSH 会话可在测试期间采集 `ppe/config`、`ppe/entries`、`ppe/bind`、
对应 conntrack 项与 nft counters。不要以测试结束后消失的短连接判断绑定失败。
保留本轮时间范围内日志，按 cookie、五元组、hash 关联两个方向。
结束后核实参数恢复、mask 为 0、测试 nft table 和两端临时路由清理完成。

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
