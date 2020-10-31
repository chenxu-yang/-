# signal

A signal is a notification to a process that an event has occurred. Signals are sometimes described as software interrupts, because in most cases, they interrupt the normal flow of execution of a program and their arrival is unpredictable.

A signal can be sent by Kernel or a Process.

A Kernel may send a signal to a process in the following scenarios:

1. When a hardware exception has occurred and that exception needs to be notified to the process. For eg. Attempting division by zero, or referencing the part of memory that is inaccessible.
2. Some software event occurred outside the process’s control but effects the process. For instance, input became available on a file descriptor, the terminal window got resized, process’s CPU time limit exceeded, etc.
3. User typed some terminal special characters like interrupt(Ctrl+C) or suspend character(Ctrl+Z).

## Pending Signals?

After a signal is generated due to some event, it does not directly get delivered to a process, and the signal remains in an intermediate state called to a pending state.

- Process is not scheduled to have CPU right now. In such a case, a pending signal is delivered as soon as the process is next scheduled to run.
-  to ensure that a signal does not arrive during execution of some critical section, a process can add signal to its process’s signal mask, which is a set of signals whose delivery is currently blocked. Process’s signal mask is a per process attribute. If a signal is generated when it is blocked, it remains in pending state until later it is unblocked. There are various system calls that allow a process to add and remove signals from its signal mask.

## What happens when a signal arrives?

When a signal is about to get delivered, one of the following *default* actions take place depending on the signal:

1. The signal is ignored, i.e., it is discarded by the kernel and has no effect on the process. (The process remains unaware that the event had even occurred.)
2. The process is terminated, a.k.a. abnormal process termination, as opposed to normal process termination that occurs when the program terminates using *exit().*
3. A core dump file is generated and the process is terminated.
4. The execution of the process is suspended or resumed.

Instead of accepting the default actions of a particular signal, a process can set the disposition of the signal by changing the action that occurs when the signal is delivered. 

*The signals SIGKILL and SIGSTOP cannot be caught, blocked or ignored.*

## How to send a signal?

A signal can be sent using either the kill system call, or the kill command, and specifying the desired pid of the process.

1. If pid > 0, then the signal is sent to a particular process with the specified pid.
2. If pid = 0, then the signal is sent to every process in the same process group.
3. If pid < -1, then the signal is sent to every process in the process group whose process group id is modulus of |pid| specified.
4. If pid = -1, then the signal is sent to all the processes for which the calling process has permission to send a signal, except *init* and the calling process itself. Signals sent in this fashion are called *Broadcast Signals*.

raise() sends a signal to the calling process itself.

One interesting use case of sending a signal is to check the existence of a process. If the *kill()* system call is called with signal argument as ‘0’, a.k.a. *null signal*, then no signal is sent, but it merely performs error checking to see if the process can be signaled.

# 2 From Boot To Panic

## Boot process

Under BIOS-based systems:

1. Power-on self-test (POST) and peripheral initializations
2. Jump to the boot code in the first 440 bytes of Master Boot Record (MBR)
3. MBR boot code locates and launches boot loader – ex) GRUB, Syslinux
4. Boot loader reads its configuration and possibly presents menu
5. Boot loader loads the kernel and launches it
6. Kernel unpacks the `initramfs` (initial RAM filesystem)
   - initial temporary root filesystem
   - contains device drivers needed to mount real root filesystem
7. Kernel switches to real root filesystem
8. Kernel launches the first user-level process `/sbin/init` (pid == 1)
   - `/sbin/init` is the mother of all processes
   - traditional SysV-style init reads `/etc/inittab`
   - in Arch, `/sbin/init` is a symlink to `systemd`

## User session

1. `init` calls `getty`
   - `getty` is called for each virtual terminal (normally 6 of them)
   - `getty` checks user name and password against `/etc/passwd`
2. `getty` calls `login`, which then runs the user’s shell – ex) `bash`
   - `login` sets the user’s environment variables
   - the user’s shell is specified in `/etc/passwd`

## Process control

### Displaying process hierarchy

```
ps axf     # display process tree

ps axfj    # with more info

ps axfjww  # even if lines wrap around terminal
```

### Creating processes

Simple shell program from APUE3, 1.6:

```
#include "apue.h"
#include <sys/wait.h>

int
main(void)
{
        char	buf[MAXLINE];	/* from apue.h */
        pid_t	pid;
        int		status;

        printf("%% ");	/* print prompt (printf requires %% to print %) */
        while (fgets(buf, MAXLINE, stdin) != NULL) {
                if (buf[strlen(buf) - 1] == '\n')
                        buf[strlen(buf) - 1] = 0; /* replace newline with null */

                if ((pid = fork()) < 0) {
                        err_sys("fork error");
                } else if (pid == 0) {		/* child */
                        execlp(buf, buf, (char *)0);
                        err_ret("couldn't execute: %s", buf);
                        exit(127);
                }

                /* parent */
                if ((pid = waitpid(pid, &status, 0)) < 0)
                        err_sys("waitpid error");
                printf("%% ");
        }
        exit(0);
}
```

Questions:

- What happens if the parent process terminates before its children?

  the child process will be a child process of init process![截屏2020-10-22 下午5.16.13](/Users/chenxu/Documents/GitHub/learning-OS/operation system/images/截屏2020-10-22 下午5.16.13.png)

- What happens if a child process has terminated, but the parent never calls `waitpid()`?

  ![截屏2020-10-22 下午5.18.29](/Users/chenxu/Documents/GitHub/learning-OS/operation system/images/截屏2020-10-22 下午5.18.29.png)

  os Wait to clean the process until the parent calls the `waitpid`,

  it become a zombie process 

## Signals

How do you terminate the following program?

```
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

static void sig_int(int signo)
{
    printf("stop pressing ctrl-c!\n");
}

int main()
{
    if (signal(SIGINT, &sig_int) == SIG_ERR) {
        perror("signal() failed");
        exit(1);
    }

    int i = 0;
    for (;;) {
        printf("%d\n", i++);
        sleep(1);
    }
}
```

How about this one?

```
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

static void sig_int(int signo)
{
    printf("stop pressing ctrl-c!\n");
}

static void sig_term(int signo)
{
    printf("stop trying to kill me!\n");
}

int main()
{
    if (signal(SIGINT, &sig_int) == SIG_ERR) {
        perror("signal() failed");
        exit(1);
    }

    if (signal(SIGTERM, &sig_term) == SIG_ERR) {
        perror("signal() failed");
        exit(1);
    }

    int i = 0;
    for (;;) {
        printf("%d\n", i++);
        sleep(1);
    }
}
```

## X session

Let’s take a look at the processes in your X session:

```
 145 ?        Ss     0:00 login -- jae     
 406 tty1     Ss     0:00  \_ -bash
 421 tty1     S+     0:00      \_ xinit /etc/xdg/xfce4/xinitrc -- /etc/X11/xinit/xserverrc
 422 ?        Ss     0:03          \_ /usr/bin/X -nolisten tcp :0 vt1
 425 tty1     S      0:00          \_ sh /etc/xdg/xfce4/xinitrc
 430 tty1     Sl     0:00              \_ xfce4-session
 448 tty1     S      0:00                  \_ xfwm4 --display :0.0 --sm-client-id 27c950fe4-6
 450 tty1     Sl     0:00                  \_ Thunar --sm-client-id 2ed5a75a5-2f67-42c8-9765-
 452 tty1     Sl     0:01                  \_ xfce4-panel --display :0.0 --sm-client-id 24a77
 494 tty1     S      0:00                  |   \_ /usr/lib/xfce4/panel/wrapper /usr/lib/xfce4
 498 tty1     S      0:00                  |   \_ /usr/lib/xfce4/panel/wrapper /usr/lib/xfce4
 454 tty1     Sl     0:00                  \_ xfdesktop --display :0.0 --sm-client-id 29a760d
 456 tty1     Sl     0:01                  \_ xfce4-terminal --geometry=100x36 --display :0.0
 505 tty1     S      0:00                      \_ gnome-pty-helper
 507 pts/0    Ss+    0:00                      \_ bash
 518 pts/1    Ss     0:00                      \_ bash
2330 pts/1    R+     0:00                      |   \_ ps afx
1218 pts/2    Ss+    0:00                      \_ bash
```

Explorations:

- Identify the function of each process
- Understand the network client-server architecture of X window system
- Try running a simpler X session – `twm` or even just `xterm`

## Kernel module

Here is a simple kernel module:

```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

/* This function is called when the module is loaded. */
int simple_init(void)
{
       printk(KERN_INFO "Loading Module\n");

       return 0;
}

/* This function is called when the module is removed. */
void simple_exit(void) {
        printk(KERN_INFO "Removing Module\n");
}

/* Macros for registering module entry and exit points. */
module_init( simple_init );
module_exit( simple_exit );

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Simple Module");
MODULE_AUTHOR("SGG");
```

Here is the Makefile:

```
obj-m += simple.o
all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

Can you modify the code to cause the kernel panic?

## References

- [Source code for this lecture](http://www.cs.columbia.edu/~jae/4118/L02/)

------

*Last updated: 2019–01–24*

# 3 UNIX File I/O

## File I/O system calls

### open

Example (taken from `man open` in Linux):

```
#include <fcntl.h>
...
int fd;
mode_t mode = S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH;
char *pathname = "/tmp/file";
...
fd = open(pathname, O_WRONLY | O_CREAT | O_TRUNC, mode);
...
```

Another example – creating a lock file:

```
fd = open("/var/run/myprog.pid", O_WRONLY | O_CREAT | O_EXCL, 0644);
```

### creat

Redundant: `creat(path, mode)` is equivalent to `open(path, O_WRONLY|O_CREAT|O_TRUNC, mode)`

And it should have been called `create`, says Ken Thompson

### close

```
int close(int fildes);
```

### lseek

```
off_t lseek(int fildes, off_t offset, int whence);
```

- If whence is SEEK_SET, the file offset shall be set to offset bytes.
- If whence is SEEK_CUR, the file offset shall be set to its current location plus offset.
- If whence is SEEK_END, the file offset shall be set to the size of the file plus offset.

### read

```
ssize_t read(int fildes, void *buf, size_t nbyte);
```

- returns number of bytes read, 0 if end of file, -1 on error

Note:

- Number of bytes read may be less than the requested nbyte

- `read()` may block forever on a “slow” read from pipes, FIFOs (aka named pipes), sockets, or keyboard

- For sockets, `read(socket, buf, nbyte)` is equivalent to `recv(socket, buf, nbyte, 0)` 0 means normal behavior

  ```
  recv(int socket, void *buffer, size_t length, int flags)
  ```

  - normally, recv() blocks until it has received at least 1 byte
  - returns num bytes received, 0 if connection closed, -1 if error

### write

```
ssize_t write(int fildes, const void *buf, size_t nbyte);
```

- returns number of bytes written, -1 on error

Note:

- Number of bytes written may be less than the requested nbyte – ex) filling up a disk

- `write()` may block forever on a “slow” write into pipes, FIFOs, or sockets

  two process, one write into pipe, two read from pipe, if two sleep and dont read, the buffer in pipe is filled, the one can not write anymore

- For sockets, `write(socket, buf, nbyte)` is equivalent to `send(socket, buf, nbyte, 0)`

  ```
  send(int socket, const void *buffer, size_t length, int flags)
  ```

  - normally, send() blocks until it sends all bytes requested
  - returns num bytes sent or -1 for error

- If the file was opened with `O_APPEND` flag, the file offset gets set to the end of the file prior to each write

  - setting of the offset and writing happen in an atomic operation **prevent two process write the same file and overwrite each other** 

    





# Process Control

Every process has a unique process ID, a non-negative integer

Process ID 0 is usually the scheduler process and is often known as the swapper. No program on disk corresponds to this process, which is part of the kernel and is known as a system process. Process ID 1 is usually the init process and is invoked by the kernel at the end of the bootstrap procedure, The program ﬁle for this process was /etc/init in older versions of the UNIX System and is /sbin/init in newer versions.

The init process never dies. It is a normal user process, not a system process within the kernel,



## fork

```
#include <unistd.h> 
pid_t fork(void);
Returns: 0 in child, process ID of child in parent, −1 on error
```

The child is a copy of the parent. For example, the child gets a copy of the parent’s data space, heap, and stack. **this is copy, not share**.The parent and the child do share the text segment,

## file sharing

1. Kernel data structures for open files

   ![Figure 3.7, APUE](/Users/chenxu/Documents/GitHub/learning-OS/operation system/images/fig3.7.jpg)Figure 3.7, APUE

2. Two independent processes with the same file open

   ![Figure 3.8, APUE](/Users/chenxu/Documents/GitHub/learning-OS/operation system/images/fig3.8.jpg)Figure 3.8, APUE

3. Kernel data structures after `dup(1)`

   ![Figure 3.9, APUE](/Users/chenxu/Documents/GitHub/learning-OS/operation system/images/fig3.9.jpg)Figure 3.9, APUE

4. Sharing of open files between parent and child after `fork`

   ![Figure 8.2, APUE](/Users/chenxu/Documents/GitHub/learning-OS/operation system/images/fig8.2.jpg)Figure 8.2, APUE

open and fork: AABBCCDD

fork and open:ABCD



## vfork

The vfork function creates the new process, just like fork, without copying the address space of the parent into the child, as the child won’t reference that address space; the child simply calls exec (or exit) right after the vfork. Instead, the child runs in the address space of the parent until it calls either exec or exit.



## exit

Five normal ways to terminate a process

1. return from the main function
2. Calling the exit function
3. Calling the _exit or _Exit function.
4. Executing a return from the start routine of the last thread in the process.
5. Calling the pthread_exit function from the last thread in the process.

three abnormal way:

1. Calling abort. This is a special case of the next item, as it generates the SIGABRT signal.
2. When the process receives certain signals.
3. The last thread responds to a cancellation request.



**Regardless of how a process terminates, the same code in the kernel is eventually executed. This kernel code closes all the open descriptors for the process, releases the memory that it was using, and so on.**

**the parent of the process can obtain the termination status from either the wait or the waitpid function**

Now we’re talking about returning a termination status to the parent. But what happens if the parent terminates before the child? The answer is that the **init process becomes the parent process of any process whose parent terminates.**This way, we’re guaranteed that every process has a parent.

For any of the preceding cases, we want the terminating process to be able to notify its parent how it terminated.

In any case, the parent of the process can obtain the termination status from either the **wait or the waitpid function**.

a process that has terminated, but whose parent has not yet waited for it, is called a zombie.

## wait and waitpid

When a process terminates, either normally or abnormally, the kernel notiﬁes the parent by sending the SIGCHLD signal to the parent.

a process that calls wait or waitpid can

• Block, if all of its children are still running

• Return immediately with the termination status of a child, if a child has terminated and is waiting for its termination status to be fetched

• Return immediately with an error, if it doesn’t have any child processes

```
#include <sys/wait.h> 
pid_t wait(int *statloc); 
pid_t waitpid(pid_t pid, int *statloc, int options); 
       Both return: process ID if OK, 0 (see later), or −1 on error
```

  wait()函数用于使父进程（也就是调用wait()的进程）**阻塞**，直到一个子进程结束或者该进程接收到了一个指定的信号为止。如果该父进程没有子进程或者它的子进程已经结束，则wait()函数就会立即返回。

  waitpid()的作用和wait()一样，但它并不一定要等待第一个终止的子进程（它可以指定需要等待终止的子进程），它还有若干选项，如可提供一个**非阻塞**版本的 wait()功能，也能支持作业控制。实际上，wait()函数只是 waitpid()函数的一个特例，在Linux 内部实现 wait()函数时直接调用的就是waitpid()函数。

the argument statloc is a pointer to an integer.If this argument is not a null pointer, the **termination status of the terminated process is stored in the location pointed to by the argument**. If we don’t care about the termination status, we simply pass a null pointer as this argument.



The interpretation of the pid argument for waitpid depends on its value:

pid == −1 pid > 0 pid == 0 pid < −1

Waits for any child process. In this respect, waitpid is equivalent to wait.

Waits for the child whose process ID equals pid.

Waits for any child whose process group ID equals that of the calling process. (We discuss process groups in Section 9.4.)

Waits for any child whose process group ID equals the absolute value of pid.



The waitpid function provides three features that aren’t provided by the wait function.

1. The waitpid function lets us wait for one particular process, whereas the wait function returns the status of any terminated child. We’ll return to this feature when we discuss the popen function.

2. The waitpid function provides a nonblocking version of wait. There are times when we want to fetch a child’s status, but we don’t want to block.

3. The waitpid function provides support for job control with the WUNTRACED and WCONTINUED options.

## waitid

```
#include <sys/wait.h> 
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options); 
                                    Returns: 0 if OK, −1 on error
```

![截屏2020-10-29 下午3.45.43](/Users/chenxu/Documents/GitHub/learning-OS/operation system/images/截屏2020-10-29 下午3.45.43.png)

The infop argument is a pointer to a siginfo structure. This structure contains detailed information about the signal generated that caused the state change in the child process.



## race conditions

a race condition occurs when multiple processes are trying to do something with shared data and the ﬁnal outcome depends on the order in which the processes run.

fork may cause race conditions.

use

```
TELL_WAIT()
WAIT_PARENT()
TELL_CHILD(pid)
```

to avoid race 

## exec

one use of the **fork** function is to create a new process (the child) that then causes another program to be executed by calling one of the **exec** functions.

When a process calls one of the exec functions, that process is completely replaced by the new program, and the new program starts executing at its main function.**pid** remain the same

**With fork, we can create new processes; and with the exec functions, we can initiate new programs.**

There are seven exec functions

![截屏2020-10-29 下午4.10.44](/Users/chenxu/Documents/GitHub/learning-OS/operation system/images/截屏2020-10-29 下午4.10.44.png)

fork函数是用于创建一个子进程，该子进程几乎是父进程的副本，而有时我们希望子进程去执行另外的程序，exec函数族就提供了一个在进程中启动另一个程序执行的方法。它可以根据指定的文件名或目录名找到可执行文件，并用它来取代原调用进程的数据段、代码段和堆栈段，在执行完之后，原调用进程的内容除了进程号外，其他全部被新程序的内容替换了。另外，这里的可执行文件既可以是二进制文件，也可以是Linux下任何可执行脚本文件。

