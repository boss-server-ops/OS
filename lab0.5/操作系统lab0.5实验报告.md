#            <div align = "center">操作系统lab0.5实验报告</div>

#### 一、实验题目及要求：

为了熟悉使用qemu和gdb进行调试工作,使用gdb调试QEMU模拟的RISCV计算机加电开始运行到执行应用程序的第一条指令（即跳转到 0x80200000）这个阶段的执行过程，说明RISC-V硬件加电后的几条指令在哪里？完成了哪些功能？要求在报告中简要写出练习过程和回答。



#### 二、实验小组成员：洪崇晏、吴魁、朱奕翔

#### 三、实验内容及结果分析：

​		使用make debug 和make gdb进入调试，输入x/10i $pc得到如下汇编代码：

```
  0x1000:  auipc   t0,0x0

  0x1004:  addi a1,t0,32

  0x1008:  csrr a0,mhartid

  0x100c:   ld  t0,24(t0)

  0x1010:  jr   t0

  0x1014:  unimp

  0x1016:  unimp

  0x1018:  unimp

  0x101a:   0x8000

  0x101c:   unimp
```

​		在qemu4.1.1/hw/riscv/vit.c文件中查看到指令如下：

```
/* reset vector */
    uint32_t reset_vec[8] = {
        0x00000297,                  /* 1:  auipc  t0, %pcrel_hi(dtb) */
        0x02028593,                  /*    addi   a1, t0, %pcrel_lo(1b) */
        0xf1402573,                  /*     csrr   a0, mhartid  */
#if defined(TARGET_RISCV32)
        0x0182a283,                  /*     lw     t0, 24(t0) */
#elif defined(TARGET_RISCV64)
        0x0182b283,                  /*     ld     t0, 24(t0) */
#endif
        0x00028067,                  /*     jr     t0 */
        0x00000000,
        memmap[VIRT_DRAM].base,      /* start: .dword memmap[VIRT_DRAM].base */
        0x00000000,
                                     /* dtb: */
};
```

​		发现上面reset-vec的汇编代码，其实就是gdb调试的汇编代码。说明成功找到了加电后的几条指令的位置。

​		至于为什么加电后PC从0x1000地址开始，是因为在cpu.c文件中有这样一段代码：

```
set_resetvec(env,DEFAULT_RSTVEC)；
```

​		表示将DEFAULT_RSTVEC赋值给resetvec，其中DEFAULT_RSTVEC在cpu.bits.h中被设置为0x1000，然后发现在cpu.c中又有如下代码：

```
env->pc=env->resetvec
```

​		在这里可以知道pc被赋值为resetvec，而resetvec已经被赋值为0x1000，所以pc的初始值就是0x1000。



​		下面说明加电后的这几条指令的含义和功能：

##### 1.**auipc t0 , 0x0**

​		auipc 是 "Add Upper Immediate to PC" 的缩写。它将立即数 0x0左移 12 位，然后加上当前PC的值，结果存储在t0寄存器中。此时t0的值为0x1000

##### 2.addi a1, t0, 32

  		表示将t0寄存器的值与立即数 32 相加，结果存储在a1寄存器中。此时a1寄存器的值为0x1020。

##### 3.**csrr a0 , mhartid**

 		 csrr是Control and Status Register Read的缩写。它读取mhartid CSR （即 CPU 的硬件线程 ID）到a0寄存器。

##### 4.**ld t0 , 24(t0)**

  		从t0寄存器指向的地址加上偏移量 24 的位置加载一个双字（64 位）到t0寄存器。而偏移量24的位置恰好储存了预设的值0x80000000

##### 5.**jr t0** 

 		 跳转到t0寄存器指定的地址。跳转到0x80000000的地址，也就是OpenSBI的启动地址。



​		复位代码到此这就完成了它的使命，将控制权交给固件，进而让OpenSBI实现内核的内存布局和入口点设置、通过sbi封装好输入输出函数，最后在0x80200000的地址上开始os.bin的运行，实现OS的启动。



#### 四、实验中重要的知识点：

1、操作系统执行之前，必然有一个bootloader ，完成把操作系统加载到内存这个工作，然后其把CPU的控制权交给操作系统。

2、CPU无法直接读取硬盘等有复杂的读写时序要求的设备，但是可以读取ROM芯片，所以bootloader需要提前预置在ROM中。

3、在计算机中，固件是一种特定的计算机软件，它为设备的特定硬件提供低级控制，也可以进一步加载其他软件。在基于 riscv 的计算机系统中，OpenSBI是固件，运行在M态。

4、地址相关代码，即指令中的访存信息在编译完成后即已成为绝对地址，那么在运行之前，需要将所需要的代码加载到指定的位置 。

5、elf 文件，包含一个文件头, 包含冗余的调试信息，指定程序每个段的内存布局，需要解析文件头才能知道各段的信息。而bin文件就只需要简单地在文件头之后解释自己应该被加载到什么起始位置。

6、链接器的作用是把输入文件(往往是 .o文件)链接成输出文件(往往是elf文件)。