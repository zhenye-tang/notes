## RT-thread 中 hardfault 处理

## 1. 背景

​	当系统进入异常时，通常会进入异常对应的处理函数，当系统中存在非法操作时，比如除0、非对齐访问、爆栈，此时便会触发 hardfault 异常，一般情况下，hardfault 中是个 while(1)，此时系统中存在非法操作，就会被这个 while(1) 永远挂着，但是此时需要查找为什么系统崩了，怎么查？从何处查？通常进入到 hardfault 后调用栈也没了，新手玩家会很头疼，不知从何处查起。因此，系统宕机时，必须拥有足够的信息帮助我们分析宕机原因

## 2. 基础知识

* cm 架构，响应异常的第一个行动，就是自动保存现场的必要寄存器信息，依次会把 psp、pc、lr、r12、r3-r0 由硬件自动压入适当的栈中
* cm 架构是双栈指针设计，当上下文环境是线程环境时，栈指针是psp，当上下文是中断环境时，栈指针为 msp，双栈指针设计天生为 os 而来，这样保证了线程环境与中断环境隔离
* cm 架构的栈是向下生长的
* 任意时刻，仅存在一个 sp 指针，要么是 psp，要么是 msp

## 3. rt-thread 构造栈帧

### 3.1 rt-thread 中的栈

1. 在 rt-thread 中每个线程拥有独立的栈空间，栈空间可以是静态的，也可以是动态的，栈就是一片连续的内存。

   ```c
   struct exception_stack_frame
   {
       rt_uint32_t r0;
       rt_uint32_t r1;
       rt_uint32_t r2;
       rt_uint32_t r3;
       rt_uint32_t r12;
       rt_uint32_t lr;
       rt_uint32_t pc;
       rt_uint32_t psr;
   };
   
   struct stack_frame // 架构不同，栈帧的大小也不同
   {
       /* r4 ~ r11 register */
       rt_uint32_t r4;
       rt_uint32_t r5;
       rt_uint32_t r6;
       rt_uint32_t r7;
       rt_uint32_t r8;
       rt_uint32_t r9;
       rt_uint32_t r10;
       rt_uint32_t r11;
   
       struct exception_stack_frame exception_stack_frame;
   };
   
   /* 初始化线程栈，构造栈帧 */
   rt_uint8_t *rt_hw_stack_init(void       *tentry,
                                void       *parameter,
                                rt_uint8_t *stack_addr,
                                void       *texit)
   {
       struct stack_frame *stack_frame;
       rt_uint8_t         *stk;
       unsigned long       i;
   
       stk  = stack_addr + sizeof(rt_uint32_t);// 栈顶，连续空间的最高地址
       stk  = (rt_uint8_t *)RT_ALIGN_DOWN((rt_uint32_t)stk, 8);
       stk -= sizeof(struct stack_frame);// 向下偏移 sizeof(struct stack_frame) 个大小
   
       stack_frame = (struct stack_frame *)stk;
   
       /* init all register */
       for (i = 0; i < sizeof(struct stack_frame) / sizeof(rt_uint32_t); i ++)
       {
           ((rt_uint32_t *)stack_frame)[i] = 0xdeadbeef;
       }
   
       stack_frame->exception_stack_frame.r0  = (unsigned long)parameter; /* r0 : argument */
       stack_frame->exception_stack_frame.r1  = 0;                        /* r1 */
       stack_frame->exception_stack_frame.r2  = 0;                        /* r2 */
       stack_frame->exception_stack_frame.r3  = 0;                        /* r3 */
       stack_frame->exception_stack_frame.r12 = 0;                        /* r12 */
       stack_frame->exception_stack_frame.lr  = (unsigned long)texit;     /* lr 线程退出时，将执行线程*/
       stack_frame->exception_stack_frame.pc  = (unsigned long)tentry;    /* entry point, pc，线程切换时，进入到用户线程函数 */
       stack_frame->exception_stack_frame.psr = 0x01000000L;              /* PSR */
   
       /* return task's current stack address */
       return stk;
   }
   ```

2. 为什么要构造一个栈帧？

   一个线程初始化完成后，怎样才能进入到线程处理函数中？是由调度器来决定的，调度器选择切换到当前时刻已经就绪的最高优先级的线程，便会手动触发一个 Pendsv 异常，进入 Pendsv 异常时，会将被切线程的上下文保存，将要执行的线程的上下文恢复到寄存器中。通过反推法，如果没有上述构造的初始栈上下文，怎么恢复到寄存器中呢？所以，构造出来一个栈帧后，将上述栈帧中的内容出栈到寄存器中，pc 寄存器中被上述恢复动作赋值成 tentry 这个函数指针，此时将要执行的线程的栈指针 sp（指向栈顶，由于该程线程首次执行，又执行了 push 操作） 更新给 psp，当异常结束返回时，就跳到了线程的处理函数中。如果线程返回了，该线程将永远不会被得到执行，因此，还需给线程收尸，也就是将该线程占用的资源给释放掉，执行构造栈帧时的 texit 函数，此函数地址在 Pendsv 异常处理函数中 push 到了 lr 寄存器中，当函数返回时，必然执行该函数。收尸动作，发生在 idle 线程中，系统空闲时，将这些僵尸线程给释放掉。

3. rt-thread 切换线程细节

   rt-thread 切换线程时，进入PendSv 异常前，psp 此时是被切线程的栈顶，此时硬件自动压入 8 个寄存器的值，还需手动将剩余寄存器压入此线程对应的栈中，由于该线程需要让出处理器的使用权，因此需要将该线程的上下文环境给保存在栈中。

   还有一个关键的步骤，被切线程的上下文已经全部保存在栈中了，此时线程控制块中的 sp 并没有更新，切线程时是根据线程控制块中的 sp 来切的，因此，还需将 psp 的值更新回线程控制块中的 sp 。此时该线程的上下文被保存在栈中了，同时该栈的栈顶也被保存在了线程控制块中，下次恢复上下文时，就有能力恢复了。

## 4. rt-thread 中 HardFault_Handler

1. rt-thread 将 HardFault_Handler 重写了，删除掉了毫无意义的 while(1)，加入了更多的信息

   ```asm
       IMPORT rt_hw_hard_fault_exception
       EXPORT HardFault_Handler
   HardFault_Handler    PROC
   
       ; get current context
       TST     lr, #0x04               ; if(!EXC_RETURN[2])
       ITE     EQ
       MRSEQ   r0, msp                 ; [2]=0 ==> Z=1, get fault context from handler.
       MRSNE   r0, psp                 ; [2]=1 ==> Z=0, get fault context from thread.
   
       STMFD   r0!, {r4 - r11}         ; push r4 - r11 register
       STMFD   r0!, {lr}               ; push exec_return register
   
       TST     lr, #0x04               ; if(!EXC_RETURN[2])
       ITE     EQ
       MSREQ   msp, r0                 ; [2]=0 ==> Z=1, update stack pointer to MSP.
       MSRNE   psp, r0                 ; [2]=1 ==> Z=0, update stack pointer to PSP.
   
       PUSH    {lr} # 压入lr的值来判断异常发生在线程中还是在中断中
       BL      rt_hw_hard_fault_exception
       POP     {lr}
   
       ORR     lr, lr, #0x04
       BX      lr
       ENDP
   
       ALIGN   4
   
       END
   ```

2. 进入异常前，硬件自动压入一些寄存器到栈中，此时的 lr 寄存器中的值代表着进入异常前的环境（线程或中断），这个环境决定着栈指针是 psp 还是 msp。因此，判断这个 lr 寄存器的 bit2 是 0 还是 1 来获取进入异常前使用的是那个栈。然后将不会被硬件压栈的寄存器给压入相应栈（psp 或 msp）中，然后将该栈的栈顶更新到 r0 寄存器中，此时跳入 C 函数中打印相关信息，入参为栈顶地址

   ```c
   struct exception_info
   {
       rt_uint32_t exc_return; // 低地址
       struct stack_frame stack_frame;
   };
   
   void rt_hw_hard_fault_exception(struct exception_info * exception_info)
   {
   #if defined(RT_USING_FINSH) && defined(MSH_USING_BUILT_IN_COMMANDS)
       extern long list_thread(void);
   #endif
       struct stack_frame* context = &exception_info->stack_frame;
   
       if (rt_exception_hook != RT_NULL)
       {
           rt_err_t result;
   
           result = rt_exception_hook(exception_info);
           if (result == RT_EOK)
               return;
       }
   
       rt_kprintf("psr: 0x%08x\n", context->exception_stack_frame.psr);
   
       rt_kprintf("r00: 0x%08x\n", context->exception_stack_frame.r0);
       rt_kprintf("r01: 0x%08x\n", context->exception_stack_frame.r1);
       rt_kprintf("r02: 0x%08x\n", context->exception_stack_frame.r2);
       rt_kprintf("r03: 0x%08x\n", context->exception_stack_frame.r3);
       rt_kprintf("r04: 0x%08x\n", context->r4);
       rt_kprintf("r05: 0x%08x\n", context->r5);
       rt_kprintf("r06: 0x%08x\n", context->r6);
       rt_kprintf("r07: 0x%08x\n", context->r7);
       rt_kprintf("r08: 0x%08x\n", context->r8);
       rt_kprintf("r09: 0x%08x\n", context->r9);
       rt_kprintf("r10: 0x%08x\n", context->r10);
       rt_kprintf("r11: 0x%08x\n", context->r11);
       rt_kprintf("r12: 0x%08x\n", context->exception_stack_frame.r12);
       rt_kprintf(" lr: 0x%08x\n", context->exception_stack_frame.lr);
       rt_kprintf(" pc: 0x%08x\n", context->exception_stack_frame.pc);
   
       if(exception_info->exc_return & (1 << 2) )
       {
           rt_kprintf("hard fault on thread: %s\r\n\r\n", rt_thread_self()->name);
   
   #if defined(RT_USING_FINSH) && defined(MSH_USING_BUILT_IN_COMMANDS)
           list_thread();
   #endif
       }
       else
       {
           rt_kprintf("hard fault on handler\r\n\r\n");
       }
   
   #ifdef RT_USING_FINSH
       hard_fault_track();
   #endif /* RT_USING_FINSH */
   
       while (1);
   }
   ```

   

