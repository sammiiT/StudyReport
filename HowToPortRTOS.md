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
## Step 1: 驗證task stack initialization
* 實做並測試Process Stack Initial是否正常:  
  * Stack alignment: 以AAPCS中定義,function call的stack top要在4 bytes alignment,進入interrupt時的stack top 必須要在8 bytes alignment; 中斷發生時的8byte對齊會自動發生, 但須設定暫存器???  
  * Stack中的register store順序 (order):  
    * 圖片:  
  * 
  
* 實做並測試

## Step 2: 驗證task level context switch是否正常
* Task與Task之間的同步(synchronization); 當兩個task同時存取一個resource時會發生。  

## Step 3: 驗證ISR level的context switch是否正常  
* ISR與Task之間的同步; Task因外部觸發(ISR)而從Pending轉態為Ready,進而轉態Running State。




