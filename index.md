# arch-lab

一个很掉头发的实验，对应了csapp中第四、五章的内容。
首先是实验的前提工作，需要进行一些配置才可以进行实验：
```
sudo apt install tcl tcl-dev tk tk-dev
sed -i "s/tcl8.5/tcl8.6/g" Makefile
sed -i "s/CFLAGS=/CFLAGS=-DUSE_INTERP_RESULT /g" Makefile
cd sim/
make clean; make
```
## Pase A
第一题共分为三个小题，要求我们以Y86-64的汇编代码实现example.c里的函数
```
typedef struct ELE {
    long val;
    struct ELE *next;
} *list_ptr;
```
首先是第一个sum_list函数，其实现的功能是求出链表val域值的总和
```
long sum_list(list_ptr ls)
{
    long val = 0;
    while (ls) {
	  val += ls->val;
	  ls = ls->next;
    }
    return val;
}
```
官网上给出了汇编实现的数据
```
ele1:
		.quad 0x00a
		.quad ele2
ele2:
		.quad 0x0b0
		.quad ele3
ele3:
		.quad 0xc00
		.quad 0
```
起初这个题刚上手的时候一脸懵逼，然后在csapp的第252页找到了一个完整的Y66-64汇编代码  
因为书上的程序是数组求总和，所以只要将其中一些地方进行一下修改就能直接拿来用了  
但是依然花了一个晚上的时间来完成这第一个小题，里面有些以前看汇编代码时没有注意到的点，经过这次编写以后算是做了一次查漏补缺了  
开头".pos 0"告诉了编译器要从地址0处开始产生代码，下一条指令实现了初始化栈的作用，在程序的最后有声明栈是从0x200处开始向下生长  
".align 8"表示8字节对齐，后面就是链表的数据了，分别对应了链表中的val和next。   
后面就是仿照书上进行的修改，其中卡住时间最长的地方应该就是实现"ls = ls->next;"这个地方了。  
next作为一个指针指向的是ele2中val的地址，%rdi+8所得到的是next的地址，进行取址操作后得到的是ele2中val的地址，进入下次循环进行(%rdi)操作时得到的就是eve2中val的值  
如果使用书上的数组操作的代码所得到的是ele2中的val的地址，后面进行相累加的话会得到一个很大的数  
当初这些地方一直没想通，数据结构的课还是要好好的上啊......

```
# Execution begins at address 0
		.pos 0
		irmovq stack, %rsp
		call main
		halt

# Sample linked list
		.align 8
ele1:
		.quad 0x00a
		.quad ele2
ele2:
		.quad 0x0b0
		.quad ele3
ele3:
		.quad 0xc00
		.quad 0

main:
		irmovq ele1, %rdi
		call sum_list
		ret
sum_list:
		xorq %rax, %rax
		jmp test
loop:
		mrmovq (%rdi), %r10
		addq %r10, %rax
		mrmovq 8(%rdi), %rdi
test:
		andq %rdi, %rdi
		jne loop
		ret

# Stack starts here and grows to lower adddresses
	.pos 0x200
stack:
```

![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image0.png)

然后第二小题的函数实现的功能与第一小题的一致，但是唯一的不同点在于其使用了递归的方式
```
long rsum_list(list_ptr ls)
{
    if (!ls)
	return 0;
    else {
	long val = ls->val;
	long rest = rsum_list(ls->next);
	return val + rest;
    }
}
```
嗯...第一次写汇编的递归，脑子经过一阵混乱之后逐渐明白了一切  
于递归而言需要保存上一个函数时变量的值，这里用栈来实现  
在rsum函数中递归的停止条件是当其指向的区域为0时则返回0，这里可以用到汇编中经常使用的一个组合"and je"对其是否为0进行判断跳转  
再往下走就是程序的关键点，与第一小题一样的操作，将ele中对应的val值赋给%r10，然后令其指向下一ele的val值  
然后便是调用rsum_list本身了，于栈上的状态而言其会将call的下一条指令的地址压入栈内，然后进入函数，如此重复直到其满足退出的条件  
此时栈上的布局如下图所示，那么我们在设置test的时候只要照着栈上的状态赋值到正确的位置就可以完成递归在汇编层面的实现

![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image0.png)

```
main:
		irmovq ele1, %rdi
		xorq %rax, %rax	
		call rsum_list
		ret
rsum_list:
		pushq %r10				
		andq %rdi, %rdi
		je test
		mrmovq (%rdi), %r10
		mrmovq 8(%rdi), %rdi
		call rsum_list
		addq %r10, %rax
test:
		popq %r10
		ret
```

![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image0.png)

第一题的最后一个小题，实现copy的功能
```
long copy_block(long *src, long *dest, long len)
{
    long result = 0;
    while (len > 0) {
	long val = *src++;
	*dest++ = val;
	result ^= val;
	len--;
    }
    return result;
}
```
观察函数参数不难发现前两个参数传入的都是数组的首地址，而关于数组的操作在csapp中有源码可以直接仿照那个来  
最后需要完成的功能是将src中的内容复制到dest的内存中，并让返回值result等于src元素的总和  
因为这个题只要照着源函数改而且跟书上的例子太像了所以感觉是这三个里最好做的一个
```
main:
		irmovq $8, %r8
		irmovq $1, %r9
		irmovq src, %rdi
		irmovq dest, %rsi
		irmovq $3, %rdx
		xorq %rax, %rax
		call copy_block1
		ret
copy_block1:
		andq %rdi, %rdi
		jmp test
copy_block2:
		mrmovq (%rdi), %r10
		rmmovq %r10, (%rsi)
		xorq %r10, %rax
		addq %r8, %rdi
		addq %r8, %rsi
		subq %r9, %rdx
test:
		jne copy_block2
		ret
```

![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image0.png)

## Pase B
修改在/sim/seq目录下的seq-full.hcl文件，使其支持iaddq功能。在书上第254页的练习题4.3中有给出iaddq的格式，如下图所示  

![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image0.png)

取指(fetch), 译码(decode), 执行(execute), 访存(memory), 写回(write back), 更新PC(PC update)  

icode(指令代码), ifun(指令功能), rA rB(表示是什么寄存器), valC(4字节常数), valP(指令地址)  
valA valB(寄存器中的值), ALU(算术逻辑单元), valM(从内存中读取出的值)  
对照着irmovq可以写出如下内容
```
fetch:
	icode: ifun <- M1[PC]
	rA:rB <- M1[PC+1]
	valC <- M8[PC+2]
	valP <- PC+10
decode:
	valB<-R[rB]
execute:
	ValE<-ValB+ValC
memory:
write back:
	R[rB]<-ValE
PC update:
	PC <- valP
```
然后在文档中修改对应位置的数值，感觉很像做英语的选词填空题
### Fetch
```
bool instr_valid = icode in 
	{ INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
	       IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, IIADDQ };

# Does fetched instruction require a regid byte?
bool need_regids =
	icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ, 
		     IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ };

# Does fetched instruction require a constant word?
bool need_valC =
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, IIADDQ };
```
### Decode
```
## What register should be used as the B source?
word srcB = [
	icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ  } : rB;
	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
	1 : RNONE;  # Don't need register
];
## What register should be used as the E destination?
word dstE = [
	icode in { IRRMOVQ } && Cnd : rB;
	icode in { IIRMOVQ, IOPQ, IIADDQ} : rB;
	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
	1 : RNONE;  # Don't write any register
];
```
### Execute
```
## Select input A to ALU
word aluA = [
	icode in { IRRMOVQ, IOPQ } : valA;
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ } : valC;
	icode in { ICALL, IPUSHQ } : -8;
	icode in { IRET, IPOPQ } : 8;
	# Other instructions don't need ALU
];

## Select input B to ALU
word aluB = [
	icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL, 
		      IPUSHQ, IRET, IPOPQ, IIADDQ } : valB;
	icode in { IRRMOVQ, IIRMOVQ } : 0;
	# Other instructions don't need ALU
];
## Should the condition codes be updated?
bool set_cc = icode in { IOPQ, IIADDQ };
```
然后在编译的时候一直失败，后来找到了解决的办法  
在ssim.c中注释掉下面两行代码  
```
extern int matherr();
int *tclDummyMathPtr = (int *) matherr;
```
然后就可以很丝滑的编译成功并测试了
```
make VERSION=full
./ssim -t ../y86-code/asumi.yo
```

![avatar](https://github.com/AmaIIl/attacklab/blob/gh-pages/image0.png)

## Pase C
