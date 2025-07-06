---
title: "Let GDB do the work"
date: 2025-06-22
author: Fiti
description: How to automate GDB debugging with Python
tags:
  - python
  - debug
  - gdb
  - automate
---

# Let GDB do the work: Automatic debugging in GDB with Python

## Introduction

In the life of an embedded developer, sometimes it's necessary to use tools for debugging, like the principal debugger: GDB (not `printf`). Some bugs are very difficult to find, and it allows us to see registers, variables, inspect the flow, perform some modifications, etc., in real time, which could be key to finding the root of the issue.

Manual debugging is fine. But why not let the debugger find the issue for us? Wouldn't it be great to automate GDB and have it assist us in finding issues?

### Why automate debugging?

Automating the debugging task can be useful for some key reasons:

- It allows us to **explore** and **execute** things we can't do manually.
- Scripts get things done faster.
- Am I really justifying why we should automate this kind of stuff? :)

In this particular case, **using the GDB CLI is slow**, since you write sequentially, usually in a very limited terminal like GDB (no recursive search, for example), and working on it can be tedious if the debugging session lasts long.

### What we would need

Of course, we'll need `python` and `gdb` installed. **Important**: GDB must have Python support. You can check if it does by running:

```shell
$ gdb --configuration | grep python
	     --with-python=/usr (relocatable)
	     --with-python-libdir=/usr/lib (relocatable)
```

The last resource we will need will be creativity and... a bit of `gdb` cli knowledge.

> I always recommend to people who are starting with C (especially debugging) to learn to use the `CLI` before the `GUI` that most IDEs integrate. (Note: This also applies to Git or anything else that a GUI makes easier to use.) The reason behind this _learn-hard_ mentality is that if you know how things work behind the scenes, you'll understand how far they can go and what the real potential of the tool is. If you're used to using the GUI, it will definitely be easier-but when things get hard, you'll be limited.

> Once you know how the CLI works, you can then take advantage of GUIs to ease your day-to-day tasks.

### Examples
#### First case: No external tool is required for automate debugging.

In this example, we'll work with a piece of software that runs a very simple example of a bad program. 

**DISCLAIMER**: This software is just an example to enforce the use of automation. Real cases tend to be a bit harder :).

>Note: To keep things fun, I've replaced the actual code with some ASM (arm64). Let's find out what `mysteryFunction()` is up to!

```c
#include <stdio.h>
#include <stdint.h>


uint32_t misteryFunction(uint32_t v) {
	uint32_t result;
	__asm__ volatile (
			"mov w1, w0\n\t"
			"mov w2, #0\n\t"
			"1:\n\t"
			"and w3, w1, #1\n\t"
			"add w2, w2, w3\n\t"
			"lsr w1, w1, #1\n\t"
			"cbnz w1, 1b\n\t"
			"mov w0, w2\n\t"
			: "=r"(result)
			: "r"(v)
			: "w1", "w2", "w3"
			);
	return result;
}

// Do some stuff.
uint32_t badFunction(uint32_t a) {
	uint32_t n = misteryFunction(a);
	if (n == 7) {
		return (int)*(int*)0; // <<<< BUG!!!
	}

	return n;
}


int main(int argc, char *argv[]) {
	printf("Let's calculate something with your number!\n");
	int r = atoi(argv[1]);

	printf("The operation with your number %d is %d\n", r, badFunction(r));
}
```

If we compile the program and execute it, we are most likely to obtain the following output:

```shell
$ gcc -g example.c -o example
```

And if we execute the program:

```shell
./example  451221231
Let's calculate something with your number!
The operation with your number 451221231 is 18
```

```shell
./example  1223
Let's calculate something with your number!
The operation with your number 1223 is 6
```

Everything seems to work fine but then...

```shell
root@ae4c4233b0cc:/workspace# ./example 1109468288
Let's calculate something with your number!
Segmentation fault
```

And what we would like to know why the mistery function returns 7 in this line, because with our GDB we discovered that the bug is happening there.

```c
if (n == 7) {
	return (int)*(int*)0; // <<<< BUG!!!
}
```

A simple case for this would be to execute a script inline to call the software:

```shell
for i in {1..100000}; do ./example "$i" 2&>1 /dev/null || echo "FAILED $i" ; done
Segmentation fault
FAILED 127
Segmentation fault
FAILED 191
Segmentation fault
FAILED 223
... more fails ...
```

> With these first numbers did you discover what the function does? :)

This is not very practical for some reasons, but main one is that this mechanism is only useful **if the program and the problem are based in the input**. Let's change the code and remove the input. Now, the variable is randomly generated inside:

```c
int main(int argc, char *argv[]) {
    printf("Let's calculate something with your number!\n");
    srand(time(NULL));
    int r = rand();

    printf("The operation with your number %d is %d\n", r, badFunction(r));
}
```

And now, the problem would be to execute in a loop, hoping it fails...

```shell
./example
Let's calculate something with your number!
The operation with your number 459098380 is 14
```

```shell
./example
Let's calculate something with your number!
The operation with your number 145642241 is 11
```


#### Second case: Automate it with Python!

If we use `GDB` for debugging this issue, we would probably do something like this:

```shell
$ gdb example
GNU gdb (Ubuntu 15.0.50.20240403-0ubuntu1) 15.0.50.20240403-git
Copyright (C) 2024 Free Software Foundation, Inc.
...
Reading symbols from example...
(gdb) b main
Breakpoint 1 at 0x960: file example.c, line 44.
(gdb) r
Starting program: /workspace/example
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/aarch64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main (argc=1, argv=0xfffffffff5d8) at example.c:44
44		printf("Let's calculate something with your number!\n");
(gdb) call misteryFunction(232323231)
$1 = 18
(gdb) call misteryFunction(232323231213)
$2 = 18
(gdb) call misteryFunction(23213213)
$3 = 12
(gdb)
```

Now let's do the same but... automatic:

Let's create a Python script that will be executed by `gdb`:

```python
import gdb

gdb.execute("file example")
gdb.Breakpoint("main")
gdb.execute("run")

for i in range(0,2000):
    p = gdb.execute("call misteryFunction("+str(i)+")", to_string=True)
    if (int(p.split("=")[1])==7):
        print(f"Found the number {i} to break the program!")
```

What it does is quite simple: It tries the output of a function with values from `0` to `2000` and reports them.

In order to execute it, we can call this script from `gdb`:

```shell
$ gdb -q -x testing.py
Breakpoint 1 at 0x960: file example.c, line 44.
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/aarch64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main (argc=1, argv=0xfffffffff5d8) at example.c:44
44		printf("Let's calculate something with your number!\n");
Found the number 127 to break the program!
Found the number 191 to break the program!
Found the number 223 to break the program!
Found the number 239 to break the program!
Found the number 247 to break the program!
Found the number 251 to break the program!
Found the number 253 to break the program!
Found the number 254 to break the program!
Found the number 319 to break the program!
Found the number 351 to break the program!
Found the number 367 to break the program!
Found the number 375 to break the program!
```


Some interesting things we can see:

- The first number to fail is `127`.
- The difference between the first and the second is `64`, the difference between the third and the second is `32`, and the difference between the fourth and the third is `16`...

As it seems to be very related to base 2, it doesn't take too much time to discover that our simple program does something quite straightforward: it calculates the number of bits using **Brian Kernighan's** algorithm.

```c
uint32_t getBits(uint32_t v) {
        uint32_t total = 0;
        do { total+=v&1; v>>=1; } while (v>0);
        return total;
}
```

Once the total number of bits has been calculated, our `uint32_t badFunction(uint32_t)` checks if the number of bits is equal to 7, and in that case, a NULL pointer dereference occurs.

## Conclusion

In this entry, we explored a new possibility: how `gdb` can be automated to improve our debugging sessions.

The possibilities of this debugging technique are multiple:

- Save the data to process: calculate metrics over the captured data that can be stored with Python in a `csv`; calculate statistics about the usage of the stack or heap.
- Detect **memory leaks** by tracking heap size or usage. For this, you can parse `malloc_stats()` or `malloc_info()`. The latter provides statistics in `xml` format, so it's easily parseable from Python.

```shell
(gdb) call malloc_info(0,stdout)
<malloc version="1">
<heap nr="0">
<sizes>
</sizes>
<total type="fast" count="0" size="0"/>
<total type="rest" count="1" size="131280"/>
<system type="current" size="135168"/>
<system type="max" size="135168"/>
<aspace type="total" size="135168"/>
<aspace type="mprotect" size="135168"/>
</heap>
<total type="fast" count="0" size="0"/>
<total type="rest" count="1" size="131280"/>
<total type="mmap" count="0" size="0"/>
<system type="current" size="135168"/>
<system type="max" size="135168"/>
<aspace type="total" size="135168"/>
<aspace type="mprotect" size="135168"/>
</malloc>
```

- A bug that only happens the `n` time it starts, we can skip those executions with python.
- etc

No conclusion in this article, but... there are never too many tools to fight against a hard-to-find bug :)



