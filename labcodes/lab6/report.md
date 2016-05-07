## [练习0]填写已有实验
> 对于lab6来说，并不是简单的拷贝前面的成果，而需要作出一些改动。
第一个修改点是在lab5中alloc_proc()函数，需要额外初始化几个与调度相关的变量：
```
proc->rq = NULL;
proc->run_link.prev = proc->run_link.next = NULL;
proc->time_slice = 0;
proc->lab6_run_pool.left = proc->lab6_run_pool.right = proc->lab6_run_pool.parent = NULL;
proc->lab6_stride = 0;
proc->lab6_priority = 0;
```
第二个修改点是在lab1的trap_dispatch(struct trapframe *tf)函数中，需要调用sched_class_proc_tick()函数：
```
sched_class_proc_tick(current);
```
主要是以上两处需要修改。

## [练习1]使用Round Robin调度算法(不需要编码)
> 用meld比较之后可以发现两者差异不大，主要是多了一些跟调度相关的函数和变量。
之后运行，错的出乎意料。仔细排查后，发现default_sched.c里的RR_enqueue()函数的assert(list_empty(&(proc->run_link)))有问题，
思考之后，初始化链表时就判断链表为空必然出错，注释掉后用make grade通过测试，除priority.c没有过去。
```
eric@eric-ThinkPad-Edge-E430:~/桌面/lab6$ make grade
badsegment:              (2.6s)
  -check result:                             OK
  -check output:                             OK
divzero:                 (1.6s)
  -check result:                             OK
  -check output:                             OK
softint:                 (1.5s)
  -check result:                             OK
  -check output:                             OK
faultread:               (1.5s)
  -check result:                             OK
  -check output:                             OK
faultreadkernel:         (1.6s)
  -check result:                             OK
  -check output:                             OK
hello:                   (1.5s)
  -check result:                             OK
  -check output:                             OK
testbss:                 (1.6s)
  -check result:                             OK
  -check output:                             OK
pgdir:                   (1.6s)
  -check result:                             OK
  -check output:                             OK
yield:                   (1.6s)
  -check result:                             OK
  -check output:                             OK
badarg:                  (1.5s)
  -check result:                             OK
  -check output:                             OK
exit:                    (1.5s)
  -check result:                             OK
  -check output:                             OK
spin:                    (1.7s)
  -check result:                             OK
  -check output:                             OK
waitkill:                (2.2s)
  -check result:                             OK
  -check output:                             OK
forktest:                (1.5s)
  -check result:                             OK
  -check output:                             OK
forktree:                (1.6s)
  -check result:                             OK
  -check output:                             OK
matrix:                  (13.2s)
  -check result:                             OK
  -check output:                             OK
priority:                (11.6s)
  -check result:                             WRONG
   -e !! error: missing 'sched class: stride_scheduler'
   !! error: missing 'stride sched correct result: 1 2 3 4 5'
  -check output:                             OK
Total Score: 163/170
make: *** [grade] 错误 1
```
RR调度算法的整个执行过程如下：首先在ucore初始化函数kern_init()里调用sched_init()，在sched_init()函数里定义调度类sched_class = &default_sched_class，而default_sched_class在schedule文件底下使用函数指针做了很详细的定义，这样就可以完成整个RR调度算法。

[练习1.1]请理解并分析sched_calss中各个函数指针的用法,并接合Round Robin调度算法描ucore的调度执行过程。
> sched_class函数定义如下：
```
struct sched_class {
    const char *name;
    void (*init)(struct run_queue *rq);
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
};
```
在sched_class当中，第一个指针指向了调度器的名字，第二个指针是执行运行队列的初始化，第三个指针函数是把一个进程加入到当前的队列当中，第四个指针函数是把一个进程从进程的队列当中删除出去，第五个指针函数是选择下一个可运行的任务，最后一个函数指针是处理时间片的更新。  
ucore当中的RR算法是通过sched_class_pick_next从进程的运行队列当中不断选择下一个进程出来进行运行，用sched_class_proc_tick进行时间片计时更新，当一个进程时间片用完后，使用sched_class_dequeue删除该进程，当来了一个新进程时，使用sched_class_enqueue使该进程进入队列，这样就可以完成整个调度过程。

[练习1.2]请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“,给出概要设计,鼓励给出详细设计。
> 关于多级反馈队列算法的实现，我们按照ucore的实际情况，设有N个队列（Q1,Q2....QN），其中各个队列对于处理机的优先级是不一样的，也就是说位于各个队列中的作业(进程)的优先级也是不一样的。一般来说，优先级Priority(Q1) > Priority(Q2) > ... > Priority(QN)。怎么讲，位于Q1中的任何一个作业(进程)都要比Q2中的任何一个作业(进程)相对于CPU的优先级要高。对于某个特定的队列来说，里面是遵循时间片轮转法。也就是说，位于队列Q2中有N个作业，它们的运行时间是通过Q2这个队列所设定的时间片来确定的（为了便于理解，我们也可以认为特定队列中的作业的优先级是按照FCFS来调度的）。各个队列的时间片是随着优先级的增加而减少的，也就是说，优先级越高的队列中它的时间片就越短。同时，为了便于那些超大作业的完成，最后一个队列QN(优先级最低的队列)的时间片一般很大(不需要考虑这个问题)。  
进程在进入待调度的队列等待时，首先进入优先级最高的Q1等待;首先调度优先级高的队列中的进程。若高优先级中队列中已没有调度的进程，则调度次优先级队列中的进程。例如：Q1,Q2,Q3三个队列，只有在Q1中没有进程等待时才去调度Q2，同理，只有Q1,Q2都为空时才会去调度Q3;对于同一个队列中的各个进程，按照时间片轮转法调度。比如Q1队列的时间片为N，那么Q1中的作业在经历了N个时间片后若还没有完成，则进入Q2队列等待，若Q2的时间片用完后作业还不能完成，一直进入下一级队列，直至完成;在低优先级的队列中的进程在运行时，又有新到达的作业，那么CPU马上分配给新到达的作业（抢占式）。这样的方法就大体实现了多级反馈队列的算法。

## [练习2]实现Stride Scheduling调度算法(需要编码)
> 在填写完代码完成调试之后得到了正确的结果，下面详细说明填写的内容：
```
#define BIG_STRIDE 4096 /* you should give a value, and is ??? */
```
第一个需要确定big_stride的值，这个值需要是一个2的若干次方，值要稍微大一些，这里我填写了4096，从最终的结果来看没有问题。
```
static void
stride_init(struct run_queue *rq) {
     /* LAB6: YOUR CODE
      * (1) init the ready process list: rq->run_list
      * (2) init the run pool: rq->lab6_run_pool
      * (3) set number of process: rq->proc_num to 0
      */
    list_init(&rq->run_list);
    rq->lab6_run_pool=NULL;
    rq->proc_num=0;
}
```
注释给的很清楚，建立运行进程的list，对于run_pool进行初始化，把运行进程的计数器设置为0。
```
static void
stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE
      * (1) insert the proc into rq correctly
      * NOTICE: you can use skew_heap or list. Important functions
      *         skew_heap_insert: insert a entry into skew_heap
      *         list_add_before: insert  a entry into the last of list
      * (2) recalculate proc->time_slice
      * (3) set proc->rq pointer to rq
      * (4) increase rq->proc_num
      */
      rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
     if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
     	proc->time_slice = rq->max_time_slice;
     }
     proc->rq = rq;
     rq->proc_num ++;
}
```
这一段是入队列代码，按照注释当中使用提示的skew_heap_insert函数进行插入即可，对于进程的time_slice变量使用rq的max_time_slice变量重新计算，把proc->rq的值置为rq，最后增加进程总数即可。
```
static void
stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE
      * (1) remove the proc from rq correctly
      * NOTICE: you can use skew_heap or list. Important functions
      *         skew_heap_remove: remove a entry from skew_heap
      *         list_del_init: remove a entry from the  list
      */
    rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
    rq->proc_num--;
}
```
这一段是出队列代码，按照提示，使用remove函数删除，随后减少进程总数即可。
```
static struct proc_struct *
stride_pick_next(struct run_queue *rq) {
     /* LAB6: YOUR CODE
      * (1) get a  proc_struct pointer p  with the minimum value of stride
             (1.1) If using skew_heap, we can use le2proc get the p from rq->lab6_run_poll
             (1.2) If using list, we have to search list to find the p with minimum stride value
      * (2) update p;s stride value: p->lab6_stride
      * (3) return p
      */
     if (rq->lab6_run_pool == NULL) return NULL;
     struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
     if (p->lab6_priority == 0)
          p->lab6_stride += BIG_STRIDE;
     else p->lab6_stride += BIG_STRIDE / p->lab6_priority;
     return p;
}
```
这个函数是挑选下一个运行的进程，队列当中stride值最小的，随后更新stride值返回相应的进程。
```
static void
stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
     /* LAB6: YOUR CODE */
     if (proc->time_slice > 0) {
          proc->time_slice --;
     }
     if (proc->time_slice == 0) {
          proc->need_resched = 1;
     }
}
```
对于进程时间tick的处理，仿照说明中写即可。

## 说明：RR调度算法在default_sched.c中，stride调度算法在default_sched_stride_c中，需要使用时直接拷进去就行。

## 参考答案有问题，make grade根本无法通过，参照注释修改和完成的。

## 本实验重要的知识点以及与对应的OS原理中的知识点
进程的调度算法，包括Round Robin算法，Stride Scheduling算法。

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点
OS原理中讲到了其他的调度算法，如多级反馈队列算法，公平共享调度等，这些实验里都没有实现。



