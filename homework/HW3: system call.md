# Part One: System call tracing
Your first task is to modify the xv6 kernel to print out a line for each system call invocation. It is enough to print the name of the system call and the return value;
这个任务是打印系统调用的名称以及起返回值

**问题1 : 用户是怎么调用到系统调用的？**

在`usys.S`中，通过了`SYSCALL(name)`宏，定义了各种系统调用，将其定义为`.globl`。如果C语言想要调用调用`open`，则直接在代码中调用`open`，然后程序计数器`eip`就会跳转到`open:`处，将`SYS_open`的值传给`eax`寄存器(`movl $SYS_##name, %ea)`，陷入中断`T_SYSCALL(64)`。之后通过中断表，跳转到`void trap(struct trapframe *tf)`，并执行`syscall`函数。
```
#define SYSCALL(name) \
  .globl name; \
  name: \
    movl $SYS_ ## name, %eax; \
    int $T_SYSCALL; \
    ret
```

**解答**：

从问题1可知，我们只需修改`syscall`函数即可。
```
static char *syscall_name[] = {
 [SYS_fork]    "sys_fork",
[SYS_exit]    "sys_exit",
[SYS_wait]    "sys_wait",
[SYS_pipe]    "sys_pipe",
[SYS_read]    "sys_read",
[SYS_kill]    "sys_kill",
[SYS_exec]    "sys_exec",
[SYS_fstat]   "sys_fstat",
[SYS_chdir]   "sys_chdir",
[SYS_dup]     "sys_dup",
[SYS_getpid]  "sys_getpid",
[SYS_sbrk]    "sys_sbrk",
[SYS_sleep]   "sys_sleep",
[SYS_uptime]  "sys_uptime",
[SYS_open]    "sys_open",
[SYS_write]   "sys_write",
[SYS_mknod]   "sys_mknod",
[SYS_unlink]  "sys_unlink",
[SYS_link]    "sys_link",
[SYS_mkdir]   "sys_mkdir",
[SYS_close]   "sys_close",
};

void
syscall(void)
{
  int num;
  struct proc *curproc = myproc();
  num = curproc->tf->eax;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    curproc->tf->eax = syscalls[num]();
+++    cprintf("%s -> %d\n", syscall_name[num], curproc->tf->eax);
  } else {
    cprintf("%d %s: unknown sys call %d\n",
            curproc->pid, curproc->name, num);
    curproc->tf->eax = -1;
  }
}
```



# Part Two: Date system call
Your second task is to add a new system call to xv6. The main point of the exercise is for you to see some of the different pieces of the system call machinery. Your new system call will get the current UTC time and return it to the user program. You may want to use the helper function, cmostime() (defined in lapic.c), to read the real time clock. date.h contains the definition of the struct rtcdate struct, which you will provide as an argument to cmostime() as a pointer.
这个任务是实现一个系统调用`date`，以此来了解系统调用的实现机制。

**问题：如何获取的到系统调用的参数**

以`sys_open`函数为例子，通过`argstr`和`argint`来获取其第一个和第二个参数。因此对于`sys_date`我们可以使用`arg_ptr`来获取入参结构体。
```
  if(argstr(0, &path) < 0 || argint(1, &omode) < 0)
    return -1;
```

**问题：如何调试**

`HW1`中已经列出了，gdb，然后`target remote localhost:26000`  以及`symbol-file kernel·`就可以调试内核程序`kernel`了。

**解答**
1. 修改Makefile，能够编译新的应用程序
```
diff --git a/Makefile b/Makefile
index 09d790c..db48968 100644
--- a/Makefile
+++ b/Makefile
@@ -181,6 +181,7 @@ UPROGS=\
 	_usertests\
 	_wc\
 	_zombie\
+	_date\
 
 fs.img: mkfs README $(UPROGS)
 	./mkfs fs.img README $(UPROGS)
@@ -249,7 +250,7 @@ qemu-nox-gdb: fs.img xv6.img .gdbinit
 
 EXTRA=\
 	mkfs.c ulib.c user.h cat.c echo.c forktest.c grep.c kill.c\
-	ln.c ls.c mkdir.c rm.c stressfs.c usertests.c wc.c zombie.c\
+	ln.c ls.c mkdir.c rm.c stressfs.c usertests.c wc.c zombie.c date.c\
 	printf.c umalloc.c\
 	README dot-bochsrc *.pl toc.* runoff runoff1 runoff.list\
 	.gdbinit.tmpl gdbutil\
```

2. `sys_date`系统调用的具体实现
```
diff --git a/lapic.c b/lapic.c
index b22bbd7..f782352 100644
--- a/lapic.c
+++ b/lapic.c
@@ -227,3 +227,19 @@ cmostime(struct rtcdate *r)
   *r = t1;
   r->year += 2000;
 }
+
+int
+sys_date(void)
+{
+  //int n;
+  char *p;
+  if (argptr(0, &p, 4) < 0) {
+    return -1;
+  }
+
+  struct rtcdate *r = (struct rtcdate *)p;
+  cmostime(r);
+
+  return 0;
+}
+
```

3. 将系统调用加入到系统调用表中

```
diff --git a/syscall.c b/syscall.c
index ee85261..9726ef3 100644
--- a/syscall.c
+++ b/syscall.c
@@ -103,6 +103,7 @@ extern int sys_unlink(void);
 extern int sys_wait(void);
 extern int sys_write(void);
 extern int sys_uptime(void);
+extern int sys_date(void);
 
 static int (*syscalls[])(void) = {
 [SYS_fork]    sys_fork,
@@ -126,8 +127,34 @@ static int (*syscalls[])(void) = {
 [SYS_link]    sys_link,
 [SYS_mkdir]   sys_mkdir,
 [SYS_close]   sys_close,
+[SYS_date]    sys_date,
 };
```

```
diff --git a/syscall.h b/syscall.h
index bc5f356..1a620b9 100644
--- a/syscall.h
+++ b/syscall.h
@@ -20,3 +20,4 @@
 #define SYS_link   19
 #define SYS_mkdir  20
 #define SYS_close  21
+#define SYS_date   22
```
```
diff --git a/user.h b/user.h
index 4f99c52..6448b6e 100644
--- a/user.h
+++ b/user.h
@@ -23,6 +23,7 @@ int getpid(void);
 char* sbrk(int);
 int sleep(int);
 int uptime(void);
+int date(struct rtcdate* r);
 
 // ulib.c
 int stat(const char*, struct stat*);
 
 ```
 ```
diff --git a/usys.S b/usys.S
index 8bfd8a1..123982f 100644
--- a/usys.S
+++ b/usys.S
@@ -29,3 +29,4 @@ SYSCALL(getpid)
 SYSCALL(sbrk)
 SYSCALL(sleep)
 SYSCALL(uptime)
+SYSCALL(date)
\ No newline at end of file
```
