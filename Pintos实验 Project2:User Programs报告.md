@[TOC](Pintos实验 Project2:User Programs)
# 安装及环境
## 实验平台
我是在ubuntu下安装的，如果你没有该系统：

 - 两种常用方式来获得这个ubuntu环境：虚拟机 or 云服务器
 - 两种常用方式来连接这个ubuntu环境：
	- 虚拟机，如vmware,finalshell
	- vscode的ssh，力荐，太好用啦

## 虚拟机
-此处的虚拟机指的是在实验平台（ubuntu）上为了连接pintos而需要的，而不是你连接实验平台所需的，官网给出的两种连接方式：
- bochs：
- qemu： 
## 安装及编译
## 系统环境变量
- 修改GDBMACROS：即将改为你的pintos的安装位置即可
- 添加`pintos`命令到环境变量：
	- `export`
	- `/etc/profile`：用户登录时
	- `root/.bashrc`：加载终端时
	- `source`: 重新加载
- 授予文件夹下命令权力：`chmod -R 777 //root/pintos/src/utils/` 
- （选）使用的是qemu的需要修改默认虚拟机
## 验证安装成功

# 做前必读
## 官网文档（ps：看英语困难的同学可以直接看文章最后的参考）
- 1.1.3 如何测试
- 3 实验二相关部分：建议全看，特别关注 “3.4 常见问题” 和 “3.5 80 x 86 调用约定”（如何启动，栈的分布），<font color="red" size=5>还有需要跟着做的一步是 “3.1.2 Using the File System” 磁盘的创建，使用和删除要学会</font>
## 命令行的使用
- linux常用命令，如`cd`，`source`等等，网上资料非常多，不再赘述
- 使用`boch -h`和`pintos -h`了解各个参数的使用
## 如何调试代码
- [（荐）利用调试工具GDB：这是链接](https://wenku.baidu.com/view/076662d769dc5022abea0006.html#:~:text=Pintos%E8%B0%83%E8%AF%95%E5%BF%83%E5%BE%97.%20Pintos%20%E8%B0%83%E8%AF%95%E5%BF%83%E5%BE%97%20%E4%B8%80%E3%80%81%E5%A6%82%E4%BD%95%E7%94%A8%20GDB%20%E8%B0%83%E8%AF%95%E5%86%85%E6%A0%B8%EF%BC%9A%20Ctrl+Alt+F1%20%E6%89%93%E5%BC%80%E7%BB%88%E7%AB%AF%EF%BC%8Ccd,enter%20%E9%94%AE%E7%BB%A7%E7%BB%AD%EF%BC%8C%E6%AD%A4%E6%97%B6%E4%B8%BA%E8%BF%9B%E5%85%A5%20gdb%20%E8%B0%83%E8%AF%95%E6%8E%A7%E5%88%B6%E5%8F%B0%20%E8%BE%93%E5%85%A5%E5%91%BD%E4%BB%A4%20target%20remote%20)
- 除了通过gdb之外的断点法是行不通的，因为bochs输出的是执行完毕后的结果。但我们还有输出大法：需要注意一点，pintos中新写了`printf()`，其要求的参数类型是`*char`，而打印出来的就是指向的字符串。所以考虑好类型的转换，和什么时候使用c中`printf()`
# LET'S BEGIN !
## 参数传递
### 实验目的
为`process_execute()`添加参数传递支持。目前，`process_execute()`不支持向新进程传递参数。通过扩展`process_execute()`来实现此功能， 而不是简单地将程序文件名作为其参数，而是将其以空格分隔成单词。第一个词是程序名称，第二个词是第一个参数，依此类推。也就是说，`process_execute("grep foo bar")`应该运行`grep`并传递两个参数`foo`和`bar`。

在命令行中，多个空格等价于一个空格，所以这`process_execute("grep foo bar")` 相当于我们原来的例子。您可以对命令行参数的长度施加合理的限制。例如，您可以将参数限制为适合单个页面 (4 kB) 的参数。（pintos实用程序可以传递给内核的命令行参数有 128 字节的无关限制。）

您可以按您喜欢的任何方式解析参数字符串。如果你迷路了，看看`strtok_r()`，在`lib/string.h` 中原型化并在`lib/string.c`中用完整的注释实现。您可以通过查看手册页（man strtok_r 在提示符下运行）找到有关它的更多信息。

### 先看看代码吧
这是我们需要修改的代码,简单来说目前做的事情就是3步：
1. 获取空页`fn_copy`，将文件名（指针）复制过去
2. 创建了一个以文件名为名，默认优先级的线程，执行`start_process`这个功能，传递的参数是`fn_copy`
3. 检查有没有出错，如果出错了就释放当时获取的页
```c
/* --------------------------- We are going to modified ----------------------------------- */
tid_t process_execute (const char *file_name) 
{
  char *fn_copy;
  tid_t tid;

  /* Make a copy of FILE_NAME.
     Otherwise there's a race between the caller and load(). */
  fn_copy = palloc_get_page (0);
  if (fn_copy == NULL)
    return TID_ERROR;
  strlcpy (fn_copy, file_name, PGSIZE);

  /* Create a new thread to execute FILE_NAME. */
  tid = thread_create (file_name, PRI_DEFAULT, start_process, fn_copy);
  if (tid == TID_ERROR)
    palloc_free_page (fn_copy); 
  return tid;
}
```
下面是上面代码所调用的功能说明：
1. 第一个函数用于获取空页，根据不同的flags进行反馈，共有3种可选
2. 第二个函数用于创建**内核线程**来执行当前的用户线程，提到了两点需要注意的，一是`function`函数指针传递的是`thread_start()`，新线程可能还没返回就被调度啦。二是优先级并没有用，是需要我们自己实现的，但这里官网给出，可以默认“只有一个核，只调度一个进程”，所以我们就先不动这里啦。
```c
/* ---------------- all the function called in the `process_execute` -------------------- */

/* Obtains a single free page and returns its kernel virtual
   address.
   If PAL_USER is set, the page is obtained from the user pool,
   otherwise from the kernel pool.  
   If PAL_ZERO is set in FLAGS, then the page is filled with zeros.  
   If no pages are available, returns a null pointer, unless PAL_ASSERT is set in
   FLAGS, in which case the kernel panics. */
void * palloc_get_page (enum palloc_flags flags);

/* Creates a new kernel thread named NAME with the given initial
   PRIORITY, which executes FUNCTION passing AUX as the argument,
   and adds it to the ready queue.  Returns the thread identifier
   for the new thread, or TID_ERROR if creation fails.

   If thread_start() has been called, then the new thread may be
   scheduled before thread_create() returns.  It could even exit
   before thread_create() returns.  Contrariwise, the original
   thread may run for any amount of time before the new thread is
   scheduled.  Use a semaphore or some other form of
   synchronization if you need to ensure ordering.

   The code provided sets the new thread's `priority' member to
   PRIORITY, but no actual priority scheduling is implemented.
   Priority scheduling is the goal of Problem 1-3. */
tid_t thread_create (const char *name, int priority, thread_func *function, void *aux);

```
很很很重要的`start_process`：
1. 先通过`memset`初始化了一个帧结构（这里称作了中断帧）
2. 对其中的一些标志位进行设置，也就是初始化中断帧啦，然后加载可执行文件，这里下面有解释
4. 失败的话，释放页，退出该线程。成功的话，通过模拟中断返回来启动用户进程，这里提到了参数的传递是通过栈指针的方式（%esp），<font color="red" size=5>看到这里就通透了！也就是我们只需要通过上面的`load`函数，把想传的参数传到栈中，这里给栈指针就好咯！</font>而获得参数的方法也在官网提示中给了，那就是`strtok_r()`。
```c
/* A thread function that loads a user process and starts it
   running. */
static void start_process (void *file_name_)
{
  char *file_name = file_name_;
  struct intr_frame if_;
  bool success;

  /* Initialize interrupt frame and load executable. */
  memset (&if_, 0, sizeof if_);
  if_.gs = if_.fs = if_.es = if_.ds = if_.ss = SEL_UDSEG;
  if_.cs = SEL_UCSEG;
  if_.eflags = FLAG_IF | FLAG_MBS;
  success = load (file_name, &if_.eip, &if_.esp);

  /* If load failed, quit. */
  palloc_free_page (file_name);
  if (!success) 
    thread_exit ();

  /* Start the user process by simulating a return from an
     interrupt, implemented by intr_exit (in threads/intr-stubs.S).  Because intr_exit takes all of its
     arguments on the stack in the form of a `struct intr_frame',
     we just point the stack pointer (%esp) to our stack frame
     and jump to it. */
  asm volatile ("movl %0, %%esp; jmp intr_exit" : : "g" (&if_) : "memory");
  NOT_REACHED ()
}
```
最后看一下`load`函数，这里给出整段的翻译以便理解：

从文件名加载`ELF`可执行文件到当前线程。将可执行文件的入口点存储到`*EIP`中，初始堆栈指针存储到`*ESP`中。成功返回true，否则返回false。

我们可以理解这个操作是把可执行文件->中断帧，再回顾上面的`start_process`后面执行的语句直接`mov`到了中断帧，所以整个过程宏观来看就是课本上学到的“硬盘->内存“。

另外，大二还没学到汇编，科普一下：
- `ELF`：可执行可链接文件格式，[格式详解](https://zhuanlan.zhihu.com/p/73114831)
- `ESP`：扩展栈指针寄存器，用于存放函数栈顶指针（下一个压入栈的活动记录的顶部），与之对应的是EBP，即帧指针寄存器，用于存放函数栈底指针（当前活动记录的底部）
- `EIP`：指令寄存器，存放当前指令的下一条指令的地址。CPU该执行哪条指令就是通过IP来指示的。
```c
/* Loads an ELF executable from FILE_NAME into the current thread.
   Stores the executable's entry point into *EIP
   and its initial stack pointer into *ESP.
   Returns true if successful, false otherwise. */
bool load (const char *file_name, void (**eip) (void), void **esp) 
```
### 着手coding
我们首先想到的第一步应该就是分离字符串，提取线程名。我之前看的几个实现版本都需要重新再复制一遍，是因为`strlcpy`这个函数同时也会传入的字符串的值。但其实仔细读一读原代码就明白人家本来给出的`file_name`就是已经复制了一遍啦，所以在`process_excute`中，只需要添加：
```c
/* Create a new thread to execute FILE_NAME. */
// Caner: `fn_copy` is used to pass the complete name+args for ushing stack, file_name is to named the thread 
char *save_ptr;
file_name = strtok_r(file_name, " ", &save_ptr);

tid = thread_create (file_name, PRI_DEFAULT, start_process, fn_copy);
```
至于为什么要把完整的名字传进去，而不是只传save_ptr，后面就懂啦

然后我们来修改`start_process`，需要修改的几个地方:

1. 和前面一样，`load()`时需要的仅仅是文件名啦，这里其实我是有一个疑惑的，从函数声明来看传的都是指针，那么有没有后面的参数是不重要的，而且进入函数体时重新指向的操作也会改变。但当我直接传改变前的`file_name`时是加载失败的，估计一层一层深究下去，取到的还是字符串。
```c
// Caner: to get the pure file_name
char *token, *save_ptr;
file_name = strtok_r(file_name, " ", &save_ptr);
success = load(file_name, &if_.eip, &if_.esp);

/* If load failed, quit. */
```
2. 加载成功后分离参数并压栈，这里我们使用断言来保证参数的个数，别忘记最后释放页~
```c
/* If load failed, quit. */
if (!success)
{
    // Caner: Not necessary. Record the exec_status of the parent thread's success and sema up parent's semaphore
    // thread_current ()->parent->success = false;
    // sema_up (&thread_current ()->parent->sema);
    thread_exit();
}
else
{
    // Caner: get the argc and argv for push_stack, limited the num under 50
    int argc = 0;
    int argv[50];
    for (token = strtok_r(fn_copy, " ", &save_ptr); token != NULL; token = strtok_r(NULL, " ", &save_ptr))
    {
        if_.esp = if_.esp - (strlen(token) + 1);
        memcpy(if_.esp, token, strlen(token) + 1);
        argv[argc++] = (int)if_.esp;
    }
    ASSERT(argc <= 50);
   // Caner: Not necessary. To align the word.
    while ((int)if_.esp % 4 != 0)
    {
        if_.esp--;
    }
    push_argument(&if_.esp, argc, argv);
    // Caner: Not necessary. Record the exec_status of the parent thread's success and sema up parent's semaphore
    // thread_current ()->parent->success = true;
    // sema_up (&thread_current ()->parent->sema);
}
palloc_free_page(file_name);
```
3. 如何压栈？我们看一下官方文档给出的栈的结构：![pintos栈的结构](https://img-blog.csdnimg.cn/20210614082325394.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0Nhbl9fZXI=,size_16,color_FFFFFF,t_70)
所以上面`for`循环中那三行代码，对应的操作就是分割字符串，然后存储参数到`argv[argc]`，这也恰好契合了我们使用`strtok_r`分割而来的字符串最后都是`\0`。
`while`中的操作不是必须的，是为了字节对齐，可以加快速度。
那下面要做的压栈操作就是**把字符串的地址，也就是参数**放进栈中。
```c
// Caner: Push arguments's address into stack
void push_stack(void **esp, int argc, int argv[])
{
    *esp = (int)*esp;
    *esp -= 4;
    *(int *)*esp = 0;
    for (int i = argc - 1; i >= 0; i--)
    {
        *esp -= 4;
        *(int *)*esp = argv[i];
    }
    *esp -= 4;
    *(int *)*esp = (int)*esp + 4;
    *esp -= 4;
    *(int *)*esp = argc;
    *esp -= 4;
    *(int *)*esp = 0;
}
```
需要注意一点，栈是向下生长的，所以每次都需要先移动栈指针（*esp参数地址），再写进去(**esp参数)。
### 测试一下
因为这是第一次涉及测试，将简述流程：先在`usrprog`中执行`make`，然后进入`build`，执行`make tests/userprog/args-single.result`，对应的是你想单项测试的项目名称。pintos会帮你编译所有相关的文件，并输出信息。

而我们这样测试一定是会出错的，因为，当你尝试一下`args-none`，即没有参数的测试，也会发现，跑不了！！！甚至你可以回到没有改之前再测试一下（我当时没有想到它居然跑不了没有测试过）。

再看看错误原因，都是一样的：`Run didn't produce any output`，它甚至没有执行到刚才修改的`process_excute`。所以我们大胆推测，很可能是`project1`中该完成的没有完成，或者某些基础的结构，甚至是入口就出现了问题。

好吧，让我们重回官方文档，或者使用gbk调试，就会发现usr_project的入口是下面这个样子的！！！
```c
void _start (int argc, char *argv[]) 
{
  exit (main (argc, argv));
}
```
但是很奇怪，这个两个参数的`main`并没有明确的定义...这里我找了好久，始终没有看懂怎么跳转到`init.c`中的`main`里的，可能是因为我还没学汇编，如果有知道的麻烦不吝赐教。

但我找到了它执行的轨迹大概是这样的：`int main (void)`->`  run_actions (argv);`->`run_task`->`  process_wait (process_execute (task));`，参考资料中有更完整的图示，这时候我们点进去，就会发现这个`process_wait`....好吧，是需要我们自己实现的。我们先不着急，向后做一做。
```c
/* Waits for thread TID to die and returns its exit status.  If
   it was terminated by the kernel (i.e. killed due to an
   exception), returns -1.  If TID is invalid or if it was not a
   child of the calling process, or if process_wait() has already
   been successfully called for the given TID, returns -1
   immediately, without waiting.

   This function will be implemented in problem 2-2.  For now, it
   does nothing. */
int process_wait (tid_t child_tid UNUSED) 
{
  return -1;
}
```

## 系统调用
# 参考资料
