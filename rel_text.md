准备记录一系列的关于elf文件的分析，今天分析第一部分，rel.text.重定位

一、代码
```
#include <stdio.h>

int main() 
{
    printf("hello, world\n");
    printf("xxooxxooxxxooo\n");
    return 0;
}
```
二、编译(分别在ubuntu12.04 和centos7.04上进行）
```
gcc -Og -S hello.c
ar -o hello.o hello.s
```
三、分析
  1、分析32 bit下
  ```
  >: readelf -r hello.o
  
   Relocation section '.rel.text' at offset 0x408 contains 4 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0000000c  00000501 R_386_32          00000000   .rodata
00000011  00000a02 R_386_PC32        00000000   puts
00000018  00000501 R_386_32          00000000   .rodata
0000001d  00000a02 R_386_PC32        00000000   puts
```
注：offset是节section（本节.text）中 符号 位置的偏移量， 节头开始数offset字节后是 符号 位置
   info 由两部分组成(一部分是TYPE，为底1个字节，8bit；另一部分是符号所在节序号，高3个字节）
   验证TYPE，第一条中的01是类型，类型为_386_32，绝对地址。
            第二条中的02是类型，类型为R_386_PC32，相对PC寻址。
   验证符号所在节的序号：
          第一个条目中序号为05，第二条条目序号为a 
          readelf —S hello.o
          There are 13 section headers, starting at offset 0x13c:
```
Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00000000 000034 000028 00  AX  0   0  4
  [ 2] .rel.text         REL             00000000 000408 000020 08     11   1  4
  [ 3] .data             PROGBITS        00000000 00005c 000000 00  WA  0   0  4
  [ 4] .bss              NOBITS          00000000 00005c 000000 00  WA  0   0  4
  [ 5] .rodata           PROGBITS        00000000 00005c 00001c 00   A  0   0  1
  [ 6] .comment          PROGBITS        00000000 000078 00002b 01  MS  0   0  1
  [ 7] .note.GNU-stack   PROGBITS        00000000 0000a3 000000 00      0   0  1
  [ 8] .eh_frame         PROGBITS        00000000 0000a4 000038 00   A  0   0  4
  [ 9] .rel.eh_frame     REL             00000000 000428 000008 08     11   8  4
  [10] .shstrtab         STRTAB          00000000 0000dc 00005f 00      0   0  1
  [11] .symtab           SYMTAB          00000000 000344 0000b0 10     12   9  4
  [12] .strtab           STRTAB          00000000 0003f4 000013 00      0   0  1
```
查表的05为.rodata节序号，
查表a为[10].shstrtab，还不知道为什么定位到这里，以前没有注意过，以后再分析puts这个。

贴出来，反汇编代码objdump -x -d hello.o
```
Disassembly of section .text:

00000000 <main>:
   0:	55                   	push   %ebp
   1:	89 e5                	mov    %esp,%ebp
   3:	83 e4 f0             	and    $0xfffffff0,%esp
   6:	83 ec 10             	sub    $0x10,%esp
   9:	c7 04 24 00 00 00 00 	movl   $0x0,(%esp)
			c: R_386_32	.rodata
  10:	e8 fc ff ff ff       	call   11 <main+0x11>
			11: R_386_PC32	puts
  15:	c7 04 24 0d 00 00 00 	movl   $0xd,(%esp)
			18: R_386_32	.rodata
  1c:	e8 fc ff ff ff       	call   1d <main+0x1d>
			1d: R_386_PC32	puts
  21:	b8 00 00 00 00       	mov    $0x0,%eax
  26:	c9                   	leave  
  27:	c3                   	ret   
```
 重点说明的是9行和15行是不同的 第一次是将   $0x0,(%esp)，第二次是 $0xd,(%esp)，这个0xd是代表了第二个字符串的位置
 ```
 readelf -x .rodata hello.o
 
 Hex dump of section '.rodata':
  0x00000000 68656c6c 6f2c2077 6f726c64 00  这里即D 78786f    hello, world.xxo
  0x00000010 6f78786f 6f787878 6f6f6f00                   oxxooxxxooo.
  
  然后看重定位的算法，csapp第7.7节
```  
  重定位符号引用
一种重定位算法的伪代码如下所示：
```
  foreach section s {
    foreach relocation entry r {
        refptr = s + r.offset;//符号所在text的地址位置位置
        if (r.type == R_386_PC32) {
            refaddr = ADDR(s) + r.offset;
            *refptr = (unsigned) (ADDR(r.symbol) + *refptr - refaddr);
        }
        if (r.type = R_386_32)
            *refptr = (unsigned)(ADDR(r.symbol) + *refptr);//符号所在的text的地址位置内存储的数值，本里中的0x0 ，0xd（第二次）
    }
}
```
注：通过这中算法确定了，符号在rodata节中的位置，从而让链接器将对应的地址写到text代码中，链接器能写text中内容，生成exec执行文件，映射到内存中就有只读属性了  
