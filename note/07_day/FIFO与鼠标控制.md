#### FIFO与鼠标控制

PS:第六章真是太恶心了 我们直接把它当成已经写好的组件来用

##### 1.获取按键编码

int.c节选（此处的int中断处理程序的缩写）

```c
#define PORT_KEYDAT 0x0060//键盘端口数据的地址
void inthandler21(int *esp)//esp是附加段寄存器 这里用来表示指向栈顶的指针
{
	struct BOOTINFO *binfo = (struct BOOTINFO*) ADR_BOOTINFO;
	unsigned char data, s[4];
	io_out8(PIC0_OCW2, 0x61); /* 这句话用来通知PIC“已经知道发生了IRQ1中断
								“0x60+IRQ号码 为中断的表示
								IRQ代表"Interrupt Request"（中断请求）
								总的来说，IRQ1 是键盘中断所使用的中断请求线。
								*/
	data = io_in8(PORT_KEYDAT);//从键盘读入一个字节的数据
	sprintf(s, "%02X", data);
	boxfill8(binfo->vram, binfo->scrnx,COL8_008484, 0, 16, 15, 31);//绘制一个与背景一个颜色的矩形
	putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);//用白色输出信息
	return;
}
```

##### 2.加快中断处理

本次显示字符的操作是在键盘的中断处理中完成的，众所周知IO是一个及其耗时的过程，所以一个中断处理过长时间可能会影响其他中断的处理。所以我们在中断中，把要输出的变量先保存下来，之后再去处理。

int.c节选

```c
struct KEYBUF {//定义一个缓冲区结构体
	unsigned char data, flag;
    //一个代表缓冲区的里的数据 一个代表缓冲区里有没有数据
};
#define PORT_KEYDAT 0x0060
struct KEYBUF keybuf;
void inthandler21(int *esp)
{
    unsigned char data;
        io_out8(PIC0_OCW2, 0x61); /* 通知PIC IRQ01已经受理完毕 */
        data = io_in8(PORT_KEYDAT);
        if (keybuf.flag == 0) {
            keybuf.data = data;
            keybuf.flag = 1;
        }
    return;
}
```

bootpack.c中HariMain函数的节选

```c
for (;;) {
    io_cli();//Clear Interrupts 禁用中断
    if (keybuf.flag == 0) {
    	io_stihlt();//"Set Interrupts and Halt 恢复中断并让处理器进入休眠状态，等待下一个中断。
        			//这里为什么要将io_sti() 和 io_hlt()写在一起呢
        			//为了原子化 防止在恢复中断之后立刻又产生新的中断,之后就直接休眠了,此时buf；里的数据就无法被处理了
    } else {
        i = keybuf.data;
        keybuf.flag = 0;
        io_sti();
        sprintf(s, "%02X", i);
        boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
        putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
    }
}
```

##### 3.制造FIFO缓冲区

> 当按下右Ctrl键时，会产生两个 字节的键码值“E0 1D”，而松开这个键之后，会 产生两个字节的键码值“E0 9D”。在一次产生两 个字节键码值的情况下，因为键盘内部电路一次只 能发送一个字节，所以一次按键就会产生两次中 断，第一次中断时发送E0，第二次中断时发送 1D。

在键盘上有的按键在按下或抬起之后会发送两个键码值而我们的缓冲区只能储存一个数据。

（终于要开始使用数据结构了ww）

int.c节选

```c
struct KEYBUF {
    unsigned char data[32];
    int next;
};
void inthandler21(int *esp)
{
    unsigned char data;
    io_out8(PIC0_OCW2, 0x61); /* 通知PIC IRQ01已经受理完毕 */
    data = io_in8(PORT_KEYDAT);
    if (keybuf.next < 32) {//next储存下一数据应该放在哪里
    keybuf.data[keybuf.next] = data;
    keybuf.next++;
}
return;
}

```

```c
for (;;) {
    io_cli();
    if (keybuf.next == 0) {
    	io_stihlt();
    } else {
    	i = keybuf.data[0];
    	keybuf.next--;//取出一个数据 next后移一格
    	for (j = 0; j < keybuf.next; j++) {//将所有数据向后移动一格
   	 		keybuf.data[j] = keybuf.data[j +1];
		}
    io_sti();
    sprintf(s, "%02X", i);
    boxfill8(binfo->vram, binfo->scrnx, OL8_008484, 0, 16, 15, 31);
    putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
}
```

##### 4.循环队列

这不是我们数据结构的循环队列吗，下次记得表明出处。

读

```c
void inthandler21(int *esp)
{
    unsigned char data;
    io_out8(PIC0_OCW2, 0x61); /* 通知 IRQ-01已经受理完毕 */
    data = io_in8(PORT_KEYDAT);
    if (keybuf.len < 32) {//其实也可以用指针是否重叠来判断满不满的ww
    	keybuf.data[keybuf.next_w] = data;
    	keybuf.len++;
    	keybuf.next_w++;
    if (keybuf.next_w == 32) {//循环
    	keybuf.next_w = 0;
	}
}
return;
}
```

取

```c
for (;;) {
    io_cli();
    if (keybuf.len == 0) {
    	io_stihlt();
    } else {
    	i = keybuf.data[keybuf.next_r];
    	keybuf.len--;
    	keybuf.next_r++;
    if (keybuf.next_r == 32) {//循环
    	keybuf.next_r = 0;
	}
    io_sti();
    sprintf(s, "%02X", i);
    boxfill8(binfo->vram, binfo->scrnx,COL8_008484, 0, 16, 15, 31);
    putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
}
}

```

#### 5.整理FIFO缓冲区

接下来是纯纯的顺序表了 数据结构人狂喜 好久没有这么轻松了 真是青青又松松啊

结构体

```c
/*
buf* 代表一个可变数组
p 代表下一个写入地址 q 代表下一个读入地址 size 代表数组的大小 free 代表还有多少空间能用
flags 代表此数组有没有发生过溢出
*/
struct FIFO8 {
	unsigned char *buf;
	int p, q, size, free, flags;
};
```

初始化函数

```c
void fifo8_init(struct FIFO8 *fifo, int size,
unsigned char *buf)
/* 初始化FIFO缓冲区 */
{
    fifo->size = size;
    fifo->buf = buf;
    fifo->free = size; /* 缓冲区的大小 */
    fifo->flags = 0;
    fifo->p = 0; /* 下一个数据写入位置 */
    fifo->q = 0; /* 下一个数据读出位置 */
    return;
}
```

put方法

```c
#define FLAGS_OVERRUN 0x0001
int fifo8_put(struct FIFO8 *fifo, unsigned chardata)
/* 向FIFO传送数据并保存 */
{
    if (fifo->free == 0) {/* 空余没有了，溢出 */
    	fifo->flags |= FLAGS_OVERRUN;
    	return -1;
    }
    fifo->buf[fifo->p] = data;
    fifo->p++;
    if (fifo->p == fifo->size) {//队列循环
    	fifo->p = 0;
    }
    fifo->free--;
    return 0;
}

```

get方法

```c
int fifo8_get(struct FIFO8 *fifo)
/* 从FIFO取得一个数据 */
{
    int data;
    if (fifo->free == fifo->size) {/* 如果缓冲区为空，则返回 -1 */
    	return -1;
    }
    data = fifo->buf[fifo->q];
    fifo->q++;
    if (fifo->q == fifo->size) {///队列循环
    	fifo->q = 0;
    }
    fifo->free++;
    return data;
}
```

status方法

```c
int fifo8_status(struct FIFO8 *fifo)
/* 报告一下到底积攒了多少数据 */
{
	return fifo->size - fifo->free;
}

```

中断内容

```c
struct FIFO8 keyfifo;
void inthandler21(int *esp)
{
    unsigned char data;
    io_out8(PIC0_OCW2, 0x61); /* 通知PIC，说IRQ-01的受理已经完成 */
    data = io_in8(PORT_KEYDAT);
    fifo8_put(&keyfifo, data);
    return;
}
```

Main函数

```c
char s[40], mcursor[256], keybuf[32];
fifo8_init(&keyfifo, 32, keybuf);
for (;;) {
    io_cli();
    if (fifo8_status(&keyfifo) == 0) {
    	io_stihlt();
    } else {
    	i = fifo8_get(&keyfifo);
    	io_sti();
    	sprintf(s, "%02X", i);
    	boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
    	putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
	}
}
```

##### 6.鼠标

> 为什么鼠标从来不去上网冲浪？
>
> 因为它们怕沉浸在网上的“陷阱”中！ - chatGPT3.5

从计算机不算短暂的历史来看，鼠标这种装置属于 新兴一族。早期的计算机一般都不配置鼠标。一个 很明显的证据就是，现在我们要讲的分配给鼠标的 中断号码，是IRQ12，这已经是一个很大的数字 了。与键盘的IRQ1比起来，那可差了好多代了。

事实上，鼠标控制电 路包含在键盘控制电路里，如果键盘控制电路的初 始化正常完成，鼠标电路控制器的激活也就完成 了。

bootpack.c节选

```c
#define PORT_KEYDAT 0x0060 //键盘数据端口
#define PORT_KEYSTA 0x0064 //键盘状态端口
#define PORT_KEYCMD 0x0064 //键盘命令端口
#define KEYSTA_SEND_NOTREADY 0x02 //用来查询键盘状态是否准备好了
#define KEYCMD_WRITE_MODE 0x60 //向键盘发送写命令的常量
#define KBC_MODE 0x47 //键盘控制电路的模式设定值 0x47代表鼠标模式
void wait_KBC_sendready(void)
{
    /* 等待键盘控制电路准备完毕 */
    /*如果 键盘控制电路可以接受CPU指令了，CPU从设备号码0x0064处所读取的数据的倒数第二位（从低位 开始数的第二位）应该是0。在确认到这一位是0之 前，程序一直通过for语句循环查询*/
    for (;;) {
    	if ((io_in8(PORT_KEYSTA) & KEYSTA_SEND_NOTREADY) == 0) {
    		break;
    	}	
    }
    return;
}
void init_keyboard(void)
{
    /* 初始化键盘控制电路 */
    wait_KBC_sendready();
    io_out8(PORT_KEYCMD, KEYCMD_WRITE_MODE);//向命令端口发送写模式命令
    wait_KBC_sendready();
    io_out8(PORT_KEYDAT, KBC_MODE); //向数据端口发送键盘控制电路的模式设定值
    return;
}
```

bookpack.c

```c
#define KEYCMD_SENDTO_MOUSE 0xd4
#define MOUSECMD_ENABLE 0xf4
void enable_mouse(void)
{
    /* 激活鼠标 */
    wait_KBC_sendready();
    io_out8(PORT_KEYCMD, KEYCMD_SENDTO_MOUSE);
    wait_KBC_sendready();
    io_out8(PORT_KEYDAT, MOUSECMD_ENABLE);
    return; /* 顺利的话,键盘控制其会返送回ACK(0xfa)*/
}
```

##### 7.从鼠标接收数据

int.c

```c
struct FIFO8 mousefifo;
void inthandler2c(int *esp)
/* 来自PS/2鼠标的中断 */
{
    unsigned char data;
    io_out8(PIC1_OCW2, 0x64); /* 通知PIC1IRQ-12的受理已经完成 
    							分配给鼠标的 中断号码，是IRQ12*/
    io_out8(PIC0_OCW2, 0x62); /* 通知PIC0IRQ-02的受理已经完成 
    							分配给从PIC的 中断号码是 IRQ2*/
    data = io_in8(PORT_KEYDAT);
    fifo8_put(&mousefifo, data);
    return;
}
```

这是因为主/从PIC的协调不能够自动完 成，如果程序不教给主PIC该怎么做，它就会忽视 从PIC的下一个中断请求。

至于传到这个设备的数据，究竟是来自键盘还是鼠 标，要靠中断号码来区分。

```c
fifo8_init(&mousefifo, 128, mousebuf);
for (;;) {
    io_cli();
    if (fifo8_status(&keyfifo) + fifo8_status(&mousefifo) == 0) {//如果鼠标和键盘的缓冲区都为0 那就HLT
        io_stihlt();
} else {
if (fifo8_status(&keyfifo) != 0) {//如果键盘不为0
    i = fifo8_get(&keyfifo);
    io_sti();
    sprintf(s, "%02X", i);
    boxfill8(binfo->vram, binfo->scrnx,COL8_008484, 0, 16, 15, 31);
    putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
} else if (fifo8_status(&mousefifo) != 0) {//如果鼠标不为0
    i = fifo8_get(&mousefifo);
    io_sti();
    sprintf(s, "%02X", i);
    boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 32, 16, 47, 31);
    putfonts8_asc(binfo->vram, binfo->scrnx, 32, 16, COL8_FFFFFF, s);
}
}
}
```

