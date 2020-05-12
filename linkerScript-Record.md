# Linker Script Record
* 透過範例來了解Linker Script撰寫

## Example_1

## Example_2

## Example_3: LMA,VMA
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
    SECTIONS  {  
        .text 0x5000:{  
            *(.text)  
        }
        .data 0x8000:{  
            *(.data)  
        }  
    }  
    
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
    
最開始01b8應該就是mov $1, $eax的instruction code。而0x3000位置的90909090顯然就是我們定義在數據段>的character了。因為linker script中没有用AT指令專門為兩個段指定lma，所以其lma与vma相等，两个段相差了0x3000 bytes的長度。.text段之前没有其他段了，所以最终的bin文件中一開始就是.text段的内容，雖然只有2個character，但仍然要过0x3000 bytes才是.data段,中間那些未知數就
填0。  

因為我們知道0x8000已經是Ram了，燒錄全域變數區域到一斷電内容就消失的Ram中？ 並且,Flash(Rom)和Ram之間相隔的0x3000 bytes不一定就對應實際的儲存區域。  
  
更新Linker Script:  
    SECTIONS{  
       .text 0x5000:{  
          *(.text)  
       }  
       .data 0x8000:AT(ADDR(.text) + SIZEOF(.text)){  
          *(.data)  
       }  
    }

* 編譯, 鏈結, 查詢其內容:  
[root@localhost]# hexdump ./a.out  
0000000 01b8 0000 9000 9090 0090  
0000009  
* 新檔案大小:只占用9 bytes  
[root@localhost]# ls -lh ./a.out  
-rwxr-xr-x 1 root root 9 11-04 20:55 ./a.out


### Reference
