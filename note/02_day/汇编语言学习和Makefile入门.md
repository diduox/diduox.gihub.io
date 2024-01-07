#### 汇编语言学习和Makefile入门

##### 1.继续开发

```assembly
; hello-os
; TAB=4

		ORG		0x7c00			; 告诉编译器，程序的数据和代码应该加载到内存的哪个位置
								; 
								
; 以下这段是标准FAT12格式磁盘专用的代码

		JMP		entry
		DB		0x90			; DB 0x90 是一条’NOP‘指令的机器码表示 不执行任何操作
								; 为了确保引导扇区的大小是精确的
		DB		"HELLOIPL"		; 启动区的名称可以是任意的字符串（8字节）
		DW		512				; 每个扇区的大小（必须为512字节）
		DB		1				; 簇的大小（必须为一个扇区1）
		DW		1				; FAT的起始位置（一般从第一个扇区开始）
		DB		2				; FAT的个数（必须为2）
		DW		224				; 根目录的大小（一般设为224项）
		DW		2880			; 该磁盘的大小（必须是2880扇区）
		DB		0xf0			; 磁盘的种类（必须是0xf0）
		DW		9				; FAT的长度（必须是9扇区）
		DW		18				; 1个磁道（track）有几个扇区（必须是18）
		DW		2				; 磁头数（必须是2）
		DD		0				; 不使用分区，必须是0
		DD		2880			; 重写一次磁盘大小
		DB		0,0,0x29		; 意义不明，固定码
		DD		0xffffffff		; （可能是）卷标号码
		DB		"HELLO-OS   "	; 磁盘的名称（11字节）
		DB		"FAT12   "		; 磁盘格式名称（8字节）
		RESB	18				; 先空出18字节

; 程序主体
entry:
		MOV		AX,0			; 初始化寄存器
		MOV		SS,AX
		MOV		SP,0x7c00
		MOV		DS,AX
		MOV		ES,AX

		MOV		SI,msg			; 将msg的地址加载到SI寄存器
								; 以后就可以通过SI对msg进行操作了
putloop:						; 此处是读取msg中的字符 并且显示
		MOV		AL,[SI]			; 读取位于SI内存处的字符
		ADD		SI,1			; 给SI加1 (相当于指针后移一位)
		CMP		AL,0
		JE		fin
		MOV		AH,0x0e			; 显示一个文字
		MOV		BX,15			; 指定字符颜色
		INT		0x10			; 调用显卡BIOS
		JMP		putloop
fin:
		HLT						; 让CPU停止，等待指令
		JMP		fin				; 无限循环

msg:							; 换行两次和显示字符串
		DB		0x0a, 0x0a		; 换行两次
		DB		"hello, world"
		DB		0x0a			; 换行
		DB		0

		RESB	0x7dfe-$		; 填写0x00,直到0x001fe

		DB		0x55, 0xaa

; 以下是启动区以外部分的输出

		DB		0xf0, 0xff, 0xff, 0x00, 0x00, 0x00, 0x00, 0x00
		RESB	4600
		DB		0xf0, 0xff, 0xff, 0x00, 0x00, 0x00, 0x00, 0x00
		RESB	1469432
```

ORG指令，这个指令会告诉汇编器，在开始执行时，将这些机器语言装载到内存的哪个地址。

另外，有了这条指令的话，美元符 （$）的含义也随之变化，它不再是指输出文件的 第几个字节，而是代表将要读入的内存地址。

JMP指令，相当于C语言的goto指令，意思是“跳转”。

entry: 相当与标签的声明，用于指定JMP指令跳转的目的地等。

MOV指令，用于赋值：”MOV AX,0“，相当于“AX=0; 同样，“MOV SS,AX”就相当 于“SS=AX;”。

***显示字符的规定***

> AH=0x0e; 
>
> AL=character code;
>
> BH=0;
>
> BL=color code;
>
> 返回值：无
>
> 注：beep、退格（back space）、CR、LF 都会被当做控制字符处理

所以，如果大家按照这里所写的步骤，往寄存器里 代入各种值，再调用INT0x10，就能顺利地在屏幕 上显示一个字符出来

8个16位寄存器（通用寄存器）

***在Intel的64位x86_64架构中，R8到R15被引入作为额外的通用寄存器。***

> AX——accumulator，累加寄存器 
>
> CX——counter，计数寄存器 
>
> DX——data，数据寄存器 
>
> BX——base，基址寄存器 
>
> SP——stack pointer，栈指针寄存器 
>
> BP——base pointer，基址指针寄存器 
>
> SI——source index，源变址寄存器 
>
> DI——destination index，目的变址寄存器

使用恰当的寄存器进行相应的操作，可以使程序变得更加简洁。

6个段寄存器

> ES——附加段寄存器（extra segment） 
>
> CS——代码段寄存器（code segment）
>
> SS——栈段寄存器（stack segment）
>
> DS——数据段寄存器（data segment） 
>
> FS——没有名称（segment part 2） 
>
> GS——没有名称（segment part 3）



##### 2.内存：

MOV指令的数据传送源和传送目的地不仅可以是 寄存器或常数，也可以是内存地址。这个时候，我 们就使用方括号（[ ]）来表示内存地址。另外， BYTE、WORD、DWORD等英文词也都是汇编语 言的保留字，下面举个例子吧。 

MOV BYTE [678],123

这个指令是要用内存的“678”号地址来保 存“123”这个数值。

###### 数据大小 [地址]

PS:像不像 p 和 *p  有种指针和指针指向的值的感觉

至于内存地址的指定方法，我们不仅可以使用常 数，还可以用寄存器。比如“BYTE [SI]”、“WORD [BX]”等等。如果SI中保存的是 987的话，“BYTE [SI]”就会被解释成“BYTE [987]”，即指定地址为987的内存。

***在此加入一张非常可爱的插图***

![](D:\30daysos\diduox.gihub.io\note\02_day\屏幕截图 2024-01-07 204414.png)

###### 注意：

MOV指令有一个规则，那就是源数据和目的 数据必须位数相同。也就是说，能向AL里代入的就 只有BYTE，这样一来就可以省略BYTE，即可以写 成： MOV AL, [SI]

  如果违反这一规则，比如写“MOV AX,CL”的话，汇编语言就找不到相对应的机器 语言，编译时会出错。

ADD指令：ADD是加法指令若以C语言的形式改写“ADD SI,1”的话，就是SI=SI+1。

CMP指令：简单说来，它是if语句的一部分。譬如Ｃ语 言会有这种语句：

​	 if(a==3){ 处理; }

 即对a和3进行比较，将其翻译成机器语言时，必须 先写“CMP a,3”，告诉CPU比较的对象，然后下 一步再写“如果二者相等，需要做什么”。

JE指令：JE是条件跳转指令中之一所谓条件跳转指令，就 是根据比较的结果决定跳转或不跳转。就JE指令而 言，如果比较结果相等，则跳转到指定的地址；而 如果比较结果不等，则不跳转，继续执行下一条指 令。

```assembly
CMP AL, 0
JE fin
```

##### 3.Makefile入门

`	Makefile` 是一个用于管理项目构建过程的文本文件，通常用于编译源代码、链接目标程序、执行测试等。它包含一组规则和命令，定义了如何构建和维护项目的过程。以下是一些关键概念和用法：

###### 基本结构：

一个简单的 `Makefile` 包含了一系列的规则，每个规则都描述了一个或多个文件之间的依赖关系以及如何生成目标文件。基本的结构如下：

```makefile
target: dependencies
    command
```

- **target：** 表示要生成的目标文件。

- **dependencies：** 表示目标文件依赖的文件或其他目标文件。

- **command：** 表示生成目标文件的命令。

  ###### 示例：

  ```makefile
  all: hello
  
  hello: main.o greet.o
      gcc -o hello main.o greet.o
  
  main.o: main.c
      gcc -c main.c
  
  greet.o: greet.c
      gcc -c greet.c
  
  clean:
      rm -f hello *.o
  ```

  `	clean` 是一个伪目标，用于清理生成的文件，执行 `make clean` 会删除 `hello` 可执行文件和所有 `.o` 目标文件。

  ------

  

​	更简略的写法 直接make run

```makefile
#该规则使用 make.exe 来生成 ipl.bin，类似于在命令行中执行 ../z_tools/make.exe -r ipl.bin
asm :
../z_tools/make.exe -r ipl.bin 
#该规则首先调用 make.exe 生成镜像文件 helloos.img，然后将其复制到 QEMU 目录下的 fdimage0.bin，最后通过 -C 选项指定目录切换到 QEMU 目录并执行 make.exe
run :
../z_tools/make.exe img
copy helloos.img..\z_tools\qemu\fdimage0.bin
../z_tools/make.exe -C ../z_tools/qemu
#该规则首先调用 make.exe 生成 helloos.img，然后使用 imgtol.com 工具将其写入磁盘（假设是软盘）。
install :
../z_tools/make.exe img
../z_tools/imgtol.com w a: helloos.img
```

Makefile，它会自动跳过没有必要的命令，这样不 管任何时候，我们都可以放心地去执行“make img”了。而且就算直接“make run”也可以顺利 运行。“make install”也是一样，只要把磁盘装 到驱动器里，这个命令就会自动作出判断，如果已 经有了最新的helloos.img就直接安装，没有的话就 先自动生成新的helloos.img，然后安装。

Makefile 相当于 make.exe + Makefile 实现批处理文件的整合与自动查找。

#### 朝花夕拾：

##### 1.为什么主引导记录的内存地址是0x7C00

深度好文：[为什么主引导记录的内存地址是0x7C00？ - 阮一峰的网络日志 (ruanyifeng.com)](https://www.ruanyifeng.com/blog/2015/09/0x7c00.html)

![](D:\30daysos\diduox.gihub.io\note\02_day\屏幕截图 2024-01-07 195607.png)

##### 2.内存分布图

<img src="D:\30daysos\diduox.gihub.io\note\02_day\屏幕截图 2024-01-07 195752.png" style="zoom:80%;" />