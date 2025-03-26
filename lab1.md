lab1分为两个部分，第一部分主要是需要理解操作系统的启动过程，第二部分是编写printk函数。

#### 怎样才算启动了操作系统？

启动操作系统其实就是要把内核加载到内存里，把PC寄存器中的地址设置成操作系统的第一个程序开始的位置就可以了。

#### 操作系统内核是什么？

我们都知道在计算机当中，所有的信息都是二进制的，这里的内核当然也就是二进制的。关键是我们如何设置这个二进制文件的格式，以方便我们读取信息。

这里的内核使用的就是ELF文件格式。这是我们在lab1当中需要去理解的。

#### 把内核加载到内存里应该怎么做？

对啊，这样一个过程不也需要程序去控制吗？但是操作系统都还没启动，要怎么控制呢？这个工作其实就是bootloader做的，bootloader先初始化相关硬件，然后寻找能用的外存，加载其中的内核到内存当中，然后把PC寄存器的地址切换到内核的入口。本来bootloader的设计是很复杂的，而且与硬件相关，但是因为我们使用QEMU模拟，所以不需要我们自己设计bootloader，这样，我们需要做的工作将大大简化，只需要将内核中每一节被加载到的位置设置正确就可以了。而这个工作又是由链接器完成的，我们只需要进行链接脚本的设置就可以了。

#### 内核应该加载到内存的哪里？

这就需要我们去理解MIPS的内存布局。这其实是架构设计者做好的事情，架构的设计者设计出这个布局，我们写操作系统就需要遵从这个布局，否则相关硬件没有办法完成工作。不同的架构有不同的内存布局，写操作系统的人需要根据不同的内存布局设计自己的操作系统。

综上，我们在lab1第一部分中的主要工作就是：

- 理解内核本质：ELF文件格式
- 理解内核应该被加载到哪里：MIPS内存布局
- 在链接脚本中进行设置，将内核调整到正确的位置。

而第二部分就是实战编写printk。

#### 为什么能用C语言写内核？

刚开始写操作系统的时候总有这样的困惑，为什么可以用C语言？没有操作系统怎么用C语言呢？怎么用编译器呢？这些不是应该都不能用吗？我们需要清楚现在是在“模拟”制作一个设计成熟的操作系统，而不是从零开始做出人类历史上第一个操作系统。我们的实验环境是成熟的linux操作系统，有成熟的编译工具和调试工具来帮助我们编写，编译，调试，最终生成一份ELF文件。这个过程非常方便，因为我们借助了成熟的工具，这些工具都是建立在成熟的操作系统之上的。不应该去考虑”如果没有这些工具应该如何制作出这个内核“这样的问题。如果没有这些工具，制作的难度会大大提高，但是也不是不可能，只不过需要我们手动编写，编译，调试，”手撕“一份二进制文件出来。不用去考虑先有鸡还是先有蛋的问题，努力弄清楚现有的系统是如何工作的就好了。

#### 内核启动的时候怎么使用printk？

所以下面我们讨论编写printk的问题。我们需要用C语言实现printk这个函数，这个函数会被编译进内核代码，内核代码会被加载到内存当中，内核代码在刚开始运行的时候是没有相应的内存管理的，但是有一片空间是专门给内核代码用的，这是内核代码被加载到的位置，也是内核代码刚开始运行的时候用到的空间。也就是说，内核在刚开始启动的时候就已经具备运行C语言程序的能力了（当然最终在内核里都是机器码），为什么能用C来写内核，或者为什么能用任何语言来写内核，原因就是最后编译出来都一样，都是ELF文件。

#### 为什么需要我们自己写printk，而不是直接使用标准库？

因为C语言的标准库都是动态库，而使用动态库用到了共享地址空间的功能，这个功能本身就需要操作系统去实现，在我们内核刚开始运行的时候我们还没有实现这个功能，这时候就不能使用动态库函数，就只能我们自己编写成一个静态库，一起放到内核代码里面才能用了。





### ELF文件格式

ELF全称Executable and Linkable Format。

即可执行可链接格式

ELF文件包含三种文件类型

可重定位文件，可执行文件，共享对象文件，

这三种类型的文件格式都是ELF，格式是一模一样的，只不过可重定位文件使用节头表的信息，剩下两个使用段头表的信息。

ELF文件从整体来说包含五个部分：

- ELF头
- 段头表
- 节头表
- 段
- 节

![image-20250326102108411](C:\Users\86135\AppData\Roaming\Typora\typora-user-images\image-20250326102108411.png)

三种结构体可以描述ELF头，一段，一节中的信息：

![image-20250326102201437](C:\Users\86135\AppData\Roaming\Typora\typora-user-images\image-20250326102201437.png)

![image-20250326102212299](C:\Users\86135\AppData\Roaming\Typora\typora-user-images\image-20250326102212299.png)

![image-20250326102222669](C:\Users\86135\AppData\Roaming\Typora\typora-user-images\image-20250326102222669.png)

而节头表和段头表的表项就是相应节和段的地址（虚拟）。

### MIPS内存布局

注意是虚拟地址

- kuseg,2GB，[0x00000000,0x800000000)
- kseg0,512MB,[0x80000000,0xA0000000)
- kseg1,512MB,[0xA0000000,0xC0000000)
- kseg2,1GB,[0xC0000000,0x100000000)

完整布局如下：

![image-20250326102959012](C:\Users\86135\AppData\Roaming\Typora\typora-user-images\image-20250326102959012.png)

![image-20250326103008522](C:\Users\86135\AppData\Roaming\Typora\typora-user-images\image-20250326103008522.png)

可以注意到内核代码（Kernel Text）其实应该放在0x80020000

### 通过Makefile看内核文件的构建过程

```makefile
$(mos_elf): $(modules) $(target_dir)
    $(LD) $(LDFLAGS) -o $(mos_elf) -N -T $(link_script) $(objects)
```

下面找到每个变量的定义：

#### mos_elf

指的就是我们要创建的内核ElF文件

```makefile
mos_elf                 := $(target_dir)/mos
```

这里有一个target_dir，

```makefile
target_dir              := target
```

 这里就是target，所以我们最后的==ELF内核文件会被创建到target\mos==

#### modules和target_dir

 这是创建mos的依赖

```makefile
modules                 := lib init kern
```

可以看到modules就是三个文件夹，target_dir就是target文件夹

#### LD、LDFLAGS

这些定义在include.mk中

```
LD             := $(CROSS_COMPILE)ld
LDFLAGS        += -$(ENDIAN) -G 0 -static -n -nostdlib --fatal-warnings
```

LD指的就是链接器

LDFLAGS指链接器选项

说明这条命令就是一个链接命令

-T $(link_script)表示使用\$(link_script)链接脚本

### link_script

```makefile
link_script             := kernel.lds
```

这里使用kernel.lds链接脚本，我们设置内存地址就是在这里设置

### objects

```makefile
objects                 := $(addsuffix /*.o, $(modules)) $(addsuffix /*.x, $(user_modules))
```

objects表示所有需要链接的.o文件

这里addsuffix的意思就是把第一个参数加到第二个参数的后面

这里我们实际得到的objects等价于

```makefile
objects					:=lib/*.o init/*.o kern/*.o $(user_modules)/*.x
```

 user_modules是用户态的模块，很显然lab1还没有涉及到，所以我们不管。

这里就可以看到objects包括了lib、init、kern里面所有的*.o文件。

### lib kern init里面的Makefile文件

```c++
INCLUDES    := -I../include/

targets     := elfloader.o print.o string.o

%.o: %.c
        $(CC) $(CFLAGS) $(INCLUDES) -c -o $@ $<

%.o: %.S
        $(CC) $(CFLAGS) $(INCLUDES) -c -o $@ $<

.PHONY: all clean

all: $(targets)

clean:
        rm -rf *~ *.o
```

这是kern的，其它两个大同小异，核心逻辑都是把汇编代码和C代码编译成.o文件.

### 解析kernel.lds链接脚本

```c++
/*
 * Set the architecture to mips.
 */
OUTPUT_ARCH(mips)

/*
 * Set the ENTRY point of the program to _start.
 */
ENTRY(_start)

SECTIONS {
        /* Exercise 3.10: Your code here. */

        /* fill in the correct address of the key sections: text, data, bss. */
        /* Hint: The loading address can be found in the memory layout. And the data section
         *       and bss section are right after the text section, so you can just define
         *       them after the text section.
         */
        /* Step 1: Set the loading address of the text section to the location counter ".". */
        /* Exercise 1.2: Your code here. (1/4) */
        . = 0x80020000;
        /* Step 2: Define the text section. */
        /* Exercise 1.2: Your code here. (2/4) */
        .text : { *(.text) }
        /* Step 3: Define the data section. */
        /* Exercise 1.2: Your code here. (3/4) */
        .data : { *(.data) }
        bss_start = .;
        /* Step 4: Define the bss section. */
        /* Exercise 1.2: Your code here. (4/4) */
        .bss : { *(.bss) }
        bss_end = .;
        . = 0x80400000;
        end = . ;
}
```

ENTRY（_start)表示程序的入口点, 这个\_start是在start.S里面定义的函数，但是我们链接用的是.o二进制文件，\_start应该已经没了呀，链接器怎么确定\_start在哪里呢？其实是因为start.S中已经使用EXPORT宏声明了\_start,\_start就会存到start.o的符号表里，符号表在.o文件的一个特定段中。