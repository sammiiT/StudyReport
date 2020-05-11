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

## Bootloader Remap

## Bootloader VTOR 
