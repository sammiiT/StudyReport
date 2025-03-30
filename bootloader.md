# bootloader for Cortex-M MCU

## ARM Cortex-M開機程序
![](https://github.com/sammiiT/Study-Report/blob/master/picture/ResetSequence.PNG)
* bin file的開頭32bits (位址0x00000000), 第二個32bits(0x00000004)儲存Reset_Handler的位址。
* 開機時,cpu會先讀取0x00000000的值,指派給sp; 接著讀取0x00000004的值, 將其指派給pc。

![](https://github.com/sammiiT/Study-Report/blob/master/picture/Initial_SP_PC.PNG)
* sp是0x20008000, pc是0x00000101
* stack pointer往下遞增
* 初始stack位址,和vector table必須從0x00000000開始存放, 因此IAP之後的APP必須 (1)Remap vector到0x00000000 或使用 (2)VTOR(Vector Table Offset Register)。

Note:  
* thumb模式下,PC代的數值的bit[0]設為1  (上圖0x00000004位址,值0x00000101; program_counter指向0x00000100)  
* 前256 bytes是一開始.asm(.s)檔案所描述的區塊, 就是圖中的0x00000000, 0x00000004 , Other_exception_vectors區塊   
* Boot Code就是ResetHandler所對應的內容 (default狀況下); 位址0x00000004的內容, 就是ResetHandler的位址; ResetHandler在default狀況,會呼叫main函式  
      

## Bootloader VTOR 
![](https://github.com/sammiiT/Study-Report/blob/master/picture/bootloader-VTOR.png)  
-----  
* bootloader將下載的user app搬移至藍色區塊; ~~需有flash controller用於燒錄user app用。~~  
* 執行跳躍到user app的位址, 須執行sample code如下:
```c
#define USER_APP_ADDRESS 0x20000000
__asm void boot_jump(uint32_t address){
  LDR SP, [R0]      //load new stack pointer address; vector table 的第一個記憶體位址(stack bottom)  ...(2)
  LDR PC, [R0,#4]   //load new program counter address; vector table中的第二個記憶體位址(Reset_Handler) ...(3)
}
void execute_user_app(void) 
{
  /* disable interrupt */
  
  SCB->VTOR = USER_APP_ADDRESS ;  //USER_APP_ADDRESS的開頭,就是sp存放位址; ...(1)
                                  //USER_APP_ADDRESS+0x04,是user app的vector table offset register  
  boot_jump(USER_APP_ADDRESS);    //重新設定stack pointer和program counter
}
/*
(1)設定VTOR
(2)設定SP
(3)做jump的動作, LDR PC,[R0,#4]
*/
```
## Bootloader Remap  
![](https://github.com/sammiiT/Study-Report/blob/master/picture/bootloader_remap.png)  
-----  
* cortex-m0 cpu不支援VTOR, 所以必須用memory remap功能。  
* 將user app vector table先拷貝一分到SRAM 的起始位址區域。   
* 設定SYSCFG register(SYSCFG_CFGR1)的bit[0]=1和bit[1]=1,將SRAM映射到0x00000000, vector table會映射到0x0。  

## 執行跳躍到user app程式:  
* (1)先執行jump 到app的程式, (2)再copy vector table 到SRAM, (3)最後再將SRAM的內容mapping到0x00000000的位址
```c  
#define APPLICATION_ADDRESS     (uint32_t)0x08004000  //UerAPP的位址
int main(void){ //BootLoader 主體
/*  boot主函數, 包含app下載,校驗, 升級(copy UserAPP)  */

//最後執行轉跳程式
	RCC->CIR = 0x00000000U;		// Mask所有中斷
	iap_load_app();						// 轉跳到app
	HAL_NVIC_SystemReset();		// 一般是不會走到這裡,會執行此行代表轉跳失敗, 重啟
} 

typedef void (*pFunction)(void);
pFunction  JumpToApplication;
void iap_load_app(void){	//轉跳程式
	uint32_t JumpAddress = 0x00;	
	//檢查Initial Stack Top是否valid;取0x08004000位址的值, 就是initial stack top	
	if (((*(__IO uint32_t*)APPLICATION_ADDRESS) & 0x2FFE0000 )==0x20000000){
		JumpAddress =*(__IO uint32_t*)(APPLICATION_ADDRESS+4);//initial stack top的下一個就是reset handler的位址。
		JumpToApplication = (pFunction)JumpAddress;
		__set_MSP(*(__IO uint32_t*) APPLICATION_ADDRESS);//設定stack pointer,此帶入值為stack top
		JumpToApplication(); //呼叫UserAPP的ResetHandler()
	}
/* 跳到ResetHandler, 此handler最後會執行到main,首先須將vector table重新填入SRAM,再從SRAM作一次mapping到0x00000000; 其sample code如下 */		
}
```
* 執行vector table remap,先拷貝vector table再從SRAM重映射到0x00000000程式:
```c
//執行User APP
#define APPLICATION_ADDRESS     (uint32_t)0x08004000  
#ifdef __CC_ARM	//armcc 編譯器
	__IO uint32_t  VectorTable[48] __attribute__((at(0x20000000)));  
#elif defined (__GNUC__)	//gnu 編譯器
	__IO uint32_t VectorTable[48] __attribute__((section(".RAMVectorTable")));
#endif

int main(void){ /* 這一段remap的動作可以放到 reset_handler scatter loading動作內, 參考linkerscript 讀書心得報告 */
	uint32_t i=0;
	//設定system clock...省略不描述
	
	//拷貝UserAPP的vector table到SRAM //上圖的(1)
	for(i=0; i<48; ++i){
		/* 前面48個word拷貝至0x2000000(SRAM的開頭位址),此48個word就是initial stack pointer和vector table */
		/* vector table中存放記憶體中的值, APPLICATION_ADDRESS位址上的值是一個function的位址 */
		VectorTable[i] = *(__IO uint32_t*)(APPLICATION_ADDRESS+(i<<2)); 
	}  
	//設定peripheral clock....省略不描述
	
	//remap動作,從0x20000000重映射到0x00000000 //上圖的(2)
	WRITE_REG(SYSCFG->MEMRMP,
	(READ_REG(SYSCFG->MEMRMP)&~(SYSCFG_MEMRMP_MEM_MODE))|(SYSCFG_MEMRMP_MEM_MODE_1|SYSCFG_MEMRMP_MEM_MODE_0));	
}
```
## BootLoader Feature  
* Configurable application Space  
* Flash erase  
* Flash Programming  
* checksum verification  
* flash protection check, write protection enable/disable  
* extended error handling, fail-safe design  


