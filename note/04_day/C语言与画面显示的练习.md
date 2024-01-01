#### C语言与画面显示的联系

##### 1.用C语言实现内存写入

naskfunc.nas里添加的部分

```assembly
;ESP + 4 存第一个参数 ESP + 8 存第二个参数
_write_mem8: ; void write_mem8(int addr, intdata);
	MOV ECX,[ESP+4] ; [ESP + 4]中存放的是地址，将其读入ECX
	MOV AL,[ESP+8] 	; [ESP + 8]中存放的是数据，将其读入AL
	MOV [ECX],AL
	RE
```

当调用C语言的 write_mem8 函数时，就会来这里寻找。

相当于汇编函数的”MOV BYTE[0X1234],0X56“

> 如果与C语言联合使用的话，有的寄存器能自由使 用，有的寄存器不能自由使用，能自由使用的只有 EAX、ECX、EDX这3个。至于其他寄存器，只能使 用其值，而不能改变其值。

bookpack.c 的内容

```c
void io_hlt(void);
void write_mem8(int addr, int data);
void HariMain(void){
	int i; /*变量声明：i是一个32位整数*/
	for (i = 0xa0000; i <= 0xaffff; i++) {// 在AL = 0x13的画面模式下 VRAM（显卡内存）是0xa0000 ~ 0xaffff 的64kb
		write_mem8(i, 15); /* MOV BYTE [i],15*/
	}
	for (;;) {
		io_hlt();
	}
}
```

因为VRAM全部都写入了15，意思是全部像素的颜色 都是第15种颜色，而第15种颜色碰巧是纯白，所 以画面就成了白色。

##### 2.条纹图案

```c
for (i = 0xa0000; i <= 0xaffff; i++) {//使输入的颜色在0~15之间循环
	write_mem8(i, i & 0x0f);
}
```

##### 3.挑战指针

为什么指针要声明类型？

因为需要让寄存器知道自己读入的是多少位的内存地址。

> MOV [ 0x1234], 0x56 
>
> 会出错。这是因为指定内存时，不知道到底 是BYTE，还是WORD，还是DWORD。只有在另 一方也是寄存器的时候才能省略，其他情况都不能 省略。

才能告诉计算机这是BYTE呢？

char *p; /*用于BYTE类地址*/

short *p; /*用于WORD类地址*/ 

int *p; /*用于DWORD类地址*/

###### 直接通过C语言来改变地址

```c
void HariMain(void){
	int i; /*变量声明。变量i是32位整数*/
	char *p; /*变量p，用于BYTE型地址*/
	for (i = 0xa0000; i <= 0xaffff; i++) {
		p = i; /*代入地址*/  //此处会出现类型转换警告 因为我们把一个int 放入了 char*中
 		*p = i & 0x0f;
		/*这可以替代write_mem8(i, i & 0x0f);*/
	}
	for (;;) {
	io_hlt();
	}
}
```

##### 4.色号设定

这个8位彩色模式，是由程序员随意指定0～255的 数字所对应的颜色的。比如说25号颜色对应 #ffffff，26号颜色对应#123456等。这种方式就 叫做调色板（palette）。

笔者通过制作OSAKA知道：要想描绘一个操作系 统模样的画面，只要有以下这16种颜色就足够了。 所以这次我们也使用这16种颜色，并给它们编上号 码0-15。

###### 本次的bootpack.c

```c
/*以下函数全部定义在汇编文件中*/
void io_hlt(void);
void io_cli(void);
void io_out8(int port, int data);
int io_load_eflags(void);
void io_store_eflags(int eflags);

/*就算写在同一个源文件里，如果想在定义前使用，还是
必须事先声明一下。*/
void init_palette(void);
void set_palette(int start, int end, unsignedchar *rgb);

void HariMain(void){
	int i; /* 声明变量。变量i是32位整数型 */
	char *p; /* 变量p是BYTE [...]用的地址 */
	init_palette(); /* 设定调色板 */
	p = (char *) 0xa0000; /* 指定地址 */
	for (i = 0; i <= 0xffff; i++) {
		p[i] = i & 0x0f;
	}
	for (;;) {
		io_hlt();
	}
}
void init_palette(void){
    //声明了一个static数组
    //声明了一个有48字符的char数组
    //汇编语言的DB，在C语言中也有类似的指示方法，那就是在声明时加上static。
	static unsigned char table_rgb[16 * 3] = {
		0x00, 0x00, 0x00, /* 0:黑 */
		0xff, 0x00, 0x00, /* 1:亮红 */
		0x00, 0xff, 0x00, /* 2:亮绿 */
		0xff, 0xff, 0x00, /* 3:亮黄 */
		0x00, 0x00, 0xff, /* 4:亮蓝 */
		0xff, 0x00, 0xff, /* 5:亮紫 */
		0x00, 0xff, 0xff, /* 6:浅亮蓝 */
		0xff, 0xff, 0xff, /* 7:白 */
		0xc6, 0xc6, 0xc6, /* 8:亮灰 */
		0x84, 0x00, 0x00, /* 9:暗红 */
		0x00, 0x84, 0x00, /* 10:暗绿 */
		0x84, 0x84, 0x00, /* 11:暗黄 */
		0x00, 0x00, 0x84, /* 12:暗青 */
		0x84, 0x00, 0x84, /* 13:暗紫 */
		0x00, 0x84, 0x84, /* 14:浅暗蓝 */
		0x84, 0x84, 0x84 /* 15:暗灰 */
	};
	set_palette(0, 15, table_rgb);
	return;
/* C语言中的static char语句只能用于数据，相当
于汇编中的DB指令 */
}
void set_palette(int start, int end, unsigned char *rgb){
	int i, eflags;
	eflags = io_load_eflags(); /* 记录中断许可标志的值*/
	io_cli(); /* 将中断许可标志置为0，禁止中断 */
	io_out8(0x03c8, start);//往指定装置里传送数据的函数
	for (i = start; i <= end; i++) {
		io_out8(0x03c9, rgb[0] / 4);
		io_out8(0x03c9, rgb[1] / 4);
		io_out8(0x03c9, rgb[2] / 4);
	rgb += 3;
	}
	io_store_eflags(eflags); /* 复原中断许可标志 */
	return;
}
/*在这个特定的代码段中，除以4的操作主要是为了调整颜色值的范围，使其适应调色板寄存器的取值范围（通常是0到63）。这样的调整可能是因为硬件或图形模式的要求。*/

```

> 调色板的访问步骤。 
>
> 首先在一连串的访问中屏蔽中断（比如 CLI）。
>
> 将想要设定的调色板号码写入0x03c8，紧 接着，按R，G，B的顺序写入0x03c9。如 果还想继续设定下一个调色板，则省略调色 板号码，再按照RGB的顺序写入0x03c9就 行了。 
>
> 如果想要读出当前调色板的状态，首先要将 调色板的号码写入0x03c7，再从0x03c9读 取3次。读出的顺序就是R，G，B。如果要 继续读出下一个调色板，同样也是省略调色 板号码的设定，按RGB的顺序读出。 
>
> 如果最初执行了CLI，那么最后要执行STI。

##### 5.绘制矩形

颜色备齐了，下面我们来画“画”吧。首先从 VRAM与画面上的“点”的关系开始说起。在当前 画面模式中，画面上有320×200（=64 000）个 像素。假设左上点的坐标是（0,0），右下点的坐 标是（319,319），那么像素坐标（x,y）对应的 VRAM地址应按下式计算。

######  0xa0000 + x + y * 320

本次的bootpack.c节选

```c
#define COL8_000000 0
#define COL8_FF0000 1
#define COL8_00FF00 2
#define COL8_FFFF00 3
#define COL8_0000FF 4
#define COL8_FF00FF 5
#define COL8_00FFFF 6
#define COL8_FFFFFF 7
#define COL8_C6C6C6 8
#define COL8_840000 9
#define COL8_008400 10
#define COL8_848400 11
#define COL8_000084 12
#define COL8_840084 13
#define COL8_008484 14
#define COL8_848484 15
void HariMain(void)
{
	char *p; /* p变量的地址 */
	init_palette(); /* 设置调色板 */
	p = (char *) 0xa0000; /* 将地址赋值进去 */
	boxfill8(p, 320, COL8_FF0000, 20, 20, 120, 120);
	boxfill8(p, 320, COL8_00FF00, 70, 50, 170, 150);
	boxfill8(p, 320, COL8_0000FF, 120, 80, 220, 180);
	for (;;) {
		io_hlt();
	}
}
void boxfill8(unsigned char *vram, int xsize, unsigned char c, int x0, int y0, int x1, int y1){
	int x, y;
	for (y = y0; y <= y1; y++) {
		for (x = x0; x <= x1; x++)
			vram[y * xsize + x] = c;
}
	return;
}
```

##### 5.绘画任务栏

HariMain

```c
void HariMain(void)
{
	char *vram;
	int xsize, ysize;
	init_palette();
	vram = (char *) 0xa0000;
	xsize = 320;
	ysize = 200;
boxfill8(vram, xsize, COL8_008484, 0, 0, xsize - 1, ysize - 29);
boxfill8(vram, xsize, COL8_C6C6C6, 0, ysize - 28, xsize - 1, ysize - 28);
boxfill8(vram, xsize, COL8_FFFFFF, 0, ysize - 27, xsize - 1, ysize - 27);
boxfill8(vram, xsize, COL8_C6C6C6, 0, ysize - 26, xsize - 1, ysize - 1);
boxfill8(vram, xsize, COL8_FFFFFF, 3, ysize - 24, 59, ysize - 24);
boxfill8(vram, xsize, COL8_FFFFFF, 2, ysize - 24, 2, ysize - 4);
boxfill8(vram, xsize, COL8_848484, 3, ysize - 4, 59, ysize - 4);
boxfill8(vram, xsize, COL8_848484, 59, ysize - 23, 59, ysize - 5);
boxfill8(vram, xsize, COL8_000000, 2, ysize - 3, 59, ysize - 3);
boxfill8(vram, xsize, COL8_000000, 60, ysize - 24, 60, ysize - 3);
boxfill8(vram, xsize, COL8_848484, xsize - 47, ysize - 24, xsize - 4, ysize - 24);
boxfill8(vram, xsize, COL8_848484, xsize - 47, ysize - 23, xsize - 47, ysize - 4);
boxfill8(vram, xsize, COL8_FFFFFF, xsize - 47, ysize - 3, xsize - 4, ysize - 3);
boxfill8(vram, xsize, COL8_FFFFFF, xsize - 3, ysize - 24, xsize - 3, ysize - 3);
for (;;) {
	io_hlt();
	}
}
```