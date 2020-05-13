# bootloader for Cortex-M MCU

## ARM Cortex-M開機程序
![](https://github.com/sammiiT/Study-Report/blob/master/picture/ResetSequence.PNG)
* bin file的開頭32bits (位址0x00000000), 第二個32bits(0x00000004)儲存Reset_Handler的位址。
* 開機時,cpu會先讀取0x00000000的值,指派給sp; 接著讀取0x00000004的值, 將其指派給pc。

![](https://github.com/sammiiT/Study-Report/blob/master/picture/Initial_SP_PC.PNG)
* sp是0x20008000, pc是0x00000101
* stack pointer往下遞增
* 初始stack位址,和vector table必須從0x00000000開始存放, 因此IAP之後的APP必須 (1)Remap vector到0x00000000 或使用 (2)VTOR(Vector Table Offset Register)。

Note: thumb模式下,PC代的數值的bit[0]設為1

## Bootloader VTOR 
![](https://github.com/sammiiT/Study-Report/blob/master/ScatterLoading-bootloader_VTOR.png)  
-----  
* bootloader將下載的user app搬移至藍色區塊; 需有flash controller用於燒錄user app用。  
* 執行跳躍到user app的位址, 並執行; sample code如下:
```c
#define USER_APP_ADDRESS 0x20000000
__asm void boot_jump(uint32_t address){
  LDR SP, [R0]      //load new stack pointer address; vector table 的第一個記憶體位址(stack bottom)
  LDR PC, [R0,#4]   //load new program counter address; vector table中的第二個記憶體位址(Reset_Handler)
}
void execute_user_app(void) 
{
  SCB->VTOR = APPLICATION_ADDRESS;  //將user app的位址指派給vector table offset register  
  boot_jump(USER_APP_ADDRESS);      //重新設定stack pointer和program counter
}
```

## Bootloader Remap  

## 簡易燒錄實作

## BootLoader Feature  
* Configurable application Space  
* Flash erase  
* Flash Programming  
* checksum verification  
* flash protection check, write protection enable/disable  
* extended error handling, fail-safe design  


