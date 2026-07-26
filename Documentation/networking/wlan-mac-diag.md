# wlan0/wlan1 MAC 地址异常维测日志说明与性能评估

## 1. 文档目的

本文说明为定位 `wlan1` 偶现 `00:00:00:00:00:00` 而增加的内核维测日志，包括：

- 每个修改文件及函数在 Wi-Fi 调用链中的位置；
- 每一处新增日志对应的代码分支、字段含义和诊断价值；
- 各函数是否属于高频路径；
- 日志对 CPU、锁竞争、启动/连接时延和用户体验的影响；
- 推荐的抓取、判读及量产保留策略。

本次修改统一使用日志标签：

```text
wlan-mac-diag
```

日志只在接口名严格等于 `wlan0` 或 `wlan1` 时输出，不会对其他网络接口输出该组日志。

## 2. 总体调用链

### 2.1 接口创建链路

```text
用户态/HDI 请求创建 wlan0 或 wlan1
    -> nl80211: _nl80211_new_interface()
    -> cfg80211 驱动回调: add_virtual_intf()
    -> mac80211: ieee80211_if_add()
    -> mac80211: ieee80211_assign_perm_addr()
    -> cfg80211: cfg80211_register_netdevice()
    -> netdevice 注册完成
```

对于不使用 mac80211 的 fullmac/vendor 驱动，可能不会经过
`ieee80211_if_add()` 和 `ieee80211_assign_perm_addr()`，但只要其使用
`cfg80211_register_netdevice()` 注册接口，仍能获得注册阶段的 MAC 快照。

### 2.2 运行期修改 MAC 链路

```text
用户态/HDI 下发 MAC
    -> ioctl/rtnetlink 等内核入口
    -> netif_set_mac_address()
    -> 驱动 ndo_set_mac_address()
    -> mac80211 驱动通常进入 ieee80211_change_mac()
    -> _ieee80211_change_mac()
    -> eth_mac_addr()
    -> 更新 net_device、vif 和 bss_conf 地址
```

`IChipIface::SetMacAddress()` 的具体 HDI/vendor 后端实现不在当前 Linux
仓库中。如果上层报告设置成功，但内核完全没有
`wlan-mac-diag: set entry`，说明调用可能没有进入通用
`netif_set_mac_address()` 路径，需要继续在 HDI/vendor 驱动实现中埋点。

## 3. 公共日志字段

| 字段 | 含义 |
| --- | --- |
| `name` | 网络接口名，只关注 `wlan0` 和 `wlan1` |
| `old` | 修改前 `net_device->dev_addr` |
| `new` | 本次请求设置的 MAC |
| `mac` | 当前分支最终得到或观察到的 MAC |
| `perm` | `net_device->perm_addr` |
| `valid` | `is_valid_ether_addr()` 结果；全零、广播或组播地址会被判为无效 |
| `zero` | `is_zero_ether_addr()` 结果；为 `1` 时表示六字节全零 |
| `source` | mac80211 为接口选择永久地址时走到的来源分支 |
| `branch` | 运行期设置或异常处理所走到的具体分支 |
| `err` | 内核或驱动返回的负错误码 |
| `present` | `netif_device_present()` 结果 |
| `running` | 接口是否处于运行状态 |
| `live` | 是否在接口运行过程中执行 MAC 切换 |

## 4. `net/wireless/nl80211.c`

### 4.1 修改函数

```c
static int _nl80211_new_interface(...)
```

该函数处理 `NL80211_CMD_NEW_INTERFACE`，属于无线接口创建控制面。

### 4.2 接口名筛选

新增 `ifname` 和 `mac_diag`：

```c
ifname = nla_data(info->attrs[NL80211_ATTR_IFNAME]);
mac_diag = !strcmp(ifname, "wlan0") || !strcmp(ifname, "wlan1");
```

目的：

- 避免对 P2P、NAN、monitor 及其他网络接口持续打印；
- 明确区分 wlan0 和 wlan1 的创建请求；
- 复用已经解析的接口名，避免每处重复访问 netlink 属性。

性能特征：

- 每次创建接口只执行最多两次短字符串比较；
- 不在报文收发路径；
- 性能开销可忽略。

### 4.3 创建请求入口日志

```text
wlan-mac-diag: new-interface request name=... type=...
mac-attr=... mac-on-create=...
```

字段含义：

- `type`：请求创建的 `nl80211_iftype`；
- `mac-attr`：netlink 请求是否携带 `NL80211_ATTR_MAC`；
- `mac-on-create`：驱动是否声明
  `NL80211_FEATURE_MAC_ON_CREATE`。

诊断价值：

- 判断 wlan1 是由 nl80211 动态创建，还是由驱动内部直接创建；
- 判断创建时用户态是否已经提供 MAC；
- 判断 STA 创建时携带的 MAC 是否会被当前驱动能力接受。

### 4.4 携带 MAC 的分支

```text
wlan-mac-diag: new-interface name=... branch=mac-attr
value=... valid=... zero=...
```

仅当接口类型/驱动能力允许在创建时携带 MAC，并且请求确实包含
`NL80211_ATTR_MAC` 时输出。

诊断价值：

- 在进入具体驱动前记录用户态提供的原始 MAC；
- 直接判断是否在 nl80211 入口就收到全零地址；
- 区分“上层传入零 MAC”和“内核后续分配成零 MAC”。

### 4.5 非法 MAC 拒绝分支

```text
wlan-mac-diag: new-interface name=... branch=reject-invalid-mac
```

该日志对应：

```c
return -EADDRNOTAVAIL;
```

如果此前 `zero=1`，说明全零 MAC 在 nl80211 层被正确拒绝，并未进入驱动。

### 4.6 未携带 MAC 的分支

```text
wlan-mac-diag: new-interface name=... branch=no-mac-attr
```

表示创建请求没有提供可用 MAC，后续地址将由驱动、wiphy 地址池或
mac80211 地址掩码逻辑决定。

该日志是判断 `wlan1` 是否继承 `wiphy->perm_addr` 的重要前置证据。

### 4.7 驱动创建结果日志

新增三个结果分支：

```text
branch=null-wdev
branch=driver-error err=...
branch=driver-success dev=... perm=... zero=...
```

含义：

- `null-wdev`：驱动异常返回空指针；
- `driver-error`：驱动返回错误指针，`err` 为错误码；
- `driver-success`：驱动成功返回，并打印 netdevice 当前地址和永久地址。

如果 `driver-success` 中 `zero=1`，说明零地址已经在驱动创建返回时出现。

### 4.8 调用频率与性能结论

| 项目 | 评估 |
| --- | --- |
| 是否高频函数 | 否 |
| 典型触发 | Wi-Fi 打开、第二 STA 创建、接口重建 |
| 是否逐包调用 | 否 |
| 成功路径日志量 | 通常 3 条 |
| 性能风险 | 低 |

只有设备反复创建/销毁接口时才会重复执行。正常连接后不会因为业务流量而调用。

## 5. `net/mac80211/iface.c`

该文件增加两类维测：

1. 接口创建时永久 MAC 的来源；
2. 接口运行期修改 MAC 的处理结果。

### 5.1 `_ieee80211_change_mac()`：运行期修改 MAC

#### 5.1.1 入口日志

```text
wlan-mac-diag: change entry name=... old=... new=...
type=... running=... zero=...
```

诊断内容：

- mac80211 实际收到的旧地址和请求地址；
- 当前接口类型；
- 是否在运行状态修改；
- 请求地址是否为全零。

该日志与 `netif_set_mac_address()` 的入口日志形成上下层对照：

- 通用 netdevice 有入口、mac80211 无入口：请求可能在到达
  `ndo_set_mac_address()` 前被拒绝；
- 两层请求 MAC 不一致：需要检查驱动封装或参数转换；
- 两层都有 `new=00:00:00:00:00:00`：零地址来自更上游。

#### 5.1.2 运行状态拒绝分支

```text
branch=powered-change-rejected err=...
```

表示接口运行过程中不满足 MAC 动态切换条件，例如接口正在认证或关联。

此时 MAC 不会被修改。

#### 5.1.3 重复地址/合法性校验拒绝分支

```text
branch=verify-rejected check-dup=... err=...
```

对应 `ieee80211_verify_mac()` 返回失败。

诊断内容：

- MAC 是否与已有虚拟接口冲突；
- 是否启用了重复地址检查；
- 地址是否因 mac80211 约束被拒绝。

#### 5.1.4 完成日志

```text
branch=complete live=... err=... final=... zero=...
```

诊断内容：

- `eth_mac_addr()` 的最终返回值；
- 是否执行了运行期 remove/add interface；
- 最终 `net_device->dev_addr`；
- 修改完成后是否仍为全零。

如果请求地址有效、`err=0`，但 `final` 仍为全零，说明 mac80211/netdevice
状态更新存在异常。

#### 5.1.5 调用频率与性能结论

| 项目 | 评估 |
| --- | --- |
| 是否高频函数 | 否 |
| 典型触发 | 随机 MAC 切换、手工设置 MAC、部分重连流程 |
| 是否逐包调用 | 否 |
| 成功路径日志量 | 2 条 |
| 性能风险 | 低 |

HarmonyOS 上层连接前可能主动重新设置随机 MAC，因此该函数可能按
“连接尝试次数”调用，而不是只在开机时调用。即使如此，它仍属于连接控制面，
不随吞吐量、包速率或扫描结果数量增长。

### 5.2 `ieee80211_assign_perm_addr()`：接口初始 MAC 来源

#### 5.2.1 函数参数变化

函数新增 `const char *name` 参数，并修改两个调用点：

```c
ieee80211_assign_perm_addr(local, wdev->address, type, name);
ieee80211_assign_perm_addr(local, ndev->perm_addr, type, ndev->name);
```

目的：

- 只为 wlan0/wlan1 输出地址选择分支；
- 在日志中明确是哪个接口发生问题；
- 不改变原有地址选择算法和返回值。

#### 5.2.2 `source=default`

```text
wlan-mac-diag: assign name=... source=default
mac=... valid=... zero=... n=...
```

触发条件：

```c
is_zero_ether_addr(wiphy->addr_mask) &&
wiphy->n_addresses <= 1
```

代码行为：

```c
memcpy(perm_addr, wiphy->perm_addr, ETH_ALEN);
return;
```

这是本问题最关键的分支。

如果 wlan1 出现：

```text
source=default mac=00:00:00:00:00:00 valid=0 zero=1
```

可以确认：

- mac80211 没有生成零地址；
- 它只是继承了已经为零的 `wiphy->perm_addr`；
- 根因应继续向芯片驱动、固件、NV/OEM MAC 读取或
  `SET_IEEE80211_PERM_ADDR()` 调用点追查。

#### 5.2.3 `source=pool`

```text
source=pool index=... mac=... valid=... zero=...
```

表示从 `wiphy->addresses[]` 地址池中选择了尚未被其他接口占用的地址。

诊断价值：

- 判断驱动是否为双 STA 提供了多个硬件地址；
- 判断 wlan1 使用了地址池中的第几个地址；
- 检查地址池本身是否包含全零项。

#### 5.2.4 `source=no-mask`

```text
source=no-mask mac=... valid=... zero=...
```

表示地址池检查完成后没有可用于派生新地址的掩码，最终沿用当前
`perm_addr`。

该日志可能紧跟在 `source=pool` 后出现，因为原算法在检查地址池后还会检查
地址掩码。应以最后一条 `assign` 日志及后续 `if-add` 日志为最终结果。

#### 5.2.5 `source=bad-mask`

```text
source=bad-mask mask=... mac=... zero=...
```

表示 `wiphy->addr_mask` 不是连续掩码，mac80211 无法按当前算法进行派生，
最终退回已有地址。

这通常属于驱动能力注册错误。

#### 5.2.6 `source=mask`

```text
source=mask mac=... valid=... zero=...
```

表示 mac80211 根据地址掩码，从永久地址或已存在接口地址派生出一个未使用地址。

该日志用于确认 wlan1 是否由 wlan0/永久地址正常派生，以及派生结果是否有效。

#### 5.2.7 `ieee80211_if_add()` 最终选择日志

```text
wlan-mac-diag: if-add name=... source=...
perm=... mac=... valid=... zero=...
```

`source` 有两种：

- `requested-mac`：创建参数中携带了有效 MAC；
- `permanent-mac`：使用 `ieee80211_assign_perm_addr()` 的结果。

字段含义：

- `perm`：永久地址选择结果；
- `mac`：真正写入 `net_device->dev_addr` 的地址。

该日志解决“永久地址正常，但创建参数覆盖成异常地址”或“永久地址本身已为零”
两类问题的区分。

#### 5.2.8 调用频率与性能结论

| 项目 | 评估 |
| --- | --- |
| 是否高频函数 | 否 |
| 典型触发 | 创建 wlan0/wlan1、重新创建虚拟接口 |
| 是否逐包调用 | 否 |
| 地址选择日志量 | 通常 1～2 条 |
| `if-add` 最终日志量 | 1 条 |
| 性能风险 | 低 |

`ieee80211_assign_perm_addr()` 只在接口创建时执行。新增的 MAC 合法性和判零检查
均只读取固定的 6 字节地址。

## 6. `net/wireless/core.c`

### 6.1 修改函数

```c
int cfg80211_register_netdevice(struct net_device *dev)
```

该函数负责将无线 netdevice 注册到通用网络子系统，并注册对应
`wireless_dev`。

### 6.2 注册入口日志

```text
wlan-mac-diag: register name=... state=entry
mac=... perm=... zero=... assign=...
```

诊断内容：

- 调用 `register_netdevice()` 前的实际地址；
- 永久地址；
- 当前地址是否为全零；
- `addr_assign_type`，用于判断地址来自永久地址、随机地址还是用户设置。

该日志不仅覆盖 mac80211，也能覆盖调用
`cfg80211_register_netdevice()` 的 fullmac/vendor 驱动。

### 6.3 注册失败日志

```text
state=failed err=... mac=... zero=...
```

表示通用 netdevice 注册失败。

如果 `zero=1` 且 `err` 表示地址不可用，可以确认零 MAC 已影响到接口注册。

### 6.4 注册成功日志

```text
state=success type=... mac=... perm=... zero=...
```

用于确认接口完成注册后 MAC 是否发生变化。

典型判读：

- `entry zero=1`、`success zero=1`：接口携带零 MAC 完成注册；
- `entry zero=0`、`success zero=1`：注册过程或 notifier 改坏地址；
- `entry zero=0`、`success zero=0`：创建阶段正常，应继续观察运行期 set MAC。

### 6.5 调用频率与性能结论

| 项目 | 评估 |
| --- | --- |
| 是否高频函数 | 否 |
| 典型触发 | 无线 netdevice 注册或重新注册 |
| 是否逐包调用 | 否 |
| 成功路径日志量 | 2 条 |
| 性能风险 | 低 |

接口注册通常发生在开机、打开 Wi-Fi 或重新加载驱动时。

## 7. `net/core/dev.c`

### 7.1 修改函数

```c
int netif_set_mac_address(...)
```

该函数是通用 netdevice MAC 设置路径，负责调用具体驱动的
`ndo_set_mac_address()`。

虽然它属于网络核心代码，但它不是 TX/RX 数据面热路径。只有明确请求修改
某个接口 MAC 时才会执行。

### 7.2 接口名筛选开销

所有调用都会新增最多两次：

```c
strcmp(dev->name, "wlan0")
strcmp(dev->name, "wlan1")
```

即使其他类型网卡调用该函数，也只承担两次长度不超过 `IFNAMSIZ` 的字符串比较，
且 MAC 设置本身就是低频管理操作。

不会影响：

- 网络报文发送和接收；
- NAPI poll；
- 中断处理；
- TCP/IP 协议栈逐包处理；
- Wi-Fi 扫描结果处理。

### 7.3 设置入口日志

```text
wlan-mac-diag: set entry old=... new=...
present=... valid=... zero=...
```

这是判断上层是否真正向内核下发零 MAC 的关键证据。

如果 `new` 为全零且 `zero=1`，问题位于该函数之前：

- communication_wifi；
- HDI 参数转换；
- IChipIface 实现；
- ioctl/rtnetlink 组包。

### 7.4 无驱动回调分支

```text
branch=no-ndo
```

表示 `net_device_ops` 没有实现 `ndo_set_mac_address()`，返回
`-EOPNOTSUPP`。

### 7.5 地址族不匹配分支

```text
branch=family-mismatch family=... type=...
```

表示传入地址族与设备类型不一致，返回 `-EINVAL`。

### 7.6 设备不存在分支

```text
branch=device-not-present
```

表示接口已经被驱动标记为不在场，返回 `-ENODEV`。

该分支可用于识别 wlan1 创建/销毁与设置 MAC 并发的问题。

### 7.7 notifier 拒绝分支

```text
branch=notifier-rejected err=...
```

表示 `NETDEV_PRE_CHANGEADDR` notifier 拒绝本次修改。

此时驱动 `ndo_set_mac_address()` 尚未执行。

### 7.8 驱动拒绝分支

```text
branch=ndo-rejected err=... current=... zero=...
```

表示请求已经到达具体 Wi-Fi 驱动，但驱动返回失败。

如果：

- `new` 有效；
- `current` 为全零；
- `ndo-rejected`；

则 wlan1 可能从创建开始就是零地址，后续修正又被驱动拒绝。

### 7.9 地址未变化分支

```text
branch=unchanged value=...
```

表示请求地址与当前地址完全一致，因此内核不会再次调用驱动。

该日志可以排除“驱动未被调用是异常”的误判。

### 7.10 设置成功日志

```text
branch=success final=... valid=... zero=...
```

诊断内容：

- 驱动返回成功后的最终地址；
- 最终地址是否合法；
- 最终地址是否仍为全零。

关键异常组合：

```text
set entry new=<有效地址> valid=1 zero=0
set-mac branch=success final=00:00:00:00:00:00 valid=0 zero=1
```

这表示驱动或驱动封装返回成功，但没有正确更新 `net_device->dev_addr`。

### 7.11 调用频率与性能结论

| 项目 | 评估 |
| --- | --- |
| 是否高频函数 | 否 |
| 典型触发 | 随机 MAC 设置、用户修改 MAC、连接前刷新 MAC |
| 是否逐包调用 | 否 |
| 成功且地址变化 | 2 条通用层日志 |
| 地址未变化 | 3 条通用层日志 |
| 错误路径 | 通常 2 条通用层日志 |
| 性能风险 | 低 |

虽然该函数可能按连接尝试调用，但正常设备不会达到毫秒级或逐包级调用频率。

## 8. 整体日志量估算

### 8.1 一次成功创建 wlan1

使用 nl80211 + mac80211 的典型路径：

| 阶段 | 预计日志条数 |
| --- | ---: |
| nl80211 请求、MAC 来源、驱动返回 | 3 |
| mac80211 永久地址选择 | 1～2 |
| mac80211 `if-add` 最终地址 | 1 |
| cfg80211 注册入口和成功 | 2 |
| 合计 | 7～8 |

接口创建完成后不会持续打印。

### 8.2 一次成功修改 MAC

mac80211 典型路径：

| 阶段 | 预计日志条数 |
| --- | ---: |
| 通用 netdevice 入口和成功 | 2 |
| mac80211 入口和完成 | 2 |
| 合计 | 4 |

如果地址未发生变化，不会进入 mac80211 驱动回调，通用层通常输出 3 条。

### 8.3 异常重连场景

假设系统因异常每秒尝试连接一次，并且每次都重新设置随机 MAC：

- 预计约 4 条 MAC 设置日志/秒；
- 如果同时销毁和重建 wlan1，还可能增加约 7～8 条创建日志/秒。

该日志量仍不属于大规模日志洪泛，但在长期故障循环中会持续占用 kernel log
ring buffer，并挤出其他早期日志。

## 9. 性能影响评估

### 9.1 CPU 开销

新增操作主要包括：

- 最多两次接口名 `strcmp()`；
- 固定 6 字节的 MAC 判零和合法性检查；
- 命中 wlan0/wlan1 时执行 `%pM` 格式化和 printk。

正常场景下调用次数很低，CPU 开销预计不可感知。

### 9.2 网络吞吐和时延

所有修改均位于接口创建或 MAC 设置控制面，不在数据报文收发路径，因此：

- 不增加每包处理指令；
- 不影响 Wi-Fi TX/RX 吞吐；
- 不影响 NAPI、中断和软中断处理；
- 不会按流量大小增加日志。

正常用户使用下，不预期出现可感知的网络性能下降。

### 9.3 锁持有时间

这些函数通常在 RTNL、wiphy 或 netdevice 操作锁保护下执行。

`pr_info()`/`netdev_info()` 会延长少量锁持有时间。通常日志只写入内核 ring
buffer，影响很小；但以下配置可能放大影响：

- 内核 `console_loglevel` 允许 INFO 日志同步输出到串口；
- 串口波特率较低；
- 接口处于高频创建/销毁或重连循环；
- 日志采集系统同步读取或落盘较慢。

在极端串口同步输出环境下，单次接口创建或 MAC 设置可能增加额外毫秒级等待。
该影响取决于控制台和日志后端，不能仅凭代码给出固定数值。

### 9.4 内存影响

- 没有新增动态内存分配；
- 没有新增长期状态或缓存；
- 仅增加少量局部变量；
- 内存影响可忽略。

### 9.5 日志安全性

日志使用 `%pM` 输出完整 MAC，便于确认全零、覆盖和派生关系，但完整 MAC
属于设备标识信息。

如果该修改进入量产版本，需要结合产品隐私规范决定是否：

- 仅在工程版本启用；
- 对正常 MAC 做脱敏，仅保留全零判断；
- 通过动态调试开关控制；
- 问题定位结束后移除。

## 10. 综合风险结论

| 评估项 | 结论 |
| --- | --- |
| 数据面性能风险 | 极低 |
| 正常连接时延风险 | 低 |
| Wi-Fi 开关时延风险 | 低 |
| 异常重连循环日志量 | 低到中 |
| 慢速串口同步打印风险 | 中，取决于设备配置 |
| 内存风险 | 极低 |
| 用户可感知风险 | 正常场景下预计不可感知 |
| 长期量产保留建议 | 建议受调试开关控制或定位结束后移除 |

本次日志适合工程版本、灰度版本或短期问题定位。由于全部是 `pr_info()`/
`netdev_info()`，默认不可通过 dynamic debug 单独关闭。若需要长期进入量产，
建议后续改为以下方案之一：

1. 使用编译宏控制，例如仅在 Wi-Fi 维测版本启用；
2. 改为 `pr_debug()`/`netdev_dbg()` 并使用 dynamic debug；
3. 对可能重复的异常日志使用 ratelimited 版本；
4. 保留 `zero=1` 异常日志，将正常成功路径降级为 debug。

## 11. 推荐日志判读

### 11.1 创建时永久地址已经为零

```text
new-interface ... branch=no-mac-attr
assign name=wlan1 source=default mac=00:00:00:00:00:00 valid=0 zero=1
if-add name=wlan1 source=permanent-mac ... zero=1
register name=wlan1 state=entry ... zero=1
```

结论：

- 零 MAC 在 mac80211 地址分配前已经存在；
- 优先检查芯片驱动设置 `wiphy->perm_addr` 的路径；
- 检查固件、NV/OEM MAC 读取和驱动初始化时序。

### 11.2 上层下发了零 MAC

```text
set entry old=<正常或异常地址>
new=00:00:00:00:00:00 valid=0 zero=1
```

结论：

- 问题位于通用 netdevice 入口之前；
- 检查 communication_wifi、HDI 和 IChipIface 参数传递。

### 11.3 驱动拒绝有效 MAC

```text
set entry new=<有效地址> valid=1 zero=0
set-mac branch=ndo-rejected err=...
```

结论：

- 请求已经到达具体驱动；
- 检查驱动运行状态限制、固件命令返回值和接口 down/up 时序。

### 11.4 驱动返回成功但地址仍为零

```text
set entry new=<有效地址> valid=1 zero=0
set-mac branch=success final=00:00:00:00:00:00 valid=0 zero=1
```

结论：

- 驱动或封装层返回值与实际状态不一致；
- 检查 `ndo_set_mac_address()` 是否更新 `dev_addr`；
- 检查设置后是否有其他异步分支重新覆盖地址。

### 11.5 上层成功但没有任何内核设置日志

上层出现：

```text
SetStaMacAddress success
```

但内核没有：

```text
wlan-mac-diag: set entry
```

结论：

- `IChipIface::SetMacAddress()` 可能绕过通用 netdevice 设置路径；
- 需要获取 HDI/vendor Wi-Fi 驱动仓库；
- 在 IChipIface 实现、ioctl 命令入口和固件命令下发处继续增加日志。

## 12. 验证结果

当前修改已完成以下静态和编译验证：

```text
git diff --check
    通过

scripts/checkpatch.pl --no-tree --strict
    0 errors
    0 warnings
    0 checks

x86_64_defconfig 对象编译
    net/core/dev.o
    net/wireless/core.o
    net/wireless/nl80211.o
    net/mac80211/iface.o
    全部通过
```

对象编译用于验证：

- 新增函数参数和调用点一致；
- 日志格式参数类型正确；
- 所用网络/MAC 辅助函数声明可见；
- 修改没有引入编译告警。

本验证不等价于目标 HarmonyOS 设备上的运行时验证。最终仍需在目标芯片和目标
内核配置上复现，并结合 communication_wifi、HDI 和驱动日志对齐时间线。
