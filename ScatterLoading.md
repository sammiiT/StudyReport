# Scatter Loading 
![](https://github.com/sammiiT/Study-Report/blob/master/picture/ScatterLoading.png)

## LMA (load memory address)
* 裝載地址: 程式尚未執行時的儲存位址; 如儲存在Flash中的ROM code

## VMA (virtual memory address)
* 運行地址: 程式執行時的實際位址, 當系統bootup時, Flash中的一部分資料會被搬移到實際的執行位址

## Scatter Loading
* 

## ld 之 Linker Script設定
SECTION{
  .text 0x0000000:{
    *(.text)  
  }
  .data 0x10000000: AT(ADDR(.text)+SIZEOF(.text)){
    *(.data)
  }
}

## ArmLink鏈結
* armlink鏈結object檔案時


## Example
