# `cfg80211_port_authorized()` 与 `__cfg80211_port_authorized()` 的关系分析

## 背景

在 `D:\下载\Code\linux\net\wireless\sme.c` 中有两个名字非常接近的函数：

```c
void __cfg80211_port_authorized(struct wireless_dev *wdev,
				const u8 *peer_addr,
				const u8 *td_bitmap,
				u8 td_bitmap_len)

void cfg80211_port_authorized(struct net_device *dev,
			      const u8 *peer_addr,
			      const u8 *td_bitmap,
			      u8 td_bitmap_len,
			      gfp_t gfp)
```

它们不是两个不同语义的 "port authorized" 事件，而是同一个事件处理链路中的两层入口：

- `cfg80211_port_authorized()` 是给驱动调用的公开 API。
- `__cfg80211_port_authorized()` 是 cfg80211 内部在已持锁上下文中实际发送 nl80211 事件的实现。

## 总体调用链

这两个函数不是直接的普通 wrapper 关系。`cfg80211_port_authorized()` 并不直接调用 `__cfg80211_port_authorized()`，而是通过事件队列和 workqueue 间接连接：

```text
driver
  -> cfg80211_port_authorized()
       -> 构造 EVENT_PORT_AUTHORIZED
       -> 挂到 wdev->event_list
       -> queue_work(cfg80211_wq, &rdev->event_work)

cfg80211 worker later
  -> cfg80211_event_work()
       -> guard(wiphy)(&rdev->wiphy)
       -> cfg80211_process_rdev_events()
            -> cfg80211_process_wdev_events()
                 -> switch EVENT_PORT_AUTHORIZED
                      -> __cfg80211_port_authorized()
                           -> nl80211_send_port_authorized()
```

## 公开入口：`cfg80211_port_authorized()`

位置：

```text
D:\下载\Code\linux\net\wireless\sme.c:1310
```

这个函数是导出的符号：

```c
EXPORT_SYMBOL(cfg80211_port_authorized);
```

它也在公开头文件中声明：

```text
D:\下载\Code\linux\include\net\cfg80211.h:9055
```

文档说明它用于通知 cfg80211：安全关联已经成功建立，例如支持 4-way handshake offload 的驱动在握手成功后调用它。

典型驱动调用示例：

```text
D:\下载\Code\linux\drivers\net\wireless\broadcom\brcm80211\brcmfmac\cfg80211.c:6468
```

调用形式类似：

```c
cfg80211_port_authorized(ndev, profile->bssid, NULL, 0, GFP_KERNEL);
```

这个公开函数做的核心事情是构造一个事件：

```c
ev->type = EVENT_PORT_AUTHORIZED;
memcpy(ev->pa.peer_addr, peer_addr, ETH_ALEN);
ev->pa.td_bitmap = ((u8 *)ev) + sizeof(*ev);
ev->pa.td_bitmap_len = td_bitmap_len;
memcpy((void *)ev->pa.td_bitmap, td_bitmap, td_bitmap_len);
```

然后把事件挂到 `wdev->event_list`：

```c
spin_lock_irqsave(&wdev->event_lock, flags);
list_add_tail(&ev->list, &wdev->event_list);
spin_unlock_irqrestore(&wdev->event_lock, flags);
queue_work(cfg80211_wq, &rdev->event_work);
```

也就是说，`cfg80211_port_authorized()` 的角色是：

```text
把驱动报告的 "port authorized" 事实封装成 cfg80211 event，然后异步投递给 cfg80211 worker。
```

## 事件结构：`EVENT_PORT_AUTHORIZED`

事件类型定义在：

```text
D:\下载\Code\linux\net\wireless\core.h:260
```

相关部分：

```c
enum cfg80211_event_type {
	EVENT_CONNECT_RESULT,
	EVENT_ROAMED,
	EVENT_DISCONNECTED,
	EVENT_IBSS_JOINED,
	EVENT_STOPPED,
	EVENT_PORT_AUTHORIZED,
};
```

事件 payload 中 `pa` 字段用于保存 port authorized 事件的数据：

```c
struct {
	u8 peer_addr[ETH_ALEN];
	const u8 *td_bitmap;
	u8 td_bitmap_len;
} pa;
```

因此，`cfg80211_port_authorized()` 和 `__cfg80211_port_authorized()` 之间真正的连接媒介是：

```text
struct cfg80211_event + EVENT_PORT_AUTHORIZED + wdev->event_list
```

## Worker 入口：`cfg80211_event_work()`

worker 初始化位置：

```text
D:\下载\Code\linux\net\wireless\core.c:649
```

```c
INIT_WORK(&rdev->event_work, cfg80211_event_work);
```

worker 函数位置：

```text
D:\下载\Code\linux\net\wireless\core.c:384
```

关键逻辑：

```c
static void cfg80211_event_work(struct work_struct *work)
{
	struct cfg80211_registered_device *rdev;

	rdev = container_of(work, struct cfg80211_registered_device,
			    event_work);

	guard(wiphy)(&rdev->wiphy);

	cfg80211_process_rdev_events(rdev);
}
```

这里的关键点是：

```c
guard(wiphy)(&rdev->wiphy);
```

它会在处理事件前拿住 wiphy 锁。后续进入 `cfg80211_process_rdev_events()` 时，代码也明确断言当前已经持有这个锁：

```c
lockdep_assert_held(&rdev->wiphy.mtx);
```

## 事件消费：`cfg80211_process_wdev_events()`

位置：

```text
D:\下载\Code\linux\net\wireless\util.c:1115
```

这个函数从 `wdev->event_list` 中取出事件，然后根据事件类型分发：

```c
switch (ev->type) {
case EVENT_CONNECT_RESULT:
	__cfg80211_connect_result(...);
	break;
case EVENT_ROAMED:
	__cfg80211_roamed(...);
	break;
case EVENT_DISCONNECTED:
	__cfg80211_disconnected(...);
	break;
case EVENT_PORT_AUTHORIZED:
	__cfg80211_port_authorized(wdev, ev->pa.peer_addr,
				   ev->pa.td_bitmap,
				   ev->pa.td_bitmap_len);
	break;
}
```

因此，`__cfg80211_port_authorized()` 是在 worker 消费事件时被调用的，而不是由驱动路径同步直接调用。

## 内部实现：`__cfg80211_port_authorized()`

位置：

```text
D:\下载\Code\linux\net\wireless\sme.c:1288
```

它开头有一个重要断言：

```c
lockdep_assert_wiphy(wdev->wiphy);
```

这表示该函数的调用者必须已经持有 wiphy 锁。

随后它会检查接口类型：

```c
if (WARN_ON(wdev->iftype != NL80211_IFTYPE_STATION &&
	wdev->iftype != NL80211_IFTYPE_P2P_CLIENT &&
	wdev->iftype != NL80211_IFTYPE_AP &&
	wdev->iftype != NL80211_IFTYPE_P2P_GO))
	return;
```

对于 station / P2P client，还会检查当前确实已经 connected，并且传入的 `peer_addr` 与当前连接地址一致：

```c
if (wdev->iftype == NL80211_IFTYPE_STATION ||
	wdev->iftype == NL80211_IFTYPE_P2P_CLIENT) {
	if (WARN_ON(!wdev->connected) ||
		WARN_ON(!ether_addr_equal(wdev->u.client.connected_addr, peer_addr)))
		return;
}
```

最后真正发送 nl80211 事件：

```c
nl80211_send_port_authorized(wiphy_to_rdev(wdev->wiphy), wdev->netdev,
			     peer_addr, td_bitmap, td_bitmap_len);
```

## 为什么要分成两个函数

### 1. 锁边界清楚

`cfg80211_port_authorized()` 是公开 API，驱动可以调用它。驱动不需要知道 cfg80211 内部何时应该持有 `wiphy->mtx`。

`__cfg80211_port_authorized()` 是内部函数，它要求调用者已经持有 wiphy 锁。

如果公开 API 直接做内部处理，就会把 cfg80211 的锁语义暴露给驱动，调用约束会更复杂，也更容易出错。

### 2. 保证事件顺序

`cfg80211_port_authorized()` 中有一段注释：

```c
/*
 * Use the wdev event list so that if there are pending
 * connected/roamed events, they will be reported first.
 */
```

这说明它故意把 port authorized 事件放进 `wdev->event_list`，这样如果前面还有 pending 的 `connected` 或 `roamed` 事件，就会先上报连接/漫游，再上报 port authorized。

这符合 userspace 对事件顺序的理解：

```text
先知道已经连接/漫游，再知道安全端口已经 authorized。
```

### 3. 统一异步事件处理

`connect_result`、`roamed`、`disconnected`、`ibss_joined`、`port_authorized` 都走同一套 event list 分发机制。

公开函数负责把外部事件记录下来，worker 在统一的 cfg80211 上下文中串行处理这些事件。

## 如何理解这种写法

这种写法可以理解成内核中常见的两层 API：

```text
foo()   : 外部可调用版本，负责参数复制、排队、上下文切换、异步化
__foo() : 内部已持锁版本，负责真正执行核心逻辑
```

在这个例子中：

```text
cfg80211_port_authorized()
  = 驱动可调用的异步入口

__cfg80211_port_authorized()
  = cfg80211 内部、已持 wiphy 锁的实际处理函数
```

它们之间的桥不是普通函数调用，而是：

```text
EVENT_PORT_AUTHORIZED event
```

因此最准确的理解是：

```text
cfg80211_port_authorized()
  -> enqueue EVENT_PORT_AUTHORIZED

later, under wiphy lock:
  EVENT_PORT_AUTHORIZED
    -> __cfg80211_port_authorized()
```

## 一句话总结

`cfg80211_port_authorized()` 是给驱动调用的公开异步 API；它把事件排入 `wdev->event_list`。cfg80211 的 worker 随后在持有 wiphy 锁的上下文中消费该事件，并调用 `__cfg80211_port_authorized()` 完成状态校验和 nl80211 事件发送。
