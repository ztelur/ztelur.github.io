---
title: Go 语言嵌入和多态机制对比
tags: Go
abbrlink: da48b455
date: 2021-09-22 22:24:00
---
### 简介

面向对象语言设计最早可以追溯到 SIMULA 67，但直到 1980 年的 Smalltalk80 将其完善，并随着 Java 的崛起而全面流行起来。面向对象设计的语言大多都支持三个关键的语言特性：抽象数据类型、继承以及多态(方法动态派发)。

本文将以 Go 语言为主，讲解一下 Go 语言在面向对象领域的一些特性以及高级编程语言的一些通用领域知识。

### 继承

在面向对象语言中，继承是代码复用和部分修改的常见手段之一。通过继承，子类型可以获得父类型的属性和行为，并且子类型的实例可以被当做父类的实例使用。



继承一般分为基于原型的继承（prototype-based inheritance）和基于类的继承（class-based inheritance），Javascript语言就实现了基于原型的继承，而其他场景后端语言多是实现了基于类的继承。



Go 语言并未实现上述两种意义上的继承，而是提供嵌入机制。**嵌入可以理解为一种组合或者代理模式的自动语法糖。**

```
type Animal struct {
   Name string
 }
 
 func (a *Animal) Eat() {
   fmt.Printf("%v is eating", a.Name)
   fmt.Println()
 }
 
 type Cat struct { 
   *Animal // 只有类型，并不定义成员变量名，表明是嵌入
 }
```



如上代码所示，Animal 类型实现了 Eat 函数，而在 Cat 类型的定义中，嵌入了 Animal 类型。上述代码基本上等同于下面这段使用组合形式创建的 Cat 结构体



```
type Cat struct {
    Animal *Animal
}
func (c *Cat) Eat() {
    c.Animal.Eat()
}

cat := &Cat{
    Animal: &Animal{
        Name: "cat",
    },
}
```



被嵌入的 Animal 类型被称为内部类型，而 Cat 被称为外部类型。外部类型会持有一个内部类型的实例，并对外提供所有内部类型的函数，而这些函数的内部实现则是直接调用了内部类型实例的对应函数。



下面代码展示了外部类型初始化和调用其对应函数。

```
cat := &Cat{
   Animal: &Animal{
     Name: "cat",
   },
 }
 cat.Eat()
```



可见，初始化 Cat 类型需要传入一个 Animal 类型，并且可以直接调用其 Eat 函数。不过需要注意的是，不同与一般的继承机制，Cat 并不是 Animal 的子类型，所以并不能作为 Animal 类型使用。



```
cat := &Cat{
   Animal: &Animal{
     Name: "cat",
   },
 }
handleAnimalVal(cat) // error Cat is not Animal
handleAnimalPrt(&val) // error *Cat is not *Animal

func handleAnimalVal(a Animal) {}
func handleAnimalPtr(a *Animal) {}
```



而在基于类的继承中，子类是可以充当父类的实例使用的，比如 Java 通过 extend 关键字来实现继承，继承类被称为子类，被继承类被称为父类或超类，凡是接收父类的地方都可以使用子类。在 Go 语言中没有父子类替换机制。



此外，继承并不是面向对象语言专属的概念，C 语言早在面向对象语言发明之前就提供了类似的机制来实现将数据结构伪装成另一种数据结构的特性，具体如下代码所示。



```
// 省略头文件
struct Animal {
  char* name;
} 
  
void eat(struct Animal* cat) {
  printf("animal %v is eating", cat.Name)
}
struct Cat {
  char* name;
  int coat_color;
}
 
struct Cat* makeCat(char* name) {
  struct Cat* cat = malloc(sizeof(struct Cat));
  cat->name = name;
  return cat;
}
 
void eat(struct Cat* cat) {
   printf("cat %v is eating", cat.Name)
}
 
int main(int ac, char** av) {
  struct Animal * animal = (struct Animal*) makeCat("Tom");
  eat(animal);
}
```



如上述代码的 main 函数实现所示，通过指针将 Cat 被伪装成 Animal 来使用，是因为 Cat 是 Animal 结构体的一个超集，二者共同拥有成员变量的顺序也是一样的，二者的结构体如下图所示。

![img](http://cdn.remcarpediem.net/2021-09-22-142538.png)


使用 Animal 指针指向一个 Cat 类型的结构体，并且可以将作为参数传递给 eat 函数，此时会调用以 Animal 类型指针为参数的 eat 函数，而不是以 Cat 类型指针为参数的 eat 函数。



同时需要注意的是，在 C 语言例子中，开发者必须强制将 Cat 向上转型为 Animal，而在真正的面向对象编程语言中，这种类型的向上转换通常是隐式的，语言运行时或者编译器为我们自动做了类型转换。

### 多态



在编程语言和类型系统中，多态（Polymorphism） 能为不同数据类型的实体提供统一的接口，或使用一个单一的符号来表示多个不同的类型。



也就是说，多态能够允许同一段代码在不同上下文中拥有不同的类型，进行不同的实现绑定，从而在不影响类型检查的情况下，为不同类型编写通用的代码。如下代码就体现了多态。



```
type IO interface {
  read() []byte
}

func getChars(io IO) []byte {
  return io.read()
}
```



getChars 函数接收 IO 接口作为参数，然后调用其 read 函数读取数据。不同的 IO 其 read 函数是不同的，比如说从标准输入读取和从文件读取。所以，传入不同的 IO 接口实现类，则会调用其不同的 read 函数实现，也就是read 函数绑定了不同的实现，也就是所谓的多态。



**不同的语言有着不同的多态实现方式，目前常见的多态实现方式一共有三类，分别是：参数多态 ( Parametric Polymorphism )、特定多态 ( Ad-hoc Polymorphism )和子类型多态 ( Subtype Polymorphism )。**



类似于类型系统，按照代码进行绑定的时间，多态可以分为静态多态（static polymorphism）和动态多态（Dynamic Polymorphism）。静态多态的代码绑定发生在编译期，而动态多态的代码绑定发生在运行时。静多态牺牲灵活性获取性能，是零成本抽象，而动多态牺牲性能获取灵活性。



动多态在运行时需要额外读取类信息等数据，花费更多时间，并占用较多空间，所以一般情况下都使用静多态。上述三种多态实现方式中，参数化多态和特定多态一般是静多态，子类型多态一般是动态多态。



Go 语言只支持子类型多态(1.18版本将支持参数多态)，Rust 语言支持参数多态和特定多态，而 Java 语言则支持参数多态和子类型多态。

#### 参数化多态



参数化多态实际上是指定义复合类型的成员变量和函数的参数时不指定其具体的类型，而是在真正使用时将其类型作为参数传入，从而使得复合类型和函数对各种具体类型都适用，从而避免大量重复性的工作，多用于队列，堆栈等容器类型和通用算法函数。



参数化多态是泛型 (generic programming) 的一种实现方式，Go 语言预计在 1.18 版本引入参数化多态实现泛型编程，从而将一直被人所诟病的缺乏泛型编程导致代码重复的短板补齐。下述代码就是 Go 语言将来参数化多态的表现。



```
type Vector[T any] []T
// 示例代码
var v Vector[int]
```



如上代码所示，定义 Vector 类型时声明了一个类型参数T，并标明它可以是任意类型( any 关键字)，然后在真正初始化 Vector 类型变量时，传入类型 int，标明其实际上是一个 int 类型数组。

#### 特定多态

特定多态是针对函数和操作符重载等特定问题的多态实现方案。它不像参数化泛型一样，并不是一种通用多态方案，也不是编程语言类型系统的基础特性。



函数重载（overloading）指的是多个函数拥有相同的名称，但是拥有不同的参数和实现。而操作符重载类似于函数重载，针对不同的参数具有不同的实现。Go 语言中只有参数不同的函数会被判定为命名重复，自然无法支持函数重载和特定多态。Java 代码为例讲述函数重载和操作符重载。



```
public void Add(String a, String b) {
  System.out.print(a + b);
}
 
public void Add(int a, int b) {
  System.out.print(a + b);
}
 
Add(1,2) // print 3
Add("1","2") // print 12
```



上述代码分别定义了参数为 string 和 int 类型的 Add 函数，在实际调用时，会根据传入的参数，调用不同的 Add函数实现。



子类型多态是指一种父子类型的包含关系，子类型可以替代父类型作为参数进行传递，当调用父类型函数时，运行时会根据调用对象的实际类型来调用不同的函数实现。

#### 子类型多态

子类型多态多存在于 Java 等面向对象语言中，Go 因其 Structural Typing 类型系统也可以实现子类型多态。



在 Go 语言中， 如果类型实现了接口定义的所有函数，该类型就被认为实现了该接口。当然，鸭子类型系统并不能精确的表达语言抽象数据类型的全部特性，因为鸭子类型系统一般不进行静态类型检查，而语言会在编译期进行类型检查，所以语言的创造者们**更喜欢用结构类型（Structural typing）一词来表述语言抽象类型系统。**



需要注意的是，这里的子类型和继承并不是同一个概念，子类型反应的是类型(接口)实现的关系，而继承则是两个对象之间的关系，所以 Go 语言并没有继承特性也能实现子类型多态。



```
type File struct {}

func (f File) read() []byte { .... } 

func main() {
  f := File{}
  getChars(f)
}
```



如上代码所示，File 实现了 IO 类型的 read 函数，从而被认为是 IO 类型的子类型，所以就可以将类型为 File 的变量f传入参数为 IO 类型的 getChars 函数中。getChars 函数中会调用参数的 read 方法，Go语言运行时会根据参数的实际类型，进行函数绑定，调用 File 类型的 read 函数。这也体现了子类型多态属于动态多态，因为上述函数绑定发生在运行时。



C语言也可以实现类似多态的代码机制，了解其具体实现方式有利于我们对多态和接口实现的本质有更好地理解。



Linux中的驱动IO设备正是使用了这一机制，每个 IO 设备都提供 open、close、read、write和seek 五个函数，在其他语言中可以将其定义为接口或抽象类，而在C语言中的定义如下所示。



```
struct FILE {
  void (open)(char name, int mode);
  void (close)();
  int (read)();
  void (write)(char);
  void (seek)(long index, int mod);
 }
```



FILE 结构体中有五个函数指针类型成员变量，分别对应上述五个函数。而不同的IO设备代码都需要各自实现自己版本的五个函数，并且将 FILE 结构体的函数指针变量指向对应的实现函数。



如下代码所示，声明了 FILE 类型的 console 变量，将对应的五个函数的指针传入结构体中，作为其成员变量。



```
#include "file.h"
 
void open(char* name, int mode) { /*...*/}
void close() {/*...*/}
int read() {int c; /*...*/ return c;}
void write(char c) {/*...*/}
void seek(long index, int mode) {/*...*/}
 
struct FILE console = {open, close, read, write, seek};
```



最后，getchar 函数接收 FILE 类型的数据作为参数，然后通过结构体的函数指针，调用对应的函数，传入的 FILE 类型的数据不同，则函数指针不同，也就调动了不同的函数实现，从而展示了多态能力。



```
extern struct FILE* STDIN;
 
int getchar() {
  return STDIN.read();
}
```



C 语言的多态能力也在 Redis 的 dict 实现中有所体现，Redis 中很多数据结构都是依赖哈希表 dictType 实现的，所以其定义了 dictType 结构体，其成员变量都是所需函数的函数指针。



```
// 定义哈希字典类型，及其所需的函数指针
typedef struct dictType {
  uint64_t (*hashFunction)(const void *key);
  void *(*keyDup)(void *privdata, const void *key);
  void *(*valDup)(void *privdata, const void *obj);
  int (*keyCompare)(void *privdata, const void *key1, const void *key2);
  void (*keyDestructor)(void *privdata, void *key);
  void (*valDestructor)(void *privdata, void *obj);
} dictType;
```



然后其具体数据结构诸如 Set 则需要实现自己版本的函数，并将其函数指针填充到对应的参数上。



```
// 具体实现，定义 Set 类型，
dictType setDictType = {
  dictSdsHash,        /* hash function */
  NULL,           /* key dup */
  NULL,           /* val dup */
  dictSdsKeyCompare,     /* key compare */
  dictSdsDestructor,     /* key destructor */
  NULL            /* val destructor */
};
```



通过这两个 C 语言的案例，我们可以发现多态是函数指针的一种应用，C 语言可以使用函数指针来模拟多态，而面向编程语言将危险的函数指针隐藏掉，内化成语言本身的特性，提供了更加安全和方便的多态实现机制。

[个人博客，欢迎来玩](http://remcarpediem.net/)


![](http://cdn.remcarpediem.net/2020-05-26-144752.png)