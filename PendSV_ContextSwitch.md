#	Context Switch by PendSV  
*	利用systick exception和PendSV exception來實現context switch。
*	目的是為了解決在執行Interrupt handler時,OS介入造成context switch,而產生usage fault的風險。  
*	參考ArmCortexM_Interrupt.md的問題描述。  

##	Example: 實作 ArmCortexM_Interrupt.md中的OS tick同步行為 
* 範例描述如下圖:  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/SystickContextSwitch.PNG)   
  *	2個task: taskA,taskB; 每一個task有自己獨立的stack。  
	*	systick timer每1ms觸發sysTick exception, 在handler中利用pendSV執行context switch; 也就是round robin schedule模式。
	*	systick exception結束後的pendSV會觸發context switch, task和task之間就是透過此機制輪流執行。  

*	**System tick**:  
```c  
/* SCB->ICSR 就是 *(uint32_t*)(0xE000ED04), 定義在header file */
#define SCB_ICSR_PENDSVSET_Msk 0x10000000
volatile uint32_t tick_cnt=0;	/* systick counter */  
void SysTick_Handler(void){
	tick_cnt++;
	switch(cur){  
	case 0: next = 1;
		break;
	case 1: next = 0;
		break;
	default:
		for(;;){;}//fault
	break;
	}
	
	//將interrup control state register(0xE000ED04)的bit[28]=1, 則當systick_handler結束之後,會進入pendsv_handler
	if (curr_task!=next_task){
		SCB->ICSR |= SCB_ICSR_PENDSVSET_Msk; //Set PendSV to pending										
	}
}
```

*	**Parameters for Task operation**:
```c  
uint32_t taskA_stack[1024], taskB_stack[1024];/* 每一個stack有1kb */
uint32_t _psp[2];       /* 紀錄stack pointer指向的位址,[0]指向taskA_stack,以此類推 */
uint32_t cur = 0;      /* current task index */
uint32_t next = 1;     /* next task index */
```  
*	**Task Function**:  
```c  
void taskA(void){  
	for(;;){  
		if (tick_cnt & 0x100){//256ms: 閃爍週期512ms  
			bsp_LedOn(1);  
		}else{  
			bsp_LedOff(1);  
		}
	}  
}	 
void taskB(void){  
	for(;;){  
		if (tick_cnt & 0x200){//512ms: 閃爍週期1024ms  
			bsp_LedOn(2);  
		}else{  
			bsp_LedOff(2);   
		}
	}
}
```  
*	**Task initial**:  
![](https://github.com/sammiiT/StudyReport/blob/master/picture/stack_initial.png)
```c  
void OS_Start(void)
{
    //taskA的Stack 如上圖
    _psp[0] = ((unsigned int) taskA_stack) + 1024 - 16*4;
    HW32_REG((_psp[0] + (14<<2))) = (unsigned long)taskA;     // PC
    HW32_REG((_psp[0] + (15<<2))) = 0x01000000;               // xPSR

    //taskB的stack
    _psp[1] = ((unsigned int) taskB_stack) + 1024 - 16*4;
    HW32_REG((_psp[1] + (14<<2))) = (unsigned long)taskB;     // PC
    HW32_REG((_psp[1] + (15<<2))) = 0x01000000;               // xPSR

    //taskA先執行
    cur = 0;

    //設定PSP(stack pointer)的指向的位址
    __set_PSP((_psp[cur] + 16*4));

    //設定PendSV的priority為最低優先層級
    NVIC_SetPriority(PendSV_IRQn, 0xFF);

    //設定systick handler的中斷時間
    SysTick_Config(SystemCoreClock / 100);

    //使用unprivileged thread mode, process stack pointer
    __set_CONTROL(0x3);

    //執行instruction synchronize barrier
    __ISB();

    //執行TaskA
    taskA();
}
```  

*	PendSV Exception: 當systick handler結束之後會接著執行PendSV handler  
	*	PendSV_Handler分為兩部分PUSH舊的task context和POP新的task context  
	*	**PUSH to Stack**  
	![](https://github.com/sammiiT/StudyReport/blob/master/picture/pendSV_push.png)  
	*	**POP from Stack**  
	![](https://github.com/sammiiT/StudyReport/blob/master/picture/pendSV_pop.png)
```c  
__asm void PendSV_Handler(void)
{
                            /* stack r0-r3, r12, lr, pc, xpsr when SystemTick handler(Tail Chain interrupt) */
    MRS R0, PSP             /* 取得當下的stack pointer值; r0-r3, r12, lr, pc, xpsr已經保存 */

    STMDB R0!,{R4-R11}      /* 保存剩下的r4-r11到_psp[]l push */
    LDR R1,=__cpp(&cur)//
    LDR R2,[R1]             /* 得到task id */
    LDR R3,=__cpp(&_psp)
    STR R0,[R3, R2, LSL #2] /* r0 = [r3 + r2<<2] */
                            /* 保存 PSP 到對應的 psp[] 中 R0 = [R3 + R2 << 2] */

    /* 左移兩位(乘四倍)是因為 _psp 是型態4bytes的array */
    /* 載入下一個任務 */
    LDR R4,=__cpp(&next)
    LDR R4,[R4] /* 得到下一個task id 得到下一个任務的ID */
    STR R4,[R1] /* 設定 cur = next */
    LDR R0,[R3, R4, LSL #2] /* 從 _psp 中取得process stack的值 */
    LDMIA R0!,{R4-R11}  /* 將新的stack內容load到R4-R11中; pop; 順便紀錄新的stack top */
    MSR PSP, R0         /* 新的stack top指派給 psp(process stack pointer)*/
    BX LR               /* 這邊的LR儲存的是EXEC_RETURN的數值,在return時通知回到thread MSP, thread PSP或 handler */ 
                        /* 這邊的xPSR, PC, LR, R12, R0-R3會自動恢復, POP到 cpu register */
    ALIGN 4
}
```
*	Systick exception --> PendSV exception 狀態圖:  
![](https://github.com/sammiiT/StudyReport/blob/master/picture/SysTickToPendSV.png)
