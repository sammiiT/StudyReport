# How To Port RTOS 
* Platform: ARM cortex-M
* RTOS: uCOSII/III

## Processor Requirement  
* Compiler可以產生Reentrant Code。  
* 需要有Systick Interrtupt。  
* 可以在C Code中 enable/disable interrupt。  
* 足夠的RAM(internal或external),提供給若干個Task作為Stack之用。  
* Processor提供指令存取stack或memory的資訊,和所有的cpu register ; 如  
```as
mrs   r0, psp         ; 讀取process stack pointer, 讀取cpu register 
stmdb r0, {r4-r11}    ; r4-r11寫入 stack, 讀寫memory stack
subs  r0, r0, #0x20   ; stack pointer 位移 0x20 bytes (r4-r11), 操作r0 register
```  
## Step 1:Porting時需要更改的相關檔案  
* Data model, Stack, Interrupt Enable/Disable Control:   
  * 確認其資料型態,並給予適當定義 如8bits cpu的data type, 16bits cpu的data type,和32bits cpu的data type。
  * Stack growth(往上或往下), Stack initalization(暫存器的順序)。  
  * 控制中斷Enable/Disable時的狀態紀錄。
  * 與上述相關的檔案是OS_CPU.h, 可由此檔案重新定義對應的資料型態和行為。
* Context Switch相關:  
  * Exception/Interrupt發生時的reigster內容儲存。  
  * Task和Task之間的轉換時,register內容儲存。  
  * 上述機制的相關檔案是OS_CPU_A.asm  
* Kernel 運作時對應的API新增功能:  
  * 函式有Hook名稱, 可新增內容提供額外的客製化功能,或加入測試code。  
  * 在OS_CPU_C.c中可以查詢到相關的API。

## Step 2: Task stack initialization
* 實做並測試Process Stack Initial是否正常:  
  * Stack alignment: 以AAPCS中定義,function call的stack top要在4 bytes alignment,進入interrupt時的stack top 必須要在8 bytes alignment; 中斷發生時的8byte對齊會自動發生。 此屬性的設定,在不cortexM同版本會有差異。  
  * Stack中的register store順序 (order):  
  ![](https://github.com/sammiiT/Study-Report/blob/master/picture/StackInitial.png)  
  * Stack Initialization Sample:  
  ```c  
  CPU_STK  *p_stk;
  p_stk = &p_stk_base[stk_size];                          //stack grows down, set stack bottom at high address;Load stack pointer 
                                                          //this part of initialization is the same as the order of exception  
  *--p_stk = (CPU_STK)0x01000000u;                        /* xPSR   set thumb bit                                                */
  *--p_stk = (CPU_STK)p_task;                             /* Entry Point                                            */
  *--p_stk = (CPU_STK)OS_TaskReturn;                      /* R14 (LR)                                               */
  *--p_stk = (CPU_STK)0x12121212u;                        /* R12                                                    */
  *--p_stk = (CPU_STK)0x03030303u;                        /* R3                                                     */
  *--p_stk = (CPU_STK)0x02020202u;                        /* R2                                                     */
  *--p_stk = (CPU_STK)p_stk_limit;                        /* R1                                                     */
  *--p_stk = (CPU_STK)p_arg;                              /* R0 : argument                                          */
                                                          /* Remaining registers saved on process stack             */
  *--p_stk = (CPU_STK)0x11111111u;                        /* R11                                                    */
  *--p_stk = (CPU_STK)0x10101010u;                        /* R10                                                    */
  *--p_stk = (CPU_STK)0x09090909u;                        /* R9                                                     */
  *--p_stk = (CPU_STK)0x08080808u;                        /* R8                                                     */
  *--p_stk = (CPU_STK)0x07070707u;                        /* R7                                                     */
  *--p_stk = (CPU_STK)0x06060606u;                        /* R6                                                     */
  *--p_stk = (CPU_STK)0x05050505u;                        /* R5                                                     */
  *--p_stk = (CPU_STK)0x04040404u;                        /* R4                                                     */
  ```  
  
## Step 3: Interrupt Enable/Disable  
* 進入critical section:  
```c  
#define  OS_ENTER_CRITICAL()  {cpu_sr = OS_CPU_SR_Save();} /* r0是caller saved所以進入之前必須存入stack */
OS_CPU_SR_Save            ;儲存RRIMASK到R0(返回值), 並關閉中斷
    MRS     R0, PRIMASK   ;Set prio int mask to mask all (except faults)
    CPSID   I             ;設定interrupt disable                    
    BX      LR
```
* 離開critical section:  
```c  
#define  OS_EXIT_CRITICAL()   {OS_CPU_SR_Restore(cpu_sr);} /* r0是caller saved所以進入之前必須存入Stack */
OS_CPU_SR_Restore         ;經過context switch之後, stack已經pop回register,所以之前存的r0狀態會pop回register.
    MSR     PRIMASK, R0   ;設定interrupt enable 之前的OS_CPU_SR_SAVE, PRIMASK是暫存器,設定Interrupt的Enable/Disable
    BX      LR
```
## Step 4: Context Switch  
* Task對Task synchronization; 外部觸發interrupt,ISR對Task發signal  
* context switch流程  
  * 進入context switch之前須將interrtup disable  
  * 取出下一個最高權限的Task control block  
  * 執行contex switch: Cortex M系列的context switch是利用PendSV(pendable service exception)來實做, 參考ArmCortexM_Interrupt.md
  ```as  
  OSCtxSw  
    LDR R0, =0xE000ED04   ;interrupt control state register
    LDR R1, =0x10000000   ;value to trigger PendSV exception
    STR R1, [R0]          ;write 0x10000000 into register 0xE000ED04
                          ;If Context switch occur during ISR is executing,the pendSV exception can't be activated until the current ISR is complete   
    BX  LR                      
  ```
  * PendSV exception handler執行:  
  ```as  
  PendSVHandler		    ; 定義在vector table中的PendSV handler
                              ; 進入PendSVHandler之前,處理器會先將r0-r3,r12,pc,lr,psr 存放到堆疊區
    CPSID   I                 ; disable interrupt
    MRS     R0,PSP            ; 紀錄當下的PSP(process stack pointer)
    SUBS    R0,R0,#0x20       ; stack growth down, 先將R0指向 R0-0x20的位址, 0x20是R4-R11共8個暫存器, 共8*4=32=0x20 
    STM     R0, {R4-R11}      ; STM=STMIA(increment after),將R4-R11由低位址像高位址做堆疊
    
    LDR     R1, =OSTCBCur     ; R1載入 OSTCBCur指標變數的位址,
    LDR     R1, [R1]          ; OSTCBCur位址的儲存值, R1是OSTCBStkPtr指標變數的位址;參考Note  
    STR     R0, [R1]          ; 將新的stack pointer位址存入[R1], 就是OSTCBStkPtr指向的位址  
    PUSH    {R14}             ; caller save, 接下來要呼叫C函式, 避免lr被修改,所以先將lr push到stack中    
    LDR     R0,=OSTaskSwHook  ;  
    BLX     R0                ; 跳到OSTaskSwHook中執行    
    POP     {R14}             ; 將原來的lr pop到register的 r14  
    
    LDR     R0, =OSPrioCur    ; 設定變數OSPrioCur = OSPrioHighRdy;
    LDR     R1, =OSPrioHighRdy
    LDRB    R2, [R1]
    STRB    R2, [R0]
    
    LDR     R0, =OSTCBCur     ; 設定變數 OSTCBCur  = OSTCBHighRdy
    LDR     R1, =OSTCBHighRdy
    LDR     R2, [R1]
    STR     R2, [R0]          ; 將OSTCBCur指標指向最高priority的task control block(OSTCBHighRdy)
    
    LDR     R0, [R2]          ; R0 is new process SP; SP = OSTCBHighRdy->OSTCBStkPtr;
    LDM     R0, {R4-R11}      ; Restore r4-11 from new process stack
    ADDS    R0, R0, #0x20     ; stack pointer往上移0x20位址
    MSR     PSP, R0           ; Load PSP with new process SP ;開始設定新process的stack pointer指向
    ORR     LR, LR, #0x04     ; Ensure exception return uses process stack
                              ; 此時的lr是EXEC_RETURN所以ORR 0x04代表返回PSP(process stack pointer)
    CPSIE   I                 ; enable interrupt
    BX      LR                ; Exception return will restore remaining context
                              ; 直接跳回下一個task, 不會執行 "OS_CPU_PendSVHandler" 以下的動作。
  ```  
  Note: Task control block的型態如下: 
  ```c
  OS_EXT  OS_TCB           *OSTCBCur;
  typedef struct os_tcb {
    OS_STK          *OSTCBStkPtr;
  .....
  }OS_EXT
  ```
  

## Step 5: 驗證OSStart()是否正常  
* 驗證OSTaskStkInit()是否正常, 參考step 2。  
* 驗證OSStartHighRdy(),在沒有create task之下,第一個High ready是IDLE_Task()
* 可以跳到IDLE_Task()則OSStart正常
* 測試程式如下:  
```c  
void main(){
 OSInit();  //會產生Idle task, 同時也會呼叫OSTaskStkInit()
 OSStart(); //會呼叫到OSStartHighRdy(),並執行第一個context switch到Idle task
}
```
* 測試方法:  
  * OSInit():   
    * 呼叫OSTaskCreate產生Idle task.  
    * OSTaskCreate呼叫OSTaskStkInit()  
    * OSTaskStkInit()裡面的stack order沒有設定好(misaligned), 則會產生當機情況
  * OSStart():  
    * 找出highest priority task  
    * 執行OSStartHighRdy(),之後會context switch到Idle task  
    * 如果不會跳到Idle task, 表示在Initial或Context Switch時的Stack Order沒有設定好  
    * 如果可以跳到Idle Task執行, 則表示OSTaskStkInit()和OSStartHighRdy()正常

* 測試實作: 利用LED  
```c  
void main(){
        OSInit();
        Turn ON LED;//執行表示OSInit()
        OSStart();
}
void OSTaskIdleHook(void){//在hook函式中加上test code
        if(LED is ON){//如果正確進入idle task則會 LED OFF
                Turn OFF LED;
        }else{
                Turn ON LED;
        }
}

```  
## Step 6: 測試context switch  
* Task對Task的context switch: OSCtxSW(), 會在OSSched()中被呼叫  
	* 測試方法:  
		*	stack initialization 必須先正常運作.  
		*	需創建一個新的task, 讓它context switch到Idle Task  
		*	OSTimeDly(1)會呼叫OSSched(),進而觸發context switch  
	*	測試程式:  
	```c    
	void main(void){
        OSInit();//產生Idle Task, 並初始化此Task Stack
        OSTaskCreate(TestTask, (void*)0, &TestTaskStk[99], 0); //創建一user task
        OSStart();
	}
	void TestTask(void* pdata){
        pdata = pdata;
        while(1){
        		OSTimeDly(1);	//時間到時會發生context switch,OSTimeDly(1)裡面會呼叫到OSCtxSW();或直接呼叫OSCtxSW()也可以
        }//因為沒有實做OSTickISR和開啟timer,所以OSTimeDly會執行context switch到Idle Task
	}
	```  
	
*	ISR對Task的context switch: OSIntCtxSw(),在OSIntExit()中被呼叫  
	*	interrupt後產生的context switch, 搭配OSTickISR()來測試  
	*	OSTickISR(); 在ARM架構中是OS_CPU_SysTickHandler(), 最後會呼叫到OSIntExit(), 進而呼叫到OSIntCtxSw(); ~任何ISR內都須加上OSIntExit,這樣才會在OSIntExit時有context switch~。  
	*	測試方法: 利用LED  
		*	在OSTickISR裡面不要呼叫OSIntExit(),並將blinking程式放到其中  
		*	如果LED會blinking,則問題發生點就會在OSIntCtxSw()  
		*	如果LED不會blinking,則問題發生點要往前推是否有將timer interrupt設定正確,而導致沒辦法進入OSTickISR  
	*	測試實作:測試OSIntCtxSw()和OSTickISR()  
	```c  
	TestTaskStk[100];
	void main(){
        OSInit();
        Turn LED OFF;
        Install the clock tick interrupt vector;
        OSTaskCreate(TestTask, (void*)0, &TestTaskStk[99],0);//stack grow down
        OSStart();
	}
	void TestTask(void *pdate){
  		BOOLEAN led_state;
    		pdata=pdata;
    		initialize the clock tick interrupt(timer);
    		Enable interrupt;
    		led_state = false;
    		Turn ON LED;
    		while(1){
    			OSTimeDly(1);//有實作tick interrupt,所以會再context switch回來
      			if(led_state==false){  
            			led_state = true;  
            			Turn ON LED;  
        		}else{  
            			led_state = false;  
            			Turn OFF LED;  
        		}  
		}  
	}
	```  
* 若stack initial, context switch, systick, ISR context switch 測試正常, 則porting完成
	
	



