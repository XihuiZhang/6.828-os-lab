# 6.828-os-lab

## lab1 系统内核如何启动
### Abstraction of booting

>1 :当你按下电脑的开机键，cpu被加电的瞬间会强行赋值 'CS = 0XF000' 'IP = 0XFFF0'.早期的内存是1M，2^(20
而地址最多是16位的，所以早期的物理地址计算等于 phy = cs<<4+ip (real mode),BIOS的程序入口就是从0xffff0
也就是启动的第一条指令。

>2 : Bios的作用就是激活显卡 检查内存 设置中断向量等等 （会有某部分代码开启 protection 模式来检查内存）
初始化结束之后，Bios就将bootloader从一个合适的位置装载到内存0x7c00,BIOS将所检查磁盘的第一个扇区(512B)载入内存，放在 0x0000:0x7c00 处， 如果该扇区的最后两个字节是“55 AA”，那么这就是一个引导扇区，这个磁盘也就是一块可引导盘。通常这个大小为 512B 的程序就称为引导程序(bootloader)。如果最后两个字节不是“55 aa”，检查下一个重复 知道找到 55 aa。
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。

>3 : bootloader的作用就是加载系统内核到0x7c00，控制权转交给系统内核

    +------------------+  <- 0xFFFFFFFF (4GB)
    |      32-bit      |
    |  memory mapped   |
    |     devices      |
    |                  |
    /\/\/\/\/\/\/\/\/\/\

    /\/\/\/\/\/\/\/\/\/\
    |                  |
    |      Unused      |
    |                  |
    +------------------+  <- depends on amount of RAM
    |                  |
    |                  |
    | Extended Memory  |
    |                  |
    |                  |
    +------------------+  <- 0x00100000 (1MB)
    |     BIOS ROM     |
    +------------------+  <- 0x000F0000 (960KB)
    |  16-bit devices, |
    |  expansion ROMs  |
    +------------------+  <- 0x000C0000 (768KB)
    |   VGA Display    |
    +------------------+  <- 0x000A0000 (640KB)
    |                  |
    |    Low Memory    |
    |                  |
    +------------------+  <- 0x00000000
 0-0X03FF是用来保存各种中断向量的存储位置，操作系统中一般都是要求多的连续内存
 所以Bootloader就被放到了内存地址的尾部，bootloader所在的sector是512kb，BootLoader的数据和stack
 也是512kb，0x007fff-0x0003fff = 0x7c000，boot.S会初始化寄存器设置code segement and data segement
 
    mov  $0x8f, %eax
    out  %al, $0x70
    in  $0x71, %al
    in %al, PortAddress    向端口地址为PortAddress的端口写入值，值为al寄存器中
    out PortAddres,%al    把端口地址为PortAddress的端口中的值读入寄存器al中
    http://bochs.sourceforge.net/techspec/PORTS.LST 
 0x71，0x70端口是控制系统中cmos的端口，0x70叫做index register，用于计算机关闭时存储信息（时钟设备 real time clock）它还可以控制
 NMI(Non-Maskable Interrupt) 这三条指令是用来关闭NMI中断的
    
    in  $0x92, %al
    or  $0x2, %al
    out  %al, $0x92
 这三步instcution 是0x92的1号bit 赋值为1，而portlist中 0x92端口的bit 1 是enable A20 （准备开始进入protect mode）

    mov  %cr0, %eax
    or  $0x1, %eax
    mov  %eax, %cr0
 计算机中包括cr0 cr1 cr2 cr3 4个控制寄存器，cr0是用来开启protect mode（cr0 的0bit 是PE位），这三步只是推测机器能否在保护模式下工作
 kernel/entry.S 设置好cr3寄存器的值为页目录的物理地址
#### ELF文件结构
                        +------------------+ 
                        |    ELF 文件头     |
                        +------------------+  
                        |    程序头表       |
                        +------------------+  
                        |    .text 节      |
                        +------------------+                  
                        |    .rodata 节    |
                        +------------------+   
                        |    .stab 节      |
                        +------------------+        
                        |    .stabstr 节   |
                        +------------------+                                 
                        |    .data 节      |
                        +------------------+  
                        |    .bss 节       |
                        +------------------+  
                        |    .comment 节   |
                        +------------------+      
                        |    节头表         |
                        +------------------+                     
    struct Elf {
        uint32_t e_magic; // 标识是否是ELF文件
        uint8_t e_elf[12]; // 魔数和相关信息 
        uint16_t e_type; // 文件类型
        uint16_t e_machine; 
        uint16_t e_version; // 版本信息
        uint32_t e_entry; // 程序入口点
        uint32_t e_phoff; // 程序头表偏移值
        uint32_t e_shoff; // 节头表偏移值
        uint32_t e_flags; 
        uint16_t e_ehsize;  // 文件头长度
        uint16_t e_phentsize; // 程序头部长度 
        uint16_t e_phnum; // 程序头部个数 
        uint16_t e_shentsize; // 节头部长度 
        uint16_t e_shnum; // 节头部个数 
        uint16_t e_shstrndx; // 节头部字符索引
    };
     struct Proghdr { 
        uint32_t p_type; // 段类型
        uint32_t p_align; // 段在内存中的对齐标志
        uint32_t p_offset; // 段位置相对于文件开始处的偏移量
        uint32_t p_va; // 段的虚拟地址
        uint32_t p_pa; // 段的物理地址
        uint32_t p_filesz; // 段在文件中长度
        uint32_t p_memsz; // 段在内存中的长度 
        uint32_t p_flags; // 段标志
    }
#### link address and load address
>在操作系统中 linkadress是保存程序变量地址的地址，编译时程序通过访问这些地址来寻找变量的值，加载地址是程序在物理内存中存放的位置0
>>real mode: link adress == load address （物理地址的直接寻址）
>>protect mode: loadaddrss == link address + kernelbase 

#### load kernel
>bootmain() 函数负责加载内核，采用的是分段加载方法。使用命令 readelf -l obj/kern/kernel 可以看到内核分段信息，kernel需要加载的有2段(LOAD标识)，以 4K (0x1000) 对齐。我们可以看到加载kernel时，是以扇区为单位读取并加载的。首先会将文件的前 4K 数据即ELF头和程序头表（注意是从扇区1开始读取，因为扇区0是bootloader）加载到物理地址 0x10000 处（注意这个不是kernel加载的地址 0x00100000，少个0），这样ELF文件头和程序头表都能读取了，接着就可以根据程序头表加载程序段到对应物理地址了。

    void
    bootmain(void)
    {
        struct Proghdr *ph, *eph;

        // 读取前4K数据，包括ELF头部和程序头表到物理地址0处
        readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

        // 判断是不是ELF文件
        if (ELFHDR->e_magic != ELF_MAGIC)
            goto bad;

        // 加载程序段
        ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
        eph = ph + ELFHDR->e_phnum;
        for (; ph < eph; ph++)
            // p_pa 是加载地址，也就是物理地址，p_memsz是段在内存中长度，p_offset段在文件中的偏移。
            readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

        // 转到kernel入口点执行，并且不再返回。
        ((void (*)(void)) (ELFHDR->e_entry))();
    }
bootmain() 加载kernel完成后，跳转到kernel的入口 e_entry（物理地址0x10000c处）执行。需要牢记的是，在代码中的地址我们称之为虚拟地址或链接地址，在开启了保护模式后(设置cr0的低位为1)，会经过全局描述符表项来生成线性地址，如果开启了分页(设置cr3的低位为1)，则会将线性地址经过分页转换为物理地址，否则，线性地址就等于物理地址。
因此，bootmain() 函数里面最后跳转的是 e_entry 的物理地址值 0x0010000c，而不是链接地址值 0xf010000c。因为 kernel 代码本身就是加载在 0x0010000c，而我们的描述符表项里面存的代码段基地址是0，所以只有在 0x0010000c 处才能找到可执行指令，此时在 0xf010000c处我们还没有做好映射。
kernel/entry.S 设置好cr3寄存器的值为页目录的物理地址，然后设置cr0开启分页，最后跳转到 relocated 执行。relocated 是类似0xf01xxxxx之类的连接地址，为什么可以执行了呢？这是因为我们已经开启分页，而且在 entrypgdir.c 中已经设置好高位地址的页目录项。在 relocated 中，先初始化 栈帧指针寄存器 ebp，用于追溯函数调用流程，然后设置 堆栈寄存器 esp 的值为内核栈栈顶，堆栈大小为32KB

