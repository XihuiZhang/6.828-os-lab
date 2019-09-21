实验三：在这个实验中你需要去实验一些基础的内核功能===》用户模式环境保护（ process && environment are interchangealbe in this lab）  
      追踪所有用户级权限环境状态  
      创造单个用户级权限环境，并将需要运行的二进制软件加载到这个环境中  
      现实操作系统 handling system call and exceptions 的功能  
prerequisite：阅读这个实验对应的source code  

```
inc/	env.h	Public definitions for user-mode environments
trap.h	Public definitions for trap handling
syscall.h	Public definitions for system calls from user environments to the kernel
lib.h	Public definitions for the user-mode support library
kern/	env.h	Kernel-private definitions for user-mode environments
env.c	Kernel code implementing user-mode environments
trap.h	Kernel-private trap handling definitions
trap.c	Trap handling code
trapentry.S	Assembly-language trap handler entry-points
syscall.h	Kernel-private definitions for system call handling
syscall.c	System call implementation code
lib/	Makefrag	Makefile fragment to build user-mode library, obj/lib/libjos.a
entry.S	Assembly-language entry-point for user environments
libmain.c	User-mode library setup code called from entry.S
syscall.c	User-mode system call stub functions
console.c	User-mode implementations of putchar and getchar, providing console I/O
exit.c	User-mode implementation of exit
panic.c	User-mode implementation of panic
user/	*	Various test programs to check kernel lab 3 code
```
