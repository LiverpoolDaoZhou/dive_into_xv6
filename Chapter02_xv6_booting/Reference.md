
### 问题1：为什么eph = 0x10094, ph = 0x10034，区间大小为6 * 16 bytes
对于如下结构体：
```c
typedef struct {
    char c;
    int i;
    double d;
} MyStruct;
```
编译器会对结构体进行内存对齐，这里char为1byte，int占用4bytes，double占用8bytes，所以结构体最宽基本类型成员是double(8 bytes)，因此结构体的大小必须是8的倍数。该结构体大小为16bytes。因此在执行结构体加法计算时是加上整个结构体大小。如果遇到想正常加减法需要采用：
```c

ph = (struct proghdr*)((uchar*)elf + elf->phoff);
eph = ph + elf->phnum;

/**
################################ elf file info ##################################
(gdb) p elf
$3 = (struct elfhdr *) 0x10000

program header table file offset
(gdb) p elf->phoff
$4 = 52

program headers' entity size
(gdb) p elf->phentsize
$6 = 32

program headers' entity counts
(gdb) p elf->phnum
$7 = 3

elf all info:
(gdb) p *elf
$6 = {magic = 1179403647, elf = "\001\001\001\000\000\000\000\000\000\000\000", type = 2, machine = 3, version = 1, entry = 1048588, phoff = 52, shoff = 212440, flags = 0, ehsize = 52, 
  phentsize = 32, phnum = 3, shentsize = 40, shnum = 18, shstrndx = 17}

################################ program header file info ##################################
(gdb) p ph
$1 = (struct proghdr *) 0x10034
(gdb) p eph
$2 = (struct proghdr *) 0x10094

参考下方.text section
(gdb) p *ph
$2 = {type = 1, off = 4096, vaddr = 2148532224, paddr = 1048576, filesz = 31077, memsz = 31077, flags = 7, align = 4096}

参考下方.data section
(gdb) p *ph
$4 = {type = 1, off = 36864, vaddr = 2148564992, paddr = 1081344, filesz = 9494, memsz = 49256, flags = 6, align = 4096}

*/
```
注意这里program header单项大小为32bytes 当表项数目为3时，其地址区间在96bytes = 16 * 6

### 问题2：如何查询Executable File的各类内存加载信息：
- 查看ELF File Header & Program Header & Section Header：
```
darrenzhou@darrenzhou-virtual-machine:~/Code/xv6-public$ readelf -h kernel
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x10000c
  Start of program headers:          52 (bytes into file)
  Start of section headers:          212440 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)

  # 32bytes each program header
  Size of program headers:           32 (bytes)
  Number of program headers:         3
  Size of section headers:           40 (bytes)                        
  Number of section headers:         18
  Section header string table index: 17


darrenzhou@darrenzhou-virtual-machine:~/Code/xv6-public$ readelf -l kernel

Elf file type is EXEC (Executable file)
Entry point 0x10000c
There are 3 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x001000 0x80100000 0x00100000 0x07965 0x07965 RWE 0x1000
  LOAD           0x009000 0x80108000 0x00108000 0x02516 0x0c068 RW  0x1000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RWE 0x10

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata .stab .stabstr 
   01     .data .bss 
   02 
```

上述是将.text, .rodata, .stab, .stabstr这些section从硬盘中0x001000开始的0x07965 bytes 装入到VirtAddr: 0x80100000大小为0x07965 bytes的内存中 并且要求和0x1000对齐 设置属性为RWE

### 问题3：如何进入entry.S中的entry地址
地址相关的元素会记录在符号表中，例如函数名，变量名和行标号；ELF可执行文件对这些变量和函数的地址按照符号来管理，协助链接时符号解析和重定位。
```bash
# 注意这里可执行文件和对象文件函数名地址不同
$ nm kernel | grep _start
0010000c T _start

$ nm -s entry.o
0000000c T entry
         U entrypgdir
         U main
00000000 T multiboot_header
00001000 C stack
8000000c T _start

$ readelf -s entry.o
    10: 8000000c     0 NOTYPE  GLOBAL DEFAULT    1 _start
    11: 0000000c     0 NOTYPE  GLOBAL DEFAULT    1 entry
```

### 问题4：分别绘制出stack中的值
```bash
(gdb) br * 0x7c00
Breakpoint 1 at 0x7c00

(gdb) info reg
esp            0x6f00              0x6f00

(gdb) x/xw 0x6f00
0x6f00: 0xf000d002

(gdb) si
=> 0x7c43:      mov    $0x7c00,%esp

# stack pointer初始化为0x7c00
(gdb) x/xw $esp
0x7c00: 0x8ec031fa

(gdb) si
=> 0x7c48:      call   0x7d49

(gdb) si
=> 0x7d49:      endbr32 

# 这里为bootmain的返回地址0x7bfc
(gdb) x/xw $esp
0x7bfc: 0x00007c4d

(gdb) x/xw $ebp
0x0:    0xf000ff53

# 观察到这里将ebp压入栈内，目的为了保存ebp，以便在函数执行期间可以使用其他寄存器进行临时计算，而不丢失对栈帧的引用
# 注意这里ebp压入栈中 esp的值执行新的栈顶（向下发展）
=> 0x7d4d:      push   %ebp
0x00007d4d in ?? ()
1: x/xw $eip  0x7d4d:   0x57e58955
2: x/xw $esp  0x7bfc:   0x00007c4d
3: x/xw $ebp  0x0:      0xf000ff53
(gdb) si
=> 0x7d4e:      mov    %esp,%ebp
0x00007d4e in ?? ()
1: x/xw $eip  0x7d4e:   0x5657e589
2: x/xw $esp  0x7bf8:   0x00000000
3: x/xw $ebp  0x0:      0xf000ff53
```
上述这些指令按照bootmain.asm的指令顺序执行

