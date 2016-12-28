 qemu_esp32
Add tensilica esp32 cpu and a board to qemu and dump the rom to learn more about esp-idf
###ESP32 in QEMU.

This documents how to add an esp32 cpu and a simple esp32 board to qemu in order to run an app compiled with the SDK in QEMU. Esp32 is a 240 MHz dual core Tensilica LX6 microcontroller.
It is a good way to learn about qemu ,esp32 and the esp32 rom.

## Setting up platformio
[I would not currentrly recomed this, Setup Platform.io](./platformio.md)
http://platformio.org/platformio-ide


## Setting up visual studio code
[Works pretty well in linux now](./VSCODE.md)


By following the instructions here, I added esp32 to qemu.
http://wiki.linux-xtensa.org/index.php/Xtensa_on_QEMU

prerequisites Debian/Ubuntu:
```
sudo apt-get install libpixman-1-0 libpixman-1-dev 
```


## qemu-esp32

```
git clone git://github.com/Ebiroll/qemu-xtensa-esp32
```

Requires that you run a patched gdb to run in qemu. If not you will get Remote 'g' packet reply is too long:
The key to using the original gdb, is to set num_regs = 104, in core-esp32.c
However debugging c-functions in the elf file requires a patched qemu.
In the bin directory there are patched versions of xtensa-esp32-elf-gdb

## Building qemu
```
mkdir ../qemu_esp32
cd ../qemu_esp32
Then run configure as,
../qemu-xtensa-esp32/configure --disable-werror --prefix=`pwd`/root --target-list=xtensa-softmmu,xtensaeb-softmmu
```

The goal was to run the app-template.elf and connect the debugger to qemu to allow debugging of the application. It works quite well and as the rom is patched slightly it its possible to run the elf file that is generated by the esp-idf qemu. However you must use theese configutation items. 
```
[*] Run FreeRTOS only on first core,    
[ ] Initialize PHY in startup code

Optimization level (Debug) 
Not sure if this is necessary,
 Xtensa timer to use as the FreeRTOS tick source (Timer 0 (int 6, level 1))
```
UART emulation is not so good as output is only on stderr. It would be much better if we could use the qemu driver with, serial_init(), however this only works in the rom part.

When running I advise that you patch gdb as described in this document or use the gdb provided in the /bin directory (linux 64 bit). This improves the debugging exerience quite a bit.

When debugging in the rom you can use,
```
    (gdb) add-symbol-file rom.elf 0x40000000
```
This also works for the original gdb and gives you all names of the functions in rom0.


To setup esp-idf do, 
```
git clone --recursive https://github.com/espressif/esp-idf.git
#To keep the esp-idf updated, do git pull & git submodule update --recursive

export PATH=$PATH:$HOME/esp/xtensa-esp32-elf/bin
export IDF_PATH=~/esp/esp-idf
```

#Dumping the ROM0 & ROM1 using esp-idf esptool.py
```
cd ..
git clone --recursive https://github.com/espressif/esp-idf.git
# dump_mem ROM0(776KB) to rom.bin
esp-idf/components/esptool_py/esptool/esptool.py --chip esp32 -b 921600 -p /dev/ttyUSB0 dump_mem 0x40000000 0x000C2000 rom.bin

# dump_mem ROM1(64KB) to rom1.bin
esp-idf/components/esptool_py/esptool/esptool.py --chip esp32 -b 921600 -p /dev/ttyUSB0 dump_mem 0x3FF90000 0x00010000 rom1.bin

Note that rom0 is smaller than the actual dump.
Those two files will be loaded by qemu and must be in same directory as you start qemu.
```

#Dumping flash content.
esp/esp-idf/components/esptool_py/esptool/esptool.py --baud 920600 read_flash 0 0x400000  esp32flash.bin
Now there is simple flash emulation in qemu. You need the file  esp32flash.bin to be in the same directory as rom.bin & rom1.bin.
Currently write and mmao functions are not implemented. Soon they should be there.
If no flashfile exists, an empty file will be created.



### Start qemu
```
  > xtensa-softmmu/qemu-system-xtensa -d guest_errors,unimp  -cpu esp32 -M esp32 -m 4M -net nic,model=vlan0 -net user,id=simnet,ipver4=on,net=192.168.1.0/24,host=192.168.1.40,hostfwd=tcp::10077-192.168.1.3:7  -net dump,file=/tmp/vm0.pcap  -kernel  ~/esp/qemu_esp32/build/app-template.elf  -s -S > io.txt

```

### Start the debugger
```
  > xtensa-esp32-elf-gdb  build/app-template.elf
  (gdb) target remote:1234

  (gdb) x/10i $pc
  (gdb) x/10i 0x40000400
```

To disassemble a specific function you can set pc from gdb,

```
  (gdb) set $pc=0x40080804

  (gdb) set $pc=<tab>  to let gdb list available functions
  (gdb) layout asm
  (gdb) si
  (gdb) ni  (skips calls)
  (gdb) finish (run until ret)
```
Setting programcounter (pc) to a function,

```
  (gdb) set $pc=call_start_cpu0
```


```
  (gdb) x/20i $pc
  (gdb) p/x $a3          (register as hex)

```

#Objdump
Dump mixed source/disassemply listing,
```
xtensa-esp32-elf-objdump -d -S build/bootloader/bootloader.elf
xtensa-esp32-elf-objdump -d build/main/main.o
```
Try adding simple assembly or functions and looḱ at the generated code,

```
void retint() {
    asm volatile (\
    	 "RFDE\n");
}

int test(int in) {
    return (in+1);
}

void jump() {
    asm volatile (\
    	 "jx a0\n");
    retint();
}
```


```
00000000 <retint>:
   0:	004136          entry	a1, 32
   3:	003200          rfde
   6:	f01d            retw.n

Disassembly of section .text.test:

00000000 <test>:
   0:	004136      entry	a1, 32
   3:	221b      	addi.n	a2, a2, 1
   5:	f01d      	retw.n

Disassembly of section .text.jump:

00000000 <jump>:
   0:	004136        	entry	a1, 32
   3:	0000a0        	jx	a0
   6:	000081        	l32r	a8, fffc0008 <jump+0xfffc0008>
   9:	0008e0          callx8	a8
   c:	f01d            retw.n
```


#Results
If you get something like this,
```
Illegal entry instruction(pc = 40080a4c), PS = 0000001f
Illegal entry instruction(pc = 40080a4c), PS = 0000001f
```
Then you have probably not downloaded rom files or put them in another directory than where you start qemu.


I got my ESP32-dev board from Adafruit. https://dl.espressif.com/dl/schematics/ESP32-Core-Board-V2_sch.pdf  I have made two dumps and mapped the dumps into the files rom.bin & rom1.bin
The code in esp32.c also patches the function ets_unpack_flash_code.
Instead of loading the code it sets the user_code_start variable (0x3ffe0400) This will later be used
by the rom to start the loaded elf file. The functions SPIEraseSector & SPIWrite are also patched to return 0.
The rom function -- ets_unpack_flash_code is located at 0x40007018.


Better flash emulation would be better. The serial device should also be emulated differently. Now serial output goes to stderr.

To get proper serial emulation you can remove the comment around this in the file esp32.c,
```
//  serial_init(system_io, 0x40000 , 2, xtensa_get_extint(env, 0),
//            115200, serial_hds[0], DEVICE_LITTLE_ENDIAN);
```
However it does not always work. You can use qemu to generate reset and it might work sometimes.

```
xtensa-softmmu/qemu-system-xtensa -d guest_errors,page,unimp  -cpu esp32 -M esp32 -m 4M  -kernel  ~/esp/qemu_esp32/build/app-template.elf  -S -s > io.txt

(gdb) b start_cpu0_default
(gdb) b app_main
(gdb) c

SR 97 is not implemented
SR 97 is not implemented
ets Jun  8 2016 00:22:57

rst:0x10 (RTCWDT_RTC_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
I (1) heap_alloc_caps: Initializing heap allocator:
I (1) heap_alloc_caps: Region 19: 3FFB3D10 len 0002C2F0 tag 0
I (1) heap_alloc_caps: Region 25: 3FFE8000 len 00018000 tag 1
check b=0x3ffb3d1c size=180948 ok
check b=0x3ffdfff0 size=0 ok
check b=0x3ffe800c size=98276 ok
I (2) cpu_start: Pro cpu up.
I (2) cpu_start: Single core mode
I (2) cpu_start: Pro cpu start user code
rtc v112 Sep 26 2016 22:32:10
XTAL 0M
TBD(pc = 40083be6): /home/olas/qemu/target-xtensa/translate.c:1354
              case 6: /*RER*/ Not implemented in qemu simulation

I (3) cpu_start: Starting scheduler on PRO CPU.
WUR 234 not implemented, TBD(pc = 400912c3): /home/olas/qemu/target-xtensa/translate.c:1887
WUR 235 not implemented, TBD(pc = 400912c8): /home/olas/qemu/target-xtensa/translate.c:1887
WUR 236 not implemented, TBD(pc = 400912cd): /home/olas/qemu/target-xtensa/translate.c:1887
RUR 234 not implemented, TBD(pc = 40091301): /home/olas/qemu/target-xtensa/translate.c:1876
RUR 235 not implemented, TBD(pc = 40091306): /home/olas/qemu/target-xtensa/translate.c:1876
RUR 236 not implemented, TBD(pc = 4009130b): /home/olas/qemu/target-xtensa/translate.c:1876

The WUR and RUR instructions are specific for the ESP32.


When configuring, choose
  Component config  --->   
    FreeRTOS  --->
      [*] Run FreeRTOS only on first core 
    ESP32-specific config  --->
      [ ] Initialize PHY in startup code  

```


##Before the rom patches
Before the rom patches we got this
```
xtensa-softmmu/qemu-system-xtensa -d guest_errors,int,page,unimp  -cpu esp32 -M esp32 -m 4M  -kernel  ~/esp/qemu_esp32/build/app-template.elf    > io.txt 
ets Jun  8 2016 00:22:57

rst:0x10 (RTCWDT_RTC_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
flash read err, 1000
Falling back to built-in command interpreter.
```
The command interpreter is Basic, Here you can read about it 
    http://hackaday.com/2016/10/27/basic-interpreter-hidden-in-esp32-silicon/


#Debugging tips
```
If you get an exception like this
Guru Meditation Error of type StoreProhibited occurred on core   0. Exception was unhandled.
Register dump:
PC      :  4008189e  PS      :  00060030  A0      :  800d05f6  A1      :  3ffe3e20  
A2      :  3ffb1134  A3      :  0002e49c  A4      :  3ffb008c  A5      :  fffffffc  
A6      :  3ffb008c  A7      :  ffffffff  A8      :  3ffe8000  A9      :  bffcffdc  
A10     :  3ffb12c4  A11     :  3ffe8000  A12     :  00000019  A13     :  00000001  
A14     :  7ffe7fe8  A15     :  00000000  SAR     :  00000004  EXCCAUSE:  0000001d  
EXCVADDR:  bffcffe0  LBEG    :  4000c46c  LEND    :  4000c477  LCOUNT  :  ffffffff  
Look at PC for the error then set a breakpoint there 
(gdb) b *0x4008189e
And reset qemu.
We break here ,   components/freertos/./heap_regions.c
(gdb) Pressing Ctrl-X and the o will open the source code if it exists. 
(gdb) where
(gdb) up

// This one we could also analyze more
ets_get_detected_xtal_freq,  0x40008588
```


#What is some of the problems with this code
Some i/o register name mapping in esp32.cis probably wrong.  The values returned are also many times wrong.
I did this mapping very quickly with grep to get a better understanding of what the rom was doing.
```
Terminal 1
>xtensa-softmmu/qemu-system-xtensa -d guest_errors,int,mmu,page,unimp  -cpu esp32 -M esp32 -m 4M  -kernel  ~/esp/qemu_esp32/build/app-template.elf   -S -s

Terminal 2
>xtensa-esp32-elf-gdb build/app-template.elf
(gdb) target remote:1234

//Uart_Init, 0x40009120
(gdb) b *0x40009120
(gdb) layout asm

xtensa-softmmu/qemu-system-xtensa -d guest_errors,int,mmu,page,unimp  -cpu esp32 -M esp32 -m 4M  -kernel  ~/esp/qemu_esp32/build/app-template.elf   -S -s 
Elf entry 400807BC
serial: event 2
tlb_fill(40000400, 2, 0) -> 40000400, ret = 0
tlb_fill(400004b3, 2, 0) -> 400004b3, ret = 0
tlb_fill(400004e3, 2, 0) -> 400004e3, ret = 0
SR 97 is not implemented
SR 97 is not implemented
tlb_fill(4000d4f8, 0, 0) -> 4000d4f8, ret = 0
tlb_fill(3ffae010, 1, 0) -> 3ffae010, ret = 0
tlb_fill(4000f0d0, 0, 0) -> 4000f0d0, ret = 0
tlb_fill(3ffe0010, 1, 0) -> 3ffe0010, ret = 0
tlb_fill(3ffe1000, 1, 0) -> 3ffe1000, ret = 0
tlb_fill(400076c4, 2, 0) -> 400076c4, ret = 0
tlb_fill(4000c2c8, 2, 0) -> 4000c2c8, ret = 0
tlb_fill(3ff9918c, 0, 0) -> 3ff9918c, ret = 0
tlb_fill(3ffe3eb0, 1, 0) -> 3ffe3eb0, ret = 0
tlb_fill(3ff00044, 0, 0) -> 3ff00044, ret = 0
io read 00000044  DPORT  3ff00044=8E6
tlb_fill(400081d4, 2, 0) -> 400081d4, ret = 0
tlb_fill(3ff48034, 0, 0) -> 3ff48034, ret = 0
io write 00000044 000008E6 io read 00048034 RTC_CNTL_RESET_STATE_REG 3ff48034=3390
io read 00048088 RTC_CNTL_DIG_ISO_REG 3ff48088=00000000
io write 00048088 00000000 RTC_CNTL_DIG_ISO_REG 3ff48088
io read 00048088 RTC_CNTL_DIG_ISO_REG 3ff48088=00000000
io write 00048088 00000400 RTC_CNTL_DIG_ISO_REG 3ff48088
tlb_fill(40009000, 2, 0) -> 40009000, ret = 0
tlb_fill(3ff40010, 1, 0) -> 3ff40010, ret = 0
serial: write addr=0x4 val=0x1
tlb_fill(3ff50010, 1, 0) -> 3ff50010, ret = 0
tlb_fill(400067fc, 2, 0) -> 400067fc, ret = 0
tlb_fill(4000bfac, 2, 0) -> 4000bfac, ret = 0
tlb_fill(3ff9c2b9, 0, 0) -> 3ff9c2b9, ret = 0
tlb_fill(3ff5f06c, 0, 0) -> 3ff5f06c, ret = 0
io write 00050010 00000001 
io read 0005f06c TIMG_RTCCALICFG1_REG 3ff5f06c=25
tlb_fill(3ff5a010, 0, 0) -> 3ff5a010, ret = 0
io read 0005a010 TIMG_T0ALARMLO_REG 3ff5a010=01
tlb_fill(40005cdc, 0, 0) -> 40005cdc, ret = 0
io read 000480b4 RTC_CNTL_STORE5_REG 3ff480b4=00000000
io read 000480b4 RTC_CNTL_STORE5_REG 3ff480b4=00000000
io write 000480b4 18CB18CB RTC_CNTL_STORE5_REG 3ff480b4

----------- break Uart_Init, 0x40009120
(gdb) finish
```

#Dumping the ROM with the main/main.c program
Please use other method (esptool.py), its easier and faster. This is saved for historical reasons and for the screen instructions.
Set the environment properly. Build the romdump app and flash it.
Use i.e screen as serial terminal.

```
screen /dev/ttyUSB0 115200
Ctrl-A  then H   (save output)
Press and hold the boot button, then press reset. This puts the ESP to download mode.
Ctrl-A  then \   (exit, detach)
make flash
screen /dev/ttyUSB0 115200
Ctrl-A  then H   (save output)
press  reset on ESP to get start.
```
When finished trim the capturefile (remove all before and after the dump data) and call it, test.log
Notice that there are two dumps search for ROM and ROM1
Compile the dump to rom program.
```
gcc torom.c -o torom
torom
If successfull you will have a binary rom dump, rom.bin
Then also create the file rom1.bin
If you start with second part and then do
mv rom.bin rom1.bin
Then you can do the first part
Those two files will be loaded by qemu and must be in same directory as you start qemu.
```

#This is head of qemu development.
Not so good for esp32 debugging.
```
git clone git://git.qemu.org/qemu.git

cd qemu
git submodule update --init dtc
```

The instruction wsr.memctl does not work in QEMU but can be patched like this (in qemu tree).
File translate.c, This is only needed for the qemu in the current directory
```
static bool gen_check_sr(DisasContext *dc, uint32_t sr, unsigned access)
{
    if (!xtensa_option_bits_enabled(dc->config, sregnames[sr].opt_bits)) {
        if (sregnames[sr].name) {
            qemu_log_mask(LOG_GUEST_ERROR, "SR %s is not configured\n", sregnames[sr].name);
        } else {
            qemu_log_mask(LOG_UNIMP, "SR %d %s is not implemented\n", sr);
        }
        //gen_exception_cause(dc, ILLEGAL_INSTRUCTION_CAUSE);
```

A better version for qemu exists here https://github.com/OSLL/qemu-xtensa,

```
git clone https://github.com/OSLL/qemu-xtensa -b  xtensa-esp32 
but requires that you run a patched gdb to run in qemu. If not you will get Remote 'g' packet reply is too long:
The key to using the original gdb, is to set num_regs = 104, in core-esp32.c 



git clone https://github.com/Ebiroll/qemu_esp32.git qemu_esp32_patch
cd qemu_esp32_patch/qemu-patch
./maketar.sh
cp qemu-esp32.tar ../../qemu

cd ../../qemu
tar xvf qemu-esp32.tar
```

in qemu source tree, manually add to makefiles:
```
hw/xtensa/Makefile.objs
  obj-y += esp32.o

target-xtensa/Makefile.objs
  obj-y += core-esp32.o
```

Interesting info. To be investigated. 
```
static HeapRegionTagged_t regions[]={
	{ (uint8_t *)0x3F800000, 0x20000, 15, 0}, //SPI SRAM, if available
	{ (uint8_t *)0x3FFAE000, 0x2000, 0, 0}, //pool 16 <- used for rom code
	{ (uint8_t *)0x3FFB0000, 0x8000, 0, 0}, //pool 15 <- can be used for BT
	{ (uint8_t *)0x3FFB8000, 0x8000, 0, 0}, //pool 14 <- can be used for BT
```

/* Use first 50 blocks in MMU for bootloader_mmap,
   50th block for bootloader_flash_read
*/
#define MMU_BLOCK0_VADDR  0x3f400000
#define MMU_BLOCK50_VADDR 0x3f720000
#define MMU_FLASH_MASK    0xffff0000
#define MMU_BLOCK_SIZE    0x00010000




#Adding flash qemu emulation.
You can assemble your own flash file with something like this,
esptool.py make_image -f app.text.bin -a 0x40100000 -f app.data.bin -a 0x3ffe8000 -f app.rodata.bin -a 0x3ffe8c00 app.flash.bin


These are some functions with associated io instructions, to help me understand.
```
From boatloader
Cache_Read_Disable(0);
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000020
io write 40,20 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
io read 58  DPORT_APP_CACHE_CTRL_REG  3ff00058=00000020
1 esp32_spi_read: +0xf8: 0x00000000
1 esp32_spi_read: +0x50: 0x00000004
1 esp32_spi_write: +0x50 = 0x00000004
written
---
Cache_Flush(0);  
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000020
io write 40,20 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000020
io write 40,30 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000030
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000030
io write 40,20  DPORT_PRO_CACHE_CTRL_REG 3ff00040
---

/**
  * @brief Set Flash-Cache mmu mapping.
  *        Please do not call this function in your SDK application.
  *
  * @param  int cpu_no : CPU number, 0 for PRO cpu, 1 for APP cpu.
  *
  * @param  int pod : process identifier. Range 0~7.
  *
  * @param  unsigned int vaddr : virtual address in CPU address space.
  *                              Can be IRam0, IRam1, IRom0 and DRom0 memory address.
  *                              Should be aligned by psize.
  *
  * @param  unsigned int paddr : physical address in Flash.
  *                              Should be aligned by psize.
  *
  * @param  int psize : page size of flash, in kilobytes. Should be 64 here.
  *
  * @param  int num : pages to be set.
  *
  * @return unsigned int: error status
  *                   0 : mmu set success
  *                   1 : vaddr or paddr is not aligned
  *                   2 : pid error
  *                   3 : psize error
  *                   4 : mmu table to be written is out of range
  *                   5 : vaddr is out of range
  */
unsigned int cache_flash_mmu_set(int cpu_no, int pid, unsigned int vaddr, unsigned int paddr,  int psize, int num);
#define MMU_BLOCK50_VADDR 0x3f720000
 int e = cache_flash_mmu_set(0, 0, MMU_BLOCK50_VADDR, map_at (0), 64, 1);
io read 44 DPORT_PRO_CACHE_CTRL1_REG  3ff00044=8E6
io read 5c  DPORT_APP_CACHE_CTRL1_REG  3ff0005C=000008E6
io read 5c  DPORT_APP_CACHE_CTRL1_REG  3ff0005C=000008E6
io write 5c,8ff   DPORT_APP_CACHE_CTRL1_REG  3ff0005C=000008FF
io read 44 DPORT_PRO_CACHE_CTRL1_REG  3ff00044=8E6
io write 44,8ff  DPORT_PRO_CACHE_CTRL1_REG  3ff00044  8ff
io write 100c8,0 
io write 44,8e6  DPORT_PRO_CACHE_CTRL1_REG  3ff00044  8e6
io write 5c,8e6  DPORT_APP_CACHE_CTRL1_REG  3ff0005C=000008E6
io read 44 DPORT_PRO_CACHE_CTRL1_REG  3ff00044=8E6
io write 44,8e6  DPORT_PRO_CACHE_CTRL1_REG  3ff00044  8e6
---
Cache_Read_Enable(0)
1 esp32_spi_read: +0x50: 0x00000004
1 esp32_spi_write: +0x50 = 0x00000005
written
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000020
io write 40,28 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
---


Cache_Read_Disable,
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000028
io write 40,20 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
io read 58  DPORT_APP_CACHE_CTRL_REG  3ff00058=00000028
---
Cache_Flush,
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000020
io write 40,28 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000028
io write 40,38 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000038
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000038
io write 40,28 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
---
cache_flash_mmu_set,
io read 44 DPORT_PRO_CACHE_CTRL1_REG  3ff00044=8E6
io read 5c  DPORT_APP_CACHE_CTRL1_REG  3ff0005C=000008E6
io read 5c  DPORT_APP_CACHE_CTRL1_REG  3ff0005C=000008E6
io write 5c,8ff 
 DPORT_APP_CACHE_CTRL1_REG  3ff0005C=000008FF
io read 44 DPORT_PRO_CACHE_CTRL1_REG  3ff00044=8E6
io write 44,8ff 
 DPORT_PRO_CACHE_CTRL1_REG  3ff00044  8ff
io write 10000,0 
 MMU CACHE  3ff10000  0
io write 44,8e6 
 DPORT_PRO_CACHE_CTRL1_REG  3ff00044  8e6
io write 5c,8e6 
 DPORT_APP_CACHE_CTRL1_REG  3ff0005C=000008E6
io read 44 DPORT_PRO_CACHE_CTRL1_REG  3ff00044=8E6
io write 44,8e6 
 DPORT_PRO_CACHE_CTRL1_REG  3ff00044  8e6
---
Cache_Read_Enable
1 esp32_spi_read: +0x50: 0x00000005
1 esp32_spi_write: +0x50 = 0x00000005
written
io read 40  DPORT_PRO_CACHE_CTRL_REG  3ff00040=00000028
io write 40,28 
DPORT_PRO_CACHE_CTRL_REG 3ff00040
---
We still get 
flash read err, 1000





This does not work.
head -c 4M /dev/zero > esp32.flash

xtensa-softmmu/qemu-system-xtensa -d guest_errors,page,unimp  -cpu esp32 -M esp32 -m 4M -pflash esp32.flash -kernel  ~/esp/qemu_esp32/build/app-template.elf  -s -S  > io.txt

This does not work at all. I dont understand qemu enough to get this working.
To add flash checkout line 502 in esp32.c
dinfo = drive_get(IF_PFLASH, 0, 0);
...
Also checkout m25p80.c driver in qemu

Dump with od,
od -t x4  partitions_singleapp.bin

xtensa-softmmu/qemu-system-xtensa -d guest_errors,page,unimp  -cpu esp32 -M esp32 -m 4M   -kernel  ~/esp/qemu_esp32/build/app-template.elf    > io.txt

```


Running two cores, works but APP CPU locks up because of missing flash emulation. Check  (gdb) info threads
```
xtensa-softmmu/qemu-system-xtensa -d guest_errors,page,unimp  -cpu esp32 -M esp32 -m 4M -smp 2 -pflash esp32.flash -kernel  ~/esp/qemu_esp32/build/app-template.elf  -s -S  > io.txt
ets Jun  8 2016 00:22:57

rst:0x10 (RTCWDT_RTC_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
I (97134) heap_alloc_caps: Initializing heap allocator:
I (97134) heap_alloc_caps: Region 19: 3FFB5B8C len 0002A474 tag 0
I (97134) heap_alloc_caps: Region 25: 3FFE8000 len 00018000 tag 1
check b=0x3ffb5b98 size=173144 ok
check b=0x3ffdfff0 size=0 ok
check b=0x3ffe800c size=98276 ok
I (97135) cpu_start: Pro cpu up.
I (97135) cpu_start: Starting app cpu, user_code_start is 0x00000000
I (97135) cpu_start: Starting app cpu, entry point is 0x40080b50
I (68034) cpu_start: App cpu up.
I (97140) cpu_start: App cpu started, user_code_start is 0x40080b50
I (97140) cpu_start: Pro cpu start user code
I (97140) rtc: rtc v160 Nov 22 2016 19:00:05
I (97141) rtc: XTAL 40M
I (97142) cpu_start: Starting scheduler on PRO CPU.
I (68253) cpu_start: Starting scheduler on APP CPU.

(gdb) info threads
  Id   Target Id         Frame 
  2    Thread 2 (CPU#1 [running]) 0x40081a75 in spi_flash_op_block_func (arg=0x1)
    at /home/olas/esp/esp-idf/components/spi_flash/./cache_utils.c:67
* 1    Thread 1 (CPU#0 [halted ]) 0x400d2444 in esp_vApplicationIdleHook ()
    at /home/olas/esp/esp-idf/components/esp32/./freertos_hooks.c:52

This was added to call_start_cpu0(),cpu_start.c for the extra user code start info
    int test= (void (*)(void))REG_READ(DPORT_APPCPU_CTRL_D_REG);
    void  *user_code_start =(void *) test;
    ESP_EARLY_LOGI(TAG, "Starting app cpu, user_code_start is %p", user_code_start);
```

## Detailed boot analaysis
[More about the boot of the romdumps](./BOOT.md)


##Patched gdb,
In the bin directory there is an improved gdb version. The following patch was applied,
https://github.com/jcmvbkbc/xtensa-toolchain-build/blob/master/fixup-gdb.sh
To use it properly you must increase the num_regs = 104 to i.e. 172 in core-esp32.c



## Another version of qemu for xtensa
As I am not alwas sure of what I am doing, I would recomend this version of the software. Currently it is not yet finnished (11//11) but will most certainly be better if Max finds the time to work on it.

#A better version of qemu with esp32 exists here,
   
    git clone https://github.com/OSLL/qemu-xtensa
    cd qemu-xtensa
    git checkout  xtensa-esp32
    git submodule update --init dtc
       Add /hw/xtensa/esp32.c as described eariler in this doc
       Edit this row
       serial_hds[0] = qemu_chr_new("serial0", "null",NULL);
    cd ..
    mkdir build-qemu-xtensa
    cd build-qemu-xtensa
    ../qemu-xtensa/configure --disable-werror --prefix=`pwd`/root --target-list=xtensa-softmmu

#Networking support
     There exists networking support of an emulated opencore network device,
     main/lwip_ethoc.c is the driver, without interrupts. A task poll_task() is created. 
     All files in the net direcory is just for reference, they are currently not used.

```  
     Dont try running   emulated_net(); on actual hardware. 
     Probably not harmful but I put the emulated hardware here, i uesd the DR_REG_EMAC_BASE

  #define	OC_BASE          0x3ff69000
  #define	OC_DESC_START    0x3ff69400
  #define	OC_BUF_START     0x3FFF8000          //pool 6 blk 1 <- can be used as trace memory

  TODO! Check if it is safe to use this memory?? 
  This could possibly screw up your data.

  Note this one  components/esp32/include/soc/soc.h
     DR_REG_EMAC_BASE                        0x3ff69000
     Here are the pins needed to get ethernet support.
     http://www.smartarduino.com/esp32/esp32_chip_pin_list_en.pdf
     Note that this could be outdated.

```  

#Qemu backend network options:
```  
    -netdev user,id=str[,ipv4[=on|off]][,net=addr[/mask]][,host=addr]
         [,ipv6[=on|off]][,ipv6-net=addr[/int]][,ipv6-host=addr]
         [,restrict=on|off][,hostname=host][,dhcpstart=addr]
         [,dns=addr][,ipv6-dns=addr][,dnssearch=domain][,tftp=dir]
         [,bootfile=f][,hostfwd=rule][,guestfwd=rule][,smb=dir[,smbserver=addr]]
                configure a user mode network backend with ID 'str',
                its DHCP server and optional services
    -netdev tap,id=str[,fd=h][,fds=x:y:...:z][,ifname=name][,script=file][,downscript=dfile]
         [,helper=helper][,sndbuf=nbytes][,vnet_hdr=on|off][,vhost=on|off]
         [,vhostfd=h][,vhostfds=x:y:...:z][,vhostforce=on|off][,queues=n]
         [,poll-us=n]
                configure a host TAP network backend with ID 'str'
                use network scripts 'file' (default=/etc/qemu-ifup)
                to configure it and 'dfile' (default=/etc/qemu-ifdown)
                to deconfigure it
                use '[down]script=no' to disable script execution
                use network helper 'helper' (default=/home/olas/qemu_esp32/root/libexec/qemu-bridge-helper) to
                configure it
                use 'fd=h' to connect to an already opened TAP interface
                use 'fds=x:y:...:z' to connect to already opened multiqueue capable TAP interfaces
                use 'sndbuf=nbytes' to limit the size of the send buffer (the
                default is disabled 'sndbuf=0' to enable flow control set 'sndbuf=1048576')
                use vnet_hdr=off to avoid enabling the IFF_VNET_HDR tap flag
                use vnet_hdr=on to make the lack of IFF_VNET_HDR support an error condition
                use vhost=on to enable experimental in kernel accelerator
                    (only has effect for virtio guests which use MSIX)
                use vhostforce=on to force vhost on for non-MSIX virtio guests
                use 'vhostfd=h' to connect to an already opened vhost net device
                use 'vhostfds=x:y:...:z to connect to multiple already opened vhost net devices
                use 'queues=n' to specify the number of queues to be created for multiqueue TAP
                use 'poll-us=n' to speciy the maximum number of microseconds that could be
                spent on busy polling for vhost net 
                
    -netdev bridge,id=str[,br=bridge][,helper=helper]
                configure a host TAP network backend with ID 'str' that is
                connected to a bridge (default=br0)
                using the program 'helper (default=/home/olas/qemu_esp32/root/libexec/qemu-bridge-helper)



     To debug lwip set #define LWIP_DEBUG 1 in cc.h
     I think there is some problem with the heaps. When disabling debug of  heaps we get an error here
     heap_alloc_caps_init()
     vPortDefineHeapRegionsTagged.

     I (32079) heap_alloc_caps: Initializing heap allocator:
     I (32079) heap_alloc_caps: Region 19: 3FFB5640 len 0002A9C0 tag 0
     I (32080) heap_alloc_caps: Region 25: 3FFE8000 len 00018000 tag 1

     http://www.freertos.org/thread-local-storage-pointers.html

     Add this for debugging in ethernet_input,ethernet.c.

    printf("ethernet_input: dest:%"X8_F":%"X8_F":%"X8_F":%"X8_F":%"X8_F":%"X8_F", src:%"X8_F":%"X8_F":%"X8_F":%"X8_F":%"X8_F":%"X8_F", type:%"X16_F"\n",
     (unsigned)ethhdr->dest.addr[0], (unsigned)ethhdr->dest.addr[1], (unsigned)ethhdr->dest.addr[2],
     (unsigned)ethhdr->dest.addr[3], (unsigned)ethhdr->dest.addr[4], (unsigned)ethhdr->dest.addr[5],
     (unsigned)ethhdr->src.addr[0], (unsigned)ethhdr->src.addr[1], (unsigned)ethhdr->src.addr[2],
     (unsigned)ethhdr->src.addr[3], (unsigned)ethhdr->src.addr[4], (unsigned)ethhdr->src.addr[5],
     (unsigned)htons(ethhdr->type));


     // Some threads when running lwip,
     "tiT"   tcpip
     "ech"
     "mai"   main task?
     "IDL"   idle task?

      Yes! Got it to work with this.
      xtensa-softmmu/qemu-system-xtensa -net nic,model=vlan0 -net user,id=simnet,ipver4=on,net=192.168.1.0/24,host=192.168.1.40,hostfwd=tcp::10077-192.168.1.3:7  -net dump,file=/tmp/vm0.pcap  -d guest_errors,unimp  -cpu esp32 -M esp32 -m 16M  -kernel  ~/esp/qemu_esp32/build/app-template.elf -s -S   > io.txt
      nc 127.0.0.1 10077 

      The options allows you to do connect to the echoserver
      in order to test the echo server that is running on port 7
      wireshark /tmp/vm0.pcap

```    





http://blog.vmsplice.net/2011/04/how-to-capture-vm-network-traffic-using.html

## Remote debugging with gdbstub.c
It is a good idea to save the original  xtensa-esp32-elf-gdb as the one in the bin directory works best tit qemu

```    
  Component config  --->
       ESP32-specific config  --->  
             Panic handler behaviour (Invoke GDBStub)  --->   
```

    esp32.c creates a thread with socket on port 8888 tto allow connect to the serial port over a socket.
    This allows connecting to panic handler gdbstub. 
    To enable this feature you must call,
        esp32_serial_init(system_io, 0x40000, "esp32.uart0",
                        xtensa_get_extint(env, 5), serial_hds[0]);
    I think the application switches the interrupt for this and thats why it is not working.

``` 
    xtensa-esp32-elf-gdb.sav  build/app-template.elf  -ex 'target remote:8888'
    Another solution if you are running qemu-gdb is to set a breakpoint in the stub.
    (gdb)b esp_gdbstub_panic_handler 
```
    This gdbstub panic handler is also nice to have when running on target.
    xtensa-esp32-elf-gdb   build/app-template.elf   -b 115200 -ex 'target remote /dev/ttyUSB0'
   
## Memory access breakpoints.

I noticed that memory got overwritten as priv_ethoc suddenly contained faulty iobase.
Memory access breakpoints comes very handy for these tyoes of errors.
```      
To track memory writes
(gdb) watch priv_ethoc.io_base
To track memory reads
(gdb) rwatch priv_ethoc.io_base
To track memory reads and writes
(gdb) awatch *0x3ffe0400
Together with add-symbol-file rom.elf 0x40000000 this allows you to easy find where  a register is accessed.
Here is an important register. DPORT_PRO_CACHE_CTRL_REG
(gdb) awatch *0x3ff00040
(gdb) info watchpoints
Dont forget that you can reset qemu hardware and start again.


## Rom symbols from rom.elf
To make debugging the rom functions there is a file rom.elf that contains debug information for the rom file. 
It was created with the help from this project, https://github.com/jcmvbkbc/esp-elf-rom
This rom.elf also works with the original gdb with panic handler gdbstub.

```      
        xtensa-softmmu/qemu-system-xtensa -d guest_errors,unimp  -cpu esp32 -M esp32 -m 4M  -kernel  ~/esp/qemu_esp32/build/app-template.elf  -s -S > io.txt
        xtensa-esp32-elf-gdb   build/app-template.elf  -ex 'target remote:1234'
       (gdb) add-symbol-file rom.elf 0x40000000
       (gdb) b start_cpu0_default
       (gdb) c
       (gdb) b app_main
```

