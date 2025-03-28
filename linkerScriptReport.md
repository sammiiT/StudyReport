# Linker Script 紀錄
* Linker的作用就是把輸入的object檔的各個區段,整合並輸出到特定的區段(section)
* 透過範例來了解Linker Script的撰寫  

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
*	**沒有被指定在linker script, 但又在檔案中存在的section稱為orphan section**: Orphan sections are sections present in the input files which are not explicitly placed into the output file by the linker script. The linker will still copy these sections into the output file by either finding, or creating a suitable output section in which to place the orphaned input section.
*	input_file => linker script => output_file (elf/axf)  沒有被指定的section的input_file會被linker指派為orphan section
*	**Orphan section放置的位可以用 .=. 來表示**: The one way to influence the orphan section placement is to assign the location counter to itself, as the linker assumes that an assignment to . is setting the start address of a following output section and thus should be grouped with that section.  
```c  
SECTIONS{
    start_of_text = . ;
    .text: {
        *(.text)
    }
    end_of_text = . ;
		
    . = . ;             //放置orphan section, 如上述的.rodata區塊(沒有被定義成.rodata{} output sectioin)
    start_of_data = . ;
    .data: {
        *(.data)
    }
    end_of_data = . ;
}
```


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
## Example_1.1: HIDDEN,PROVIDE 指令  
*	**HIDDEN**: 跟static有點相似,表示這個symbol的scope僅在linker script模組之中。 
*	**PROVIDE**:使用PROVIDE的symbol代表此symbol可以在c模組中被重新宣告, 且進行link時會用c模組中的定義; 若沒有在c模組中宣告,則link時使用linker script的定義。  
*	**PROVIDE_HIDDEN**: 如同PROVIDE, 但其scope僅限於當下的elf檔案。

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


## **Example_3**: 比較下列兩者差異  
*	**.text :{*(.text)}**  
將.text區塊 設定成locale counter, 且位址會經linker做alignment的動作,變成 4byte alignment; 如:
```c
.=0x103
.text:{
        *(.text)
}
```
最後.text的section會放到0x104, 會經過linker作alignment
*	**.text . :{*(.text)}**  
有多一個 . 符號, 表示.text區塊不管alignment, 就直接設定到當下的locale counter; 如:
```c
.= 0x103;
.text . :{
        *(.text)
}
```
上述.text的 section會放到0x103, linker不會將其作alignment的動作  
![](https://github.com/sammiiT/StudyReport/blob/master/picture/local_counter_%26_alignment.png)  


##	**Example_4**: output section順序 
![](https://github.com/sammiiT/Study-Report/blob/master/picture/LinkerOrganize.png)  
*	利用linker script來規劃各個section擺放的區域。以上圖(a),(b),(c)來做解說:  
	*	圖(a)的排列, 可用以下描述的linker script來做表示:  
	```c  
	SECTIONS{ # output輸出共3個section, 分別為outputa(從0x10000開始), outputb(接續outpua) 和 outputc(接續outputb)
		outputa 0x10000:{         # outputa區塊,位於0x10000  
			all.o                   # all.o的所有區塊放到outputa(.text,.data,...)  
			foo.o(.input1)          # foo.o檔案中的 .input1 區段  
		}  
		outputb:{  
			foo.o(.input2)          # foo.o檔案中的.input2區段  
			foo1.o(.input1)         # foo1.o檔案中的.input1區段  
		}	 
		outputc:{
			*(.input1) #所有檔案中的.input1區段, 被放到outputc
			*(.input2) #所有被歸類為.input2 section的區塊(section)檔案, 被放到outputc  
		}
	}
	```  
	*	圖(b)的排列,可以用下列描述的linker script來做表示: 僅討論.sec1和.sec2為input section  
		*	*(.sec1 .sec2) 以檔案為主, 把所有檔案中的.sec1和.sec2區塊作輸入  
	* 圖(c)的排列,可以用下列描述的linker script來做表示: 僅討論.sec1和.sec2為input section  
		*	*(.sec1) *(.sec2) 以section為主, 把所有檔案的.sec1集中擺放到一個區塊, 再把所有檔案的.sec2集中放到一個區塊
  *	每一個object file(如foo.o, foo1.o),在程式撰寫時,可定義其個別區段;如 foo.o的input1區段和input2區段; 如:
  ```as
  .section .input1   #foo的input1 區段 
	_symbol_in_input1:
	mov $1, %eax

	.section .input2   #foo的input2 區段   
	_symbol_in_intput2:
	.long 0x90909090
  ```  



## Example_5: Source Code Refernce    
*	C模組中宣告變數, 編譯器對此變數的處理:  
	*	編譯器建立symbol table時,會將變數名稱轉換程另外一種名稱; 如fortran, C++的編譯器都會進行這種轉換。 
	*	當在C語言中宣告symbol並編譯, 編譯器會作兩件事.編譯器會保留一個空間給給此symbol,並賦值; 編譯器會建立symbol table紀錄此symbol的位址; 所以此symbol table會紀錄一個位址, 此位址指向一個空間, 空間儲存一個value; 如:  
		```c  
		int foo =1000;
		```
		*	在symbol table 產生一個叫做foo的entry, 這個entry存放int型態的位址, 指向記憶體空間,此記憶體空間的儲存值為1000; 當程式參考foo這個symbol, 編譯器會先存取symbol table找到此記憶體空間的位址, 然後code去讀取此記憶體空間的值; foo=1000。  
		*	若要取得foo這個變數的address;取得symbol的位址;然後寫入數值1到此位址:  
		```c  
		int* a = &foo;  
		*a = 1;
		```
		參考symbol table中的foo entry,取得其位址,在將其位址存入a這個記憶體空間。
		
*	從linker script中宣告symbol(變數):  
	*	linker script symbol並不等同於C語言中的變數宣告, 他是一個沒有數值的symbol(只代表一個位址)。  
	*	在linker script中宣告symbol, 僅會在symbol table中產生一個entry,此entry僅紀錄位址(symbol僅代表一個位址而已),但不會分配任何記憶體空間給此symbol; **(linker script中的symbol是會被編譯到rom code中, 代表程式編排的資訊??)**。  
	*	如以下在linker script中的描述會產生一個叫做"foo"的symbol table, 此symbol table紀錄的值為1000, 但不會在1000位址上初始化任何值; 這表示使用者無法存取linker script中所宣告的symbol的位址, 僅能讀取symbol所代表的位址。  
	```c  
	foo = 1000;  
	```  
	*	linker script中宣告變數例子; 其中start_of_ROM, end_of_ROM和 start_of_FLASH會被記錄在symbol table中  
	```c  
	start_of_ROM = .ROM;  
	end_of_ROM = .ROM + sizeof(.ROM);  
	start_of_FLASH = .FLASH;  
	```  
	在c code中取得linker script宣告的symbol:  
	```c  
	extern char start_of_ROM;  
	extern char end_of_ROM;  
	extern char start_of_FLASH;  
	memory(&start_of_FLASH, &start_of_ROM, &end_of_ROM - &start_of_ROM);//使用&來取得linker script symbol的指派位址  
	```
	若在linker script中的symbol宣告成array模式,則上述可改寫成:  
	```c 
	extern char start_of_ROM[], end_of_ROM[], start_of_FLASH[];
	memory(start_of_FLASH, start_of_ROM, end_of_ROM - start_of_ROM);
	```  
	
## Example_6: Source Code Reference實例  
*	ARM Cortext-M 為例, 實做了在linker script中宣告變數, 並參考此變數執行scatter loading; 以下只針對這兩像做討論:  
	*	在c模組中(startup.c)參考的linker script symbol:   
	```c    
	extern unsigned long _etext;  //.text section結束的位址
	
	//經過初始化的global或static global擺放的位址。
	extern unsigned long _sidata; //.data section的LMA起始位址 
	extern unsigned long _sdata;  //.data section的VMA起始位址
	extern unsigned long _edata;  //.data section的VMA結束位址值

	//要放在RAM中執行的模組定義為.fastcode section; 在RAM中執行比ROM執行快。
	extern unsigned long _sifastcode; //.fastcode section的LMA起始位址
	extern unsigned long _sfastcode;  //.fastcode section的VMA起始位址值
	extern unsigned long _efastcode;  //.fastcode section的VMA結束位址值
	
	//尚未初始化的global或stacic global擺放的位址。
	extern unsigned long _sbss;  //.bss的起始位址值
	extern unsigned long _ebss;  //.bss的結束位址值
	extern void _estack;         //stack的結束位址值
	```  
	*	**Linker Script**:  
		*	**.text section**: 擺放到IROM
		```c  
		.text :{
			KEEP(*(.isr_vector .isr_vector.*))
			*(.text .text.*)
			*(.gnu.linkonce.t.*)
			*(.glue_7)
			*(.glue_7t)
			*(.gcc_except_table)
			*(.rodata .rodata*)
			*(.gnu.linkonce.r.*)
		} >IROM  
		/*** other sections ***/
		.dtors :{
			.=ALIGN(4);
			PROVIDE(__dtors_start = .);
			KEEP(*(SORT(.dtors.*)))
			KEEP(*(.dtors))
			PROVIDE(__dtors_end = .);
			. = ALIGN(4);
			/* End Of .text section */
			_etext = .; 			//_etext和_sifastcode是一樣的位址??
			_sifastcode = .; //start of fastcode section in LMA
		}> IROM  
 		```  
		*	**.fastcode section**: 此區域的擺放的程式碼是要在RAM直型, 因為比在ROM執行快, 所以稱為.fastcode  
		```c  
		.fastcode :{  
			.= ALIGN (4);  
			_sfastcode = . ;  
			*(.glue_7t) *(.glue_7)  
			*(.fastcode)  
			/* 把要放在RAM的模組放到這邊 ... */  
			/* 模組用__attribute__ ((section (".text.fastcode")))*/  
			*(.text.fastcode)  
			/* 或自訂義section,如: __attribute__  (.text.Blinky_shift) */  
			*(.text.Blinky_shift)  
			. = ALIGN (4);  
			_efastcode = . ;  
			_sidata = .;  
		}	>IRAM0 AT>IROM	//VMA 在IRAM0區域,  LMA是在IROM區域
		```  
		在c模組中對應的上述fastcode section,將屬性(.text.fastcode)作為input放入,如:  
		```c  
		__attribute__ ((section (".text.fastcode"))) void Blinky_flash(Blinky *me, uint8_t n) {
			...;
		}
		```  
		在c模組中對應的上述fastcode section,將屬性(.text.Blinky_shift)作為input放入,如:  
		```c  
		__attribute__ ((section (".text.Blinky_shift"))) void Blinky_Shift(){
			...;
		}
		```  
		*	**.data section**: 初始化的global, static global 擺放的區域 
		```c  
		.data :{
			_sidata = LOADADDR (.data);	//_sidata的數值為.data section的LMA
			. = ALIGN(4);
			_sdata = .;				//.data 起始位址
			*(vtable vtable.*)
			*(.data .data.*)	// global和static global存放位址
			*(.gnu.linkonce.d*)
			. = ALIGN(4);
			_edata = . ;			// .data結束位址
		}>IRAM0 AT>IROM			//VMA 在IRAM0區域,  LMA是在IROM區域 check
		```  
		*	**.bss section**
		```c
		.bss(NOLOAD):{
			. = ALIGN(4);
			_sbss = . ;				// .bss的起始位址
			*(.bss .bss.*)
			*(.gnu.linkonce.b*)
			*(COMMON)
			. = ALIGN(4);
			_ebss = . ;				// .bss的結束位址	
		}>IRAM0					// VMA在 IRAM0
		```
		
	*	ARM cortex M開機執行的第一個函示為Reset_Handler(Exception), 此函示會先做scatter loading, 在data section成功拷貝到VMA之後,再進入main執行。 會用到 上述 linker script中的_sidata,_sdata,_edata,_sifastcode,_sfastcode,_efastcode,_sbss,_ebss
	```c
	void __attribute__((__interrupt__)) Reset_Handler(void){
        SystemInit();
        unsigned long *pulDest;
        unsigned long *pulSrc;
        if (&_sidata != &_sdata) {// 拷貝 data section 從flash到SRAM  
            pulSrc = &_sidata;	//在.data section中有定義 _sidata = LOADADDR (.data); LOADADDR為取LMA位址指令
            for(pulDest = &_sdata; pulDest < &_edata; ) { *(pulDest++) = *(pulSrc++); }
        }
        if (&_sifastcode != &_sfastcode) {// 拷貝.fastcode section 從flash到SRAM
            pulSrc = &_sifastcode; //在 .dtors Section中有描述到_sifastcode的 LMA
            for(pulDest = &_sfastcode; pulDest < &_efastcode; ) { *(pulDest++) = *(pulSrc++); }
        }
        for(pulDest = &_sbss; pulDest < &_ebss; ){//初始化 .bss section 
            *(pulDest++) = 0;
        }  
        /* 這邊可以再加入 vector table remap的程式; cortex m0 cpu.................. */
        main(); // main函式進入點
	}
	```

		




### Reference
