
# gdb
## gdb 시작하기

실행절차
```
#include <stdio.h>
int main(int argc, char *argv[]) {
  int  i,sum=0;
  for (i=0; i<5; i++){
        sum +=i;
        printf("[%d]\n",sum);
  }
  printf("[%d]\n",sum);
}
```
* 디버그 모드 컴파일 
```
good@code:~/code/t$ gcc -g t.c  -o t
good@code:~/code/t$ gdb t
(gdb) list
1       #include <stdio.h>
2
3       int main(int argc, char *argv[]) {
4         int  i,sum=0;
5         for (i=0; i<5; i++){
6               sum +=i;
7               printf("[%d]\n",sum);
8               }
9         printf("[%d]\n",sum);
10      }
(gdb)
```
* break/run
```
(gdb) b 4
Breakpoint 1 at 0x115c: file t.c, line 4.
(gdb) run
Starting program: /home/good/code/t/t

Breakpoint 1, main (argc=1, argv=0x7fffffffe4b8) at t.c:4
4         int  i,sum=0;
```
* step은 sub루팅이 있으면 그 안으로 들어간다. 
* next는 sub루틴을 호출하면서 다음으로 진행한다
```
(gdb) s
5         for (i=0; i<5; i++){
(gdb) s
6               sum +=i;
(gdb) s
7               printf("[%d]\n",sum);
(gdb) s
__printf (format=0x555555556004 "[%d]\n") at printf.c:28
28      printf.c: 그런 파일이나 디렉터리가 없습니다.
(gdb) list
23      in printf.c
(gdb) n
[1]
5         for (i=0; i<5; i++){
(gdb) n
6               sum +=i;
(gdb) n
7               printf("[%d]\n",sum);
(gdb) p sum
$6 = 3
(gdb) p i
$7 = 2
```
* info

```
(gdb) info locals //local 변수들 
i = 2
sum = 3

(gdb) info stack  //stack 
#0  main (argc=1, argv=0x7fffffffe4b8) at t.c:5

(gdb) info registers
rax            0x4                 4
rbx            0x5555555551b0      93824992235952
rcx            0x0                 0
rdx            0x0                 0
rsi            0x5555555592a0      93824992252576
rdi            0x7ffff7faf4c0      140737353807040
rbp            0x7fffffffe3c0      0x7fffffffe3c0
rsp            0x7fffffffe3a0      0x7fffffffe3a0
r8             0x0                 0
r9             0x4                 4
r10            0x555555556007      93824992239623
r11            0x246               582
r12            0x555555555060      93824992235616
r13            0x7fffffffe4b0      140737488348336
r14            0x0                 0
r15            0x0                 0
rip            0x555555555188      0x555555555188 <main+63>
eflags         0x202               [ IF ]
cs             0x33                51
ss             0x2b                43
ds             0x0                 0
es             0x0                 0
fs             0x0                 0
gs             0x0                 0
```

## gdb in xv6

 We provide a file called `.gdbinit` which automatically sets up GDB for use with QEMU.
- Must run GDB from the lab or xv6 directory
- Edit ~/.gdbinit to allow other gdbinits

Use make to start QEMU with or without GDB.
- With GDB: run make qemu[-nox]-gdb, then start GDB in a second shell
- Use make qemu[-nox] when you don’t need GDB
- to exit consol `ctl-a,x`

## gdb command
Run help <command-name> if you’re not sure how to use a command.
- All commands may be abbreviated(축약) if unambiguous:
  * c = co = cont = continue
- Some additional abbreviations are defined, e.g.
  * s = step and si = stepi


### stepping
* step runs one line of code at a time. When there is a function call, it steps into the called function.
* next does the same thing, except that it steps over function calls.
* stepi and nexti : do the same thing for assembly instructions rather than lines of code.

* All take a numerical argument to specify repetition.
* Pressing the enter key repeats the previous command.

### runnign 
* continue runs code until a breakpoint is encountered or
you interrupt it with Control-C.
* finish runs code until the current function returns
* advance <location> runs code until the instruction  pointer gets to the specified location.


### breakpoints 

* break <location> sets a breakpoint at the specified location.

* Locations can be memory addresses (“*0x7c00”) or names (“mon backtrace”, “monitor.c:71”).
* Modify breakpoints using delete, disable, enable.

### Conditional breakpoints 

* break <location> if <condition> sets a breakpoint at the specified location, but only breaks if the condition is satisfied.

* cond <number> <condition> adds a condition on an existing breakpoint.

### watch points 

* Like breakpoints, but with more complicated conditions.
* watch <expression> will stop execution whenever the expression’s value changes.
* watch -l <address> will stop execution whenever  the contents of the specified memory address change.
* What’s the difference between wa var and wa -l &var?
* rwatch [-l] <expression> will stop execution whenever the value of the expression is read.

* `info registers`: prints the value of every register.
* `info frame` :  prints the current stack frame
* `list <location>` :  prints the source code of the function
at the specified location.


### Examining 

* x prints the raw contents of memory in whatever format
you specify (x/x for hexadecimal, x/i for assembly, etc).
* print evaluates a C expression and prints the result as
its proper type. It is often more useful than x.
* The output from p *((struct elfhdr *) 0x10000)
is much nicer than the output from x/13x 0x10000.
* backtrace might be useful as you work on lab 1!

### Layout
* GDB has a text user interface that shows useful
information like code listing, disassembly, and register
contents in a curses UI.
* layout <name> switches to the given layout.

### Other tricks 
* You can use the set command to change the value of a
variable during execution.
* You have to switch symbol files to get function and
variable names for environments other than the kernel.
* For example, when debugging JOS:
  - symbol-file obj/user/<name>
  - symbol-file obj/kern/kernel






## homework solution

```
From bootasm.S:
# Set up the stack pointer and call into C.
movl $start, %esp
call bootmain

# Set up the stack pointer and call into C.
movl $start, %esp
call bootmain
Later, in bootmain():

// Call the entry point from the ELF header.
// Does not return!
entry = (void(*)(void))(elf->entry);
entry();

````

* call bootmain pushes a return address
* The prologue in bootmain() makes a stack frame

````
push %ebp
mov %esp,%ebp
push %edi
push %esi
push %ebx
sub $0x1c,%esp
````

call bootmain pushes a return address
The prologue in bootmain() makes a stack frame
push %ebp
mov %esp,%ebp
push %edi
push %esi
push %ebx
sub $0x1c,%esp
The call to entry() pushes a return address
