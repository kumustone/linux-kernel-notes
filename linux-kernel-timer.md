# linux定时管理

## 定时器与时间管理

**内核中的时间概念**

内核必须在硬件的帮助下才能计算和管理时间。系统定时器以某种频率自行触发时钟中断，该中断可以通过编程设定，称为节拍率（tick rate).



linux2.5之前的节拍率是100， 2.5之后就变成1000了。即每个1ms产生一个时钟中断，那么时钟中断产生后，kernel做些神马事呢？（这个中断是谁产生的，看下面的RTC部分）

- 更新系统时间
- 更新实际时间
- 在smp系统上，均衡调度程序中各处理器上的运行队列
- 检查当前进程是否用尽了自己的时间片，如果用尽，重新进行调度。
- 运行超时的动态定时器
- 更新资源消耗和处理器时间统计值。

> 如何理解 usleep(10) 这种需求呢，难道每次变成睡眠1ms吗？即系统中微秒级别的定时器是如何实现的?
>
> [High-resolution timing](http://tldp.org/HOWTO/IO-Port-Programming-4.html)
>
> - sleep级别的
>
> sleep usleep 使用的进程级别切换，只能维持10ms级别的准确度；
>
> 现在usleep是使用unsleep实现的，unsleep有改进 -> 前提是设置了`sched_setscheduler()`
>
> In the 2.0.x series of Linux kernels, there is a new system call, `nanosleep()` (see the `nanosleep(2)` manual page), that allows you to sleep or delay for short times (a few microseconds or more).
>
> For delays <= 2 ms, if (and only if) your process is set to soft real time scheduling (using `sched_setscheduler()`), `nanosleep()` uses a busy loop; otherwise it sleeps, just like `usleep()`.
>
>
>
> - 使用中断来实现实现定时器
>
>   Inputting or outputting any byte from/to port 0x80 (see above for how to do it) should wait for almost exactly 1 microsecond independent of your processor type and speed.
>
> - 使用汇编指令来实现时钟级别的定时器
>
> ```
> Instruction   i386 clock cycles   i486 clock cycles
> xchg %bx,%bx          3                   3
> nop                   3                   1
> or %ax,%ax            2                   1
> mov %ax,%ax           2                   1
> add %ax,0             2                   1
> ```

[时钟和定时器电路](http://guojing.me/linux-kernel-architecture/posts/time-system/)

- 实时时钟  RTC 

  > 独立于时间CPU和其他芯片，即使当PC被切断电源，RTC依旧继续工作，因为它靠一个小电池或蓄电池供电。CMOS TAM和RTC被继承在一个芯片[1](http://guojing.me/linux-kernel-architecture/posts/time-system/#fn:1)上。
  >
  >  
  >
  > RTC能在IRQ8上发出周期性的中断，频率在2～8192Hz之间，也可以对RTC进行编程以使当RTC到达某个特定的值时激活IRQ8线，也就是作为一个闹钟来工作。

- 时间戳计时器 TSC gettimeofday就是从TSC中获取的，所以可以统计很短的时间流逝。

  > 所有的80x86微处理器都包含一条CLK输入引线，它接收玩不振荡器的时钟信号。从Pentium开始，80x86微处理器就都包含一个计数器，它在每个时钟信号到来时加1。该计数器是利用64位的时间戳计数器（*Time Stamp Counter TSC*）寄存器来实现的，可以通过汇编语言指令*rdtsc*来读这个寄存器。当使用这个寄存器时，内核必须考虑到时钟信号的频率，例如时钟节拍的频率时1GHz，那么时间戳计数器每那庙增加一次。

- PIT Programmable Interval Timer

- CPU本地计时器