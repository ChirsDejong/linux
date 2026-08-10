# `synchronize_rcu()` 定义分析

## 背景

仓库里有 5 处 `synchronize_rcu` 的"定义"，取决于编译目标：

- `tools/include/linux/kernel.h:95` — 用户态工具编译用的空宏桩（无操作）
- `tools/virtio/linux/kernel.h:149` — virtio 用户态桩（`assert(0)`，调用即 abort）
- `include/linux/rcupdate.h:53` — 内核统一声明（函数原型）
- `kernel/rcu/tree.c:3349` — TREE RCU 主实现（SMP / 通用内核）
- `kernel/rcu/tiny.c:141` — TINY RCU 实现（单核非抢占内核）

真实内核实现由配置二选一：`CONFIG_TREE_RCU` 走 `tree.c`，否则走 `tiny.c`。

## 语义契约（来自 tree.c 函数头注释）

`tree.c:3311-3348` 定义了完整契约：

- 返回时，调用前已开始的**所有 RCU 读端临界区**都已完成。
- 但返回后可能与**返回前新开始的临界区**并发运行（等待一个宽限期，非全同步）。
- 读端临界区由 `rcu_read_lock()` / `rcu_read_unlock()` 界定，可嵌套；v5.0 起关中断/关抢占/关软中断的代码段也算临界区（含硬件中断、软中断、NMI 处理器）。
- 内存屏障保证：返回时每个 CPU 都执行过完整内存屏障（含离线、空闲、用户态 CPU）；调用方 A 与被唤醒方 B 在调用期间都执行过内存屏障。
- 实现细节参考 `Documentation/RCU/Design/Memory-Ordering/Tree-RCU-Memory-Ordering.rst`。

## 用户态空宏桩：`tools/include/linux/kernel.h:95`

```c
#define current_gfp_context(k) 0
#define synchronize_rcu()
```

编译 tools/ 用户态工具（perf、testing/nvdimm 等）时使用的内核头文件副本。`synchronize_rcu()` 展开为空操作。

原因：工具链复用了内核的链表/RCU 头文件（如 `rculist.h`），其中会调用 `synchronize_rcu()`；用户态没有真正的 RCU 机制，所以编译成 no-op。

对比 `tools/virtio/linux/kernel.h:149` 的 virtio 桩则严格得多：

```c
static inline void synchronize_rcu(void)
{
	assert(0);
}
```

调用即 abort —— virtio 测试环境不允许出现未实现的 RCU 语义。

## 内核统一声明：`include/linux/rcupdate.h:53`

```c
/* Exported common interfaces */
void call_rcu(struct rcu_head *head, rcu_callback_t func);
void rcu_barrier_tasks(void);
void synchronize_rcu(void);
```

所有内核调用方看到的原型。具体实现由配置编译期分派。

## 主实现：`kernel/rcu/tree.c:3349`（TREE RCU）

真实系统绝大多数情况走到的实现。函数体：

```c
void synchronize_rcu(void)
{
	unsigned long flags;
	struct rcu_node *rnp;

	RCU_LOCKDEP_WARN(lock_is_held(&rcu_bh_lock_map) ||
			 lock_is_held(&rcu_lock_map) ||
			 lock_is_held(&rcu_sched_lock_map),
			 "Illegal synchronize_rcu() in RCU read-side critical section");
	if (!rcu_blocking_is_gp()) {
		if (rcu_gp_is_expedited())
			synchronize_rcu_expedited();
		else
			synchronize_rcu_normal();
		return;
	}

	// 空宽限期（vacuous GP）路径，仅在调度器启动前
	rcu_poll_gp_seq_start_unlocked(&rcu_state.gp_seq_polled_snap);
	rcu_poll_gp_seq_end_unlocked(&rcu_state.gp_seq_polled_snap);

	local_irq_save(flags);
	WARN_ON_ONCE(num_online_cpus() > 1);
	rcu_state.gp_seq += (1 << RCU_SEQ_CTR_SHIFT);
	for (rnp = this_cpu_ptr(&rcu_data)->mynode; rnp; rnp = rnp->parent)
		rnp->gp_seq_needed = rnp->gp_seq = rcu_state.gp_seq;
	local_irq_restore(flags);
}
EXPORT_SYMBOL_GPL(synchronize_rcu);
```

### 第一条路径：常规路径（调度器已启动）

1. **Lockdep 检查**：若在 RCU 读端临界区内调用（`rcu_lock_map` 等被持有），触发 `RCU_LOCKDEP_WARN` —— 死锁场景。
2. `rcu_blocking_is_gp()`（`tree.c:3265`）返回 false（即 `rcu_scheduler_active != RCU_SCHEDULER_INACTIVE`）→ 需要真正等待：
   - `rcu_gp_is_expedited()` 为真 → 快速路径 `synchronize_rcu_expedited()`（IPI 强制各 CPU 退出临界区）。
   - 否则 → 常规路径 `synchronize_rcu_normal()`。

### 第二条路径：空宽限期（仅调度器启动前）

`rcu_blocking_is_gp()` 返回 true 意味着调度器未启动、只有引导 CPU、无并发，任何宽限期都"真空"立即完成。只做记账：

- 用 `rcu_poll_gp_seq_start_unlocked/end_unlocked` 记录一次轮询式 GP 的起止。
- 关中断下手动把 `rcu_state.gp_seq` 推进一格（`RCU_SEQ_CTR_SHIFT = 2`，每次 +4 表示一个完整 GP 序号），沿 `rcu_data->mynode` 向上同步 `rcu_node` 层次每级的 `gp_seq` / `gp_seq_needed`。
- `WARN_ON_ONCE(num_online_cpus() > 1)` 防御性验证单 CPU 场景。

### 常规路径核心：`synchronize_rcu_normal()`（`tree.c:3277-3309`）

```c
static void synchronize_rcu_normal(void)
{
	struct rcu_synchronize rs;

	trace_rcu_sr_normal(rcu_state.name, &rs.head, TPS("request"));

	if (READ_ONCE(rcu_normal_wake_from_gp) < 1) {
		wait_rcu_gp(call_rcu_hurry);   // 旧式：等回调真正执行
		goto trace_complete_out;
	}

	init_rcu_head_on_stack(&rs.head);
	init_completion(&rs.completion);

	if (IS_ENABLED(CONFIG_PROVE_RCU))
		get_state_synchronize_rcu_full(&rs.oldstate);

	rcu_sr_normal_add_req(&rs);          // 加入 per-CPU llist 请求队列

	(void) start_poll_synchronize_rcu(); // 触发一个宽限期

	wait_for_completion(&rs.completion); // 睡眠直到宽限期回调完成
	destroy_rcu_head_on_stack(&rs.head);

trace_complete_out:
	trace_rcu_sr_normal(rcu_state.name, &rs.head, TPS("complete"));
}
```

基于经典 "`struct rcu_synchronize` = 栈上 `rcu_head` + `completion`" 模式：

- 把 `&rs.head` 作为回调挂到 RCU 上，宽限期结束时回调 `complete(&rs.completion)` 唤醒睡眠中的调用者。
- 调用中可能被抢占，`CONFIG_PROVE_RCU` 下先做 `get_state_synchronize_rcu_full()` 快照以校验语义。
- `rcu_normal_wake_from_gp` 是 sysfs 旋钮（`/sys/kernel/rcu_normal`）。默认 <1 走 `wait_rcu_gp(call_rcu_hurry)` 老路径；≥1 启用"从宽限期唤醒"优化，改用 `start_poll_synchronize_rcu()` + completion 等待。

## 极小实现：`kernel/rcu/tiny.c:141`（TINY RCU）

```c
void synchronize_rcu(void)
{
	RCU_LOCKDEP_WARN(lock_is_held(&rcu_bh_lock_map) ||
			 lock_is_held(&rcu_lock_map) ||
			 lock_is_held(&rcu_sched_lock_map),
			 "Illegal synchronize_rcu() in RCU read-side critical section");
	preempt_disable();
	WRITE_ONCE(rcu_ctrlblk.gp_seq, rcu_ctrlblk.gp_seq + 2);
	preempt_enable();
}
EXPORT_SYMBOL_GPL(synchronize_rcu);
```

没有任何真正的等待。原因：非抢占 + 单核，`rcu_read_lock()` / `rcu_read_unlock()` 之间不可能有另一个执行流穿插，临界区要么完整在调用前、要么完整在调用后。于是关抢占下把 `rcu_ctrlblk.gp_seq` **加 2**（低 2 位为 GP 状态位，`+2` 把状态从"进行中"翻转为"已完成"），让 `poll_state_synchronize_rcu()` 等检查看到新宽限期已结束。整个调用 O(1)。

## 关联符号

| 符号 | 位置 | 说明 |
|---|---|---|
| `synchronize_rcu_mult(...)` | `include/linux/rcupdate_wait.h:59` | 宏，并行等待 `synchronize_rcu` 与 `call_rcu_tasks*` 等 |
| `synchronize_rcu_tasks` | `include/linux/rcupdate.h:203` | `#define synchronize_rcu_tasks synchronize_rcu`，别名 |
| `cond_synchronize_rcu(oldstate)` | `kernel/rcu/tree.c:3629` | 按需同步：快照宽限期未结束才调 `synchronize_rcu()` |
| `synchronize_rcu_expedited()` | `kernel/rcu/tree.c` | 快速路径：IPI 强制各 CPU 退出临界区 |
| `rcu_blocking_is_gp()` | `kernel/rcu/tree.c:3265` | 调度器未启动时判定空宽限期 |
| `rcu_sr_normal_add_req` | `kernel/rcu/tree.c` | 常规路径 per-CPU llist 请求入队（配合 `rcu_normal_wake_from_gp`） |

## 调用者（Blast radius）

- 内核态 `tree.c` 版：约 432 个调用者（arch 下 virt、电源管理、驱动卸载路径等）。
- 工具态 `tools/virtio` 版：1 个调用者 —— nvdimm 测试 `tools/testing/nvdimm/test/iomap.c` 的 `nfit_test_teardown`。

## 小结

- `tools/include/linux/kernel.h:95` 是**用户态工具编译用的空宏** —— `synchronize_rcu()` 变成 no-op。
- 内核真实语义在 `kernel/rcu/tree.c:3349`（SMP 通用内核，完整宽限期等待 + 快速路径 + 空宽限期）与 `kernel/rcu/tiny.c:141`（单核极小内核，O(1) 伪等待）。
- 各处定义与 `include/linux/rcupdate.h` 的统一声明配合，靠 `CONFIG_TREE_RCU` / `CONFIG_PREEMPT_RCU` 等配置在编译期分派。
