> ***2021.10.23\***

------

## 3.2 Program Encodings

命令gcc指的是GCC的C编译器。编译选项`-Og`告诉编译器使用符合原始C代码整体结构的机器代码和优化等级。实际从考虑性能考虑，较高级别的优化如`-O1`或`-O2`被认为是较好的选项。

### 3.2.1 Machine-Level Code

对于机器级编程，有两种抽象特别重要：

- 第一种是由指令集体系结构或指令集架构（Instruction Set Architecture，ISA）来定义机器级程序的格式和行为，它定义了处理器状态、指令的格式，以及每条指定对状态的影响。
- 第二种是机器级程序使用的内存地址是虚拟地址，提供的内存模型是看起来非常大的字节数组。

x86-64处理器状态：

- 程序计数器（“PC”，在x86-64中用`%rip`表示）给出将要执行的下一条指定在内存中的地址。
- 整数寄存器包含16个命名位置，分别存储64位的值。可以存储地址（C中的指针）或者整形数据。
- 条件码寄存器保存最近执行的算数或者逻辑指令的状态信息。用来实现控制或者数据流中的条件变化，比如`if`和`while`语句。
- 向量寄存器存放一个或者多个整数或浮点数值。

机器代码只是简单的将内存看成一个很大的、按字节寻址的数组。

程序内存包括：

- 程序的可执行机器代码；
- 操作系统需要的一些信息；
- 用来管理过程调用和返回的运行时栈；
- 以及用户分配的内存块（比如malloc）。

一条机器指令只执行一个非常基本操作。编译器必须产生这些指令的序列，从而实现程序结构。

### 3.2.2 Code Examples

示例代码`mstore.c`，包含如如下函数：

```c
long mult2(long, long);

void multstore(long x, long y, long *dest) {
		long t = mult2(x, y);
		*dest = t;
}
```

在命令行使用“-S”选项，就能看到GCC生成的汇编代码：

```bash
linux> gcc -Og -s mstore.c
```

会产生一个汇编文件`mstore.s`，但不做进一步工作。

汇编代码如下，每个缩进行对应一条机器指令，例如`pushq`：

```scss
multstore:
	pushq   %rbx
	movq    %rdx, %rbx
	call    mult2
	movq    %rax, (%rbx)
	popq    %rbx
	ret
```

如果使用“-c”选项，GCC会编译并汇编代码：

```bash
linux> gcc -Og -x mstore.c
```

会产生目标代码文件中`mstore.o`，为二进制格式。文件中有一段14字节序列，十六进制表示为：

```
53 48 89 d3 e8 00 00 00 00 48 89 03 5b c3
```

就是上面汇编指令对应的目标代码。所以机器指令只是一个字节序列。

在Linux中，`OBJDUMP`带`-d`命名可以作为反汇编器：

```bash
linux> objdump -d mstore.o
```

结果如下：

```scss
1 0000000000000000 <multstore>:
2    0:  53              push   %rbx
3    1:  48 89 d3        mov    %rdx,%rbx
4    4:  e8 00 00 00 00  callq  9 <multstore+0x+9>
5    9:  48 89 03        mov    %rax,(%rbx)
6    c:  5b              pop    %rbx
7    d:  c3              retq
```

机器代码与它的反汇编的表示：

- x86-64的指令长度从1到15个字节不等。常用指令以及操作数较少的指令所需字节少，不常用或者操作数较多的指令所需字节较多。
- 设计指令格式的方式是，从每个给定位置开始，可以将字节唯一解码成机器指令。
- 反汇编器只是基于机器代码文件中的字节序列来确定汇编代码。
- 反汇编器使用的指令命令规则与GCC生成的汇编代码使用有细微差别。

生成实际可执行的代码需要对一组目标代码文件运行链接器，而这组目标代码文件中必须含有一个`main`函数。假设有`mian.c`，可用如下方法生成prog：

```bash
linux> gcc -Og -o prog main.c mstore.c
```

反汇编prog文件：

```bash
linux> objdump -d prog
1 0000000000400540 <multstore>:
2    400540:  53              push   %rbx
3    400541:  48 89 d3        mov    %rdx,%rbx
4    400544:  e8 00 00 00 00  callq  40058b <mult2>
5    400549:  48 89 03        mov    %rax,(%rbx)
6    40054c:  5b              pop    %rbx
7    40054d:  c3              retq
8    40054e:  90              nop
9    40054f:  90              nop
```

与之前比较：

- 地址不同
- 第4行链接器天上了call器指定调用函数mult2的地址
- 多出8、9行，为了是函数代码变为16字节

### 3.2.3 Notes on Formatting

汇编代码中，以‘.‘开头的都是伪指令。

我们在汇编代码中，省略了大部分伪指令，但包含行号和解释性说明。
