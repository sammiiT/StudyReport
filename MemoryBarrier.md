# Memory Barrier  
* 程式在運行時memory被存取的順序往往會與程式碼存取的順序不一定一致,因而造成memory存取時的錯亂, Memory barrier可以讓CPU或編譯器在存取memory時有序。  

* 以下參考於Keil infocenter
* ARM處理器會按照程序的順序來執行指令或訪問數據。而最新的ARM處理器會執行指令和訪問數據的順序進行優化, ARMv6/v7架構處理器會對以下指令進行優化, 例如:  
```as
LDR r0, [r1]    
STR r2, [r3]  
``` 
* 假設第一條LDR指令cache miss,這樣cache就會填充, 這個動作一般會暫用好幾個clock周期的時間。舊架構的ARM處理器(帶cache的), 比如ARM926EJ-S會等待這個動作完成, 再執行下一條STR指令。而ARM v6/v7處理器會識別出下一條指令(STR)並不需要等待第一條指令(LDR)完成(不依賴r0的值), 於是就會先執行STR指令, 而不是等待LDR指令完成。  

* 在有些情況下,類似上面提到的這種推測讀取或者亂序執行的處理器優化並不是我們所期望的, 因為可能使程序不按我們的預期執行。在這種情況下, 就有必要“Traditional ARM”行為的程序中插入內存隔離指令。ARM提供了3種內存隔離指令。  
    * **DMB(data memory barrier)**:在DMB之後的內存訪執行前, 保存所有在DMB指令之前的內存訪問完成。DMB前面的LOAD/STORE讀寫的最新值的acknowledgement在時間上一定先於DMB之後的指令;**(在DMB之後的data memory transfer都必須等待DMB之前的data memory transfer完成
)**。   
    * **DSB(data synchronize barrier)**:DSB前面所有的指令,都必須執行完程, 才會執行DSB後面的指令;**(在DSB之後的任何指令,都必須等待DSB之前的data memory transfer完成)**。  
    * **ISB(instruction synchronize barrier)**:清除(flush)pipeline, 使得所有ISB之後的執行指令都是從cache或內存中獲得的(而不是從pipeline中)。  
 
**Note**: DMB和DSB區別在於DMB可以繼續執行之後的指令, 只要這條指令不是內存訪問指令(在interrupt中所執行的內存運作, 用DMB隔離不夠嚴謹)。DSB不管他後面是甚麼指令,都會強迫CPU等待DSB之前的指令執行完畢。 而ISB不僅做了DSB所做的事情, 還會將流水線(pipeline)清空。  

## 使用時機:  
* 使用Memory Barrier(DMB,ISB)情況(1):**Memory Remap**  
    ![](https://github.com/sammiiT/Study-Report/blob/master/picture/memRemap.PNG)
    * 在系統初始化之後, 你可能希望關閉flash的映射, 這樣就可以將底部的位置給RAM使用(如上圖右側)。下面的程式碼(永遠在flash中執行)在運行內存複製立成將一些數據複製到內存底部(RAM)前關閉了flash的映射。  
    ```as  
    mov r0, #0
    mov r1, #REMAP_REG  
    str r0, [r1]              ;關閉flash映射 (透過data bus寫入數值至io map register)
    dmb                       ;保證str完成 (確認寫入完成)
    bl  block_copy_routine()  ;複製程式碼到上右圖RAM區域(對底下那一塊做操作)
    isb                       ;清空pipeline(防止 block_copy_routine還沒結束就執行 copied_routine)
    bl  copied_routine()      ;執行複製後的程式碼(RAM裡面)
    ```  
    * 如果沒有STR和BL中間的DMB,  就不能保證STR在複製代碼到底部內存(block_copy_routine)之前已經完成, 因為複製過程可能會通過STR寫入的數據還在寫緩衝過程中就運行。DMB指令強迫所有DMB之前的數據訪問完成。而ISB指令防止了在複製代碼結束前就從RAM中取指令。  
    * 此範例常發生在bootloader完跳躍到User App之後的動作, 開機時bootloader從flash remap到下面的區塊, 初始化系統之後到User App就可以將此區塊重新利用(如copy vector table)。重新利用之前的事前處理,就是上述的程式碼。  

**補充**: The definitive guide to ARM cortexM0 M0+ , p674 
Some cortexM MCU have memory remapping feature to enable the processor to boot up from a boot loader at address 0 and the switch the memory map t execute program in flash alsom from address 0. The memory remapping function might also be able to remap some of the SRAM to address 0. **When switching memory maps**, a DSB and then an ISB should be used  after the switch to ensure that the processor fetches instructions with the new memory map.  
在轉換memory map時,必須使用DSB和ISB。
 
 
* 使用Memory Barrier(DSB)情況(2):**Interrupt**  
    * 上圖表示了通用中斷控制器(GIC)的系統結構。當中斷控制器檢測到中斷發生時, 他會發送nIRQ信號給處理器。這會觸發一系列事件包括運行ISR, 並屏蔽IRQ中斷源。  
    ![](https://github.com/sammiiT/Study-Report/blob/master/picture/InterruptDMB.PNG)  

    * 下面的中斷服務例程通過讀取中斷確認暫存器(IAR)來確認中斷, 不僅返回掛起中高優先級的中斷ID, 還告訴中斷控制器解除nIRQ的信號。在這之後, 中斷服務例程需要重新enable中斷, 以使更高等級的中斷能搶佔(preempt)當前的中斷(NVIC)。  
    ```as  
    Interrupt_Handler
      ;....
      ldr r0,=GIC_CPUIF_BASE  ;獲得中斷控制器base address
      ldr r1, [r0,#0x0c]      ;讀取IAR, 同時解除nIRQ信號(read pending status)
      DSB                     ;確認register;位址 (r0 + 0x0c);存取結束, 並沒有其他指令運行
      CPSIE I                 ;繼續運行前, 開啟中斷
      ;....
      RFE sp!
    ```  
    * 在上述過程中,正確操作需要在CPSIE重新enble前完成IAR的讀取。因為這兩條指令沒有數據依賴,所以CPU會在ldr完成之前就執行CPSIE。 這會導致處理器再次響應相同的中斷,因此在這兩指令中插入一條DSB。
    
* 使用Memory Barrier(DSB)情況(3): **Operation Mode Changing**  
    * 使用CONTROL[0]暫存器來切換operation mode時, 必須確保模式正確切換, 才能往下繼續執行。  
    ```as  
    ldr r0, #0        ; r0寫入 0x00
    msr CONTROL, r0   ; 切換operation mode
    dsb               ; dsb確保, CONTROL register正確寫入數值, operation mode切換完成
    ;...              ; 模式切換之後的動作
    ```
    * 若沒有dsb(data synchronize barrier), 則有可能發生模式尚未完全切換就執行接下來的動作(變成是privileged); 原本是要在user mode執行,卻變成在privileged mode執行。


* 使用Memory Barrier情況(ISB)(4):  
    * Pipeline中能包含過期的指令, 此情況下必須使用ISB。  
    ```as  
    Overlay_manager:
      ;...
      bl  block_copy    ;將新的user app從ROM複製到RAM
      b relocated_code  ;轉跳到
    ```  
    * 如果enable分支預測(branch prediction), 並像上面的例子一樣重定位代碼, 處理器會預測到第二條分支指令即將執行, 然後從其指示例程中取指令, 這樣就會導致處理器運行舊的程式碼(pipeline中的fetch->decode->execute; 優化所造成的結果)。 如果複製例程和跳轉到新例程的分支指令很近, 這樣的問題就會發生。  
    * 為了確保這樣的優化不要發生, 必須在新的重定位的程式碼運行前插入一條ISB指令, 以此來保證預指緩衝在處理器重新取指令前已經被清除:  
    ```as  
    Overlay_manager
      ;...
      bl  block_copy      ;將新例程從ROM複製到RAM
      isb                 ;保證pipeline被清除
      b   relocated_code  ;轉跳到新例程  
    ```
## ARM v7-m Architecture Application Level Reference Manual  
*  DMB:
      *  All explicit memory accesses by instructions occurring in program order before this instruction are globally observed before any explicit memory accesses due to instructions occuring in program order after this instruction are observed.  
      *  The DMB instructinon has no effect on the ordering of other instructions executing on the processor.  
      *  DMB ensures the apparent order of the explicit memory operations before and after the instruction, without ensuring their completion.
*  DSB:  
      *  The DSB instruction operation acts as a special kind of Data Memory Barrier. The DSB operation complete when all explicit memory accesses before this instruction complete.  
      *  The DSB operation complete when all explicit memory access (before this instruction) complete.  
      *  In addition, no instruction subsequent to the DSB can be execute until the DSB completes.  
*  ISB:  
      * ensures that any branches which appear in program order after the ISB are always written into the branch prediction logic with context that is visible after the ISB. This is required to ensurer correct execution of the instruction stream.

## 結論:  
  * 外部記憶體存取(data),內部指令和暫存器操作(instruction),這兩種操作非常接近時, 必須要有memory barrier機制來做保護。  
  * 模式切換前後; 如CONTROL[0], CONTROL[1]的操做; 必須模式完整切換才能繼續執行接下來的指令。  
  * 以pipeline(fetch->decode->execute)的機制來思考memory barrier指令操作會比較容易理解。  

    
