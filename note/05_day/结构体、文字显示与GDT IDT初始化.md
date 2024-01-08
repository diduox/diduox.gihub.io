#### 结构体、文字显示与 GDT/IDT初始化

##### 1.接收启动信息

我们将每个所需要的地址保存到变量里面，便于后续的操作。

HariMain节选

```c
void HariMain(void)
{
	char *vram;
	int xsize, ysize;
	short *binfo_scrnx, *binfo_scrny;
	int *binfo_vram;
	init_palette();
	binfo_scrnx = (short *) 0x0ff4;//这些地址仅仅是为了与asmhead. nas保持一致才出现的。
	binfo_scrny = (short *) 0x0ff6;//也许我们可以将它当成一个特定的预置数ww
	binfo_vram = (int *) 0x0ff8;//图像缓冲区的开始地址
	xsize = *binfo_scrnx;
	ysize = *binfo_scrny;
	vram = (char *) *binfo_vram;
}
```

##### 2.试用结构体

我们把需要的前置变量全部整合到一个结构体中

```c
struct BOOTINFO {
	char cyls, leds, vmode, reserve;
	short scrnx, scrny;
	char *vram;
};
void HariMain(void)
{
	char *vram;
	int xsize, ysize;
    //在c语言中，如果不使用typedef为结构体定义别名，需要在声明结构体变量时，带上'struct'关键字。
	struct BOOTINFO *binfo;
	init_palette();
	binfo = (struct BOOTINFO *) 0x0ff0;
	xsize = (*binfo).scrnx;
	ysize = (*binfo).scrny;
	vram = (*binfo).vram;
}
```

> 另外，我们还把画面的像素数、颜色数，以及从 BIOS取得的键盘信息都保存了起来。
>
> 保存位置是在 内存0x0ff0附近。
>
> 保存在这里是前面规定好的ww

##### 3.显示字符

以前我们显示字符可以依靠BIOS函数，但这次是32位模式，不能再依赖BIOS了，只能自力更生。

字符可以用8×16的长方形 像素点阵来表示。

像这种描画文字形状的数据称为字体（font）数 据，那这种字体数据是怎样写到程序里的呢？有一 种临时方案：

```c
static char font_A[16] = {
0x00, 0x18, 0x18, 0x18, 0x18, 0x24, 0x24,0x24,
0x24, 0x7e, 0x42, 0x42, 0x42, 0xe7, 0x00,0x00
};
```

输出字符的函数：//好久没有看到如此耿直的函数实现了，有一种力大砖飞的美感。

```c
void putfont8(char *vram, int xsize, int x, int y, char c, char *font)
{
	int i;
	char d; /* data */
	for (i = 0; i < 16; i++) {
		d = font[i];
        /*
       	0x80 = 1000 0000
       	0x40 = 0100 0000
       	0x20 = 0010 0000
       	.
       	.
       	.
        */
        //用8个if来判断每一位是不是1 如果是1 则输出一个像素
		if ((d & 0x80) != 0) { vram[(y + i) * xsize + x + 0] = c; }
		if ((d & 0x40) != 0) { vram[(y + i) * xsize + x + 1] = c; }
		if ((d & 0x20) != 0) { vram[(y + i) * xsize + x + 2] = c; }
		if ((d & 0x10) != 0) { vram[(y + i) * xsize + x + 3] = c; }
		if ((d & 0x08) != 0) { vram[(y + i) * xsize + x + 4] = c; }
		if ((d & 0x04) != 0) { vram[(y + i) * xsize + x + 5] = c; }
		if ((d & 0x02) != 0) { vram[(y + i) * xsize + x + 6] = c; }
		if ((d & 0x01) != 0) { vram[(y + i) * xsize + x + 7] = c; }
	}
	return;
}
```

通过指针让代码变得更加精简一些//果然程序等于数据结构 + 算法啊

```c
void putfont8(char *vram, int xsize, int x, int y, char c, char *font)
{
	int i;
	char *p, d /* data */;
	for (i = 0; i < 16; i++) {
		p = vram + (y + i) * xsize + x;
        d = font[i];
        if ((d & 0x80) != 0) { p[0] = c; }
        if ((d & 0x40) != 0) { p[1] = c; }
        if ((d & 0x20) != 0) { p[2] = c; }
        if ((d & 0x10) != 0) { p[3] = c; }
        if ((d & 0x08) != 0) { p[4] = c; }
        if ((d & 0x04) != 0) { p[5] = c; }
        if ((d & 0x02) != 0) { p[6] = c; }
        if ((d & 0x01) != 0) { p[7] = c; }
	}
	return;
}
```

##### 4.增加字体

我们这次就将hankaku.txt这个文本文件加入到我 们的源程序大家庭中来。这个文件的内容如下：

###### hankaku.txt的内容

```
char 0x41
........
...**...
...**...
...**...
...**...
..*..*..
..*..*..
..*..*..
..*..*..
.******.
.*....*.
.*....*.
.*....*.
***..***
........
........
```

> 当然，这既不是C语言，也不是汇编语言，所以需 要专用的编译器。
>
> 新做一个编译器很麻烦，所以我 们还是使用在制作OSASK时曾经用过的工具 （makefont.exe）。
>
> 说是编译器，其实有点言过其 实了，只不过是将上面这样的文本文件（256个字 符的字体文件）读进来，然后输出成 16×256=4096字节的文件而已。 
>
> 编译后生成hankaku.bin文件，但仅有这个文件还 不能与bootpack.obj连接，因为它不是目标 （obj）文件。所以，还要加上连接所必需的接口 信息，将它变成目标文件。
>
> 这项工作由bin2obj.exe 来完成。它的功能是将所给的文件自动转换成目标 程序，就像将源程序转换成汇编那样。

也就是说， 好像将下面这两行程序编译成了汇编：

```assembly
_hankanku:
	DB 各种数据（共4096字节）
```

如果在C语言中使用这种字体数据，只需要写上以下语句就可以了。

```c
extern char hankaku[4096];
```

OSASK的字体数据，依照一般的ASCII字符编码， 含有256个字符。A的字符编码是0x41，所以A的 字体数据，放在自“hankaku + 0x41 * 16”开始 的16字节里。C语言中A的字符编码可以用’A’来 表示，正好可以用它来代替0x41，所以也可以写 成“hankaku + ‘A’ * 16”。

***解释：***

为什么存放’A‘的地址是 hankaku + ‘A’ * 16，因为一个字符占16个字节，前面存在了65(0 ~ 64)个字符每个各占了16字节，所以轮到'A'的地址就是hankaku + 'A' * 16.(ASCII 字符编码是从 0 开始的,所以'A'前面有64个字符)

本次的HariMain的内容

```c
void HariMain(void)
{
	struct BOOTINFO *binfo = (struct BOOTINFO*) 0x0ff0;
	extern char hankaku[4096];
	init_palette();
	init_screen(binfo->vram, binfo->scrnx,binfo->scrny);
	putfont8(binfo->vram, binfo->scrnx, 8,  8, COL8_FFFFFF, hankaku + 'A' * 16);
	putfont8(binfo->vram, binfo->scrnx, 16, 8, COL8_FFFFFF, hankaku + 'B' * 16);
	putfont8(binfo->vram, binfo->scrnx, 24, 8, COL8_FFFFFF, hankaku + 'C' * 16);
	putfont8(binfo->vram, binfo->scrnx, 40, 8, COL8_FFFFFF, hankaku + '1' * 16);
	putfont8(binfo->vram, binfo->scrnx, 48, 8, COL8_FFFFFF, hankaku + '2' * 16);
	putfont8(binfo->vram, binfo->scrnx, 56, 8, COL8_FFFFFF, hankaku + '3' * 16);
	for (;;) {
		io_hlt();
	}
}
```

##### 5.显示字符串

仅仅显示6个字符，就要写这么多代码，实在不太好看。

所以我们制作一个函数来显示字符串

```c
void putfonts8_asc(char *vram, int xsize, int x, int y, char c, unsigned char *s)
{
	extern char hankaku[4096];
	for (; *s != 0x00; s++) {
    //在这里解释一下变量含义吧
    /*	vram 显存地址 
    	xsize 一行的宽度，用以计算正确的vram，与字符串的位置无关
    	x 输出字符串的位置
    	y 输出字符串的位置
    	c 调色板颜色编号 (char只是为了限制0~255 和字符没关系)
    	s 字符数组（字符串）*/    
		putfont8(vram, xsize, x, y, c, hankaku + *s * 16);
		x += 8;//下个字的位置
	}
	return;
}
```

##### 6.显示变量值

以使用sprintf函数。它是printf函 数的同类，与printf函数的功能很相近。在开始的 时候，我们曾提到过，自制操作系统中不能随便使 用printf函数，但sprintf可以使用。因为sprintf不 是按指定格式输出，只是将输出内容作为字符串写 在内存中。

它在制作者的精心设计之下能够不使 用操作系统的任何功能。

```c
#include <stdio.h>//位于这个头文件中
sprintf(s, "scrnx = %d", binfo->scrnx);	//sprintf（地址，格式，值，值，值，……）
										//可以通过这个将变量'binfo->scrnx'以字符串的形式输出出来
putfonts8_asc(binfo->vram, binfo->scrnx, 16, 64, COL8_FFFFFF, s);
```

##### 7.显示鼠标指针

先画出一个鼠标再说：

```c
void init_mouse_cursor8(char *mouse, char bc)//输入鼠标数组的保存位置 和 背景颜色
/* 准备鼠标指针（16×16） */
{
    static char cursor[16][16] = {
    "**************..",
    "*OOOOOOOOOOO*...",
    "*OOOOOOOOOO*....",
    "*OOOOOOOOO*.....",
    "*OOOOOOOO*......",
    "*OOOOOOO*.......",
    "*OOOOOOO*.......",
    "*OOOOOOOO*......",
    "*OOOO**OOO*.....",
    "*OOO*..*OOO*....",
    "*OO*....*OOO*...",
    "*O*......*OOO*..",
    "**........*OOO*.",
    "*..........*OOO*",
    "............*OO*",
    ".............***"
    };
    int x, y;
    for (y = 0; y < 16; y++) {
    	for (x = 0; x < 16; x++) {
    	if (cursor[y][x] == '*') {
    		mouse[y * 16 + x] = COL8_000000;//黑色
    }
    	if (cursor[y][x] == 'O') {
    		mouse[y * 16 + x] = COL8_FFFFFF;//白色
    }
    	if (cursor[y][x] == '.') {
    		mouse[y * 16 + x] = bc;//bc就是背景颜色 
    }
  }
 }
    return;
}
```

要将背景色显示出来，还需要作成下面这个函数。 其实很简单，只要将buf中的数据复制到vram中去 就可以了。

```c
//vram 显卡内存 vxsize 显示窗口的长度（用来换行用）
//pxzize pysize 显示图形的大小
//px0 py0 指定图形在画面上的显示位置 
//buf和bxsize 分别指定图形的存放地址和每一行含有的像素数
//bxsize 和 pxsize 大体相同，但有时候想放入不同的值 
void putblock8_8(char *vram, int vxsize, int pxsize,int pysize, int px0, int py0, char *buf,int bxsize){
    int x, y;
    for (y = 0; y < pysize; y++) {
    	for (x = 0; x < pxsize; x++) {
    		vram[(py0 + y) * vxsize + (px0 + x)] = buf[y * bxsize + x];
    	}
    }
	return;
}
```

先初始化鼠标

然后将鼠标部署在vram（显存）上

```c
//相当于将鼠标画在图形上 
init_mouse_cursor8(mcursor, COL8_008484);//在mcursor储存鼠标的表示数据
putblock8_8(binfo->vram, binfo->scrnx, 16, 16, mx, my, mcursor, 16);
```

##### 8.GDT和IDT的初始化

###### 分段：

***为什么要分段？***

防止内存的使用范围重叠，造成内存地址冲突。

> 所谓分段，打个比方说，就是按照自己喜欢的方式，将合计4GB的内存分成很多块（block），每一块的起始地址都看作0来处理。这很方便，有了 这个功能，任何程序都可以先写上一句ORG 0。像这样分割出来的块，就称为段（segment）。

> 需要注意的一点是，我们用16位的时候曾经讲解过的段寄存器。这里的分段，使用的就是这个段寄存 器。但是16位的时候，如果计算地址，只要将地址 乘以16就可以了。但现在已经是32位了，不能再 这么用了。如果写成“MOV AL,[DS:EBX]”，CPU 会往EBX里加上某个值来计算地址，这个值不是DS 的16倍，而是DS所表示的段的起始地址。即使省略段寄存器（segment register）的地址，也会自 动认为是指定了DS。这个规则不管是16位模式还 是32位模式，都是一样的。

按照这种分段方法，为了表示一个段，需要有以下信息。

- 段的大小是多少
- 段的起始地址在哪里
- 段的管理属性（禁止写入，禁止执行。系统专用等）

**CPU用8个字节（=64位）的数据来表示这些信息（也称段描述符）。**

尽管 CPU 的数据宽度为 64 位，但段寄存器仍然只有 16 位。

且段寄存器中的低 3 位不能使用，因此实际可用的段号范围为 0 到 8191。

为了克服这个限制，采用了类似图像调色板的方式。

即先将一个 13 位的段号存放在段寄存器中，然后通过预先设定的对应关系来找到实际的段描述符。

就是说用一个段号，来对应一个段描述符。

因为段寄存器能使用13位，所以段寄存器可以储存0~8191范围内的8192个段号，而每个段号又有8byte的数据来表示自身信息，这些8192 * 8 = 65536Byte(64KB)的数据将被存到内存中，被称为GDT。

###### GDT：

GDT是“global（segment）descriptor table”的 缩写，意思是全局段号记录表。

将这些数据整齐地排列在内存的某个地方，然后将内存的起始地址和有效设定个数放在CPU内被称作GDTR的特殊寄存器中，设定就完成了。

###### IDT:

IDT是“interrupt descriptor table”的缩 写，直译过来就是“中断记录表”。

当CPU遇到外部状况变化，或者是内部偶然发生某些错误时，会临时切换过去处理这种突发事件。这就是中断功能。

> 讲了这么长，其实总结来说就是：要使用鼠标，就 必须要使用中断。所以，我们必须设定IDT。IDT记 录了0～255的中断号码与调用函数的对应关系， 比如说发生了123号中断，就调用○×函数，其设定 方法与GDT很相似（或许是因为使用同样的方法能 简化CPU的电路）。 如果段的设定还没顺利完成就设定IDT的话，会比 较麻烦，所以必须先进行GDT的设定。

有种作者也不想讲了，之后翻桌子的感觉ww.

又到了我最喜欢的小插图时间了。

<img src="D:\30daysos\diduox.gihub.io\note\05_day\屏幕截图 2024-01-08 230820.png" style="zoom: 50%;" />

**本次的*bootpack.c节选**

```c
struct SEGMENT_DESCRIPTOR{//存放GDT中8字节的内容
	short limit_low, base_low;
	char base_mid, access_right;
	char limit_high, base_high;
};
struct GATE_DESCRIPTOR {//存放IDT中8字节的内容
	short offset_low, selector;
	char dw_count, access_right;
	short offset_high;
};
void init_gdtidt(void)
{
    //这两个地址只是单纯的找了两个没有被使用过的地址，并没有特定的含义
	struct SEGMENT_DESCRIPTOR *gdt = (struct SEGMENT_DESCRIPTOR *) 0x00270000;
	struct GATE_DESCRIPTOR *idt = (struct GATE_DESCRIPTOR *) 0x0026f800;
	int i;
	/* GDT的初始化 */
	for (i = 0; i < 8192; i++) {//让GDT中每个段的上限，基址，访问权限都设为0
		set_segmdesc(gdt + i, 0, 0, 0);
	}
	set_segmdesc(gdt + 1, 0xffffffff, 0x00000000, 0x4092);//对段1进行设定 它表示的是CPU所能管理的全部内存本身
	set_segmdesc(gdt + 2, 0x0007ffff, 0x00280000, 0x409a);//对段2进行设定，它的大小是512KB，地址是0x280000。这正好是为bootpack.hrb而准备的
	load_gdtr(0xffff, 0x00270000);//通过汇编语言给GDTR赋值，因为C语言里不能进行这个操作
	/* IDT的初始化 */
	for (i = 0; i < 256; i++) {
		set_gatedesc(idt + i, 0, 0, 0);
	}
	load_idtr(0x7ff, 0x0026f800);
	return;
}
void set_segmdesc(struct SEGMENT_DESCRIPTOR *sd, unsigned int limit, int base, int ar)
{
	if (limit > 0xfffff) {
		ar |= 0x8000; /* G_bit = 1 */
		limit /= 0x1000;
	}
	sd->limit_low = limit & 0xffff;
	sd->base_low = base & 0xffff;
	sd->base_mid = (base >> 16) & 0xff;
	sd->access_right = ar & 0xff;
	sd->limit_high = ((limit >> 16) & 0x0f) |
	((ar >> 8) & 0xf0);
	sd->base_high = (base >> 24) & 0xff;
	return;
}
void set_gatedesc(struct GATE_DESCRIPTOR *gd, int offset, int selector, int ar)
{
	gd->offset_low = offset & 0xffff;
	gd->selector = selector;
	gd->dw_count = (ar >> 8) & 0xff;
	gd->access_right = ar & 0xff;
	gd->offset_high = (offset >> 16) & 0xffff;
	return;
}
```



#### 朝花夕拾：

##### 1.为什么有这么多提前规定好的地址？

让ChatGPT来告诉你吧。

> 为了保持员工秩序，公司制定了一份规定：
>
> 规定1：上班时间9点，迟到一次扣50元。 规定2：下班时间6点，早退一次扣50元。 规定3：中午12点吃饭，迟到扣50元。
>
> 一天，小明迟到了，早退了，还中午1点才吃饭。结果工资发现有问题，于是他去找HR。
>
> 小明问：“为什么我的工资被扣了这么多？”
>
> HR回答：“根据规定，你迟到、早退、而且中午迟到吃饭，所以扣了150元。”
>
> 小明：“可是，这都是你们公司的规定啊！”
>
> HR微笑道：“是的，但是你没看到规定4吗？”
>
> 小明一脸疑惑：“哪来的规定4？”
>
> HR拿出规定书，指着规定4说：“规定4：不得质疑公司规定，否则扣全部工资！