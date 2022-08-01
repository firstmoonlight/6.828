# 参考资料
[Mit6.828 HW2 Shell](https://blog.csdn.net/bysui/article/details/51435476)

# 概述
[6.828 shell](https://pdos.csail.mit.edu/6.828/2017/homework/sh.c)
文章前面的简介中，已经向我们提供了大部分的代码，但是一些功能需要我们进行补充。因此我这边先简要梳理下[6.828 shell](https://pdos.csail.mit.edu/6.828/2017/homework/sh.c)的主要逻辑。
1. 通过`gecmd`获取命令，获取用户从界面输入的命令，最大为100个字节。
2. `parsecmd`函数解析命令
3. 系统调用`fork`创建新的进程，来执行命令


# Executing simple commands
使`shell`程序能够简单地执行`ls`等命令。
直接修改`runcmd`下的代码：
```
  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      _exit(0);
    //fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    if (-1 == execvp(ecmd->argv[0], ecmd->argv)) {
      fprintf(stderr, "execvp error : %s\n", strerror(errno));
    }
    break;
```
`execvp`系统调用会切到环境变量`PATH`下的路径，然后查找可执行文件。
![Image84.png](/homework/graphs/Image84.png)

# I/O redirection
这道题目我没有想出来，肤浅了。我想着打开一个文件，然后将cmd的执行结果write到这个文件中。
看了下其他人的答案[Mit6.828 HW2 Shell](https://blog.csdn.net/bysui/article/details/51435476)：**关闭重定向命令的文件描述符，然后使用open命令打开重定向文件，由于open命令会自动选择最小的文 件标识符0(标准输入)或者1(标准输出)，实现重定向**。这个答案很赞。

```
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    //fprintf(stderr, "redir not implemented\n");
    // Your code here ...
    close(rcmd->fd);
    if (open(rcmd->file, rcmd->flags, 0664) < 0) {
      fprintf(stderr, "open file %s error, %s\n", rcmd->file, strerror(errno));
    }
    runcmd(rcmd->cmd);
    break;
```

# Implement pipes
难点在于我对于`pipe`的系统调用不熟悉，以及如何获取当前`cmd`的执行结果。查看别人写的代码，发现很巧妙：**将标准输入输出分别映射到一个管道的两端，标准输出对应到管道的写端，标准输入对应管道的读端，如此当前`cmd`的输出会直接床给下一个`cmd`的输入。**
方法一，创建两个子进程：
```
  case '|':
    pcmd = (struct pipecmd*)cmd;
    //fprintf(stderr, "pipe not implemented\n");
    // Your code here ...
    if (pipe(p) < 0) {
      fprintf(stderr, "create pipe error, %s\n", strerror(errno));
    }

    if (fork1() == 0) {
      //close read fd
      //p[0]是读端，p[1]是写端
      
      dup2(p[1], 1);
      close(p[0]);
      //p[1]已经被stdout取代了，因此可以关闭了
      close(p[1]);
      runcmd(pcmd->left);
    } 
    if (fork1() == 0) {
      
      dup2(p[0], 0);
      close(p[1]);
      close(p[0]);
      runcmd(pcmd->right);
    }

    close(p[0]);
    close(p[1]);
    wait(&r);
    wait(&r);
    break;
  } 
```

方法二，父进程传到子进程：
```
case '|':
    pcmd = (struct pipecmd*)cmd;
    //fprintf(stderr, "pipe not implemented\n");
    // Your code here ...
    if (pipe(p) < 0) {
      fprintf(stderr, "create pipe error, %s\n", strerror(errno));
    }
    int pid = fork1();
    if (pid > 0) {
      //close read fd
      //p[0]是读端，p[1]是写端
      close(p[0]);
      dup2(p[1], 1);
      //p[1]已经被stdout取代了，因此可以关闭了
      close(p[1]);
      runcmd(pcmd->left);
    } else if (pid == 0) {
      close(p[1]);
      dup2(p[0], 0);
      close(p[0]);
      runcmd(pcmd->right);
    }
    wait(&r);
    break;
  }    


```
