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
    * 0 = MSP (default), 此pointer會指向linker定義的initial_sp的起始位址。  
    * 1 = PSP  
    * 在unprivileged thead 模式之下只能存取PSP, 在privileged thread 模式可以存取MSP和PSP。  
    * Stack模式可用CONTROL register來控制, 也可以用EXEC_RETURN的 bit[2]來設定用MSP或PSP。(Chapter 8)
*  當User mode發生exception時, PSP和MSP之間是自動切換。(Chapter8)  
      
## CONTROL[0]與 operation mode switch  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/OperationMode.png)  
*  系統一開機時,是在privileged mode, 經由設定CONTROL[0]=1, 會切換到User Mode(unprivileged)。  
*  在User mode沒有存取CONTROL暫存器的權限, 只有在Exception/Interrupt時才能再次設定CONTROL暫存器。  
*  當control[0]設定為1時, control[1]會變成多少,會自動切換到PSP嗎??? 




Reference: The Definitive Guide To The ARM Cortex-M3
