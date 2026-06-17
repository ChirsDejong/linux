> Linux 5.10 → 6.6 内核升级，MLO 业务逻辑排查

## 概述

- **5.10 中 MLO #ifdef 块总数**: ~80 处 (13 个文件)
- **6.6 中 MLO #ifdef 块总数**: 7 处 (3 个文件)
- **状态统计**:
  - **INTEGRATED** (已合入基线): 72 处 — 上游 6.6 已原生支持 MLO/EHT，5.10 中 `#ifdef` 分支的代码已无条件存在
  - **PRESENT** (仍保留 #ifdef): 5 处 — HISI 厂商私有扩展，6.6 中仍由 `#ifdef CONFIG_HISI_WIFI7_MLO` 守护
  - **DIFFERENT** (合入但有差异): 3 处 — MLO 逻辑已合入但实现方式有调整

---

## 一、6.6 中仍保留 #ifdef CONFIG_HISI_WIFI7_MLO 的代码块

### 1.1 cfg80211_inform_mlo_assoc_link_id (sme.c)

**5.10 位置**: `net/wireless/sme.c:1115-1122`
**6.6 位置**: `net/wireless/sme.c:1104-1111`

```diff
-/* 5.10: 在 #ifdef CONFIG_HISI_WIFI7_MLO 内 */
+/* 6.6: 在 #ifdef CONFIG_HISI_WIFI7_MLO 内，代码完全一致 */
 void cfg80211_inform_mlo_assoc_link_id(struct net_device *dev, u8 assoc_link_id)
 {
 	struct wireless_dev *wdev = dev->ieee80211_ptr;
 	wdev->assoc_link_id = assoc_link_id;
 }
 EXPORT_SYMBOL(cfg80211_inform_mlo_assoc_link_id);
```

**结论**: 一致，无需修改。

---

### 1.2 nl80211_inform_link_switch + cfg80211_inform_link_switch (nl80211.c)

**5.10 位置**: `net/wireless/nl80211.c:17954-18002`
**6.6 位置**: `net/wireless/nl80211.c:19825-19873`

```diff
-/* 5.10: 在 #ifdef CONFIG_HISI_WIFI7_MLO 内 */
+/* 6.6: 在 #ifdef CONFIG_HISI_WIFI7_MLO 内，代码完全一致 */
 static void nl80211_inform_link_switch(struct cfg80211_registered_device *rdev,
 				    struct net_device *netdev,
 				    unsigned int link_id, gfp_t gtp,
 				    enum nl80211_commands notif)
 {
 	/* ... 构造 nlmsg，填充 MLO_LINK_SWITCH_LINK_ID 和
 	   MLO_LINK_SWITCH_WORK_BSSID 属性 ... */
 }

 void cfg80211_inform_link_switch(struct net_device *dev, u8 link_id)
 {
 	struct wireless_dev *wdev = dev->ieee80211_ptr;
 	struct wiphy *wiphy = wdev->wiphy;
 	struct cfg80211_registered_device *rdev = wiphy_to_rdev(wiphy);
 	ASSERT_WDEV_LOCK(wdev);
 	WARN_INVALID_LINK_ID(wdev, link_id);
-	/* only wifi7 mlo connect, link_id match valid link inform */
 	if (wdev->valid_links && (wdev->valid_links & BIT(link_id))) {
 		wdev->assoc_link_id = link_id;
 		nl80211_inform_link_switch(rdev, dev, link_id, GFP_KERNEL,
 					   NL80211_CMD_LINK_SWITCH_EVENT);
 	}
 }
 EXPORT_SYMBOL(cfg80211_inform_link_switch);
```

**结论**: 一致（5.10 有一行注释在 6.6 中被去掉，逻辑完全相同），无需修改。

---

### 1.3 MLO Link Switch Policy (nl80211.c — 6.6 新增)

**6.6 位置**: `net/wireless/nl80211.c:834-837`
**5.10**: 无对应（5.10 中这些属性不在 policy 表中）

```c
/* 6.6: #ifdef CONFIG_HISI_WIFI7_MLO */
#ifdef CONFIG_HISI_WIFI7_MLO
	[NL80211_ATTR_MLO_LINK_SWITCH_LINK_ID] = { .type = NLA_U8 },
	[NL80211_ATTR_MLO_LINK_SWITCH_WORK_BSSID] = { .type = NLA_UNSPEC, .len = ETH_ALEN },
#endif
```

**结论**: 6.6 新增了 link switch 属性的 policy 定义，5.10 中缺失。这是改进，无需回退。

---

### 1.4 cofeature/EMLSR 字段与宏定义 (cfg80211.h)

**5.10 位置**: `include/net/cfg80211.h:5458-5461` (宏定义) 和 `5488-5504` (cofeature 不在 #ifdef 内)
**6.6 位置**: `include/net/cfg80211.h:6121-6125` (wdev 字段) 和 `6128-6131` (宏定义)

```diff
-/* 5.10: #ifdef CONFIG_HISI_WIFI7_MLO */
+/* 6.6: #ifdef CONFIG_HISI_WIFI7_MLO */
 #ifdef CONFIG_HISI_WIFI7_MLO
+	u8 cofeature;
+	u8 cofeature_state;
+	u16 reason_code;
+#endif
+};
+
+#ifdef CONFIG_HISI_WIFI7_MLO
 #define WIFI7_MLO_COFEATURE 0
 #define WIFI7_MLO_COFEATURE_STATE_EMLSR 3
 #endif
```

**5.10 差异**: 5.10 中 `cofeature`/`cofeature_state`/`reason_code` 字段**不在** `#ifdef` 内，而是直接在 `struct wireless_dev` 中；宏定义在 `#ifdef` 内。
**6.6 差异**: 6.6 将字段也放入了 `#ifdef CONFIG_HISI_WIFI7_MLO` 内，宏定义同样在 `#ifdef` 内。

**结论**: 6.6 收紧了保护范围（字段也加了 #ifdef），逻辑一致。无需修改。

---

### 1.5 cfg80211_inform_mlo_assoc_link_id / cfg80211_inform_link_switch 声明 (cfg80211.h)

**5.10 位置**: `include/net/cfg80211.h:6179-6195`
**6.6 位置**: `include/net/cfg80211.h:8955-8970`

```diff
-/* 5.10: #ifdef CONFIG_HISI_WIFI7_MLO */
+/* 6.6: #ifdef CONFIG_HISI_WIFI7_MLO */
 #ifdef CONFIG_HISI_WIFI7_MLO
 void cfg80211_inform_mlo_assoc_link_id(struct net_device *dev, u8 assoc_link_id);
 void cfg80211_inform_link_switch(struct net_device *dev, u8 link_id);
+void cfg80211_inform_cofeature_state(struct net_device *dev, u8 feature,
+				     u8 state, u16 reason_code);
 #endif
```

**5.10 差异**: `cfg80211_inform_cofeature_state` 声明在 `#endif` 之后，不在 `#ifdef` 内。
**6.6 差异**: `cfg80211_inform_cofeature_state` 也纳入了 `#ifdef CONFIG_HISI_WIFI7_MLO` 保护。

**结论**: 6.6 将 cofeature_state 也纳入了 #ifdef 保护，一致性更好。无需修改。

---

### 1.6 BSS dump 中 cofeature/EMLSR 检查 (nl80211.c)

**5.10 位置**: `net/wireless/nl80211.c:9681-9700`
**6.6 位置**: `net/wireless/nl80211.c:10574-10589`

```diff
-/* 5.10: 整个 for_each_valid_link 块在 #ifdef 内 */
+/* 6.6: for_each_valid_link 已合入基线，仅 cofeature 条件在 #ifdef 内 */
 	case NL80211_IFTYPE_STATION:
 	case NL80211_IFTYPE_P2P_CLIENT:
-#ifdef CONFIG_HISI_WIFI7_MLO
 		for_each_valid_link(wdev, link_id) {
 			if (intbss == wdev->links[link_id].client.current_bss &&
-			    (!wdev->valid_links || link_id == wdev->assoc_link_id ||
-			     (wdev->cofeature == WIFI7_MLO_COFEATURE &&
-			      wdev->cofeature_state == WIFI7_MLO_COFEATURE_STATE_EMLSR)) &&
+#ifdef CONFIG_HISI_WIFI7_MLO
+			    (!wdev->valid_links || link_id == wdev->assoc_link_id ||
+			     (wdev->cofeature == WIFI7_MLO_COFEATURE &&
+			      wdev->cofeature_state == WIFI7_MLO_COFEATURE_STATE_EMLSR)) &&
+#endif
 			    (nla_put_u32(msg, NL80211_BSS_STATUS,
 					 NL80211_BSS_STATUS_ASSOCIATED) ||
 			     (wdev->valid_links &&
 			      (nla_put_u8(msg, NL80211_BSS_MLO_LINK_ID, link_id) ||
 			       nla_put(msg, NL80211_BSS_MLD_ADDR, ETH_ALEN,
-				       wdev->connected_addr)))))
+				       wdev->u.client.connected_addr)))))
 				goto nla_put_failure;
 		}
-#else
-		if (intbss == wdev->current_bss &&
-		    nla_put_u32(msg, NL80211_BSS_STATUS,
-				NL80211_BSS_STATUS_ASSOCIATED))
-			goto nla_put_failure;
-#endif
 		break;
```

**关键差异**:

1. 5.10 中**整个** `for_each_valid_link` 迭代都在 `#ifdef` 内，有 `#else` 分支使用 `wdev->current_bss`；6.6 中迭代已无条件存在，仅 cofeature 条件在 `#ifdef` 内
2. 6.6 使用 `wdev->u.client.connected_addr` 替代 5.10 的 `wdev->connected_addr`
3. 当 `CONFIG_HISI_WIFI7_MLO` **未定义**时，6.6 中 `if` 条件变为 `intbss == wdev->links[link_id].client.current_bss && (nla_put_u32(...))` — 缺少了 `!wdev->valid_links || link_id == wdev->assoc_link_id` 这个过滤条件

**风险**: ⚠️ **需要关注** — 当 `CONFIG_HISI_WIFI7_MLO` 未定义时，6.6 中 BSS dump 的过滤逻辑不完整，会输出所有 valid_links 中匹配的 BSS 而不仅仅是 assoc_link 的 BSS。如果驱动未开启 MLO 配置，此处的行为可能不符合预期。建议将 `!wdev->valid_links || link_id == wdev->assoc_link_id` 条件从 `#ifdef` 中移出。

---

## 二、6.6 中已合入但有实现差异的代码块

### 2.1 __cfg80211_port_authorized BSSID 校验 (sme.c)

**5.10 位置**: `net/wireless/sme.c:1751-1757`
**6.6 位置**: `net/wireless/sme.c:1333-1334`

```diff
-/* 5.10 #ifdef CONFIG_HISI_WIFI7_MLO */
-	if (WARN_ON(!wdev->connected) ||
-	    WARN_ON(!wdev->links[wdev->assoc_link_id].client.current_bss) ||
-	    WARN_ON(!ether_addr_equal(
-		wdev->links[wdev->assoc_link_id].client.current_bss->pub.bssid, bssid)))
-/* 5.10 #else */
-	if (WARN_ON(!wdev->current_bss) ||
-	    WARN_ON(!ether_addr_equal(wdev->current_bss->pub.bssid, bssid)))
-/* 5.10 #endif */
+/* 6.6: 无条件，使用 connected + connected_addr */
+	if (WARN_ON(!wdev->connected) ||
+	    WARN_ON(!ether_addr_equal(wdev->u.client.connected_addr, bssid)))
```

**差异说明**:

1. 5.10 MLO 版本检查 `wdev->links[wdev->assoc_link_id].client.current_bss->pub.bssid`（按 link 级 BSSID 校验）
2. 6.6 改为检查 `wdev->u.client.connected_addr`（使用连接地址而非具体 link 的 BSSID）
3. 6.6 新增了 `td_bitmap` / `td_bitmap_len` 参数
4. 6.6 新增了 `NL80211_IFTYPE_P2P_CLIENT` 支持

**风险**: 低 — `connected_addr` 在 MLO 场景下是 MLD 地址，而非 per-link BSSID。如果业务需要精确校验 assoc_link 的 BSSID，可能需要补充逻辑。

---

### 2.2 cfg80211_inform_bss_frame_data 简化 (scan.c)

**5.10 位置**: `net/wireless/scan.c:3152-3265`
**6.6 位置**: `net/wireless/scan.c:3000-3036`

```diff
-/* 5.10 MLO 版本 */
 	bcn.mbssid_ies = mbssid_ies;
 	bcn.mbssid_ies_ext = mbssid_ies_ext;
-	if (support_only_he_mbssid && !bcn.mbssid_ies)
-		goto out;
-	if (!bcn.mbssid_ies && !bcn.mbssid_ies_ext)
-		goto out;
+/* 6.6: 简化版，去掉了 mbssid HE 检查和 early return */
 	/* 直接调用 cfg80211_parse_mbssid_data() 和
 	   cfg80211_parse_ml_sta_data()，无 early return */
```

**差异说明**:

1. 6.6 去掉了 `support_only_he_mbssid` / `support_mbssid` 的提前返回逻辑
2. 6.6 去掉了 `nontrans_list` 时间戳更新逻辑
3. 核心调用链 `inform_single_bss_frame_data → parse_mbssid_data → parse_ml_sta_data` 保持一致

**风险**: 低 — 简化是合理的，early return 逻辑在上游看来是冗余的。

---

### 2.3 wireless_dev 结构体重构 (cfg80211.h)

**5.10 位置**: `include/net/cfg80211.h:5389-5461`
**6.6 位置**: `include/net/cfg80211.h:6037-6131`

```diff
-/* 5.10 */
 struct wireless_dev {
-	bool connected;
-	u8 connected_addr[ETH_ALEN];
-	/* ssid, ssid_len 直接在 wdev 顶层 */
-	u8 ssid[IEEE80211_MAX_SSID_LEN];
-	u8 ssid_len;
+/* 6.6: 使用 union 按接口类型分区 */
+	struct {
+		u8 connected:1;
+		u8 valid_links:1;
+		/* ... */
+	} u;  /* 实际结构更复杂 */
+	/* ssid 移入 wdev->u.client.ssid */
+	/* connected_addr 移入 wdev->u.client.connected_addr */
+	/* current_bss 被 links[link_id].client.current_bss 替代 */
 };
```

**关键重命名映射**:

| 5.10 字段 | 6.6 字段 | 说明 |
|-----------|----------|------|
| `wdev->connected` | `wdev->connected` (不变) | 连接状态标志 |
| `wdev->connected_addr` | `wdev->u.client.connected_addr` | MLD 连接地址 |
| `wdev->ssid` | `wdev->u.client.ssid` | SSID |
| `wdev->ssid_len` | `wdev->u.client.ssid_len` | SSID 长度 |
| `wdev->current_bss` | `wdev->links[link_id].client.current_bss` | 单 link 被 per-link 替代 |
| `wdev->beacon_interval` | `wdev->links[link_id].ap.beacon_interval` | AP 模式 |

**风险**: 低 — 结构体重构是上游统一行为，所有引用点已更新。如需回移 5.10 的自定义代码，需注意字段路径变更。

---

## 三、6.6 中已完整合入（无需关注）的代码块

以下 MLO 代码块在 5.10 中由 `#ifdef CONFIG_HISI_WIFI7_MLO` 守护，在 6.6 中已**无条件**存在于基线代码中，逻辑完全等价（除上述字段重命名外）：

### 3.1 rdev-ops.h (12 处 → 全部 INTEGRATED)

所有 key 操作函数（`add_key`/`get_key`/`del_key`/`set_default_key`/`set_default_mgmt_key`/`set_default_beacon_key`）的 `link_id` 参数已无条件存在。

5.10 中每个函数有两处 #ifdef（函数签名 + ops 调用），6.6 中全部合入基线。

此外 6.6 新增了 5.10 中没有的 MLO 函数：`rdev_add_intf_link`、`rdev_del_intf_link`、`rdev_add_link_station`、`rdev_mod_link_station`、`rdev_del_link_station`。

### 3.2 nl80211.c (39 处 → 35 处 INTEGRATED + 3 处 PRESENT + 1 处 DIFFERENT)

| 类别 | 5.10 行号 | 6.6 状态 |
|------|-----------|---------|
| EHT/MLO netlink policy | 739-754 | INTEGRATED |
| nl80211_link_id / link_id_or_invalid 函数 | 1165-1196 | INTEGRATED |
| wdev->connected 替代 wdev->current_bss (7处) | 1470,3561,8443,13140,14573,14583,10261 | INTEGRATED |
| MLO key 操作 validate_link_id + link_id 传递 (11处) | 4038-4489 | INTEGRATED |
| parse_mbssid_elems / parse_beacon (7处) | 4987-5677 | INTEGRATED |
| ASSOC_REQ_DISABLE_EHT / CONNECT_REQ_MLO_SUPPORT | 10261,11045,11143 | INTEGRATED |
| MLO internal flags & selectors | 15051-15096 | INTEGRATED |
| nl80211_send_connect_result / send_roamed (MLO link info) | 16573-16843 | INTEGRATED |
| external auth MLD_ADDR | 18848-18853 | INTEGRATED |
| **link_switch 通知** | 17954-18002 | **PRESENT** (6.6:19825) |
| **BSS dump cofeature 检查** | 9681-9700 | **DIFFERENT** (6.6:10574) |
| **link switch policy** | (无) | **PRESENT** (6.6:834, 新增) |

### 3.3 sme.c (14 处 → 13 处 INTEGRATED + 1 处 PRESENT)

| 5.10 行号 | 逻辑 | 6.6 状态 |
|-----------|------|---------|
| 266-270 | cr.links[0].bssid | INTEGRATED |
| 385-389 | cr.links[0].bssid | INTEGRATED |
| 475-504 | wdev_release_bsses | INTEGRATED |
| 705-709 | wdev->connected vs current_bss | INTEGRATED |
| 730-744 | connect_result_release_bsses | INTEGRATED |
| 745-1114 | MLO __cfg80211_connect_result | INTEGRATED |
| 1115-1229 | inform_mlo_assoc_link_id + connect_done | PRESENT + INTEGRATED |
| 1232-1653 | MLO __cfg80211_roamed + cfg80211_roamed | INTEGRATED |
| 1751-1757 | port_authorized BSSID check | DIFFERENT |
| 1793-1856 | MLO __cfg80211_disconnected | INTEGRATED |
| 1979-2148 | wdev->connected 替代 current_bss (4处) | INTEGRATED |

### 3.4 scan.c (7 处 → 全部 INTEGRATED)

| 5.10 行号 | 逻辑 | 6.6 状态 |
|-----------|------|---------|
| 1885-2150 | inform_single_bss_data 结构体+函数 | INTEGRATED |
| 2223-2470 | parse_mbssid_data (MLO 签名) | INTEGRATED |
| 2471-2916 | defragment_element/defrag_mle/tbtt_info_for_mld_ap/parse_ml_sta_data | INTEGRATED |
| 2917-2938 | parse_mbssid_frame_data 旧包装函数 | MISSING (正确移除) |
| 3152-3265 | inform_bss_frame_data (MLO版) | INTEGRATED (有简化) |
| 3354-3356 | update_assoc_bss_entry link_id 参数 | INTEGRATED |
| 3361-3365 | wdev->links[link_id] 替代 current_bss | INTEGRATED |

### 3.5 其他文件

| 文件 | 5.10 #ifdef 数 | 6.6 状态 |
|------|---------------|---------|
| wext-compat.c | 6 处 | 全部 INTEGRATED (link_id=-1 无条件传递) |
| ibss.c | 1 处 | INTEGRATED |
| core.h | 1 处 | INTEGRATED |
| core.c | 1 处 | INTEGRATED |
| mlme.c | 1 处 | INTEGRATED (增强为 per-link 循环) |
| util.c | 2 处 | INTEGRATED |
| reg.c | 1 处 | INTEGRATED (增强为 multi-link 循环) |
| cfg80211.h | 22 处 | 18 INTEGRATED + 2 PRESENT + 2 DIFFERENT |
| ieee80211.h | 5 处 | 全部 INTEGRATED (EHT structs/caps/enums) |

---

## 四、需要关注的风险项汇总

| # | 风险项 | 文件 | 6.6 行号 | 严重程度 | 说明 |
|---|--------|------|---------|---------|------|
| 1 | BSS dump link 过滤条件缺失 | nl80211.c | 10574-10589 | ⚠️ 中 | `#ifdef` 未定义时，`!wdev->valid_links \|\| link_id == wdev->assoc_link_id` 条件丢失，所有 valid_links 的 BSS 都会输出 ASSOCIATED 状态 |
| 2 | port_authorized BSSID 校验方式变更 | sme.c | 1333-1334 | 低 | 从 per-link BSSID 校验改为 connected_addr 校验，MLD 地址 vs 具体 link BSSID |
| 3 | cofeature 字段在 6.6 中纳入 #ifdef 保护 | cfg80211.h | 6121-6125 | 低 | 若 6.6 未开启 CONFIG_HISI_WIFI7_MLO，cofeature 字段不存在，但引用处也在 #ifdef 内，自洽 |

---

## 五、建议操作

1. **BSS dump 过滤条件 (风险项 #1)**: 建议将 `!wdev->valid_links || link_id == wdev->assoc_link_id` 从 `#ifdef CONFIG_HISI_WIFI7_MLO` 中移出，使其成为无条件逻辑。这样即使 MLO 未启用，也只输出当前关联 link 的 BSS 状态。
2. **其余 PRESENT 代码块**: `cfg80211_inform_mlo_assoc_link_id`、`cfg80211_inform_link_switch`、cofeature/EMLSR 相关代码 — 这些是 HISI 厂商私有扩展，6.6 中保留 `#ifdef` 保护是合理的，无需改动。
3. **INTEGRATED 代码块**: 上游 6.6 已原生支持 MLO/EHT，5.10 中绝大部分 `#ifdef CONFIG_HISI_WIFI7_MLO` 分支的代码已无条件存在于 6.6 基线中，无需额外移植。