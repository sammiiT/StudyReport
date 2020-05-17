# Scatter Loading Implement
* Platform : ARM Cortex-M   

## C Implementation:
* 參考linkerScript-Record.md的Symbol章節和Example_5:  
  * 下列的變數定義為undefined symbol, 其實體定義在linker script中
```c  
extern unsigned long _etext;  // 
extern unsigned long _sidata; //.data section的起始位址的初始值 
extern unsigned long _sdata;  //.data section的起始位址值
extern unsigned long _edata;  //.data section的結束位址值

extern unsigned long _sifastcode; //.fastcode section的起始位址的初始值
extern unsigned long _sfastcode;  //.fastcode section的起始位址值
extern unsigned long _efastcode;  //.fastcode section的結束位址值

extern unsigned long _sbss;  //.bss的起始位址值
extern unsigned long _ebss;  //.bss的結束位址值
extern void _estack;         //stack的結束位址值
```

```
void __attribute__((__interrupt__)) Reset_Handler(void){
        SystemInit();
        unsigned long *pulDest;
        unsigned long *pulSrc;
        
        if (&_sidata != &_sdata) {// Copy the data segment initializers from flash to SRAM in ROM mode   
            pulSrc = &_sidata;
            for(pulDest = &_sdata; pulDest < &_edata; ) {
                *(pulDest++) = *(pulSrc++);
            }
        }
        
        if (&_sifastcode != &_sfastcode) {/* Copy the .fastcode code from ROM to SRAM */
            pulSrc = &_sifastcode;
            for(pulDest = &_sfastcode; pulDest < &_efastcode; ) {
                *(pulDest++) = *(pulSrc++);
            }
        }
        
        for(pulDest = &_sbss; pulDest < &_ebss; ){/* Zero fill the bss segment */ 
            *(pulDest++) = 0;
        }
        
        main(); /* Call the application's entry point */
}
```



note: 參考linkerScript-Record.md

## Assembler Implementation
