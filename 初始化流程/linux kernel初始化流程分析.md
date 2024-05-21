## 环境说明
版本： linux-5.10.131

## start_kernel
linux kernel初始化函数，是linux初始化过程中，汇编代码切换到C代码的入口

## set_task_stack_end_magic(&init_task)
init_task是内核启的第一个线程。set_task_stack_end_magic函数在线程的栈底设置一个魔数STACK_END_MAGIC，用于检测栈溢出
```
void set_task_stack_end_magic(struct task_struct *tsk)
{
	unsigned long *stackend;

	stackend = end_of_stack(tsk);
	*stackend = STACK_END_MAGIC;	/* for overflow detection */
}
```
值得一提的是，配置项CONFIG_STACK_GROWSUP用于控制栈的增长方向，默认不使能，及栈的默认增长方向向下

## smp_setup_processor_id( )
对称多处理系统相关的初始化，不同arch之间的实现有所差异。大致流程为：1、从当前CPU寄存器获取cpuid；2、初始化`__cpu_logical_map[ ]`；3、配置当前CPU的offset到寄存器中，与percpu的实现有关
```
void __init smp_setup_processor_id(void)
{
	u64 mpidr = read_cpuid_mpidr() & MPIDR_HWID_BITMASK;
	set_cpu_logical_map(0, mpidr);

	/*
	 * clear __my_cpu_offset on boot CPU to avoid hang caused by
	 * using percpu variable early, for example, lockdep will
	 * access percpu variable inside lock_release
	 */
	set_my_cpu_offset(0);
	pr_info("Booting Linux on physical CPU 0x%010lx [0x%08x]\n",
		(unsigned long)mpidr, read_cpuid_id());
}
```

## debug_objects_early_init( )
初始化debug objects。debug objects模块用于跟踪和调试内核对象的生命周期和状态的
```
void __init debug_objects_early_init(void)
{
	int i;

	for (i = 0; i < ODEBUG_HASH_SIZE; i++)
		raw_spin_lock_init(&obj_hash[i].lock);

	for (i = 0; i < ODEBUG_POOL_SIZE; i++)
		hlist_add_head(&obj_static_pool[i].node, &obj_pool);
}
```
配置项CONFIG_DEBUG_OBJECTS控制debug objects模块是否使能。默认不使能

## cgroup_init_early( )
初始化cgroup根控制组以及遍历初始化cgroup_subsys数组

配置项CONFIG_CGROUPS控制是否使能cgroup

## local_irq_disable（ ）
屏蔽当前CPU上的所有中断。在此操作之后，还会把early_boot_irqs_disabled置true

值得一提，配置项CONFIG_TRACE_IRQFLAGS用于启用中断标志（IRQ flags）的跟踪或调试功能，当启用此配置项时，内核会在修改中断标志（如使用 `local_irq_save()`, `local_irq_disable()`, `local_irq_restore()` 等函数）时记录额外的日志或跟踪信息

## boot_cpu_init( )
初始化和激活当前CPU（即启动CPU），配置当前CPU的状态
```
void __init boot_cpu_init(void)
{
	int cpu = smp_processor_id();

	/* Mark the boot cpu "present", "online" etc for SMP and UP case */
	set_cpu_online(cpu, true);
	set_cpu_active(cpu, true);
	set_cpu_present(cpu, true);
	set_cpu_possible(cpu, true);

#ifdef CONFIG_SMP
	__boot_cpu_id = cpu;
#endif
}
```
CPU的四种基础状态：  
1、possible：置位`__cpu_possible_mask`中对应的bit，表示存在这个CPU资源，但还没有纳入kernel的管理范围，需要为此CPU的per-cpu 变量分配启动时内存
2、present：置位`__cpu_present_mask`中的对应bit，表示此CPU已经被kernel接管，但不一定处于在线状态
3、active：置位`__cpu_active_mask`中对应的bit，表示允许任务迁入此CPU
4、online：置位`__cpu_online_mask`中对应的bit，表示此CPU可以被调度器使用，或者说此CPU正在被内核使用
