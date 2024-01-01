#### 结构体、文字显示与 GDT/IDT初始化

##### 1.接收启动信息

我们试着用指针取得地址，为了当画面模式改变时，系统仍然能正确运行。

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

	binfo_scrny = (short *) 0x0ff6;
	binfo_vram = (int *) 0x0ff8;
	xsize = *binfo_scrnx;
	ysize = *binfo_scrny;
	vram = (char *) *binfo_vram;
```

##### 2.试用结构体

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
	struct BOOTINFO *binfo;
	init_palette();
	binfo = (struct BOOTINFO *) 0x0ff0;
	xsize = (*binfo).scrnx;
	ysize = (*binfo).scrny;
	vram = (*binfo).vram;

```

##### 3.显示字符

字符可以用8×16的长方形 像素点阵来表示。

像这种描画文字形状的数据称为字体（font）数 据，那这种字体数据是怎样写到程序里的呢？有一 种临时方案：

```c
static char font_A[16] = {
0x00, 0x18, 0x18, 0x18, 0x18, 0x24, 0x24,0x24,
0x24, 0x7e, 0x42, 0x42, 0x42, 0xe7, 0x00,0x00
};
```

输出字符的函数：

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

通过指针让代码变得更加精简一些

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

##### 4.显示变量值

以使用sprintf函数。它是printf函 数的同类，与printf函数的功能很相近。在开始的 时候，我们曾提到过，自制操作系统中不能随便使 用printf函数，但sprintf可以使用。因为sprintf不 是按指定格式输出，只是将输出内容作为字符串写 在内存中。

它在制作者的精心设计之下能够不使 用操作系统的任何功能。

```c
sprintf(s, "scrnx = %d", binfo->scrnx);
putfonts8_asc(binfo->vram, binfo->scrnx, 16, 64, COL8_FFFFFF, s);
```

##### 5.显示鼠标指针

先画出一个鼠标再说：

```c
void init_mouse_cursor8(char *mouse, char bc)
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
    		mouse[y * 16 + x] = COL8_000000;
    }
    	if (cursor[y][x] == 'O') {
    		mouse[y * 16 + x] = COL8_FFFFFF;
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
//vram 显卡内存 vxsize 显示窗口的长度（用来换行用） pxzize pysize 显示图形的大小
//px0 py0 指定图形在画面上的显示位置 buf和bxsize 分别指定图形的存放地址和每一行含有的像素数
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
init_mouse_cursor8(mcursor, COL8_008484);//这背景颜色怎么是固定的，鼠标移动改怎么办？
putblock8_8(binfo->vram, binfo->scrnx, 16, 16, mx, my, mcursor, 16);
```

##### 6.GDT和IDT的初始化

GDT是“global（segment）descriptor table”的 缩写，意思是全局段号记录表。将这些数据整齐地 排列在内存的某个地方，然后将内存的起始地址和 有效设定个数放在CPU内被称作GDTR 5的特殊寄存 器中，设定就完成了。

IDT是“interrupt descriptor table”的缩 写，直译过来就是“中断记录表”。当CPU遇到外 部状况变化，或者是内部偶然发生某些错误时，会 临时切换过去处理这种突发事件。这就是中断功 能。