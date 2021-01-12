# 系统调用

## 系统调用流程

代码模板和执行过程，或者叫：增加一个系统调用时的步骤，也是代码的执行过程。

1. 增加中断向量和对应的调用。

   1. ```assembly
      get_ticks:
      	mov	eax, _NR_get_ticks
      	int	INT_VECTOR_SYS_CALL
      	ret
      ```

   2. 

   3. ```c
      PUBLIC void init_prot()
      {
      	init_8259A();
      
      	// 全部初始化成中断门(没有陷阱门)
      	// 其他，被省略
      
      	init_idt_desc(INT_VECTOR_SYS_CALL,	DA_386IGate,
      		      sys_call,			PRIVILEGE_USER);
      
      	/* 填充 GDT 中 TSS 这个描述符 */
      	memset(&tss, 0, sizeof(tss));
      	tss.ss0		= SELECTOR_KERNEL_DS;
      	init_descriptor(&gdt[INDEX_TSS],
      			vir2phys(seg2phys(SELECTOR_KERNEL_DS), &tss),
      			sizeof(tss) - 1,
      			DA_386TSS);
      	tss.iobase	= sizeof(tss);	/* 没有I/O许可位图 */
      
      	// 填充 GDT 中进程的 LDT 的描述符
      	int i;
      	PROCESS* p_proc	= proc_table;
      	u16 selector_ldt = INDEX_LDT_FIRST << 3;
      	for(i=0;i<NR_TASKS;i++){
      		init_descriptor(&gdt[selector_ldt>>3],
      				vir2phys(seg2phys(SELECTOR_KERNEL_DS),
      					proc_table[i].ldts),
      				LDT_SIZE * sizeof(DESCRIPTOR) - 1,
      				DA_LDT);
      		p_proc++;
      		selector_ldt += 1 << 3;
      	}
      }
      ```

   4. ```c
      init_idt_desc(INT_VECTOR_SYS_CALL,	DA_386IGate,
      		      sys_call,			PRIVILEGE_USER);
      ```

2. ```assembly
   ; ====================================================================================
   ;                                 sys_call
   ; ====================================================================================
   sys_call:
           call    save
   
           sti
   
           ; eax 是 0。不知道在哪里赋值的。[sys_call_table + eax * 4] 是 sys_call_table 第一个元素。
           ; 根据 PUBLIC	system_call		sys_call_table[NR_SYS_CALL] = {sys_get_ticks}; ， [sys_call_table + eax * 4] 是 sys_get_ticks。
           call    [sys_call_table + eax * 4]
           ; 函数的返回地址，完全不理解。
           ; 没有这句，没发现有什么影响。
           mov     [esi + EAXREG - P_STACKBASE], eax
   
           cli
   
           ret
   ```

3. ```c
   PUBLIC	system_call		sys_call_table[NR_SYS_CALL] = {sys_get_ticks};
   ```

4. ```c
   /*======================================================================*
                              sys_get_ticks
    *======================================================================*/
   PUBLIC int sys_get_ticks()
   {
   	disp_str("+");
   	return 0;
   }
   ```



这种套路，是如何形成的？最终，我需要能够不看书中的代码独立写出操作系统代码。除了记忆，我不知道如何才能让我顺其自然地写出符合上面流程的代码。

上述流程的文字版。

1. 声明并定义个系统调用函数。这个函数并不是真正实现系统调用功能的函数。它的首要功能是，定义中断向量。
2. 将中断向量和门描述符对应起来，同时将中断向量和函数sys_call对应起来。
3. 定义sys_call，在其中调用真正的实现系统调用功能的函数。也可以在这个函数实现系统调用功能。
4. 实现系统调用功能的函数。

这就算理解代码了吗？我总是觉得差点什么。