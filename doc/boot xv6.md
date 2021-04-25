# boot xv6

## xv6 커널 컴파일

### kernel 

Makefile

```makefile
kernel: $(OBJS) entry.o entry initcode kernel.ld
	$(LD) $(LDFLAGS) -T kernel.ld -o kernel entry.o $(OBJS) -b binary initcode entryother
	$(OBJDUMP) -S kernel > kernel.asm
	$(OBJDUMP) -t kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > kernel.sym
```

* $(LD) 단계에서 kernel.ld  설정 참조
* ENTRY(_start) 로 정의되어 있고, `.txt`는   `.txt: AT(0x100000)` 로 정의되어 있음
* ELF란 Excutable하고 Linkable한 File의 format

###  xv6 start 

* qemu boot file: xv6.img

```makefile
QEMUOPTS = -drive file=fs.img,index=1,media=disk,format=raw -drive file=xv6.img,index=0,media=disk,format=raw -smp $(CPUS) -m 512 $(QEMUEXTRA)

qemu: fs.img xv6.img
	$(QEMU) -serial mon:stdio $(QEMUOPTS)
```

* booting 디스크 xv6.img 생성

```makefile
xv6.img: bootblock kernel
	dd if=/dev/zero of=xv6.img count=10000
	dd if=bootblock of=xv6.img conv=notrunc
	dd if=kernel of=xv6.img seek=1 conv=notrunc   // 한블럭을 skip하고 kernel 파일을 기록한다.
```

* /dev/zero 통해서 512byte*10,000 크기의 파일을 생성 5MB
* rootblock 파일을 xv6.img에 기록
* 1블럭을 skip하고 kernel 파일을 xv6.im에 기록

QEMU에서 기동을 시작할 boot b파일에 boot block과 kernel 위치에 기록해서 부팅 준비

### bootasm.S

```makefile
bootblock: bootasm.S bootmain.c
	$(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
	$(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
	$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o
	$(OBJDUMP) -S bootblock.o > bootblock.asm
	$(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
	./sign.pl bootblock
```

* 여기서 bootblock 을 ELF 형태로 만들때  start 위치를 -Ttext 0x7c00으로 정의한다.
* 따라서 bootblokc이 로딩되고 시작되는 첫 주소는 0x7c00이 된다.
* 시스템 부팅이 시작되면 BIOS가 부팅 디스크의 첫번째 sector를  물리주소 `0x7c00` 주소에 올려 놓는다.
* 그리고  real mode에서`%cs=0 %ip=0x7c00`로 설정해 놓기 때문에 CPU가 real mode에서 시작할 수 있게 된다.
* `bootasm`-은 stack point를 start32로 설정하고  -> bootmain.c를 호출한다.

### bootmain.c

```c
void bootmain(void)
{
  struct elfhdr *elf;
  struct proghdr *ph, *eph;
  void (*entry)(void);
  uchar *pa;

  elf = (struct elfhdr *)0x10000; // scratch space

  // Read 1st page off disk
  readseg((uchar *)elf, 4096, 0);

  // Is this an ELF executable?
  if (elf->magic != ELF_MAGIC)
    return; // let bootasm.S handle error

  // Load each program segment (ignores ph flags).
  ph = (struct proghdr *)((uchar *)elf + elf->phoff);
  eph = ph + elf->phnum;
  for (; ph < eph; ph++)
  {
    pa = (uchar *)ph->paddr;
    readseg(pa, ph->filesz, ph->off);
    if (ph->memsz > ph->filesz)
      stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
  }

  // Call the entry point from the ELF header.
  // Does not return!
  entry = (void (*)(void))(elf->entry);
  entry();
}
```

* bootmain의 핵심 역할은 kernel 파일을 실행 할 수 있는 준비를 하고 enry()를 호출 하는 것이다.

* kernel의 entry는 `_start`이고  이것은 kernel 파일 생성할 때 그 위치를 ld 스크립트에 정의 했다.

* kernel 파일의 _start 위치는 0x0010000c로 설정된다. 1MB+12bytes

* ```c
  OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
  OUTPUT_ARCH(i386)
  ENTRY(_start)
  SECTIONS
  {
  	. = 0x80100000;
  	.text : AT(0x100000) {
  		*(.text .stub .text.* .gnu.linkonce.t.*)
  	}
  ```



## Entry Point of Kernel:

Find the address of _start

```sh
$ nm kernel | grep _start
8010a48c D _binary_entryother_start
8010a460 D _binary_initcode_start
0010000c T _start
```



