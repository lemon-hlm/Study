时间的一种保持方式是通过时钟中断计数，进而换算得到，这种方式在虚拟机里有问题，因为有时运行vpcu的cpu被调度出来使时钟中断不能准时到达guest os。

另外一种方式，如模拟HPET，guest os当需要的时候会去读当前的时间，这种方式会使得虚拟机频繁的退出(VM-exit)，严重影响性能。

为此kvm引入了基于半虚拟化的时钟kvmclock,这种方式需要在guest上实现一个kvmclock驱动，建立guest os 到VMM的通道，这样通过这个通道 guest os 向 vmm 查询时间。kvmclock是一个半虚拟化的时钟，guest感知。guest向hypervisor询问时间，同时保证时钟的稳定和准确。默认情况下guest os会使用kvmclock，如fedora17

```
1. kvm clock 时钟
    1.1. kvmclock 原理
    1.2. KVM clock 在 Guest 中
    1.2. KVM clock 在host中
2. kvm_guest_time_update分析
3. 关于use_master_clock
4. KVMCLOCK的优点
5. Cpu Steal time
```

## 1. kvm clock 时钟

### 1.1. kvmclock 原理

guest注册内存页，里面包含kvmclock数据，该页在guest整个生命周期内存在，hypervisor会持续写这个页。

```
include\xen\arm\interface.h

/* TODO: Move pvclock definitions some place arch independent */
struct pvclock_vcpu_time_info {
	u32   version;
	u32   pad0;
	//guest的TSC时间戳，在kvm_guest_time_update中会被更新
	u64   tsc_timestamp;
	//guest的墙上时间（1970年距今的绝对日期），和上者在一起更新
    //system_time = kernel_ns + v->kvm->arch.kvmclock_offset
    //系统启动后的时间减去VM init的时间，即guest init后到现在的时间
    //最近一次从host读到的时间，通过当前注册的时钟源读取
	u64   system_time;
	//KVMCLOCK时钟频率固定在1GHZ
	//存放当前一nanoseconds多少个counter值，cpu频率调整后该值也会变
	u32   tsc_to_system_mul;
	s8    tsc_shift;
	u8    flags;
	u8    pad[2];
} __attribute__((__packed__)); /* 32 bytes */

/* It is OK to have a 12 bytes struct with no padding because it is packed */
struct pvclock_wall_clock {
	u32   version;
	u32   sec;
	u32   nsec;
	u32   sec_hi;
} __attribute__((__packed__));
```

#### 时钟源 kvm_clock 的定义

```
static struct clocksource kvm_clock = {

    .name ="kvm-clock",

    .read = kvm_clock_get_cycles,

    .rating = 400, //rating400为理想时钟源

    .mask =CLOCKSOURCE_MASK(64),

    .flags =CLOCK_SOURCE_IS_CONTINUOUS,

};
```

### 1.2. KVM clock 在 Guest 中

源码路径： arch\x86\kernel\kvmclock.c pvclock.c.

kvmclock_init负责在[guest启动过程](http://oenhan.com/kvm-src-2-vm-run)中初始化kvm clock（kvmclock 驱动），首先更新了两个MSR值：

```
#define MSR_KVM_WALL_CLOCK_NEW  0x4b564d00
#define MSR_KVM_SYSTEM_TIME_NEW 0x4b564d01
```

然后为每个CPU分配struct pvclock_vsyscall_time_info内存，获取了首要CPU的pvti对应的物理地址，将其写入到msr_kvm_system_time MSR中

```
void __init kvmclock_init(void)
{
	···
	//更新两个MSR值
	if (kvmclock && kvm_para_has_feature(KVM_FEATURE_CLOCKSOURCE2)) {
		msr_kvm_system_time = MSR_KVM_SYSTEM_TIME_NEW;
		msr_kvm_wall_clock = MSR_KVM_WALL_CLOCK_NEW;
    }
    ···
    //注册kvmclock，下面详细描述
	if (kvm_register_clock("primary cpu clock")) {

	//启用STEAL time机制
	if (kvm_para_has_feature(KVM_FEATURE_CLOCKSOURCE_STABLE_BIT))
		pvclock_set_flags(PVCLOCK_TSC_STABLE_BIT);
    ···
    
    //见下面详细描述，里面会获取当前时钟偏移
	kvm_sched_clock_init(flags & PVCLOCK_TSC_STABLE_BIT);
	
	put_cpu();
	//写改x86的函数指针，在guest虚拟的硬件平台上指定函数
	//重点看下面这四个函数
	x86_platform.calibrate_tsc = kvm_get_tsc_khz;
	x86_platform.calibrate_cpu = kvm_get_tsc_khz;
	//get_wallclock本质是获取1970年到现在的时间差，本质是绝对日期
	//例如x86_platform.get_wallclock
	//默认为mach_get_cmos_time（从cmos取得wallclock).
    //wallclock指的是操作系统从开机开始的绝对时间。
	x86_platform.get_wallclock = kvm_get_wallclock;
	x86_platform.set_wallclock = kvm_set_wallclock;
    //注册系统时钟源
	clocksource_register_hz(&kvm_clock, NSEC_PER_SEC);
	pv_info.name = "KVM";
}
```

#### 函数 kvm_register_clock

```
int kvm_register_clock(char *txt)
{
	int cpu = smp_processor_id();
	int low, high, ret;
	struct pvclock_vcpu_time_info *src;

	if (!hv_clock)
		return 0;

	src = &hv_clock[cpu].pvti;
	low = (int)slow_virt_to_phys(src) | 1;
	high = ((u64)slow_virt_to_phys(src) >> 32);
	//获取了首要CPU的pvti对应的物理地址，
	//将其写入到msr_kvm_system_time MSR中
	//通过msr寄存write的方式将hv_clock[cpu].pvti的gpa通知给vmm.
	ret = native_write_msr_safe(msr_kvm_system_time, low, high);
	printk(KERN_INFO "kvm-clock: cpu %d, msr %x:%x, %s\n",
	       cpu, high, low, txt);

	return ret;
}
```

关键函数**native_write_msr_safe**：计算出来pvit的物理地址（这个“物理地址”，其实是Guest Physical Address，即GPA），写MSR寄存器，通过msr_kvm_system_time（MSR_KVM_SYSTEM_TIME），告诉Host它的pvit地址。

#### 函数 kvm_sched_clock_init

```
static inline void kvm_sched_clock_init(bool stable)
{
	if (!stable) {
		pv_time_ops.sched_clock = kvm_clock_read;
		clear_sched_clock_stable();
		return;
	}
	//获取当前的时钟偏移
	kvm_sched_clock_offset = kvm_clock_read();
	pv_time_ops.sched_clock = kvm_sched_clock_read;

	printk(KERN_INFO "kvm-clock: using sched offset of %llu cycles\n",
			kvm_sched_clock_offset);

	BUILD_BUG_ON(sizeof(kvm_sched_clock_offset) >
	         sizeof(((struct pvclock_vcpu_time_info *)NULL)->system_time));
}
```

#### 函数 clocksource_register_hz：时钟资源注册

```
static inline int clocksource_register_hz(struct clocksource *cs, u32 hz)
{
	return __clocksource_register_scale(cs, 1, hz);
}

int __clocksource_register_scale(struct clocksource *cs, u32 scale, u32 freq)
{

	/* Initialize mult/shift and max_idle_ns */
	__clocksource_update_freq_scale(cs, scale, freq);

	/* Add clocksource to the clocksource list */
	mutex_lock(&clocksource_mutex);
	clocksource_enqueue(cs);
	clocksource_enqueue_watchdog(cs);
	//将guest os的curr_clocksource设为kvmclock
	clocksource_select();
	clocksource_select_watchdog(false);
	mutex_unlock(&clocksource_mutex);
	return 0;
}
```

下面详细看上面复写的四个函数

#### a. 函数 kvm_get_tsc_khz

```
//获取pv clock信息转换成hz
static unsigned long kvm_get_tsc_khz(void)
{
	struct pvclock_vcpu_time_info *src;
	int cpu;
	unsigned long tsc_khz;

	cpu = get_cpu();
	src = &hv_clock[cpu].pvti;
	tsc_khz = pvclock_tsc_khz(src);
	put_cpu();
	return tsc_khz;
}
```

#### b. 函数 kvm_get_wallclock

```
static void kvm_get_wallclock(struct timespec *now)
{
	struct pvclock_vcpu_time_info *vcpu_time;
	int low, high;
	int cpu;
	//获取wall clock变量的物理地址，写入到msr中
	//写完msr后wall_clock就是更新后的墙上时间，即guest启动的日期。
	low = (int)__pa_symbol(&wall_clock);
	high = ((u64)__pa_symbol(&wall_clock) >> 32);
	//通知vmm要取wall_clock并将wall_clock的gpa告诉vmm.
	native_write_msr(msr_kvm_wall_clock, low, high);

	cpu = get_cpu();

	vcpu_time = &hv_clock[cpu].pvti;
	//然后再加上 pvclock_clocksource_read(vcpu_time) 即 guest 启动时间，
	//则就是当前 guest 中的 real time。
	//返回 vmm 设置号的 wallclock.wall_clock 在返回前相当于是 guest 与 vmm 间的共享内存.
	pvclock_read_wallclock(&wall_clock, vcpu_time, now);

	put_cpu();
}
```

#### 函数pvclock_read_wallclock的访问

```
void pvclock_read_wallclock(struct pvclock_wall_clock *wall_clock,
			    struct pvclock_vcpu_time_info *vcpu_time,
			    struct timespec *ts)
{
	u32 version;
	u64 delta;
	struct timespec now;

	/* get wallclock at system boot */
	//等待vmm设置好wall_clock, 用version来标记数据是否更新
	do {
		version = wall_clock->version;
		rmb();		/* fetch version before time */
		now.tv_sec  = wall_clock->sec;
		now.tv_nsec = wall_clock->nsec;
		rmb();		/* fetch time before checking version */
	} while ((wall_clock->version & 1) || (version != wall_clock->version));
	//这时wall_clock记录的是系统开机时的时间

	//取得系统运行的时间, vcpu_time作为共享内存，其地址在kvm_register_clock通知了vmm
	delta = pvclock_clocksource_read(vcpu_time);	/* time since system boot */

	//两者相加为wall_clock,
	delta += now.tv_sec * (u64)NSEC_PER_SEC + now.tv_nsec;

	now.tv_nsec = do_div(delta, NSEC_PER_SEC);
	now.tv_sec = delta;

	set_normalized_timespec(ts, now.tv_sec, now.tv_nsec);
}
```



```
u64 pvclock_clocksource_read(struct pvclock_vcpu_time_info *src)
{
	unsigned version;
	u64 ret;
	u64 last;
	u8 flags;

	do {
		version = pvclock_read_begin(src);
		ret = __pvclock_read_cycles(src, rdtsc_ordered());
		flags = src->flags;
	} while (pvclock_read_retry(src, version));

	if (unlikely((flags & PVCLOCK_GUEST_STOPPED) != 0)) {
		src->flags &= ~PVCLOCK_GUEST_STOPPED;
		pvclock_touch_watchdogs();
	}

	if ((valid_flags & PVCLOCK_TSC_STABLE_BIT) &&
		(flags & PVCLOCK_TSC_STABLE_BIT))
		return ret;

	/*
	 * Assumption here is that last_value, a global accumulator, always goes
	 * forward. If we are less than that, we should not be much smaller.
	 * We assume there is an error marging we're inside, and then the correction
	 * does not sacrifice accuracy.
	 *
	 * For reads: global may have changed between test and return,
	 * but this means someone else updated poked the clock at a later time.
	 * We just need to make sure we are not seeing a backwards event.
	 *
	 * For updates: last_value = ret is not enough, since two vcpus could be
	 * updating at the same time, and one of them could be slightly behind,
	 * making the assumption that last_value always go forward fail to hold.
	 */
	last = atomic64_read(&last_value);
	do {
		if (ret < last)
			return last;
		last = atomic64_cmpxchg(&last_value, last, ret);
	} while (unlikely(last != ret));

	return ret;
}
```

### 1.2. KVM clock 在host中

VMM 中 kvmclock实现

Guest中使用了wrmsr指令，Host会感知，然后进入到kvm_set_msr_common 中处理。

#### （1） MSR_KVM_WALL_CLOCK

msr的实现：

arch\x86\kvm\x86.c

kvm_set_msr_common ==>  case MSR_KVM_WALL_CLOCK ==> kvm_write_wall_clock

```
case MSR_KVM_WALL_CLOCK:
	//wall_clock是guest中对应的wall_clock地址
	vcpu->kvm->arch.wall_clock = data;
	kvm_write_wall_clock(vcpu->kvm, data);
```

```
static void kvm_write_wall_clock(struct kvm *kvm, gpa_twall_clock)
{
    .......
    //a. 读guest version，下面详细
    r = kvm_read_guest(kvm,wall_clock, &version, sizeof(version));
    if (r)
       return;
    if (version & 1)
       ++version;  /* first time write, random junk */
    ++version;
    kvm_write_guest(kvm,wall_clock, &version, sizeof(version));//更新version
    //此处获取的是host的启动的绝对时间，即从1970到现在的时间差
    getboottime(&boot);//得到系统的boot时间
    if(kvm->arch.kvmclock_offset) {
       //kvmclock_offset是host启动后到guest启动的相对值，
       //即几分几秒
       struct timespec ts =ns_to_timespec(kvm->arch.kvmclock_offset);
       boot =timespec_sub(boot, ts);
    }
    wc.sec = boot.tv_sec;
    wc.nsec = boot.tv_nsec;
    wc.version = version;
    //更新guest wall_clock
    kvm_write_guest(kvm,wall_clock, &wc, sizeof(wc));
    version++;
    kvm_write_guest(kvm,wall_clock, &version, sizeof(version)); //更新version，完成通讯
}
```

在kvm_arch_init_vm中有

```
arch\x86\kvm\x86.c

kvm->arch.kvmclock_offset = -ktime_get_boot_ns();
```

因为kvmclock_offset为负值，相减即相加，host启动日期加上guest启动的距离host启动时间的差值等于guest启动的日期，所以write msr的结果就是这样。

kvm_read_guest/kvm_write_guest 的工作原理是通过gpa得到对应page 的hva和页内偏移，然后就能读写内存了

```
int kvm_read_guest(struct kvm *kvm, gpa_t gpa, void *data, unsigned long len)
{
	gfn_t gfn = gpa >> PAGE_SHIFT;
	int seg;
	int offset = offset_in_page(gpa);
	int ret;

	while ((seg = next_segment(len, offset)) != 0) {
		ret = kvm_read_guest_page(kvm, gfn, data, offset, seg);
		if (ret < 0)
			return ret;
		offset = 0;
		len -= seg;
		data += seg;
		++gfn;
	}
	return 0;
}

int kvm_read_guest_page(struct kvm *kvm, gfn_t gfn, void *data, int offset,
			int len)
{
	struct kvm_memory_slot *slot = gfn_to_memslot(kvm, gfn);

	return __kvm_read_guest_page(slot, gfn, data, offset, len);
}

static int __kvm_read_guest_page(struct kvm_memory_slot *slot, gfn_t gfn,
				 void *data, int offset, int len)
{
	int r;
	unsigned long addr;

	addr = gfn_to_hva_memslot_prot(slot, gfn, NULL);
	if (kvm_is_error_hva(addr))
		return -EFAULT;
	r = __copy_from_user(data, (void __user *)addr + offset, len);
	if (r)
		return -EFAULT;
	return 0;
}
```

#### （2） MSR_KVM_SYSTEM_TIME

```
case MSR_KVM_SYSTEM_TIME: {
    ···
	vcpu->arch.time = data;
	kvm_make_request(KVM_REQ_GLOBAL_CLOCK_UPDATE, vcpu);

	/* we verify if the enable bit is set... */
	if (!(data & 1))
		break;
	//kvm_gfn_to_hva_cache_init会得到guest os 的hv_clock[cpu].pvti
	//time就是kvm clock对应的guest pvclock_vsyscall_time_info地址，
	//通过kvm_gfn_to_hva_cache_init函数，会把传过来的地址参数data记录到pv_time中。
	//这样子就可以通过pv_time来直接修改Guest中的pvit。
	if (kvm_gfn_to_hva_cache_init(vcpu->kvm,
	     &vcpu->arch.pv_time, data & ~1ULL,
	     sizeof(struct pvclock_vcpu_time_info)))
		vcpu->arch.pv_time_enabled = false;
	else
		vcpu->arch.pv_time_enabled = true;

	break;
}
```

vcpu_enter_guest==> KVM_REQ_GLOBAL_CLOCK_UPDATE 

kvm_gen_kvmclock_update(vcpu);

```
static void kvm_gen_kvmclock_update(struct kvm_vcpu *v)
{
	struct kvm *kvm = v->kvm;

	kvm_make_request(KVM_REQ_CLOCK_UPDATE, v);
	schedule_delayed_work(&kvm->arch.kvmclock_update_work,
					KVMCLOCK_UPDATE_DELAY);
}
```

由于在kvm_arch_init_vm时：

```
INIT_DELAYED_WORK(&kvm->arch.kvmclock_update_work, kvmclock_update_fn);
INIT_DELAYED_WORK(&kvm->arch.kvmclock_sync_work, kvmclock_sync_fn);
```

所以kvm->arch.kvmclock_update_work==》

```
static void kvmclock_update_fn(struct work_struct *work)
{
    ···
    //对每个vcpu设置KVM_REQ_CLOCK_UPDATE
    kvm_for_each_vcpu(i,vcpu, kvm) {
    	kvm_make_request(KVM_REQ_CLOCK_UPDATE, vcpu);
    	kvm_vcpu_kick(vcpu);
    }
}
```

vcpu_enter_guest==>KVM_REQ_CLOCK_UPDATE 

kvm_guest_time_update(vcpu);

kvm_guest_time_update会将时间更新到vcpu->pv_time

下面详细讲解下函数kvm_guest_time_update

## 2. kvm_guest_time_update分析

```
static int kvm_guest_time_update(struct kvm_vcpu *v)
{
	unsigned long flags, tgt_tsc_khz;
	struct kvm_vcpu_arch *vcpu = &v->arch;
	struct kvm_arch *ka = &v->kvm->arch;
	s64 kernel_ns;
	u64 tsc_timestamp, host_tsc;
	u8 pvclock_flags;
	bool use_master_clock;

	kernel_ns = 0;
	host_tsc = 0;

	/*
	 * If the host uses TSC clock, then passthrough TSC as stable
	 * to the guest.
	 */
	spin_lock(&ka->pvclock_gtod_sync_lock);
	use_master_clock = ka->use_master_clock;
	if (use_master_clock) {
		host_tsc = ka->master_cycle_now;
		kernel_ns = ka->master_kernel_ns;
	}
	spin_unlock(&ka->pvclock_gtod_sync_lock);

	/* Keep irq disabled to prevent changes to the clock */
	local_irq_save(flags);
	//获取host中tsc与TSC时钟1KHZ的比例
	tgt_tsc_khz = __this_cpu_read(cpu_tsc_khz);
	if (unlikely(tgt_tsc_khz == 0)) {
		local_irq_restore(flags);
		kvm_make_request(KVM_REQ_CLOCK_UPDATE, v);
		return 1;
	}
	if (!use_master_clock) {
		host_tsc = rdtsc();
		kernel_ns = ktime_get_boot_ns();
	}
	//读取guest中当前tsc
	tsc_timestamp = kvm_read_l1_tsc(v, host_tsc);

	/*
	 * We may have to catch up the TSC to match elapsed wall clock
	 * time for two reasons, even if kvmclock is used.
	 *   1) CPU could have been running below the maximum TSC rate
	 *   2) Broken TSC compensation resets the base at each VCPU
	 *      entry to avoid unknown leaps of TSC even when running
	 *      again on the same CPU.  This may cause apparent elapsed
	 *      time to disappear, and the guest to stand still or run
	 *	very slowly.
	 */
	if (vcpu->tsc_catchup) {
		u64 tsc = compute_guest_tsc(v, kernel_ns);
		if (tsc > tsc_timestamp) {
			adjust_tsc_offset_guest(v, tsc - tsc_timestamp);
			tsc_timestamp = tsc;
		}
	}

	local_irq_restore(flags);

	/* With all the info we got, fill in the values */

	if (kvm_has_tsc_control)
		tgt_tsc_khz = kvm_scale_tsc(v, tgt_tsc_khz);
	//将1KHZ TSC转换成guest TSC
	//如果当前guest时钟（kvmclock）的频率不同，则更新转换比例
	if (unlikely(vcpu->hw_tsc_khz != tgt_tsc_khz)) {
		kvm_get_time_scale(NSEC_PER_SEC, tgt_tsc_khz * 1000LL,
				   &vcpu->hv_clock.tsc_shift,
				   &vcpu->hv_clock.tsc_to_system_mul);
		//即是guest tsc转换成kvmclock的比例
		vcpu->hw_tsc_khz = tgt_tsc_khz;
	}

	//当前kvmclock下的TSC值
	vcpu->hv_clock.tsc_timestamp = tsc_timestamp;
	//当前kvmclock下的guest了多少ns
	vcpu->hv_clock.system_time = kernel_ns + v->kvm->arch.kvmclock_offset;
	vcpu->last_guest_tsc = tsc_timestamp;

	/* If the host uses TSC clocksource, then it is stable */
	pvclock_flags = 0;
	if (use_master_clock)
		pvclock_flags |= PVCLOCK_TSC_STABLE_BIT;

	vcpu->hv_clock.flags = pvclock_flags;

	if (vcpu->pv_time_enabled)
		kvm_setup_pvclock_page(v);
	if (v == kvm_get_vcpu(v->kvm, 0))
		kvm_hv_setup_tsc_page(v->kvm, &vcpu->hv_clock);
	return 0;
}
```

因为Host和Guest，使用的是相同的tsc，这里还需要说明一个问题：

参考arch/x86/kvm/x86.c文件中kvm_guest_time_update函数：

关于Guest中的时间的计算，pv time中有两个重要参数：tsc_timestamp和system_time

Guest Sytem Time = Guest Sytem Time + offset；

为什么要这么麻烦的计算？

因为热迁移。两台Host的TSC不一样，如果Dst Host的TSC比Src Host的TSC小，那么可能会让Windows蓝屏或者linux panic。 如果Dst Host的TSC比Src Host的TSC大，那么在Guest中看到tsc瞬间跳变。所以需要计算offset来调整。

另外，在Guest中，还需要做一次计算delta：

arch/x86/include/asm/pvclock.h文件中的__pvclock_read_cycles函数中：

计算Host中读取到Tsc和Guest中读取到的Tsc的差值，再计算出来delta，最后再计算出来Guest中的“kvmclock”。这里把Host和Guest中前后两次的Tsc微小差值都计算进去了，可见精度确实很高。

## 3. 关于use_master_clock

在kvm_write_tsc中，本次tsc写和上次的tsc写比较，得到elapsed和usdiff

```
ns = ktime_get_boot_ns();

elapsed = ns - kvm->arch.last_tsc_nsec;

usdiff = data - kvm->arch.last_tsc_write;
```

用usdiff与elapsed进行对冲，如果二者差值小于usdiff < USEC_PER_SEC则证明tsc是稳定的

因为last_tsc_write和last_tsc_nsec都是在KVM下而非vcpu下，就是证明所有tsc是稳定的意义

```
if (!matched) {

kvm->arch.nr_vcpus_matched_tsc = 0;

} else if (!already_matched) {

kvm->arch.nr_vcpus_matched_tsc++;

}
```

## 4. KVMCLOCK的优点

kvm_get_wallclock替代mach_get_cmos_time获取rtc时间，mach_get_cmos_time函数在guest中执行需要多个pio vmexit才能完成，而kvm_get_wallclock只需要一个msr write即可，简便了操作，也不要在QEMU RTC的支持。

通过0x70，0x71端口操作。
参考LINUX内核：mach_get_cmos_time()，启动的时候获取日期时间。虽然内核也可以在每次需要的得到当前时间的时候读取 RTC，但这是一个 IO 调用，性能低下。实际上，在得到了当前时间后，Linux 系统会立即启动 tick 中断。此后，在每次的时钟中断处理函数内，Linux 更新当前的时间值，并保存在全局变量 xtime 内。比如时钟中断的周期为 10ms，那么每次中断产生，就将 xtime 加上 10ms。

```
unsigned char rtc_cmos_read(unsigned char addr)

{

unsigned char val;

lock_cmos_prefix(addr);

outb(addr, RTC_PORT(0));

val = inb(RTC_PORT(1));

lock_cmos_suffix(addr);

return val;

}
```

对于calibrate_tsc和calibrate_cpu同理。因为kvmclock效率只在启动的时候有体现，整体看替代效率并不明显。

关键在于时钟源的读取不再依赖于xtime的中断：

```
static struct clocksource kvm_clock = {

.name = "kvm-clock",

.read = kvm_clock_get_cycles,

.rating = 400,

.mask = CLOCKSOURCE_MASK(64),

.flags = CLOCK_SOURCE_IS_CONTINUOUS,

};

static u64 kvm_clock_read(void)

{

struct pvclock_vcpu_time_info *src;

u64 ret;

int cpu;

preempt_disable_notrace();

cpu = smp_processor_id();

src = &hv_clock[cpu].pvti;

ret = pvclock_clocksource_read(src);

preempt_enable_notrace();

return ret;

}
```

直接获取虚拟的clock时间。

而tsc的时钟源是

```
static struct clocksource clocksource_tsc = {

.name                   = "tsc",

.rating                 = 300,

.read                   = read_tsc,

.mask                   = CLOCKSOURCE_MASK(64),

.flags                  = CLOCK_SOURCE_IS_CONTINUOUS |

CLOCK_SOURCE_MUST_VERIFY,

.archdata               = { .vclock_mode = VCLOCK_TSC },

.resume= tsc_resume,

};
```

和TSC相比，kvmclock优势并不明显，除非TSC进行了迁移。

## 5. Cpu Steal time

Cpu Steal time指的是vcpu 等待 real cpu 的时间, 因为vcpu会发生vm-exit而进入vmm;进入vmm 后到重新vm-entry的时间就是一次cpu steal time. 该指标是衡量vm性能的重要指标。 通过半虚拟化技术guest os能得到cpu steal time. VMM与guest通讯机制与上一节类似，本节就不讨论了。

（1） Guest os 实现

1. kvm_guest_init注册函数指针pv_time_ops.steal_clock =kvm_steal_clock; 对非guest而言

该函数为native_steal_clock， 直接返回0



2. Guest os 通过kvm_register_steal_time 通知vmm 共享内存地址：

wrmsrl(MSR_KVM_STEAL_TIME,(slow_virt_to_phys(st) | KVM_MSR_ENABLED));

 

内核kernel\core.c update_rq_clock ==> update_rq_clock_task ==>

paravirt_steal_clock(cpu_of(rq))==> pv_time_ops.steal_clock;

 

（2） vmm 实现

kvm_set_msr_common ==》　case MSR_KVM_STEAL_TIME

  a. kvm_gfn_to_hva_cache_init得到guest os gpa -> hva

  b. vcpu->arch.st.last_steal= current->sched_info.run_delay;

  c. accumulate_steal_time(vcpu);

```
static void accumulate_steal_time(struct kvm_vcpu *vcpu)

{

    .......

    delta =current->sched_info.run_delay - vcpu->arch.st.last_steal;

    vcpu->arch.st.last_steal= current->sched_info.run_delay;

    vcpu->arch.st.accum_steal= delta;

}
```

第一调用时delta会为0， 但当以后vcpu_load时kvm_arch_vcpu_load会重新调用accumulate_steal_time

 

  d. kvm_make_request(KVM_REQ_STEAL_UPDATE,vcpu);

 

vcpu_enter_guest ==> record_steal_time(vcpu);

```
static void record_steal_time(struct kvm_vcpu *vcpu)

{

  

    ............ //kvm_read_guest_cached

    vcpu->arch.st.steal.steal+= vcpu->arch.st.accum_steal;

    vcpu->arch.st.steal.version+= 2;

    vcpu->arch.st.accum_steal= 0;

    ......... //kvm_write_guest_cached

}
```