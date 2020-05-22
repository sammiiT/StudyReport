# ARM CortexM Interrupt Chap7.8.9

## Exception Type:
*  type 1-15的 excetpion 和 type 16-255的external interrupt  
*  沒有type 0的exception  
*  excetpion發生時會將pending state register舉起來,直到對應的handler執行為止; 即使 exception被mask,其pending state還是會舉起。 傳統的ARM processor是hold住IRQ或FIQ一直到執行完成, 現在用pending state register來取代。 REQUEST是外部訊號,hold住才會執行interrupt handler;若沒有hold住,就不會執行???

## Vector Table:  
*  vector table列表所有interrupt/exception handler的位址,和initial stack top; 通常是放在VMA的0x00000000位址。  
*  vector table可以relocate到其他位址,再透過VTOR暫存器重新指向。可利用此機制設計bootloader。  

## Interrupt Input & Pending behavior  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/InterruptPendingBehavior.PNG)  
*  當interrupt發生時,會先進入pending state,待cpu處理此interrupt request, 當cpu進入handller mode處理此中斷,此pending state會被自動清除,如左圖。processor進入handler模式, 則pending state 被清除。  
*  如果cpu一直不執行此中斷要求, pending state會一直維持在waiting 狀態。  
*  若cpu尚未進入handler模式, pending state就被手動清除, 則此interrupt request不會執行對應的響應。如右圖。  
*  即使interrupt被mask, interrupt request發生Pending state還是會舉起來。這個時候pending state需要手動清除。  
*  pending state可用來做為software interrupt的開關, 設定pending state = 1則會發出對應的中斷。  

![](https://github.com/sammiiT/Study-Report/blob/master/picture/InterruptPendingBehavior2.png)  
*  interrupt active bit:執行handler mode時會自動舉起, interrupt return時會自動清除。如左圖。  
*  若interrupt request維持不清除, 當handler mode結束時會pending state會再舉起, 則interrupt handler會再執行一次。如右圖。  

![](https://github.com/sammiiT/Study-Report/blob/master/picture/InterruptPendingBehavior3.png)  
*  在pending state舉起時, 若interrupt request連續觸發, handler只會執行一次。左圖。  
*  在執行handler mode時, 若相同的interrupt request再次觸發, 此次觸發會讓pending state再次舉起, 待handler mode執行結束時會在進入執行(Tail chainning interrupt)。 如右圖。

### 結論
*  request 控制 pending status。  
*  pending status 控制cpu是否進入handler mode; pending status可以由使用者來控制。 
*  pending status 同時也控制active status。  
*  active status與handler的動作時間是一樣的。

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
    * 發生interrupt時, LR會儲存EXC_RETURN的數值; 而Function Call時,LR會記錄的caller下一個執行指令,返回時用到。          

## Interrupt/Exception結束:
* Unstacking: MSP或PSP的內容pop回CPU register  
* NVIC register update:  
    
note: interrupt/exception 觸發,結束; 上述動作都是自動完成, 使用者不須自己寫程式完成。  
note: cortex-m3 rev.2之前的版本的stack frame是4byte aligned, rev.2之後的版本是8 bytes aligned; 可經由NVIC暫存器的STKALIGN bit來設定。  
note: Exception不支援重入(reentrant)。  

## Tail-chaining Interrupt:  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/Tail-Chaining.PNG)
* 當processor在handler mode時, 發生了相同權限或較低權限的exception, 這些exception會進入pending state(chap7)一直到當下的interrupt handler完成。   
* handler結束,會直接執行pending的exception, 不會執行stack/unstack的動作, 因為與前一個結束的handler處於同一個狀態。
* 可改善interrupt latency,因為少了一次stack的push/pop。

## Late Arrival: 後到先執行  
*  當系統因interrupt/exception而執行stacking動作時, 又發生了權限更高的exception, 會先將stacking的動作完成(low and ?high?), 接著執行高權限的handler。  

## PendSV (Pendable Service Call):  
*	在討論PendSV之前,需討論因OS system tick產生的同步行為:  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/SystickContextSwitch.PNG)   
*	上圖是兩個task, 經由systick觸發而交換CPU的執行權。在沒有任何其他外部觸發(external interrupt)的情況下, 此模型是正確的。  
*	Systick exception執行權預設是高於其他的external interrupt, 則當external interrupt handler執行時發生Systick Exception, 會有如下問題:  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/ProblemContextSwIRQ.PNG)
	*	當system tick exception結束會執行context switch, 此Context switch會造成原本被preempt的interrupt handler延遲; 如上圖。  
	*	當interrupt active status=1的情況下, cortex-M 並不允許執行thread mode, 執行系統會產生usage fault exception.  

*	解決上述問題:  
	*	使用PendSV功能, 將所產生的context switch延遲到ISR執行完畢,在接著執行system tick handler執行的context switch。  
	*	system tick Exception的priority設定為最低權限, 可以讓權限較高的Interrupt先執行。 
	

*	Question: 若ISR執行結果要發生context switch, system tick exception執行結果也要發生context switch,結果會如何??

* Reference: The Definitive Guide to The ARM cortex-M3 Second Edition
