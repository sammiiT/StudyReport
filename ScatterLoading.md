# Scatter Loading 
![](https://github.com/sammiiT/Study-Report/blob/master/picture/ScatterLoading.png)

## LMA (load memory address)
* 裝載地址: 程式尚未執行時的儲存位址; 如儲存在Flash中的ROM code

## VMA (virtual memory address)
* 運行地址: 程式執行時的實際位址, 當系統bootup時, Flash中的一部分資料會被搬移到實際的執行位址

## (RO + RW + ZI) & (.text + .data + .bss) 
* arm的編譯器: RO=(程式和const data) ; RW=(初始化的global和global static variable) ; ZI=(未經初始化的global及global static), 其中ZI會在程式執行main函式之前被設定為0
* gnu arm編譯器: .text=程式和const data ; .data=初始化的global和global static variable ; .bss=未經初始化的global及global static, 此區域會初始化為0

## Scatter Loading
* 程式開始執行時,會將對應的RW(initialized glabol or static global variable)從LMA搬移到實際運作的VMA區塊,此搬移動作稱為Scatter Loading。用armlink時,搬移的動作會自動加在main函式的開頭; 參照; 用gnu arm ld 必須加上搬移.data 到VMA的描述。

## ArmLink鏈結
* armlink鏈結object檔案時,

## ld 之 Linker Script設定
SECTION{  

  .text 0x0000000:{
    *(.text)  
  }
  .data 0x10000000: AT(ADDR(.text)+SIZEOF(.text)){
    *(.data)
  }
}



## Example
