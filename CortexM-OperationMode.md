# Cortex-M Operation Mode  Chapter 3  
* cortex-M處理器支援兩種工作模式,兩種權限層級  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/Mode%26Privileged.PNG)

## Privileged & Unprivileged Mode
* Priviledged Mode: 高權限模式,此模式下可存取cpu的任何資源, 主要是special function register。
* Unprivileged Mode:  
* 兩個層級由CONTROL register的bit[0]來控制:  
    * 0 = privileged。  
    * 1 = unprivileged (user mode)。  
    * 當系統在handler mode, 則一定是在privileged層級。



## Handler & Thread Mode  
* Handler Mode: Exception/Interrupt發生時,ISR的執行就是在Handler Mode。  
* Thread Mode: 在一般狀態下, 屬於Thread Mode







## Stack Mode:  
* 堆疊有兩種模式, MSP(main stack pointer), PSP(process stack pointer)  
* Stack modeu由CONTROL register的bit[1]來控制:  
    * 0 = MSP (default)  
    * 1 = PSP  
    * 在unprivileged 模式之下只能存取PSP, 在privileged 模式可以存取MSP和PSP



Reference: The Definitive Guide To The ARM Cortex-M3
