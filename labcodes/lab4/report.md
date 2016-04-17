## [练习0]填写已有实验

## [练习1]分配并初始化一个进程控制块(需要编码)
> 
本练习主要就是对进程控制块进行一些初始化操作，有几个比较特殊，参照实验参考书，如：
proc->state=PROC_UNINIT;//设置进程为“初始”态   
proc->pid=-1;//设置进程pid的未初始化值   
proc->cr3=boot_cr3;//使用内核页目录表的基址   
其他runs/kstack/need_resched/parent/mm/context/tf/flags/name类型一般为0或者null。   

[练习1.1]请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥?(提示通过看代码和编程调试可以判断出来)
> 
```
struct context当中保存的是八个寄存器的值
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```
属于进程的上下文,用于进程切换,当进行进程切换的时候我们保存context结构，使用context保存寄存器的目的就在于在内核态中能够进行上下文之间的切换。   
tf是中断帧的指针,总是指向内核栈的某个位置    
```
struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} 
```
它主要记录了进程在用户态和内核态切换时进程在被中断前的状态。与context的区别在于它属于一个进程内部状态的变化而不属于进程间的状态变化。


## [练习2]为新创建的内核线程分配资源(需要编码)
> 
代码很简单，如下：
```
proc=alloc_proc();
setup_kstack(proc);
copy_mm(clone_flags, proc);
copy_thread(proc, stack, tf);
proc->pid=get_pid();
hash_proc(proc);
list_add_before(&proc_list, &(proc->list_link));
wakeup_proc(proc);
ret=proc->pid;
```
过程是这样的：首先调用alloc_proc,首先获得一块用户信息块;然后为进程分配一个内核栈;接着复制原进程的内存管理信息到新进程(但内核线程不必做此事);再复制原进程上下文到新进程;再将新进程添加到进程列表;再唤醒新进程;最后返回新进程号。如果出现了复制内存的问题，调用put_kstack(proc)进行处理;如果出现了建立内核堆栈的问题，则调用kfree(proc)进行异常处理。

  
[练习2.1] 请说明ucore是否做到给每个新fork的线程一个唯一的id?请说明你的分析和理由。
> 
由代码proc->pid = get_pid()可知，查看get_pid()的函数即可，如下：
```
// get_pid - alloc a unique pid for process
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}
```
整个有0-4095编号，这里的last_pid是最终的编号，可以看出它一直在递增，
如果这些进程号码被装满的话，这里的repeat结构会起到作用，它会一直监
听直到一个进程被释放掉，last_pid就会返回这个进程号码，因而有了这
一套机制，ucore是一定可以保证进程可以被分配唯一的id。


## [练习3]阅读代码,理解	proc_run函数和它调用的函数如何完成进程切换的。(无编码工作)
> 
proc_run函数如下：
```
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);
    }
}
```
这个函数的执行过程首先是判断当前的线程是不是要切换到的线程，如果是就省事了，如果不是就进行如下的操作： 第一个是要进行中断，随后把线程更新为要切换到的线程，之后修改esp0和cr3，设置任务状态段ts中特权态0下的栈顶指针esp0为next内核线程，设置CR3寄存器的值为next内核线程initproc的页目录表起始地址next->cr3,这实际上是完成进程间的页表切换，由switch_to函数完成具体的两个线程的执行现场切换,即切换各个寄存器,当switch_to函数执行完“ret”指令后,就切换到initproc执行了。在整个实验的过程当中，实际上是创建了两个内核进程，第一个proc_init当中的id为0的进程，即idleproc。而第二个内核进程是id为1的initproc。

## 本实验重要的知识点以及与对应的OS原理中的知识点
内核线程创建/执行的管理过程  
内核线程的切换和基本调度过程  

## 列出你认为OS原理中很重要,但在实验中没有对应上的知识点



