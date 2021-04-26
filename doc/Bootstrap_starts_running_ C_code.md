# Bootstrap starts running C code

## main

```c
int main(void)
{
  kinit1(end, P2V(4 * 1024 * 1024));          // phys page allocator
  kvmalloc();                                 // kernel page table
  mpinit();                                   // detect other processors
  lapicinit();                                // interrupt controller
  seginit();                                  // segment descriptors
  picinit();                                  // disable pic
  ioapicinit();                               // another interrupt controller
  consoleinit();                              // console hardware
  uartinit();                                 // serial port
  pinit();                                    // process table
  tvinit();                                   // trap vectors
  binit();                                    // buffer cache
  fileinit();                                 // file table
  ideinit();                                  // disk
  startothers();                              // start other processors
  kinit2(P2V(4 * 1024 * 1024), P2V(PHYSTOP)); // must come after startothers()
  userinit();                                 // first user process
  mpmain();                                   // finish this processor's setup
}
```

## kinit1() 



* Initialization happens in two phases. 
* main() calls kinit1() while still using entrypgdir to place just  the  pages mapped by entrypgdir on free list. 
* main() calls kinit2() with the rest of the physical pages after installing a full page table that maps them on all cores.

```
void kinit1(void *vstart, void *vend)
{
  initlock(&kmem.lock, "kmem");
  kmem.use_lock = 0;
  freerange(vstart, vend);
}
```





## kvmalloc()

* Allocate one page table for the machine for the kernel address
* space for scheduler processes.

``` c
void kvmalloc (void)
{
  kpgdir = setupkvm();
  switchkvm();
}
```



### setupkvm ()

* setup kernel part of a page table.

```c
pde_t * setupkvm(void)
{
  pde_t *pgdir;
  struct kmap *k;
  if ((pgdir = (pde_t *)kalloc()) == 0)  return 0;
  memset(pgdir, 0, PGSIZE);
  if (P2V(PHYSTOP) > (void *)DEVSPACE) panic("PHYSTOP too high");
  for (k = kmap; k < &kmap[NELEM(kmap)]; k++)
    if (mappages(pgdir, k->virt, k->phys_end - k->phys_start, (uint)k->phys_start, k->perm) < 0)
    {
      freevm(pgdir);
      return 0;
    }
  return pgdir;
}
```



#### kalloc ()

* Allocate one 4096-byte page of physical memory.
*  Returns a pointer that the kernel can use.
*  Returns 0 if the memory cannot be allocated

```c
char * kalloc(void)
{
  struct run *r;

  if (kmem.use_lock)
    acquire(&kmem.lock);
  r = kmem.freelist;
  if (r)
    kmem.freelist = r->next;
  if (kmem.use_lock)
    release(&kmem.lock);
  return (char *)r;
}
```



##### kmap

* This table defines the kernel's mappings, which are present in every process's page table.

```c
static struct kmap
{
  void *virt;
  uint phys_start;
  uint phys_end;
  int perm;
} kmap[] = {
    {(void *)KERNBASE, 0, EXTMEM, PTE_W},            // I/O space
    {(void *)KERNLINK, V2P(KERNLINK), V2P(data), 0}, // kern text+rodata
    {(void *)data, V2P(data), PHYSTOP, PTE_W},       // kern data+memory
    {(void *)DEVSPACE, DEVSPACE, 0, PTE_W},          // more devices
};
```



## mpinit() detect other processors

```c

void mpinit(void)
{
  uchar *p, *e;
  int ismp;
  struct mp *mp;
  struct mpconf *conf;
  struct mpproc *proc;
  struct mpioapic *ioapic;

  if ((conf = mpconfig(&mp)) == 0)
    panic("Expect to run on an SMP");
  ismp = 1;
  lapic = (uint *)conf->lapicaddr;
  for (p = (uchar *)(conf + 1), e = (uchar *)conf + conf->length; p < e;)
  {
    switch (*p)
    {
    case MPPROC:
      proc = (struct mpproc *)p;
      if (ncpu < NCPU)
      {
        cpus[ncpu].apicid = proc->apicid; // apicid may differ from ncpu
        ncpu++;
      }
      p += sizeof(struct mpproc);
      continue;
    case MPIOAPIC:
      ioapic = (struct mpioapic *)p;
      ioapicid = ioapic->apicno;
      p += sizeof(struct mpioapic);
      continue;
    case MPBUS:
    case MPIOINTR:
    case MPLINTR:
      p += 8;
      continue;
    default:
      ismp = 0;
      break;
    }
  }
  if (!ismp)
    panic("Didn't find a suitable machine");

  if (mp->imcrp)
  {
    // Bochs doesn't support IMCR, so this doesn't run on Bochs.
    // But it would on real hardware.
    outb(0x22, 0x70);          // Select IMCR
    outb(0x23, inb(0x23) | 1); // Mask external interrupts.
  }
}
```



## lapicinit() : interrupt controller



## seginit() : segment descriptors

* Set up CPU's kernel segment descriptors.
*  Run once on entry on each CPU.

```c
void seginit(void)
{
  struct cpu *c;

  // Map "logical" addresses to virtual addresses using identity map.
  // Cannot share a CODE descriptor for both kernel and user
  // because it would have to have DPL_USR, but the CPU forbids
  // an interrupt from CPL=0 to DPL=3.
  c = &cpus[cpuid()];
  c->gdt[SEG_KCODE] = SEG(STA_X | STA_R, 0, 0xffffffff, 0);
  c->gdt[SEG_KDATA] = SEG(STA_W, 0, 0xffffffff, 0);
  c->gdt[SEG_UCODE] = SEG(STA_X | STA_R, 0, 0xffffffff, DPL_USER);
  c->gdt[SEG_UDATA] = SEG(STA_W, 0, 0xffffffff, DPL_USER);
  lgdt(c->gdt, sizeof(c->gdt));
}
```





## pincinit(): disable pic

*  Don't use the 8259A interrupt controllers.  Xv6 assumes SMP hardware.

```c
void picinit(void)
{
  // mask all interrupts
  outb(IO_PIC1+1, 0xFF);
  outb(IO_PIC2+1, 0xFF);
}
```



## ipapicinit() : another interrupt controller

```c

void ioapicinit(void)
{
  int i, id, maxintr;

  ioapic = (volatile struct ioapic *)IOAPIC;
  maxintr = (ioapicread(REG_VER) >> 16) & 0xFF;
  id = ioapicread(REG_ID) >> 24;
  if (id != ioapicid)
    cprintf("ioapicinit: id isn't equal to ioapicid; not a MP\n");

  // Mark all interrupts edge-triggered, active high, disabled,
  // and not routed to any CPUs.
  for (i = 0; i <= maxintr; i++)
  {
    ioapicwrite(REG_TABLE + 2 * i, INT_DISABLED | (T_IRQ0 + i));
    ioapicwrite(REG_TABLE + 2 * i + 1, 0);
  }
}
```



## consoleinit : console hardware

```c
void consoleinit(void)
{
  initlock(&cons.lock, "console");

  devsw[CONSOLE].write = consolewrite;
  devsw[CONSOLE].read = consoleread;
  cons.locking = 1;

  ioapicenable(IRQ_KBD, 0);
}

```



## uartinit : serial port



```c
void uartinit(void)
{
  char *p;

  // Turn off the FIFO
  outb(COM1 + 2, 0);

  // 9600 baud, 8 data bits, 1 stop bit, parity off.
  outb(COM1 + 3, 0x80); // Unlock divisor
  outb(COM1 + 0, 115200 / 9600);
  outb(COM1 + 1, 0);
  outb(COM1 + 3, 0x03); // Lock divisor, 8 data bits.
  outb(COM1 + 4, 0);
  outb(COM1 + 1, 0x01); // Enable receive interrupts.

  // If status is 0xFF, no serial port.
  if (inb(COM1 + 5) == 0xFF)
    return;
  uart = 1;

  // Acknowledge pre-existing interrupt conditions;
  // enable interrupts.
  inb(COM1 + 2);
  inb(COM1 + 0);
  ioapicenable(IRQ_COM1, 0);

  // Announce that we're here.
  for (p = "xv6...\n"; *p; p++)
    uartputc(*p);
}
```



## pinit: process table

```c
void pinit(void)
{
  initlock(&ptable.lock, "ptable");
}

void initlock(struct spinlock *lk, char *name)
{
  lk->name = name;
  lk->locked = 0;
  lk->cpu = 0;
}
```



## tvinit : trap vectors 

```c
void tvinit(void)
{
  int i;

  for (i = 0; i < 256; i++) SETGATE(idt[i], 0, SEG_KCODE << 3, vectors[i], 0);
  SETGATE(idt[T_SYSCALL], 1, SEG_KCODE << 3, vectors[T_SYSCALL], DPL_USER);

  initlock(&tickslock, "time");
}
```

### gate descriptors for interrupts and traps

```c
// Gate descriptors for interrupts and traps
struct gatedesc {
  uint off_15_0 : 16;   // low 16 bits of offset in segment
  uint cs : 16;         // code segment selector
  uint args : 5;        // # args, 0 for interrupt/trap gates
  uint rsv1 : 3;        // reserved(should be zero I guess)
  uint type : 4;        // type(STS_{IG32,TG32})
  uint s : 1;           // must be 0 (system)
  uint dpl : 2;         // descriptor(meaning new) privilege level
  uint p : 1;           // Present
  uint off_31_16 : 16;  // high bits of offset in segment
};

// Set up a normal interrupt/trap gate descriptor.
// - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate.
//   interrupt gate clears FL_IF, trap gate leaves FL_IF alone
// - sel: Code segment selector for interrupt/trap handler
// - off: Offset in code segment for interrupt/trap handler
// - dpl: Descriptor Privilege Level -
//        the privilege level required for software to invoke
//        this interrupt/trap gate explicitly using an int instruction.
#define SETGATE(gate, istrap, sel, off, d)                \
{                                                         \
  (gate).off_15_0 = (uint)(off) & 0xffff;                \
  (gate).cs = (sel);                                      \
  (gate).args = 0;                                        \
  (gate).rsv1 = 0;                                        \
  (gate).type = (istrap) ? STS_TG32 : STS_IG32;           \
  (gate).s = 0;                                           \
  (gate).dpl = (d);                                       \
  (gate).p = 1;                                           \
  (gate).off_31_16 = (uint)(off) >> 16;                  \
}

#endif
```





## binit : buffer cache

```c
struct
{
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // head.next is most recently used.
  struct buf head;
} bcache;

void binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  //PAGEBREAK!
  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for (b = bcache.buf; b < bcache.buf + NBUF; b++)
  {
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}
```



## fileinit() : file table

```c
struct
{
  struct spinlock lock;
  struct file file[NFILE];
} ftable;

void fileinit(void)
{
  initlock(&ftable.lock, "ftable");
}

// Allocate a file structure.
struct file *
filealloc(void)
{
  struct file *f;

  acquire(&ftable.lock);
  for (f = ftable.file; f < ftable.file + NFILE; f++)
  {
    if (f->ref == 0)
    {
      f->ref = 1;
      release(&ftable.lock);
      return f;
    }
  }
  release(&ftable.lock);
  return 0;
}
```



## ideinit() : disk

```c
extern uchar _binary_fs_img_start[], _binary_fs_img_size[];

static int disksize;
static uchar *memdisk;

void
ideinit(void)
{
  memdisk = _binary_fs_img_start;
  disksize = (uint)_binary_fs_img_size/BSIZE;
}

```



## startothers:  start other processors 

```c

// Start the non-boot (AP) processors.
static void
startothers(void)
{
  extern uchar _binary_entryother_start[], _binary_entryother_size[];
  uchar *code;
  struct cpu *c;
  char *stack;

  // Write entry code to unused memory at 0x7000.
  // The linker has placed the image of entryother.S in
  // _binary_entryother_start.
  code = P2V(0x7000);
  memmove(code, _binary_entryother_start, (uint)_binary_entryother_size);

  for (c = cpus; c < cpus + ncpu; c++)
  {
    if (c == mycpu()) // We've started already.
      continue;

    // Tell entryother.S what stack to use, where to enter, and what
    // pgdir to use. We cannot use kpgdir yet, because the AP processor
    // is running in low  memory, so we use entrypgdir for the APs too.
    stack = kalloc();
    *(void **)(code - 4) = stack + KSTACKSIZE;
    *(void (**)(void))(code - 8) = mpenter;
    *(int **)(code - 12) = (void *)V2P(entrypgdir);

    lapicstartap(c->apicid, V2P(code));

    // wait for cpu to finish mpmain()
    while (c->started == 0)
      ;
  }
}
```



## kinit2 : must com after startothers 

```c
void kinit2(void *vstart, void *vend)
{
  freerange(vstart, vend);
  kmem.use_lock = 1;
}
```



## userinit  : first user process

```c
// Set up first user process.
void userinit(void)
{
  struct proc *p;
  extern char _binary_initcode_start[], _binary_initcode_size[];

  p = allocproc();

  initproc = p;
  if ((p->pgdir = setupkvm()) == 0)
    panic("userinit: out of memory?");
  inituvm(p->pgdir, _binary_initcode_start, (int)_binary_initcode_size);
  p->sz = PGSIZE;
  memset(p->tf, 0, sizeof(*p->tf));
  p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
  p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
  p->tf->es = p->tf->ds;
  p->tf->ss = p->tf->ds;
  p->tf->eflags = FL_IF;
  p->tf->esp = PGSIZE;
  p->tf->eip = 0; // beginning of initcode.S

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  // this assignment to p->state lets other cores
  // run this process. the acquire forces the above
  // writes to be visible, and the lock is also needed
  // because the assignment might not be atomic.
  acquire(&ptable.lock);

  p->state = RUNNABLE;

  release(&ptable.lock);
}
```



## mpmain : finish this processors' setup

```c
// Common CPU setup code.
static void
mpmain(void)
{
  cprintf("cpu%d: starting %d\n", cpuid(), cpuid());
  idtinit();                    // load idt register
  xchg(&(mycpu()->started), 1); // tell startothers() we're up
  scheduler();                  // start running processes
}
```



### ### scheduler(void)

```c
//PAGEBREAK: 42
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run
//  - swtch to start running that process
//  - eventually that process transfers control
//      via swtch back to the scheduler.
void scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  c->proc = 0;

  for (;;)
  {
    // Enable interrupts on this processor.
    sti();

    // Loop over process table looking for process to run.
    acquire(&ptable.lock);
    for (p = ptable.proc; p < &ptable.proc[NPROC]; p++)
    {
      if (p->state != RUNNABLE)
        continue;

      // Switch to chosen process.  It is the process's job
      // to release ptable.lock and then reacquire it
      // before jumping back to us.
      c->proc = p;
      switchuvm(p);
      p->state = RUNNING;

      swtch(&(c->scheduler), p->context);
      switchkvm();

      // Process is done running for now.
      // It should have changed its p->state before coming back.
      c->proc = 0;
    }
    release(&ptable.lock);
  }
}
```

