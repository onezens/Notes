---
title: ARM汇编
date: 2017.08.07 11:08
tags: iOS逆向和安全
---

### 一. ARM 寄存器

>ARM共有37个32位寄存器,其中31个为通用寄存器,6个为状态寄存器.这些寄存器不能被同时访问,但在任何时候,通用寄存器R0~R14,程序计数器PC,一个或两个状态寄存器都是可访问的.


![ARM寄存器](http://upload-images.jianshu.io/upload_images/1216462-545cdcb90b1f22a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* arm64有32个64bit长度的通用寄存器x0～x30，sp，可以只使用其中的32bit w0～w30。 arm32只有16个32bit的通用寄存器r0~r12, lr, pc, sp. arm64有32个128bit SIMD寄存器v0～v31，arm32有16个128bit SIMD寄存器Q0～Q15，又可细分为32个64bit SIMD寄存器D0～D31 

* 函数调用：arm64前面8个参数都是通过寄存器来传递x0～x7， arm32前面4个参数通过寄存器来传递r0～r3，其他通过栈传递

* iOS中ARMv7以及ARM64下的ABI：
  * ARMv7中，对于通用寄存器，自己写的过程中需要保护R4、R5、R6、R7、R8、R9、R10、R11以及R14寄存器；NEON寄存器需要保存Q4、Q5、Q6、Q7寄存器。

   * ARM64模式下，通用寄存器X18、X30不能被使用。而需要被自己写的过程所保护的是：X19、X20、X21、X22、X23、X24、X25、X26、X27、X28、X29寄存器；而SIMD寄存器需要保护的是V8、V9、V10、V11、V12、V13、V14、V15。

### 二. ARM汇编指令简介

1. 常用寄存器
    
    |寄存器 | 位数|描述|
    |--------- | -------------|-------------|
    |X0-X30 | 64bit |64位通用寄存器|
    |WO-W30 | 32bit |32位通用寄存器|
    |FP(x29) | 64bit |栈底指针|
    |LR (X30) | 64bit |程序链接寄存器，保存跳转返回信息地址|
    |SP | 64bit |保存栈指针|
    |PC | 64bit |程序计数器，又称PC指针，指向下一条指令|

2. 通用寄存器名称

    | 位数  | 通用 | 0寄存器 | 栈指针 |
    |------|------|------|------|
    | 32bit | Wn | WZR| WSP|
    | 64bit | Xn | XZR | SP |

3. 通用寄存器用法

    ```
    X0-X7   用于函数调用时参数传递，X0一般用作返回值
    X8      间接寻址结果
    X29     通常用作FP
    X30     通常用作LR
    ```
4. 常用的汇编指令

    ```
    MOV X1, X0  ;寄存器X0的值传给X1
    ADD X0, X1, X2  ;寄存器X1和X2的值相加后给X0
    SUB X0, X1, X2  ;寄存器X1和X2的值相减后给X0
    
    AND X0, X0, #0xF    ;X0和0xF相与后的值给X0
    ORR X0, X0, #0x10   ;X0和0x10相或后的值给X0
    EOR X0, X0, #0x11   ;X0和0x11相异或后的值给X0
    
    LDR(LDUR) X5, [X6, #0x8]  ;X6寄存器的值(地址)加0x8的地址内的值给X5
    STR(STUR) X0, [SP, #0x8]  ;X0的值给(SP+0x8)地址指向的空间
    
    STP X29, X30, [SP, #0x1]    ;入栈操作
    LDP X29, X30, [SP, #0x1]    ;出栈操作
    
    CBZ     比较，如果结果为0，就跳转到后面的指令
    CBNZ    比较，如果结果非0，就跳转到后面的指令
    
    CMP     比较指令，结果影响CSPR状态
    
    B/BL    绝对跳转,无返回值/绝对跳转,返回值地址保存到LR(X30)
    RET     子程序返回，返回地址保存到LR(X30)
    
    ADRP    用来定位数据段中的数据, 因为ASLR会导致代码及数据的地址随机化, 用ADRP来根据PC做辅助定位
    
    ```
    
### 三. 反汇编

MacOS终端编译文件命令：将1.m 编译成 arm1

```
clang -O0 -arch arm64 -isysroot `xcrun --sdk iphoneos --show-sdk-path` -o arm1 1.m   
```

1. 赋值、加法和减法运算

    **C语言：**

    ```
    #include <stdio.h>
    int main(int argc, char* argv[])
    {
        int a = 0x1234;
        int b = a + 0x1122;
        int c = a + 0x2233;
        return 0;
    }
    ```

    **Hopper查看反汇编代码：**

    ```
                        _main:
    0000000100007f70    sub     sp, sp, #0x20 ;分配0x20个的内存空间
    0000000100007f74    movz    w8, #0x0      ;w8=0x0
    0000000100007f78    movz    w9, #0x2233   ;w9=0x2233
    0000000100007f7c    movz    w10, #0x1122  ;w10=0x1122
    0000000100007f80    movz    w11, #0x1234  ;w11=0x1234
    0000000100007f84    str     wzr, [sp, #0x1c]  ;(sp+0x1c)=wzr
    0000000100007f88    str     w0, [sp, #0x18]  ;(sp+0x18)=w0
    0000000100007f8c    str     x1, [sp, #0x10]  ;(sp+0x10)=x1
    0000000100007f90    str     w11, [sp, #0xc]  ;(sp+0xc)=w11
    0000000100007f94    ldr     w11, [sp, #0xc]  ;w11 = (sp+oxc)
    0000000100007f98    add     w10, w11, w10    ;w10 = w11+w10
    0000000100007f9c    str     w10, [sp, #0x8]  ;(sp+0x8)=w10
    0000000100007fa0    ldr     w10, [sp, #0xc]  ;w10=(sp+0xc)
    0000000100007fa4    add     w9, w10, w9      ;w9 = w10+w9
    0000000100007fa8    str     w9, [sp, #0x4]   ;(sp+0x4) = w9
    0000000100007fac    mov     x0, x8           ;x0 = x8
    0000000100007fb0    add     sp, sp, #0x20  ;释放分配0x20个空间
    0000000100007fb4    ret        
                        ; endp    
    
    ```
    

2. 与、或和异或运算

    **C语言：**

    ```
    #include <stdio.h>
    int main(int argc, char* argv[])
    {
        int a = 0x3;
        int b = a & 0xf;
        int c = a | 0xe;
        int d = a ^ 0xd;
        int e = ~a;
        return 0;
    }
    
    ```

    **反汇编代码：**

    ```
                        _main:
    0000000100007f58    sub   sp, sp, #0x30 
    0000000100007f5c    movz  w8, #0x0      ;w8=0
    0000000100007f60    movn  w9, #0x0      ;w9=0;
    0000000100007f64    movz  w10, #0xd     ;w10=0xd;
    0000000100007f68    orr   w11, wzr, #0x3    ;w11 = wzr | 0x3 = 0x3;
    0000000100007f6c    str   wzr, [sp, #0x2c]  ;(sp+0x2c) = wzr
    0000000100007f70    str   w0, [sp, #0x28]   ;(sp+0x28) = w0
    0000000100007f74    str   x1, [sp, #0x20]   ;(sp+0x20) = x1
    0000000100007f78    str   w11, [sp, #0x1c]  ;(sp+0x1c) = w11
    0000000100007f7c    ldr   w11, [sp, #0x1c]  ;w11 = (sp+0x1c)
    0000000100007f80    and   w11, w11, #0xf    ;w11 = w11 & 0xf
    0000000100007f84    str   w11, [sp, #0x18]  ;(sp+0x18) = w11
    0000000100007f88    ldr   w11, [sp, #0x1c]  ;w11 = (sp+0x1c) = 0x3
    0000000100007f8c    orr   w11, w11, #0xe    ;w11 = w11 | 0x3 = 0x3 | 0xe
    0000000100007f90    str   w11, [sp, #0x14]  ;(sp+0x14) = w11
    0000000100007f94    ldr   w11, [sp, #0x1c]  ;w11 = (sp+0x1c) = 0x3
    0000000100007f98    eor   w10, w11, w10     ;w10 = w11 ^ w10 = 0xd ^ 0x3
    0000000100007f9c    str   w10, [sp, #0x10]  ;(sp+0x10) = w10
    0000000100007fa0    ldr   w10, [sp, #0x1c]  ;w10 = (sp+0x1c) = 0x3
    0000000100007fa4    eor   w9, w10, w9       ;w9 = w10 ^ w9 = 0x3 ^ 0 = ~a
    0000000100007fa8    str   w9, [sp, #0xc]    ;(sp+0xc) = w9
    0000000100007fac    mov   x0, x8            ;x0 = x8 = 0
    0000000100007fb0    add   sp, sp, #0x30
    0000000100007fb4    ret 
                        ; endp
    
    ```


3. 比较

    **CSPR条件码标记：**
    N、Z、C、V均为条件码标志位。它们的内容可被算术或逻辑运算的结果所改变，并且可以决定某条指令是否被执行。
    
    | 标志位 | 描述 |
    |------|------|
    | N | 当两个有符号整数运算时： <br>N=1表示运算的结果为负数；<br> N=0表示运算的结果为正数或零 |
    | Z | Z=1表示运算的结果为零<br> Z=0表示运算的结果非零<br> 对于CMP指令，Z=1表示进行比较的两个数相等 |
    | C | 可以有4种方法设置C的值： <br> 1. 在加法指令中(包括CMP指令)，当结果产生了进位,则C=1,表示无符号运算发生上溢出；其他情况C=0 <br>2. 在减法指令中(包括CMP指令)，当运算中发生借位，则C=0，表示无符号运算数发生下溢出；其他情况下C=1  <br>3. 对于包含移位操作的非加减运算指令，C中包含最后一次溢出的位的数值 <br>4.对于其他非加减运算指令，C位的值通常不受影响 |
    | V | 对于加减运算指令，当操作数和运算结果为二进制的补码表示的带符号数时，V=1表示符号位溢出；通常其他指令不影响V位 |
    
    **常见的指令的条件码**

    | 条件码助记符  | 标志 | 描述 |
    | ------ |------|------|
    | NE	| Z清零	 | 不相等 |
    | EQ	| Z置位	 | 相等|
    | LE	| Z清零或（N不等于V） | 	带符号数小于或等于 |
    | GE	| N等于V	 | 带符号数大于或等于 |
    | LT	| N不等于V	 | 带符号数小于|
    | GT	| Z清零且（N等于V）	 | 带符号数大于|
    | HI	| C置位Z清零	 | 无符号数大于|
    | LS	| Z置位C清零	 | 无符号数小于或等于|
    | CC(LO)	| C清零	 | 无符号数小于 |
    | CS(HS)	| C置位	 | 无符号数大于或等于 |

    **C语言：**

    ```
    #include <stdio.h>
    
    int main(int argc, char* argv[])
    {
       int a = 3;
       unsigned int b;
       if(a>3) b = 1;
       if(a<3) b = 2;
       if(a==3) b = 0;
       unsigned int c = 4;
       if(b>c) a  = 1;
       else a = 2;
       return 0;
    }
    
    ```

    **反汇编代码：**

    ```
                        _main:
    0000000100007f2c    sub     sp, sp, #0x20 
    0000000100007f30    orr     w8, wzr, #0x3
    0000000100007f34    str     wzr, [sp, #0x1c]    ;(sp + 0x1c) = wzr = 0
    0000000100007f38    str     w0, [sp, #0x18]     ;(sp + 0x18) = wo 
    0000000100007f3c    str     x1, [sp, #0x10]     ;(sp + 0x10) = x1
    0000000100007f40    str     w8, [sp, #0xc]      ;(sp + 0xc) = w8 = 0x3
    0000000100007f44    ldr     w8, [sp, #0xc]      ;w8 = (sp + 0xc) = 0x3
    0000000100007f48    cmp     w8, #0x3            ;w8和3比较
    0000000100007f4c    b.le    0x100007f58 ;如果有符号数w8小于等于3则跳转执行 0x100007f58指令
    
    ;如果w8>3则执行这里(按顺序执行)
    0000000100007f50    orr     w8, wzr, #0x1 ;w8 = wzr | 0x1 = 1
    0000000100007f54    str     w8, [sp, #0x8] ;(sp+0x8) = w8 = 1
    
    0000000100007f58    ldr     w8, [sp, #0xc]  ;w8 = (sp+0xc) = 3
    0000000100007f5c    cmp     w8, #0x3        ;w8和3进行比较
    0000000100007f60    b.ge    0x100007f6c ;如果有符号数w8大于等于3则跳转执行 0x100007f6c
    
    ;如果w8<3则执行这里
    0000000100007f64    orr     w8, wzr, #0x2  ;w8 = wzr | 0x2 = 2
    0000000100007f68    str     w8, [sp, #0x8] ;(sp+0x8) = w8 = 2
    
    0000000100007f6c    ldr     w8, [sp, #0xc]  ;w8 = (sp+0xc) = 3
    0000000100007f70    cmp     w8, #0x3        ;w8和3比较
    0000000100007f74    b.ne    0x100007f7c ;如果有符号数w8不等于3则跳转执行 0x100007f7c

    ;如果w8==3则执行这里
    0000000100007f78    str     wzr, [sp, #0x8] ; (sp+0x8) = wzr = 0 即： b=0
    
    0000000100007f7c    orr     w8, wzr, #0x4  ; w8 = wzr | 0x4 = 0x4
    0000000100007f80    str     w8, [sp, #0x4] ;（sp+0x4) = w8 = 0x4
    0000000100007f84    ldr     w8, [sp, #0x8] ; w8 = (sp+0x8) = 0
    0000000100007f88    ldr     w9, [sp, #0x4] ; w9 = (sp+0x4) = 0x4
    0000000100007f8c    cmp     w8, w9 ;w8和w9进行比较
    0000000100007f90    b.ls    0x100007fa0 ;如果无符号数w8小于等于w9，则跳转执行0x100007fa0
    
    0000000100007f94    orr     w8, wzr, #0x1  ;w8 = wzr | 0x1 = 1
    0000000100007f98    str     w8, [sp, #0xc] ;(sp+oxc) = w8
    0000000100007f9c    b       0x100007fa8    ;跳转执行 0x100007fa8
    
    0000000100007fa0    orr     w8, wzr, #0x2 ; w8 = wzr | 0x2 = 2
    0000000100007fa4    str     w8, [sp, #0xc]; (sp+0xc) = w8 = 2 即：a=2
    
    0000000100007fa8    movz    w8, #0x0  
    0000000100007fac    mov     x0, x8
    0000000100007fb0    add     sp, sp, #0x20
    0000000100007fb4    ret     
                        ; endp
    
    ```


4. for循环

    **C语言：**

    ```
    #include <stdio.h>
    
    int main(int argc, char* argv[])
    {
        for(int a=0; a<10; a++){
    	printf("a = %d\n",a);
        }
        return 0;
    }
    
    ```

    **反汇编代码：**

    ```
                         _main:
    0000000100007f14   sub    sp, sp, #0x30 
    0000000100007f18   stp    x29, x30, [sp, #0x20]
    0000000100007f1c   add    x29, sp, #0x20
    0000000100007f20   stur   wzr, [x29, #0xfffffffc]
    0000000100007f24   stur   w0, [x29, #0xfffffff8]
    0000000100007f28   str    x1, [sp, #0x10]
    0000000100007f2c   str    wzr, [sp, #0xc] ;(sp+0xc) = wzr = 0
    
    0000000100007f30   ldr    w8, [sp, #0xc] ;w8 = (sp+0xc) = 0
    0000000100007f34   cmp    w8, #0xa    ;w8和0xa进行比较
    0000000100007f38   b.ge   0x100007f6c ;w8>=10 则跳转执行0x100007f6c
    
    0000000100007f3c   ldr    w8, [sp, #0xc] ;w8= (sp+0xc) = 0
    0000000100007f40   mov    x0, x8         ;x0 = x8
    0000000100007f44   mov    x9, sp         ;x9 = sp
    0000000100007f48   str    x0, [x9]       ;(x9) = x0 
    0000000100007f4c   adrp   x0, #0x100007000 
    0000000100007f50   add    x0, x0, #0xfb0 ;x0 = x0+0xfb0
    0000000100007f54   bl     imp___stubs__printf ;调用打印
    0000000100007f58   str    w0, [sp, #0x8]    ;(sp+0x8) = w0 
    0000000100007f5c   ldr    w8, [sp, #0xc]    ;w8 = (sp+0xc) = 0
    0000000100007f60   add    w8, w8, #0x1      ;w8 = w8+0x1
    0000000100007f64   str    w8, [sp, #0xc]    ;(sp+0xc) = w8
    0000000100007f68   b      0x100007f30       ;无条件跳转 0x100007f30
    
    0000000100007f6c   movz   w8, #0x0 
    0000000100007f70   mov    x0, x8
    0000000100007f74   ldp    x29, x30, [sp, #0x20]
    0000000100007f78   add    sp, sp, #0x30
    0000000100007f7c   ret        
                      ; endp
    
    ```


5. while

    **C语言：**

    ```
    #include <stdio.h>
    int main(int argc, char* argv[])
    {
        int a = 0;
        while(a<10){
    	printf("a = %d\n",a);
    	a++;
        }
        return 0;
    }
    
    ```

    **反汇编代码：**

    ```
                         _main:
    0000000100007f14   sub    sp, sp, #0x30   
    0000000100007f18   stp    x29, x30, [sp, #0x20]
    0000000100007f1c   add    x29, sp, #0x20
    0000000100007f20   stur   wzr, [x29, #0xfffffffc]
    0000000100007f24   stur   w0, [x29, #0xfffffff8]
    0000000100007f28   str    x1, [sp, #0x10]
    0000000100007f2c   str    wzr, [sp, #0xc]  ; (sp+0xc) = wzr = 0
    
    0000000100007f30   ldr    w8, [sp, #0xc]    ;w8 = (sp+0xc) 
    0000000100007f34   cmp    w8, #0xa          ;w8和10进行比较
    0000000100007f38   b.ge   0x100007f6c       ;如果大于等于10则跳转到0x100007f6c
    
    0000000100007f3c   ldr    w8, [sp, #0xc]    ;w8 = (sp+0xc)
    0000000100007f40   mov    x0, x8    ;输出相关逻辑
    0000000100007f44   mov    x9, sp
    0000000100007f48   str    x0, [x9]
    0000000100007f4c   adrp   x0, #0x100007000 imp___stubs__printf
    0000000100007f50   add    x0, x0, #0xfb0  
    0000000100007f54   bl     imp___stubs__printf ;输出逻辑结束
    0000000100007f58   ldr    w8, [sp, #0xc]    ;w8 = (sp+0xc) 
    0000000100007f5c   add    w8, w8, #0x1      ;w8 = w8+1
    0000000100007f60   str    w8, [sp, #0xc]    ;(sp+0xc) = w8
    0000000100007f64   str    w0, [sp, #0x8]    ;(sp+0x8) = w0
    0000000100007f68   b      0x100007f30       ;跳转 -> 0x100007f30
    
    0000000100007f6c   movz   w8, #0x0   
    0000000100007f70   mov    x0, x8
    0000000100007f74   ldp    x29, x30, [sp, #0x20]
    0000000100007f78   add    sp, sp, #0x30
    0000000100007f7c   ret    
                        ; endp
    
    ```


6. switch语句

    **C语言：**

    ```
    #include <stdio.h>
    
    int main(int argc, char* argv[])
    {
        int a = 3;
        int b = 5;
        switch(a){
            case 0:  b=6; break;
            case 1:  b=7; break;
            case 2:  b=8; break;
            case 3:  b=9; break;
            default:  b=0; break;
        }
        printf("b = %d\n", b);
        return 0;
    }
    ```

    **反汇编代码：**

    ```
                       _main:
    0000000100007ea8   sub    sp, sp, #0x40  
    0000000100007eac   stp    x29, x30, [sp, #0x30] ;分配0x30个内存空间
    0000000100007eb0   add    x29, sp, #0x30 ;栈底指针
    0000000100007eb4   movz   w8, #0x0               ; w8 = 0
    0000000100007eb8   stur   w8, [x29, #0xfffffffc] ;(x29+0xfffffffc) = w8 = 0
    0000000100007ebc   stur   w0, [x29, #0xfffffff8] ;(x29+0xfffffff8) = w0
    0000000100007ec0   stur   x1, [x29, #0xfffffff0] ;(x29+0xfffffff0) = x1
    0000000100007ec4   orr    w8, wzr, #0x3          ;w8 = wzr | 0x3 = 0x3
    0000000100007ec8   stur   w8, [x29, #0xffffffec] ;(x29+0xffffffec) = w8 = 0x3
    0000000100007ecc   movz   w8, #0x5      ;w8 = 0x5
    0000000100007ed0   str    w8, [sp, #0x18]   ;(sp+0x18) = w8 = 0x5
    0000000100007ed4   ldur   w8, [x29, #0xffffffec]    ;w8=(x29+0xffffffec)=0x3
    0000000100007ed8   mov    x1, x8
    0000000100007edc   mov    x8, x1
    0000000100007ee0   subs   w8, w8, #0x3  ;w8 = w8-0x3 = 0   #0x3 为switch语句中条件的个数-1
    0000000100007ee4   str    x1, [sp, #0x10]   ;(sp+0x10) = x1
    0000000100007ee8   str    w8, [sp, #0xc]    ;(sp+0xc) = w8 = 0
    0000000100007eec   b.hi   0x100007f38   ;无符号数大于则跳转0x100007f38,即跳default
    
    0000000100007ef0   adrp   x8, #0x100007000
    0000000100007ef4   add    x8, x8, #0xf70     ; 0x100007f70
    0000000100007ef8   ldr    x9, [sp, #0x10]   ;x9 = (sp+0x10)
    0000000100007efc   ldrsw  x10, [x8, x9, lsl #2] 
    0000000100007f00   add    x8, x10, x8   ;x8 = x10 + x8
    0000000100007f04   br     x8    ;跳转x8
    0000000100007f08   orr    w8, wzr, #0x6     ; w8 = 0x6
    0000000100007f0c   str    w8, [sp, #0x18]   ;(sp + 0x18) = w8
    0000000100007f10   b      0x100007f3c       ;跳转
    
    0000000100007f14   orr    w8, wzr, #0x7     ;w8 = 0x7
    0000000100007f18   str    w8, [sp, #0x18]   ;(sp+0x18) = 28
    0000000100007f1c   b      0x100007f3c
    
    0000000100007f20   orr    w8, wzr, #0x8     ;w8 = 0x8
    0000000100007f24   str    w8, [sp, #0x18]   ;(sp+0x18)=w8
    0000000100007f28   b      0x100007f3c
    
    0000000100007f2c   movz   w8, #0x9          ;w8=9
    0000000100007f30   str    w8, [sp, #0x18]   ;(sp+0x18) = w8
    0000000100007f34   b      0x100007f3c
    
    0000000100007f38   str    wzr, [sp, #0x18]  ;(sp+0x18)=0  default
    
    0000000100007f3c   ldr    w8, [sp, #0x18]   ;w8=(sp+0x18)
    0000000100007f40   mov    x0, x8    
    0000000100007f44   mov    x9, sp
    0000000100007f48   str    x0, [x9]
    0000000100007f4c   adrp   x0, #0x100007000  ;argument #1 for method imp___stubs__printf
    0000000100007f50   add    x0, x0, #0xfb0   ; "b = %d\\n"
    0000000100007f54   bl     imp___stubs__printf
    
    0000000100007f58   movz   w8, #0x0
    0000000100007f5c   str    w0, [sp, #0x8]
    0000000100007f60   mov    x0, x8
    0000000100007f64   ldp    x29, x30, [sp, #0x30]
    0000000100007f68   add    sp, sp, #0x40
    0000000100007f6c   ret    
                       ; endp
    0000000100007f70   dd     0xffffff98, 0xffffffa4, 0xffffffb0, 0xffffffbc 
    ```
    
    
### 四. OC代码反汇编测试
创建一个APP项目，编译的时候选择真机
**OC代码**

```
#import "AppDelegate.h"

@interface Person : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
- (instancetype)initWithName:(NSString *)name age:(NSInteger)age;
@end

@implementation Person

- (instancetype)initWithName:(NSString *)name age:(NSInteger)age {
    
    if (self = [super init]) {
        _name = name;
        _age = age;
    }
    return self;
}

@end

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    [self personTestCode];
    return YES;
}

- (void)personTestCode {
    
    Person *xiaoming = [[Person alloc] initWithName:@"xiaoming" age:20];
    xiaoming.age = 21;
    Person *zhangfei = [[Person alloc] init];
    zhangfei.name = @"zhangfei";
    zhangfei.age = 24;
    
}

@end
```

**Hopper 反汇编代码：**

```
================ B E G I N N I N G   O F   P R O C E D U R E ================



                     -[Person initWithName:age:]:
0000000100006584         sub        sp, sp, #0x60                               ; Objective C Implementation defined at 0x1000081f8 (instance)  0x1000081f8 是数据段的地址，Person类的实现
0000000100006588         stp        x29, x30, [sp, #0x50]
000000010000658c         add        x29, sp, #0x50 ;x29=sp+0x50
0000000100006590         sub        x8, x29, #0x18 ;x8 = x29-0x18
0000000100006594         movz       x9, #0x0        ;x9 = 0
0000000100006598         stur       x0, [x29, #0xfffffff8] ;(x29+0xfffffff8) = x0
000000010000659c         stur       x1, [x29, #0xfffffff0] 
00000001000065a0         stur       x9, [x29, #0xffffffe8] 
00000001000065a4         mov        x0, x8
00000001000065a8         mov        x1, x2
00000001000065ac         str        x3, [sp, #0x18] ;(sp+0x18)=x3 传进来的age参数
00000001000065b0         bl         imp___stubs__objc_storeStrong
00000001000065b4         add        x0, sp, #0x20   ;x0 = sp+0x20
00000001000065b8         adrp       x8, #0x100008000 ;地址生成指令 代表Person(声明)这个类，和类实现地址差不多0x1000081f8
00000001000065bc         add        x8, x8, #0xe30    ; @selector(init) x8 = x8+0xe30 即：x8 = [Super init] 
00000001000065c0         adrp       x9, #0x100008000 ;x9 = Person
00000001000065c4         add        x9, x9, #0xe78  ; 0x100008e78
00000001000065c8         movz       x1, #0x0    ;x1 = 0
00000001000065cc         ldr        x2, [sp, #0x18] = x2 = (sp+0x18)
00000001000065d0         stur       x2, [x29, #0xffffffe0] ;(x29+0xffffffe0) = x2
00000001000065d4         ldur       x3, [x29, #0xfffffff8] ;(x29+0xfffffff8) = x3
00000001000065d8         stur       x1, [x29, #0xfffffff8] ;x1 = x3
00000001000065dc         str        x3, [sp, #0x20] ;(sp+0x20) = x3
00000001000065e0         ldr        x9, [x9]    ;（x9) = x9 x9指向的地址里的值，赋值x9
00000001000065e4         str        x9, [sp, #0x28] ;(sp+0x28) = x9
00000001000065e8         ldr        x1, [x8] ;x1 = (x8)
00000001000065ec         bl         imp___stubs__objc_msgSendSuper2
00000001000065f0         sub        x8, x29, #0x8 ;x8 = x29+0x8
00000001000065f4         mov        x9, x0
00000001000065f8         stur       x9, [x29, #0xfffffff8]
00000001000065fc         mov        x1, x0
0000000100006600         str        x0, [sp, #0x10]
0000000100006604         mov        x0, x8
0000000100006608         bl         imp___stubs__objc_storeStrong self = [super init] 赋值存储完成
000000010000660c         ldr        x8, [sp, #0x10]
0000000100006610         cbz        x8, 0x100006654 ;如果x8==0，跳转到0x100006654

0000000100006614         adrp       x8, #0x100008000 ;x8 = self
0000000100006618         add        x8, x8, #0xe80   ;地址 x8 = x8+0xe80=_name _OBJC_IVAR_$_Person._name
000000010000661c         ldur       x9, [x29, #0xffffffe8] ; x9 = (x29+0xffffffe8)
0000000100006620         ldur       x10, [x29, #0xfffffff8];x10 = (x29+0xfffffff8)
0000000100006624         ldrsw      x8, [x8] ;x8 = (x8) 
0000000100006628         add        x8, x10, x8 ;x8 = x10+x8
000000010000662c         mov        x0, x8 ;x0 = x8
0000000100006630         mov        x1, x9 ;x1 = x9
0000000100006634         bl         imp___stubs__objc_storeStrong 存储name完成

0000000100006638         adrp       x8, #0x100008000 ;x8=self
000000010000663c         add        x8, x8, #0xe84   ;_age = age    _OBJC_IVAR_$_Person._age
0000000100006640         ldur       x9, [x29, #0xffffffe0]
0000000100006644         ldur       x10, [x29, #0xfffffff8]
0000000100006648         ldrsw      x8, [x8]
000000010000664c         add        x8, x10, x8
0000000100006650         str        x9, [x8] 存储age完成

0000000100006654         ldur       x8, [x29, #0xfffffff8]                      ; XREF=-[Person initWithName:age:]+140
0000000100006658         mov        x0, x8
000000010000665c         bl         imp___stubs__objc_retain
0000000100006660         movz       x8, #0x0
0000000100006664         sub        x30, x29, #0x18
0000000100006668         str        x0, [sp, #0x8]
000000010000666c         mov        x0, x30
0000000100006670         mov        x1, x8
0000000100006674         bl         imp___stubs__objc_storeStrong
0000000100006678         movz       x8, #0x0
000000010000667c         sub        x0, x29, #0x8
0000000100006680         mov        x1, x8
0000000100006684         bl         imp___stubs__objc_storeStrong
0000000100006688         ldr        x0, [sp, #0x8]
000000010000668c         ldp        x29, x30, [sp, #0x50]
0000000100006690         add        sp, sp, #0x60
0000000100006694         ret        
                        ; endp


================ B E G I N N I N G   O F   P R O C E D U R E ================

                     -[AppDelegate personTestCode]:
0000000100006860         sub        sp, sp, #0x30                               ; Objective C Implementation defined at 0x100008cf0 (instance)
0000000100006864         stp        x29, x30, [sp, #0x20]
0000000100006868         add        x29, sp, #0x20
000000010000686c         adrp       x8, #0x100008000  ;x8为Person类
0000000100006870         add        x8, x8, #0xe40                              ; @selector(alloc) x8 = [Person alloc]
0000000100006874         adrp       x9, #0x100008000
0000000100006878         add        x9, x9, #0xe68                              ; objc_cls_ref_Person  x9为person对象
000000010000687c         stur       x0, [x29, #0xfffffff8]
0000000100006880         str        x1, [sp, #0x10]
0000000100006884         ldr        x9, [x9]
0000000100006888         ldr        x1, [x8]
000000010000688c         mov        x0, x9
0000000100006890         bl         imp___stubs__objc_msgSend ;调用init
0000000100006894         adrp       x8, #0x100008000
0000000100006898         add        x8, x8, #0x70                               ; @"xiaoming"  ;参数name
000000010000689c         movz       x3, #0x14 ;参数age=20
00000001000068a0         adrp       x9, #0x100008000
00000001000068a4         add        x9, x9, #0xe48                              ; @selector(initWithName:age:) ;x9为初始化方法的地址
00000001000068a8         ldr        x1, [x9]
00000001000068ac         mov        x2, x8
00000001000068b0         bl         imp___stubs__objc_msgSend ;调用initWithName:age:
00000001000068b4         movz       x2, #0x15 ; 参数x2=age=21
00000001000068b8         adrp       x8, #0x100008000
00000001000068bc         add        x8, x8, #0xe50                              ; @selector(setAge:) ;x8为setAge:方法地址
00000001000068c0         str        x0, [sp, #0x8]
00000001000068c4         ldr        x9, [sp, #0x8]
00000001000068c8         ldr        x1, [x8]
00000001000068cc         mov        x0, x9
00000001000068d0         bl         imp___stubs__objc_msgSend ;调用setAge:


00000001000068d4         adrp       x8, #0x100008000
00000001000068d8         add        x8, x8, #0xe40   ; @selector(alloc) [Person alloc]
00000001000068dc         adrp       x9, #0x100008000
00000001000068e0         add        x9, x9, #0xe68                              ; objc_cls_ref_Person
00000001000068e4         ldr        x9, [x9]
00000001000068e8         ldr        x1, [x8]
00000001000068ec         mov        x0, x9
00000001000068f0         bl         imp___stubs__objc_msgSend ;调用alloc

00000001000068f4         adrp       x8, #0x100008000
00000001000068f8         add        x8, x8, #0xe30                              ; @selector(init)
00000001000068fc         ldr        x1, [x8]
0000000100006900         bl         imp___stubs__objc_msgSend ;调用init

0000000100006904         adrp       x8, #0x100008000
0000000100006908         add        x8, x8, #0x90                               ; @"zhangfei" ;参数name
000000010000690c         adrp       x9, #0x100008000
0000000100006910         add        x9, x9, #0xe58  ; @selector(setName:) ;person setName: 方法地址
0000000100006914         str        x0, [sp]
0000000100006918         ldr        x0, [sp]
000000010000691c         ldr        x1, [x9]
0000000100006920         mov        x2, x8
0000000100006924         bl         imp___stubs__objc_msgSend ; 调用setName: 

0000000100006928         orr        x2, xzr, #0x18 ;参数age=24
000000010000692c         adrp       x8, #0x100008000
0000000100006930         add        x8, x8, #0xe50                              ; @selector(setAge:) ;setAge:方法地址
0000000100006934         ldr        x9, [sp]
0000000100006938         ldr        x1, [x8]
000000010000693c         mov        x0, x9
0000000100006940         bl         imp___stubs__objc_msgSend ;调用setAge:

0000000100006944         movz       x8, #0x0
0000000100006948         mov        x9, sp
000000010000694c         mov        x0, x9
0000000100006950         mov        x1, x8
0000000100006954         bl         imp___stubs__objc_storeStrong
0000000100006958         movz       x8, #0x0
000000010000695c         add        x9, sp, #0x8
0000000100006960         mov        x0, x9
0000000100006964         mov        x1, x8
0000000100006968         bl         imp___stubs__objc_storeStrong

000000010000696c         ldp        x29, x30, [sp, #0x20]
0000000100006970         add        sp, sp, #0x30
0000000100006974         ret        
                        ; endp
```
    
### 五. Xcode汇编分析

```

#import <Foundation/Foundation.h>

int sum(int a, int b){
    int c = 3;
    int d = 4;
    return a+b+c+d;
}

int main(int argc, const char * argv[]) {

    int a = sum(1, 2);
    printf("%d\n", a);
    
    return 0;
}
```

汇编代码

```
demo1`main:
    0x100000f40 <+0>:  pushq  %rbp     ;入栈保护bp指针
    0x100000f41 <+1>:  movq   %rsp, %rbp    ; bp指向sp
    0x100000f44 <+4>:  subq   $0x20, %rsp    ;  声明32字节的空间作为局部变量空间
    0x100000f48 <+8>:  movl   $0x1, %eax ;局部变量1给%eax寄存器
    0x100000f4d <+13>: movl   $0x2, %ecx  ;局部变量2给%ecx寄存器
    0x100000f52 <+18>: movl   $0x0, -0x4(%rbp) ;0入栈
    0x100000f59 <+25>: movl   %edi, -0x8(%rbp) ;%edi入栈
    0x100000f5c <+28>: movq   %rsi, -0x10(%rbp); %rsi入栈
    0x100000f60 <+32>: movl   %eax, %edi  ;%eax值给%edi
    0x100000f62 <+34>: movl   %ecx, %esi   ;%ecx值给%esi
    0x100000f64 <+36>: callq  0x100000f10 ; sum at main.m:11  调用sum函数
    0x100000f69 <+41>: leaq   0x3a(%rip), %rdi          ; "%d\n"
    0x100000f70 <+48>: movl   %eax, -0x14(%rbp)
->  0x100000f73 <+51>: movl   -0x14(%rbp), %esi
    0x100000f76 <+54>: movb   $0x0, %al
    0x100000f78 <+56>: callq  0x100000f8a               ; symbol stub for: printf
    0x100000f7d <+61>: xorl   %ecx, %ecx
    0x100000f7f <+63>: movl   %eax, -0x18(%rbp)
    0x100000f82 <+66>: movl   %ecx, %eax
    0x100000f84 <+68>: addq   $0x20, %rsp ;释放局部变量空间
    0x100000f88 <+72>: popq   %rbp
    0x100000f89 <+73>: retq   

demo1`sum:
    0x100000f10 <+0>:  pushq  %rbp     ;入栈保护bp指针
    0x100000f11 <+1>:  movq   %rsp, %rbp  ; bp指向sp
    0x100000f14 <+4>:  movl   %edi, -0x4(%rbp) ;将%edi的值入栈
    0x100000f17 <+7>:  movl   %esi, -0x8(%rbp) ;将%esi的值入栈
->  0x100000f1a <+10>: movl   $0x3, -0xc(%rbp) ;将局部变量3入栈
    0x100000f21 <+17>: movl   $0x4, -0x10(%rbp);将局部变量4入栈
    0x100000f28 <+24>: movl   -0x4(%rbp), %esi ;将外部参数给%esi
    0x100000f2b <+27>: addl   -0x8(%rbp), %esi; 外部参数与%esi相加后放入%esi
    0x100000f2e <+30>: addl   -0xc(%rbp), %esi ;局部变量和%esi相加后放入%esi
    0x100000f31 <+33>: addl   -0x10(%rbp), %esi  ;局部变量和%esi相加后放入%esi
    0x100000f34 <+36>: movl   %esi, %eax ;将返回值放入%eax
    0x100000f36 <+38>: popq   %rbp ;恢复bp
    0x100000f37 <+39>: retq    ;返回

```

###引用
[1. 在iOS中如何使用汇编语言](http://www.cnblogs.com/zenny-chen/archive/2011/10/31/2229731.html)


