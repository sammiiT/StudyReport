# ARM CortexM Interrupt Chap7.8.9

p172 , p59
## Interrupt/Exception Sequences:  
* **Stacking**:   
    * 將r0-r3,r12,lr,pc,psr, 8個暫存器堆疊至stack; 此stack是main stack 或 process stack, 由當下Control Register決定; CONTROL[1]。  
    * 在Interrupt/Exception handler中, 永遠用main stack。
    * 依據AAPCS, function call 或 Interrupt/Exception發生時, stack pointer指向的位址必須要在double word alignment。

* **Vector fetch**:  
    * Instruction bus會到vector table取出對應的interrupt handler。  
    * Data bus在執行stacking動作的同時, instruction bus就會開始對vector table作取址的動作。  
 
* **Update Stack pointer, link register, program counter**:  
    * SP更新: 更新MSP或PSP  
    * PSR更新:  
    * LR更新:link register會被更新為；  
        * 0xFFFFFFF1: 表示return時回到handler mode。  
        * 0xFFFFFFF9: 表示return時回到thread mode用main stack。  
        * 0xFFFFFFFD: 表示return時回到thread mode,用process stack。

## Interrupt/Exception結束:
* Unstacking: MSP或PSP的內容pop回CPU register  
* NVIC register update:  
    
note: interrupt/exception 觸發,結束; 上述動作都是自動完成, 使用者不須自己寫程式完成。
note: cortex-m3 rev.2之前的版本的stack frame是4byte aligned, rev.2之後的版本是8 bytes aligned; 可經由NVIC暫存器的STKALIGN bit來設定。
note: Exception不支援重入(reentrant)。


# Tail-chaining Interrupt:  
* 當processor在handler mode時, 發生了相同權限或較低權限的exception, 這些exception會進入pending state(chap7)一直到當下的interrupt handler完成。   
* handler結束,會直接執行pending的exception, 不會執行stack/unstack的動作, 因為與前一個結束的handler處於同一個狀態。
* 可改善interrupt latency,因為少了一次stack的push/pop。


# Late Arrival:







* Reference: The Definitive Guide to The ARM cortex-M3 Second Edition
