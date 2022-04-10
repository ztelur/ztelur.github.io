---
title: Go 结构体函数调用底层实现
tags: Go
abbrlink: 293e04b1
date: 2021-11-01 21:12:00
---

在[《Go 语言嵌入和多态机制对比》](http://remcarpediem.net/article/da48b455/)一文中我们了解了 Go 语言的类型系统特征，下面，我们就来了解一下 Go 语言是如何实现上述类型系统特性的，我们将会深入到 Go 语言运行时和最终机器码层面对 Go 语言的结构体、嵌入、抽象接口实现以及函数动态绑定原理进行了解。

上文已经提及，Go 语言结构体并非 Java 和 C++ 语言中 class 的概念，下面我们来了解一下结构体变量声明和相关函数调用在机器码或汇编层面的体现。我们以下面代码为案例进行分析。

```go
func (u User) addAgeVal(a int32) int32 {
  n := u.Age + a
  return n
 }
 func (u *User) addAgePtr(a int32) int32 {
  n := u.Age + a
  return n
 }

 func main() {
  u := User{ID: 1, Name: "Tom", Age: 23}
  s1 := u.addAgeVal(1)
  s2 := u.addAgePtr(2)
  println(s1 == s2)
 }

```

将上述代码使用如下命令编译成机器码，其中 GOOS 指定目标操作系统，GOARCH 指定 CPU 架构，-S 表示打印机器码，-N 是禁止编译器优化，-l 是禁止内联，本机 Go 版本为 go1.16.4。

```go
GOOS=linux GOARCH=amd64 go tool compile -S -N -l main.go
```

#### 变量声明和初始化

我们首先来看 main 函数中 u 变量的声明和初始化过程。汇编代码较大，下面只截取部分内容展示，具体如下所示。

```
"".main STEXT size=217 args=0x0 locals=0x60 funcid=0x0
  ....
  // 声明并清空 u**变量所需空间**
  0x0029 00041 (main.go:17) MOVQ $0, "".u+64(SP)
  0x0032 00050 (main.go:17) MOVL $0, "".u+80(SP)
  0x003a 00058 (main.go:17) MOVQ $1, "".u+88(SP)
  // 将值1**加载到栈低56字节位，也就是给User的ID赋值** 
  0x003a 00058 (main.go:17) MOVQ $1, "".u+64(SP)
  // 将 Tom 字面量地址加载到栈低64**字节位置**
  0x0043 00067 (main.go:17) LEAQ go.string."Tom"(SB), AX
  0x004a 00074 (main.go:17) MOVQ AX, "".u+72(SP)
  // string**类型不同于其他基础类型，需要将其长度也加载到栈中**
  0x004f 00079 (main.go:17) MOVQ $3, "".u+80(SP)
  // 将23**加载到栈低80字节位，给Age赋值**
  0x0058 00088 (main.go:17) MOVL $23, "".u+88(SP)
```

由上可见，结构体真的就是基础类型变量的集合，并没有额外其他信息的加载，对于类型为 User 的 u 变量的声明并初始化语句，首先将对应的栈内空间清零，然后依次处理三个初始化参数值，并加载到对应的栈空间位置，完成初始化过程。

其中 ID 和 Age 由于是基础类型，所以较为简单，而 Name 字段涉及到 string 类型，稍有区别，String类型的运行时表达，具体如下所示。

```go
type** StringHeader struct {
  Data uintptr
  Len int
}
```

由此可见上述汇编中首先将 Tom 字面量地址加载到栈内空间，Tom 字面量则存储在内存数据段中，给 Data 变量赋值，然后将字面量的长度 3 加载到对应位置，给 Len 变量赋值，具体如下图所示。

![stack1](http://cdn.remcarpediem.net/2021-11-01-123253.png)

SP 代表栈顶指针，而 "".u +64(SP) 代表相对于栈顶偏移 64 字节的位置，u 则是引用地址的别名，也真是变量 u 的名称。如图所示，在栈空间中，并不存在结构体 User，而是由基础类型数值和指针等组成的一段空间，这段空间就代表着结构体 User。

从栈顶向栈底方向依次为占 8 字节的代表 User.ID 的常量值1，占据 16 字节的代表 User.Name 的字符串 Tom 值地址和占据 8 字节的代表 User.Age 的常量 23，其中字符串 Tom 又由 8 字节的 Data 指针和 8 字节的 Len 组成。

上述代码中变量 u 未发生逃逸，所以分配在栈中，如果将变量声明成指针类型并且符合逃逸规则，该结构体就会分配在堆上。

```go
func makeUser() *User {
  u := &User{ID: 1, Name: "Tom", Age: 23}
  return u
 }
```

上述指针变量声明和初始化过程的汇编如下所示。

```
u := &User{ID: 1, Name: "Tom", Age: 23}**
"".makeUser STEXT size=181 args=0x8 locals=0x28 funcid=0x0
  // 将 User 类型加载到栈首，作为 newObject**的参数**
  0x002a 00042 (main.go:29) LEAQ type."".User(SB), AX
  0x0031 00049 (main.go:29) MOVQ AX, (SP)
  0x0035 00053 (main.go:29) CALL runtime.newobject(SB)
  // 将newObject**的返回值也就是User结构体的指针加载到对应寄存器和栈内地址**
  0x003a 00058 (main.go:29) MOVQ 8(SP), AX  
  0x003f 00063 (main.go:29) MOVQ AX, ""..autotmp_2+24(SP)
  // 使用指针，将值1**设置到结构体的id位置，其他字段设置类似**
  0x0044 00068 (main.go:29) MOVQ $1, (AX)
```

可以看出汇编代码会首先将 Cat 结构体的类型指针加载到栈顶，作为参数；然后调用 newObject 函数来在堆上按照 Cat 结构体类型分配对应的空间，并返回空间的起始地址；最后使用该起始地址设置结构体的变量。

分配在堆上的结构体示意图在上一个图的右侧显示。我们可以看到，当结构体分配在栈上时，其内部成员变量会依次排列，占据各自固定的空间；而结构体分配在堆上时，其在栈上只会存在一个指向堆地址的指针，该指针指向结构体在堆上的起始位置。

#### 值接收器函数

下面我们来看一下结构体作为函数接收器如何进行函数调用，包括如何如何传递参数和返回值，如何进行值接收器和指针接收器转换等。上述例子中涉及函数调用的片段如下所示：

```
"".main STEXT size=217 args=0x0 locals=0x60 funcid=0x0
  ....
  // 将User**类型变量u的数值copy到寄存器，然后再copy到对应栈空间**
  0x0060 00096 (main.go:20) MOVQ "".u+64(SP), AX
  0x0065 00101 (main.go:20) MOVQ "".u+72(SP), CX
  0x006a 00106 (main.go:20) MOVQ "".u+80(SP), DX
  0x006f 00111 (main.go:20) MOVQ AX, (SP)
  0x0073 00115 (main.go:20) MOVQ CX, 8(SP)
  0x0078 00120 (main.go:20) MOVQ DX, 16(SP)
  0x007d 00125 (main.go:20) MOVL $23, 24(SP)
  // 声明值为值为1**的int32参数，加载到对应位置** 
  0x0085 00133 (main.go:20)  MOVL $1, 32(SP)
  // 调用对应函数
  0x008d 00141 (main.go:20) CALL "".User.addAgeVal(SB)
  // 从+40(SP)**加载返回值到 AL** 寄存器，并保存到+60(SP)
  0x0092 00146 (main.go:20) MOVL 40(SP), AX
  0x0096 00150 (main.go:20) MOVL AX, "".s1+60(SP)
```

Go 的调用规约要求函数参数和返回值都通过栈来传递，这部分空间由调用方在其栈帧(stack frame)上提供。

- 函数接收器是隐式的第一个函数参数，所以上述代码片段的第一步就是讲变量 u 拷贝到对应的栈空间上，这也正对应了值接收器的拷贝机制；

- 然后第二步则是声明 int32 类型的值为 1 的参数 a 并分配到指定位置；

- 接着是使用 CALL 指令调用 User 的 addAgeVal 函数，CALL 指令会将函数的返回值地址推到栈顶，也就是会存储栈的 +40(SP) 位置上；

- 而最后会将其值加载到 +60(SP) 上，也就是将函数返回值赋值给变量 s1。

  

  下面，我们来看一下被调用函数 addAgeVal 函数的相关机器码表达。

```
"".User.addAgeVal STEXT nosplit size=48 args=0x30 locals=0x10 funcid=0x0
  // 栈增长16**字节，存储当前BP值到8(SP)，然后使用LEAQ计算出新的栈帧指针到BP寄存器**
  0x0000 00000 (main.go:9) SUBQ $16, SP
  0x0004 00004 (main.go:9) MOVQ BP, 8(SP)
  0x0009 00009 (main.go:9) LEAQ 8(SP), BP
  // 清零返回值栈空间
  0x000e 00014 (main.go:9) MOVL $0, "".~r1+64(SP)
  // 将 user**的age变量加载到 AX寄存器，然后将其和参数a进行累加**
  0x0016 00022 (main.go:10) MOVL "".u+48(SP), AX
  0x001a 00026 (main.go:10) ADDL "".a+56(SP), AX
  // 将累计后的值从寄存器加载到 +4(SP)**上，也就是复制给变量 n**
  0x001e 00030 (main.go:10) MOVL AX, "".n+4(SP)
  // 设置返回值
  0x0022 00034 (main.go:11) MOVL AX, "".~r1+64(SP)
  // 恢复 BP 指针，栈缩短16**字节，调用RET指令返回**
  0x0026 00038 (main.go:11) MOVQ 8(SP), BP
  0x002b 00043 (main.go:11) ADDQ $16, SP
  0x002f 00047 (main.go:11) RET
```

`addAgeVal` 函数大致分为四个步骤：

- 使用 SUBQ 指令将 SP 减少 16，代表栈增长 16 字节，因为栈帧是向低位增长，其中 8 个字节用于存储当前的栈帧指针，并使用 LEAQ 计算出新的栈帧指针存到BP中；
- 初始化函数返回值，因为是其类型是 int32，所以将其设置为对应的零值，栈空间地址是 +64(SP)；
- 从 +48(SP) 位置加载函数接收器 User 的变量 Age 到 AX 寄存器，然后将其和函数参数 a 累加，其位置为 +56(SP)
- 将二者的和赋值给变量 n，并且将二者的和保存到返回值所在栈空间，也就是 +64(SP)；
- 从 8(SP) 中取出旧栈帧指针，并且将栈帧缩小 16 字节，并调用 RET 指令返回。

综上，main 函数调用 User 的 `addAgeVal` 函数的过程如下图所示。

![function](http://cdn.remcarpediem.net/2021-11-01-123830.png)     
如上图所示，我们看到在 main 函数执行 call 指令前，为调用函数 `addAgeVal` 的参数和返回值准备好了空间，然后将函数接收器 u 和对应的参数 a 按照顺序拷贝到该空间上，然后预留 +40(SP) 的位置给函数调用的返回值。也正是因为值接收器和函数参数发生拷贝，所以函数内对其修改不会影响原值。

调用 call 指令时，会将指令返回地址压入栈首，然后再执行 addAgeVal 函数的指令，将栈顶增长 16 字节，从而导致函数接收器、参数和返回值的相对于SP的地址发生变化，增加了 16 字节，所以大家会发现 addAgeVal 函数中指令操作的相对地址发生了变化。

#### 指针接收器函数

下面，我们来看调用指针接收器函数 addAgePtr 相关的具体指令，体会它与值接收器函数的区别。

```
"".main STEXT size=241 args=0x0 locals=0x68 funcid=0x0
  // 将接收器u**的起始栈地址加载到栈顶**
  0x009a 00154 (main.go:21) LEAQ "".u+64(SP), AX
  0x009f 00159 (main.go:21) MOVQ AX, (SP)
  // 将参数a**的值1加载到8(SP)**
  0x00a3 00163 (main.go:21) MOVL $2, 8(SP)
  // 调用
  0x00ab 00171 (main.go:21) CALL "".(*User).addAgePtr(SB)

```

可以看到调用 addAgePtr 时不会对接收器 u 进行拷贝，而只是将 u 的起始栈地址加载到栈顶，这其实就相当于传递了指向 u 的指针。然后是设置参数 a 的值，最后使用 CALL 指令调用 addAgePtr 函数。

而 addAgePtr 函数的指令和 addAgeVal 类似，唯一不同的是要使用指针来获取接收器 u 的 Age 变量的值，具体如下所示。

```
"".(*User).addAgePtr STEXT nosplit size=54 args=0x18 locals=0x10 funcid=0x0
  ....
  // 前后指令和addAgePtr**类似**
  // 从+24(SP)**取出u的指针到寄存器**
  0x0016 00022 (main.go:14) MOVQ "".u+24(SP), AX
  // 将u**的age变量加载到集群器，**
  // 24(AX)**就是表示相对于 u** 指针记录的起始位置偏移24**字节，也就是age变量的位置**
  0x001d 00029 (main.go:14) MOVL 24(AX), AX

```

从对应的栈空间取到接收器 u 的指针，也就是其起始地址，从起始地址偏移 24 字节就是接收器 u 的 Age 变量位置。整个流程如下图所示。

![function2](http://cdn.remcarpediem.net/2021-11-01-124351.png)

如上图所示，可以看到指针接收器的函数调用时，只需要将其地址作为默认参数进行传递，所以在函数内的对接收器的修改，都是直接修改在原值上。

此外，调用 addAgePtr 的场景是在值变量上调用指针接收器函数，我们看到编译器将值的地址取出作为接收器参数进行传递，而如果是指针变量调用值接收器函数的话，则会先对指针进行取地址，然后再将指针指向的值数据进行拷贝。

综上，我们了解了 Go 语言中结构器和结构体函数在机器层级方面的底层实现，后续文章我们再继续了解 Go 语言相关特性的底层实现。

[个人博客，欢迎来玩](http://remcarpediem.net/)


![](http://cdn.remcarpediem.net/2020-05-26-144752.png)