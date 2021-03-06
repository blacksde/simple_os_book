# 基于时间事件的等待与唤醒

Clock（时钟）中断(irq0，可回顾第二章2.4节) 可给操作系统提供有一定间隔的时间事件， 操作系统将其作为基本的计时单位，这里把两次时钟中断之间的时间间隔为一个时间片（timer splice）。基于此时间片，操作系统得以向上提供基于时间点的事件，并实现基于固定时间长度的等待和唤醒机制。在每个时钟中断发生时，操作系统可产生对应时间长度的时间事件，这样操作系统和应用程序可基于这些时间事件来构建基于时间的软件设计。在proj10.4中，实现了定时器timer的支持。

## timer数据结构

sched.h定义了有关timer数据结构，

    typedef struct {
        unsigned int expires;          //到期时间
        struct proc_struct *proc;     //等待时间到期的进程
        list_entry_t timer_link;      //链接到timer_list的链表项指针
    } timer_t;

这几个成员变量描述了一个timer（定时器）涉及的相关因素，首先一个expires表明了这个定时器何时到期，而第二个成员变量描述了定时器到期后需要唤醒的进程，最后一个参数是一个链表项，用于把自身挂到系统的timer链表上，以便于扫描查找特定的timer。

## timer相关操作

一个 timer在ucore中的生存周期可以被描述如下：

1. 某进程创建和初始化timer_t结构的一个timer，并把timer被加入系统timer管理列表timer_list中，进程设置为基于timer事件的阻塞态（即睡眠了），这样这个timer就诞生了，；
2. 系统时间通过时钟中断被不断累加，且ucore定期检查是否有某个timer的到期时间已经到了，如果没有到期，则ucore等待下一次检查，此timer会挂在timer_list上继续存在； 
3. 如果到期了，则对应的进程又处于就绪态了，并从系统timer管理列表timer_list中移除该 timer，自此timer就死亡退出了。

基于上述timer生存周期的流程，与timer相关的函数如下：

* timer init：对timer的成员变量进行初始化，设定了在expires时间之后唤醒proc进程
* add_timer：向系统timer链表timer_list添加某个初始化过的timer，这样该timer计时器将在指定时间expires后被扫描到，如果等待则这个定时器timer的进程处在等待状态，并将将进程唤醒，进程将处于就绪态。
* del_timer：向系统timer链表timer_list删除（或者说取消）某一个计时器。该计时 器在取消后，对应的进程不会被系统在指定时刻expires唤醒。
* run_timer_list：被trap函数调用，遍历系统timer链表timer_list中的timer计时器，找出所有应该到时的timer计时器，并唤醒与此计时器相关的等待进程，再删除此timer计时器。在lab4/proj13以后，还增加了对进程调度器在时间事件产生后的处理函数的调用（在后续有进一步分析）。

有了这些函数的支持，我们就可以实现进程睡觉并被定时唤醒的功能了。比如ucore在用户函数库中提供了sleep函数，当用户进程调用sleep函数后，会进一步调用sys_sleep系统调用，在内核中完成sys_sleep系统调用服务的是do_sleep内核函数，其实现如下：

    int
    do_sleep(unsigned int time) {
    ……
        timer_t __timer, *timer = timer_init(&__timer, current, time);
        current->state = PROC_SLEEPING;
        current->wait_state = WT_TIMER;
        add_timer(timer);
    ……
        schedule();
        del_timer(timer);
        return 0;
    }

可以看出，do_sleep首先初始化了一个定时器timer，设置了timer的proc是当前进程，到期时间expires是参数time；然后把当前进程的状态设置为等待状态，且等待原因是等某个定时器到期；再调用schedule完成进程调度与切换，这时当前进程已经不占用CPU执行了。当定时器到期后，run_timer_list会删除timer且唤醒timer对应的当前进程，从而使得当前进程可以继续执行。