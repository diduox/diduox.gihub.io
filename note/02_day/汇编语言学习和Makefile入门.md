#### 汇编语言学习和Makefile入门

##### 1.继续开发

```assembly
; hello-os
; TAB=4

		ORG		0x7c00			; 指明程序的装载地址

; 以下这段是标准FAT12格式磁盘专用的代码

		JMP		entry
		DB		0x90
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

		MOV		SI,msg
putloop:
		MOV		AL,[SI]
		ADD		SI,1			; 给SI加1
		CMP		AL,0
		JE		fin
		MOV		AH,0x0e			; 显示一个文字
		MOV		BX,15			; 指定字符颜色
		INT		0x10			; 调用显卡BIOS
		JMP		putloop
fin:
		HLT						; 让CPU停止，等待指令
		JMP		fin				; 无限循环

msg:
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

8个16位寄存器

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

##### 2.内存：

MOV指令的数据传送源和传送目的地不仅可以是 寄存器或常数，也可以是内存地址。这个时候，我 们就使用方括号（[ ]）来表示内存地址。另外， BYTE、WORD、DWORD等英文词也都是汇编语 言的保留字，下面举个例子吧。 

MOV BYTE [678],123

这个指令是要用内存的“678”号地址来保 存“123”这个数值。

###### 数据大小 [地址]

至于内存地址的指定方法，我们不仅可以使用常 数，还可以用寄存器。比如“BYTE [SI]”、“WORD [BX]”等等。如果SI中保存的是 987的话，“BYTE [SI]”就会被解释成“BYTE [987]”，即指定地址为987的内存。

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

Makefile就像是一个非常聪明的批处理文件。

```makefile
asm :
../z_tools/make.exe -r ipl.bin
run :
../z_tools/make.exe img
copy helloos.img
..\z_tools\qemu\fdimage0.bin
../z_tools/make.exe -C ../z_tools/qemu
install :
../z_tools/make.exe img
../z_tools/imgtol.com w a: helloos.img
```

Makefile，它会自动跳过没有必要的命令，这样不 管任何时候，我们都可以放心地去执行“make img”了。而且就算直接“make run”也可以顺利 运行。“make install”也是一样，只要把磁盘装 到驱动器里，这个命令就会自动作出判断，如果已 经有了最新的helloos.img就直接安装，没有的话就 先自动生成新的helloos.img，然后安装。

Makefile 相当于 make.exe + Makefile 实现批处理文件的整合与自动查找。