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
## Step 1: 
* 實做並測試Process Stack Initial是否正常:  
  * Stack alignment
* 實做並測試

## Step 2:
* 測試
