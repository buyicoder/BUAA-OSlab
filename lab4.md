前面lab3说中断是一种异常，现在系统调用也是一种异常。

其实异常可以说是一种陷入内核的手段。

先回忆一下异常的处理方式，是

主要是要跳转到异常分发代码处（当然前面还需要设置EPC，EXL和Cause，分别是记录返回位置，陷入内核禁止中断，记录异常原因）

异常分发代码根据异常类型跳转到不同的处理程序

8号异常就是系统调用的处理程序

- kern

  - entry.S

    - exc_gen_entry：异常分发代码的入口
      - exception_handlers，异常向量组，定义在trap.c中

  - traps.c（trap是陷阱的意思）

    - exception_handlers,异常向量组，存着所有异常类型，在entry.S中用到，这些全部都在genex.S中定义
      - handle_reserved,应该表示什么也不干，直接恢复？
      - handle_int，处理时钟中断
      - handle_tlb，处理tlb load异常
      - handle_mod，处理tlb store异常
      - handle_sys，处理系统调用
    - do_reserved(struct Trapframe *tf)genex.S中handle_reserved要用

  - genex.S

    - handle_int定义
      - schedule最后跳转到调度函数，定义在sched.c中
    - handle_reserved定义
      - do_reserved
    - handle_sys定义
      - do_syscall，定义在syscall_all.c里
    - handle_mod定义
      - do_tlb_mod，定义在syscall_all.c里

  - syscall_all.c

    - sys_putchar(int c)向屏幕打印输出单个字符

      - printcharc()

    - sys_print_cons（const void *s, u_int num)输出指定长度的字节串

    - sys_getenvid（void)获取当前进程的env_id

    - sys_yield

    - sys_env_destroy(u_int envid)销毁指定进程

    - sys_set_tlb_mod_entry

    - is_illegal_va

    - is_illegal_va_range

    - sys_mem_alloc(u_int envid, u_int va, u_int perm)（分配物理页并映射到指定虚拟地址）

      - is_illegal_va(va)检查用户空间地址合法性
      - envid2env检查操作权限
      - page_alloc
      - page_insert

    - sys_mem_map

    - sys_mem_unmap

    - sys_exofork（void)创建子进程

      - env_alloc分配新的Env结构
      - 复制父进程的Trapframe
      - 设置子进程返回值寄存器
      - 标记位不可立即运行

    - sys_set_env_status

    - sys_set_trapframe

    - sys_panic

    - sys_ipc_recv(u_int dstva)阻塞等待接收消息

    - sys_ipc_try_send向指定进程发送消息

    - sys_cgetc

    - sys_write_dev设备寄存器写

    - sys_read_dev设备寄存器读

    - void *syscall_table[MAX_SYSNO]

    - do_syscall(struct Trapframe *tf)系统调用分发程序

      - 找到tf->refs[4]，是系统调用号
      - 找到syscall_table里对应的系统调用函数，
      - 组织好函数的参数，调用函数然后把返回结果放到tf->regs[2]里面，这一句是真正调用系统调用，系统调用的所有定义也就在syscall_all.c里面，系统调用号的定义在别的地方
      
      

### 进程间通信机制（IPC）

```
struct Env {
	 // lab 4 IPC
     u_int env_ipc_value;//进程传递的具体数值
     u_int env_ipc_from;//发送方进程的ID
     u_int env_ipc_recving;//1：等待接受数据中，0：不可接受数据
     u_int env_ipc_dstva;接收到的页面需要与自身的哪个虚拟页面完成映射
     u_int env_ipc_perm;传递的页面的权限位设置
 };
```

sys_ipc_recv(u_int dstva)接受消息，放到dstva这一页里面，接受的值放到env_ipc_value这里面

- 将自身env_ipc_recving设置为1，表明盖进程准备接受发送方的消息
- 之后给env_ipc_dstva赋值，表明自己将接收到的页面与dstva完成映射
- 阻塞当前进程
- 放弃CPU

sys_ipc_try_send（u_int envid, u_int value, u_int srcva, u_int perm)发送消息给envid这个进程，发送srcva这一页，发送value这个值，

- 根据envid找到相应进程，如果指定进程为可接受状态，发送成功
- 否则，函数返回-E_IPC_NOT_RECV,表示目标进程未处于接受状态
- 清除接受进程的接收状态，将相应数据填入进程控制块，传递物理页面的映射关系
- 修改进程控制块中的进程状态

使用srcva为0的调用来表示值传value值

### Fork

- user/lib/fork.c

  - int fork(void)

    - cow_entry处理写实复制的函数
    - env_user_tlb_mod_entry
    - syscall_set_tlb_mod_entry(0,cow_entry)
    - syscall_exofork()定义在kern/syscall_all.c里面
      - 分配env结构体
      - 设置env->tf为父进程上下文
      - 设置新进程返回值env_tf.regs[2]=0,表明是子进程
      - 设置env_status为ENV_NOT_RUNNABLE
    - ENVX(syscall+getenvid())
    - UXSTACKTOP
    - vpd[i]
    - duppage

    - syscall_set_tlb_mod_entry

    - syscall_set_env_status

  - duppage(u_int envid, u_int vpn)复制地址空间

    - syscall_mem_map
    - syscall_mem_unmap

  - static void \__attribute__((noreturn)) cow_entry(struct Trapframe *tf)实现写时复制

    - syscall_mem_alloc()
    - memcpy
    - syscall_set_trapframe(0,tf)

