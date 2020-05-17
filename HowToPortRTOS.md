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
  * 函式有Hook名稱, 可新增內容提供額外的客製化功能。  
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
  *--p_stk = (CPU_STK)0x01000000u;                        /* xPSR                                                   */
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
#define  OS_ENTER_CRITICAL()  {cpu_sr = OS_CPU_SR_Save();}
OS_CPU_SR_Save            ;儲存RRIMASK到R0(返回值), 並關閉中斷
    MRS     R0, PRIMASK   ;Set prio int mask to mask all (except faults)
    CPSID   I             ;設定interrupt disable                    
    BX      LR
```
* 離開critical section:  
```c  
#define  OS_EXIT_CRITICAL()   {OS_CPU_SR_Restore(cpu_sr);}
OS_CPU_SR_Restore         ; 
    MSR     PRIMASK, R0   ;設定interrupt enable 之前的OS_CPU_SR_SAVE, PRIMASK是暫存器,設定Interrupt的Enable/Disable
    BX      LR
```



## Step 3: 驗證task level context switch是否正常
* Task與Task之間的同步(synchronization); 當兩個task同時存取一個resource時會發生。  

## Step 4: 驗證ISR level的context switch是否正常  
* ISR與Task之間的同步; Task因外部觸發(ISR)而從Pending轉態為Ready,進而轉態Running State。




