# Cortex-M Operation Mode  Chapter 3  
* cortex-M處理器支援兩種工作模式,兩種權限層級  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/Mode%26Privileged.PNG)

## Privileged & User(Unprivileged) Mode
* Priviledged Mode: 高權限模式,此模式下可存取cpu的任何資源, 主要是special function register。
* Unprivileged Mode: 無法存取 system control space(SCS),存取special function register的指令(如MSR,MRS)無法使用,在User mode存取SCS,會發生fault exception。
* 兩個層級由CONTROL register的bit[0]來控制:  
    * 0 = privileged。  
    * 1 = unprivileged (user mode)。  
    * 當系統在handler mode, 則一定是在privileged層級。


## Handler & Thread Mode  
* Handler Mode: Exception/Interrupt發生時,ISR的執行就是在Handler Mode。  
* Thread Mode: 在一般狀態下, 屬於Thread Mode。

## Stack Mode:  
* 堆疊有兩種模式, MSP(main stack pointer), PSP(process stack pointer)  
* Stack modeu由CONTROL register的bit[1]來控制:  
	*	0 = MSP (default), 此pointer會指向linker定義的initial_sp的起始位址。 
  * 1 = PSP  
  * 在privileged thread 模式可以存取MSP和PSP, unprivileged thead 模式之下也支持此屬性, 但不建議在unprivileged mode下使用MSP,目的是為了將OS kernel(用MSP)和一般的Appllication(用PSP)做區分 。  
  *	Stack模式可用CONTROL register來控制, 也可以用EXEC_RETURN的 bit[2]來設定用MSP或PSP。(Chapter 8)   
	*	EXEC_RETRUN是在系統在handler mode(exception/interrupt)時的LR數值, 可藉由return回thread mode時設定其bit[2]來控制是回到MSP或PSP。 參考HowToPortRTOS.md 的context switch最後的描述, 即為從handler mode在跳回thread mode時,利用EXEC_RETURN來決定MSP或PSP設定。  
			
	```as  
	LDM     R0, {R4-R11}      ; Restore r4-11 from new process stack  
	ADDS    R0, R0, #0x20     ; stack pointer往上移0x20位址  
	MSR     PSP, R0           ; Load PSP with new process SP ;開始設定新process的stack pointer指向  
	ORR     LR, LR, #0x04     ; Ensure EXEC_RETURN process stack  
	                          ; 此時的lr是EXEC_RETURN所以ORR 0x04代表返回PSP(process stack pointer)  
	                          ; 之前的r0~r3, r12, lr,pc,PSR會自動從PSP, pop回register.
	```
*  當User mode發生exception時, PSP和MSP之間是自動切換。(Chapter8)  
      
## CONTROL[0]與 operation mode switch  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/OperationMode.png)  
*  系統一開機時,是在privileged mode, 經由設定CONTROL[0]=1, 會切換到User Mode(unprivileged)。  
*  在User mode沒有存取CONTROL暫存器的權限, 只有在Exception/Interrupt時才能再次設定CONTROL暫存器。  
*  當control[0]設定為1時(unprivileged thread), control[1]會變成多少,會自動切換到PSP? 在unprivileged thread mode是可以使用MSP,但不建議;目的是為了將OS kernel(用MSP)和一般的Appllication(用PSP)做區分。

## Stack and Unstacking while changing operation mode:  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/Stack%26Unstack.PNG)  
*  Interrupt/Exception發生時的context switch屬於caller save。  
*  context 會先push到caller的stack, 若當下caller是用PSP, 則會將register push到PSP,在接著執行handler mode。如上圖。若caller當下是用MSP, 則register會被push到MSP之後再處理handler。
*  Context返回會經由EXEC_RETURN來判斷是從MSP或是從PSP pop back回cpu register。(Chapter 9)


Reference: The Definitive Guide To The ARM Cortex-M3
