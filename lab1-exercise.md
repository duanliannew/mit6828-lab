# Part 1: Bootstrap PC
## Exercise 1
```
Familiarize yourself with the assembly language materials available on the 6.828 reference page.
You don't have to read them now, but you'll almost certainly want to refer to some of this material when reading and writing x86 assembly.
We do recommend reading the section "The Syntax" in Brennan's Guide to Inline Assembly.
It gives a good (and quite brief) description of the AT&T assembly syntax we'll be using with the GNU assembler in JOS.
```

## Exercise 2
```
Use GDB's si (Step Instruction) command to trace into the ROM BIOS for a few more instructions, and try to guess what it might be doing.
You might want to look at Phil Storrs I/O Ports Description, as well as other materials on the 6.828 reference materials page.
No need to figure out all the details - just the general idea of what the BIOS is doing first.

Answer: It does some device initialization.
(gdb) x/20i 0xfec0b
   0xfec0b:	cli
   0xfec0c:	cld
   0xfec0d:	mov    %ax,%cx
   0xfec10:	mov    $0x8f,%ax
   0xfec14:	add    %al,(%eax)
   0xfec16:	out    %al,$0x70
   0xfec18:	in     $0x71,%al
   0xfec1a:	in     $0x92,%al # FAST A20 enabling
   0xfec1c:	or     $0x2,%al
   0xfec1e:	out    %al,$0x92
   0xfec20:	mov    %cx,%ax
   0xfec23:	lidtl  %cs:(%esi)
   0xfec27:	sbb    %ah,0x2e(%edx)
   0xfec2a:	lgdtl  (%esi)
   0xfec2d:	fsubs  0xf(%ecx)
   0xfec30:	and    %al,%cl
   0xfec32:	and    $0xffff,%cx
   0xfec37:	lcall  *(%edi)
   0xfec39:	or     $0x1,%cx
   0xfec3d:	mov    %ecx,%cr0
```

## Exercise 3
```
Take a look at the lab tools guide, especially the section on GDB commands. Even if you're familiar with GDB, this includes some esoteric GDB commands that are useful for OS work.

Set a breakpoint at address 0x7c00, which is where the boot sector will be loaded. Continue execution until that breakpoint. Trace through the code in boot/boot.S, using the source code and the disassembly file obj/boot/boot.asm to keep track of where you are. Also use the x/i command in GDB to disassemble sequences of instructions in the boot loader, and compare the original boot loader source code with both the disassembly in obj/boot/boot.asm and GDB.

Trace into bootmain() in boot/main.c, and then into readsect(). Identify the exact assembly instructions that correspond to each of the statements in readsect(). Trace through the rest of readsect() and back out into bootmain(), and identify the begin and end of the for loop that reads the remaining sectors of the kernel from the disk. Find out what code will run when the loop is finished, set a breakpoint there, and continue to that breakpoint. Then step through the remainder of the boot loader.

disassemble instruction in gdb:
(gdb) x/30i 0x7c00
=> 0x7c00:	cli
   0x7c01:	cld
   0x7c02:	xor    %eax,%eax
   0x7c04:	mov    %eax,%ds
   0x7c06:	mov    %eax,%es
   0x7c08:	mov    %eax,%ss
   0x7c0a:	in     $0x64,%al
   0x7c0c:	test   $0x2,%al
   0x7c0e:	jne    0x7c0a
   0x7c10:	mov    $0xd1,%al
   0x7c12:	out    %al,$0x64
   0x7c14:	in     $0x64,%al
   0x7c16:	test   $0x2,%al
   0x7c18:	jne    0x7c14
   0x7c1a:	mov    $0xdf,%al
   0x7c1c:	out    %al,$0x60
   0x7c1e:	lgdtl  (%esi)
   0x7c21:	fs jl  0x7c33
   0x7c24:	and    %al,%al
   0x7c26:	or     $0x1,%ax
   0x7c2a:	mov    %eax,%cr0 # turn on protected mode
   0x7c2d:	ljmp   $0xb866,$0x87c32 # protected mode takes effective now
   0x7c34:	adc    %al,(%eax)
   0x7c36:	mov    %eax,%ds
   0x7c38:	mov    %eax,%es
   0x7c3a:	mov    %eax,%fs
   0x7c3c:	mov    %eax,%gs
   0x7c3e:	mov    %eax,%ss
   0x7c40:	mov    $0x7c00,%esp # setup stack for function call
   0x7c45:	call   0x7d19 # this is bootmain()

disassemble bootmain in gdb:
(gdb) x/34i 0x7d19
=> 0x7d19:	push   %ebp
   0x7d1a:	mov    %esp,%ebp
   0x7d1c:	push   %esi
   0x7d1d:	push   %ebx
   0x7d1e:	push   %edx
   0x7d1f:	push   $0x0
   0x7d21:	push   $0x1000
   0x7d26:	push   $0x10000
   0x7d2b:	call   0x7cda # call readseg to read elf header
   0x7d30:	add    $0x10,%esp
   0x7d33:	cmpl   $0x464c457f,0x10000 # check elf header magic number
   0x7d3d:	jne    0x7d77
   0x7d3f:	mov    0x1001c,%eax # read ELFHDR->e_phoff
   0x7d44:	movzwl 0x1002c,%esi # read ELFHDR->e_phnum
   0x7d4b:	lea    0x10000(%eax),%ebx # ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
   0x7d51:	shl    $0x5,%esi # sizeof Proghdr = 32
   0x7d54:	add    %ebx,%esi # the guard to exit loop
   0x7d56:	cmp    %esi,%ebx
   0x7d58:	jae    0x7d71
   0x7d5a:	push   %eax
   0x7d5b:	add    $0x20,%ebx
   0x7d5e:	push   -0x1c(%ebx) # ph->p_offset
   0x7d61:	push   -0xc(%ebx)  # ph->p_memsz
   0x7d64:	push   -0x14(%ebx) # ph->p_pa
   0x7d67:	call   0x7cda      # call readseg to read a section from elf file
   0x7d6c:	add    $0x10,%esp  # restore call stack
   0x7d6f:	jmp    0x7d56      # loop
   0x7d71:	call   *0x10018    # kernel entry: ((void (*)(void)) (ELFHDR->e_entry))();
   0x7d77:	mov    $0x8a00,%edx
   0x7d7c:	mov    $0xffff8a00,%eax
   0x7d81:	out    %ax,(%dx)
   0x7d83:	mov    $0xffff8e00,%eax
   0x7d88:	out    %ax,(%dx)
   0x7d8a:	jmp    0x7d8a

trace into os kernel:
(gdb) b *0x7d30
Breakpoint 1 at 0x7d30
(gdb) c
Continuing.
The target architecture is set to "i386".
=> 0x7d30:	add    $0x10,%esp

Breakpoint 1, 0x00007d30 in ?? ()
(gdb) x/1xw 0x10018
0x10018:	0x0010000c
(gdb) b *0x0010000c
Breakpoint 2 at 0x10000c
(gdb) c
Continuing.
=> 0x10000c:	movw   $0x1234,0x472  # entry.S:entry

Breakpoint 2, 0x0010000c in ?? ()


Question: At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?
Answer: From label protcseg, CPU starts executing in 32-bit mode.
        The exact mechanism that causes the switch from 16-bit to 32-bit mode is below code fragment:
        orl     $CR0_PE_ON, %eax
        movl    %eax, %cr0
        followed by a "ljmp    $PROT_MODE_CSEG, $protcseg"

Question: What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded?
Answer: The last instruction of the bootloader executed by processor is "call   *0x10018".
        The first instruction of th kernel loaded by boot loader is "movw   $0x1234,0x472".

Question: Where is the first instruction of the kernel?
Answer: It is located in source file entry.S:entry.

Question: How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
Answer: Boot loader reads kernel code in elf-format, it first reads elf header and find out how many sections are there inside the elf file, what the offset of each section is, what the size of each section is, at what address each section should be loaded.
```

## Exercise 4
```
Read about programming with pointers in C. The best reference for the C language is The C Programming Language by Brian Kernighan and Dennis Ritchie (known as 'K&R').
We recommend that students purchase this book (here is an Amazon Link) or find one of MIT's 7 copies.


Answer: Actually I am quite familiar with C programming language, no need to read that stuff.
```

## Exercise 5
```
Trace through the first few instructions of the boot loader again and identify the first instruction that would "break" or otherwise do the wrong thing if you were to get the boot loader's link address wrong. Then change the link address in boot/Makefrag to something wrong, run make clean, recompile the lab with make, and trace into the boot loader again to see what happens. Don't forget to change the link address back and make clean again afterward!

Answer:
I guess the first instruction inside boot.S that breaks is "lgdt    gdtdesc", and we can observe the effect at instruction "ljmp    $PROT_MODE_CSEG, $protcseg", if you mess up its link address. Try to verify that, after I change the link address from 0x7c00 to 0x8c00 and trace through boot loader, gdb outputs like below, which demonstrates that we cannot jump to the right place to continue executing.

[   0:7c1e] => 0x7c1e:	lgdtl  (%esi) # This output might be wrong, cannot figure out why.
0x00007c1e in ?? ()
(gdb) info registers esi
esi            0x0                 0
(gdb) si
[   0:7c23] => 0x7c23:	mov    %cr0,%eax
0x00007c23 in ?? ()
(gdb) 
[   0:7c26] => 0x7c26:	or     $0x1,%ax
0x00007c26 in ?? ()
(gdb) 
[   0:7c2a] => 0x7c2a:	mov    %eax,%cr0
0x00007c2a in ?? ()
(gdb) 
[   0:7c2d] => 0x7c2d:	ljmp   $0xb866,$0x88c32  # oops, something goes wrong
0x00007c2d in ?? ()
(gdb) 
[f000:e05b]    0xfe05b:	cmpw   $0x28,%cs:(%esi)


```

## Exercise 6
```
We can examine memory using GDB's x command. The GDB manual has full details, but for now, it is enough to know that the command x/Nx ADDR prints N words of memory at ADDR. (Note that both 'x's in the command are lowercase.) Warning: The size of a word is not a universal standard. In GNU assembly, a word is two bytes (the 'w' in xorw, which stands for word, means 2 bytes).

Reset the machine (exit QEMU/GDB and start them again). Examine the 8 words of memory at 0x00100000 at the point the BIOS enters the boot loader, and then again at the point the boot loader enters the kernel. Why are they different? What is there at the second breakpoint? (You do not really need to use QEMU to answer this question. Just think.)

Answer: 
(gdb) x/8xb 0x100000  # just enter the boot loader
0x100000:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00

(gdb) x/8xb 0x100000 # about to enter the kernel, the memory content is little endian encoded MULTIBOOT_HEADER_MAGIC
0x100000:	0x02	0xb0	0xad	0x1b	0x00	0x00	0x00	0x00  # #define MULTIBOOT_HEADER_MAGIC (0x1BADB002)

(gdb) x/8xb 0x10000 # this is where elf header is stored, #define ELF_MAGIC 0x464C457FU
0x10000:	0x7f	0x45	0x4c	0x46	0x01	0x01	0x01	0x00
```