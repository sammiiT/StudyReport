# Linker Script 紀錄
* Linker的作用就是把輸入的object檔的各個區段,整合並輸出到特定的區段(section)
* 透過範例來了解Linker Script撰寫

## Object File
* .obj檔案, 經編譯器編譯後的檔案
* 經編譯後的檔案(obj)基本上分為三個區段:程式區段, 經初始化的全域變量, 未經初始化的全域變量
* 每一個obj檔案的每一個區段,會經由linker script放置到其所描述的區域, 最後輸出bin檔,或elf/axf執行檔

## Symbol
* 每一個object檔案都有一個symbol列表, 稱為symbol table. 這些symbol有些是defined, 有些是undefined。 
* Defined symbol都有一個固定的地址(存放symbol的地址),defined symbol可以是一個函式,一個global variable, 或一個static global。
* Undefined symbol是在object檔案中以extern描述的外部定義變量; 如.c檔案中的 extern int a; extern void foo(); 這些參數是必須經由linker鏈結才能參考。
* 可以利用binutil的nm指令查詢檔案中的所有symbol.

## Location Counter
* .是location counter, 用來表示symbol的位址值
## Orphan Section

## Example_1: SECTION , MEMORY 指令
* MEMORY: 描述target device的記憶體區塊位址,大小; 可以用此指令來劃分記憶體的region,告訴linker哪一塊可以存放資料,程式或變量。可以將SECTION中的各個區塊指派給MEMORY中的region。   
```c
MEMORY{  
        FLASH(rx):ORIGIN=0x00000000,LENGTH=128K  //FLASH region從0x00000000位址開始, 大小為128k, 屬性為讀和執行  
        RAM(rwx):ORIGIN=0x20000000,LENGTH=32K   //RAM region從0x20000000位址開始,大小為32K,屬性為讀寫和執行  
}  
```
* SECTION:此指令告訴linker如何組織各個輸入的section, 並將其輸出; 同時也定義各個output section在memory中的位址。  
```c
SECTIONS{  
        .text:{  
                *(.text)        //各個 obj檔案的.text區段放到.text區段   
                *(.text.*)      //各個obj檔案的.text.*區段放到.text區段          
                *(.rodata)  
                _sromdev=.;     //_sromdev 符號(symbol)    
                _eromdev=.;
                _sidata=.;
        }>FLASH             //.text區段會放到MEMORY的FLASH region
        .data:AT(_sidata){      //.data區段的LMA會放到_sidata起始的位址
                _sdata=.;
                *(.data)
                *(.data*)
                _edata=.;  
        }>RAM              //.data會放到MEMORY的RAM region
        .bss:{  
                _sbss=.;  
                *(.bss)  
                _ebss=.;  
        }>RAM              //.bss會放到MEMORY的RAM region,接續在.data的後面
        _estack = ORIGIN(RAM) + LENGTH(RAM);
}
```  
## Example_2: LMA,VMA
* 以下assembler, 編譯gcc -o xx.o -c xxx.S
```as
.section .text
_symbol_in_text:
mov $1, %eax

.section .data
_symbol_in_data:
.long 0x90909090
```
* Linker script, 鏈結 ld ./xxx.o -T ./xxx.lds; 鏈結完之後是elf格式檔案  
```c  
SECTIONS  {  
        .text 0x5000:{  
            *(.text)  
        }
        .data 0x8000:{  
            *(.data)  
        }  
}  
```
    
在上述的ld script指定義了VMA; 根據ld的規則, 如果沒有AT指令定義LMA, 則LMA默認為VMA。這裡兩段的VMA不一樣, 因為嵌入式系統常常會遇到下述情況, 即Flash(ROM)空間較大,RAM空間相對較小,於是我們只希望數據裝載至RAM空間(0x8000), 其他代碼就運行在Flash(0x5000)。  

* 產生執行檔, objcopy -O binary ./a.out ;此文件為bin檔,會占用13k:  
    [root@localhost]# ls -lh ./a.out  
    -rwxr-xr-x 1 root root 13K 11-04 20:55 ./a.out    

* 利用hexdump查看bin檔內容:  
    [root@localhost]# hexdump ./a.out  
    0000000 01b8 0000 0000 0000 0000 0000 0000 0000  
    0000010 0000 0000 0000 0000 0000 0000 0000 0000  
    *  
    0003000 9090 9090  
    
最開始01b8應該就是mov $1, $eax的instruction code。而0x3000位置的90909090顯然就是我們定義在數據段的character了。因為linker script中沒有用AT指令專門為兩個段指定LMA，所以其LMA與VMA相等，兩區段相差了0x3000 bytes的長度。.text段之前沒有其他區段了，所以最終的bin文件中一開始就是.text段的內容，雖然只有2個character，但仍然要跨過0x3000 bytes才是.data段,中間那些未知數就
填0。  

因為我們知道0x8000已經是Ram了，燒錄全域變數區域到一斷電, 其容就消失的Ram中？ 並且,Flash(Rom)和Ram之間相隔的0x3000 bytes不一定就對應實際的儲存區域。  
  
更新Linker Script:下面的ld script在定義.data段時增加了AT指令來描述其LMA，這樣表示.data區段的LMA緊接在.text區段的後面, 但.data的VMA還是落在 0x8000位址上:  
```c
SECTIONS{  
        .text 0x5000:{  
          *(.text)  
       }  
       .data 0x8000:AT(ADDR(.text) + SIZEOF(.text)){  
          *(.data)  
       }  
}
```
* 編譯, 鏈結, 查詢其內容:  
[root@localhost]# hexdump ./a.out  
0000000 01b8 0000 9000 9090 0090  
0000009  
* 新檔案大小:只占用9 bytes  
[root@localhost]# ls -lh ./a.out  
-rwxr-xr-x 1 root root 9 11-04 20:55 ./a.out


### Reference
