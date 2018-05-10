---
title: scala基础
date: 2018/3/16 08:28:25
category:
- scala
  tag:
- scala
  comments: true  
---

Scala 是一门多范式（multi-paradigm）的编程语言，设计初衷是要集成面向对象编程和函数式编程的各种特性。

Scala 运行在Java虚拟机上，并兼容现有的Java程序。

Scala 源代码被编译成Java字节码，所以它可以运行于JVM之上，并可以调用现有的Java类库。

## Scala 特性

### 面向对象特性

Scala是一种纯面向对象的语言，每个值都是对象。对象的数据类型以及行为由类和特质描述。

类抽象机制的扩展有两种途径：一种途径是子类继承，另一种途径是灵活的混入机制。这两种途径能避免多重继承的种种问题。

### 函数式编程

Scala也是一种函数式语言，其函数也能当成值来使用。Scala提供了轻量级的语法用以定义匿名函数，支持高阶函数，允许嵌套多层函数，并支持柯里化。Scala的case class及其内置的模式匹配相当于函数式编程语言中常用的代数类型。

更进一步，程序员可以利用Scala的模式匹配，编写类似正则表达式的代码处理XML数据。

### 静态类型

Scala具备类型系统，通过编译时检查，保证代码的安全性和一致性。类型系统具体支持以下特性：

- 泛型类
- 协变和逆变
- 标注
- 类型参数的上下限约束
- 把类别和抽象类型作为对象成员
- 复合类型
- 引用自己时显式指定类型
- 视图
- 多态方法

### 扩展性

Scala的设计秉承一项事实，即在实践中，某个领域特定的应用程序开发往往需要特定于该领域的语言扩展。Scala提供了许多独特的语言机制，可以以库的形式轻易无缝添加新的语言结构：

- 任何方法可用作前缀或后缀操作符
- 可以根据预期类型自动构造闭包。

### 并发性

Scala使用Actor作为其并发模型，Actor是类似线程的实体，通过邮箱发收消息。Actor可以复用线程，因此可以在程序中可以使用数百万个Actor,而线程只能创建数千个。在2.10之后的版本中，使用Akka作为其默认Actor实现。

## 数据类型

Scala 与 Java有着相同的数据类型，下表列出了 Scala 支持的数据类型：

| 数据类型 | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Byte     | 8位有符号补码整数。数值区间为 -128 到 127                    |
| Short    | 16位有符号补码整数。数值区间为 -32768 到 32767               |
| Int      | 32位有符号补码整数。数值区间为 -2147483648 到 2147483647     |
| Long     | 64位有符号补码整数。数值区间为 -9223372036854775808 到 9223372036854775807 |
| Float    | 32位IEEE754单精度浮点数                                      |
| Double   | 64位IEEE754单精度浮点数                                      |
| Char     | 16位无符号Unicode字符, 区间值为 U+0000 到 U+FFFF             |
| String   | 字符序列                                                     |
| Boolean  | true或false                                                  |
| Unit     | 表示无值，和其他语言中void等同。用作不返回任何结果的方法的结果类型。Unit只有一个实例值，写成()。 |
| Null     | null 或空引用                                                |
| Nothing  | Nothing类型在Scala的类层级的最低端；它是任何其他类型的子类型。 |
| Any      | Any是所有其他类的超类                                        |
| AnyRef   | AnyRef类是Scala里所有引用类(reference class)的基类           |

上表中列出的数据类型都是对象，也就是说scala没有java中的原生类型。在scala是可以对数字等基础类型调用方法的。

------

Scala 基础字面量

Scala 非常简单且直观。接下来我们会详细介绍 Scala 字面量。

### 整型字面量

整型字面量用于 Int 类型，如果表示 Long，可以在数字后面添加 L 或者小写 l 作为后缀。：

```
0
035
21 
0xFFFFFFFF 
0777L
```

### 浮点型字面量

如果浮点数后面有f或者F后缀时，表示这是一个Float类型，否则就是一个Double类型的。实例如下：

```
0.0 
1e30f 
3.14159f 
1.0e100
.1
```

### 布尔型字面量

布尔型字面量有 true 和 false。

### 符号字面量

符号字面量被写成： **'<标识符>** ，这里 **<标识符>** 可以是任何字母或数字的标识（注意：不能以数字开头）。这种字面量被映射成预定义类scala.Symbol的实例。

如： 符号字面量 

'x

 是表达式 

scala.Symbol("x")

 的简写，符号字面量定义如下：

```
package scala
final case class Symbol private (name: String) {
   override def toString: String = "'" + name
}
```

### 字符字面量

在scala中字符类型表示为半角单引号(')中的字符，如下：

```
'a' 
'\u0041'
'\n'
'\t'
```

其中 **\** 表示转移字符，其后可以跟 **u0041** 数字或者 **\r\n** 等固定的转义字符。

### 字符串字面量

字符串表示方法是在双引号中(") 包含一系列字符，如：

```
"Hello,\nWorld!"
"菜鸟教程官网：www.runoob.com"
```

### 多行字符串的表示方法

多行字符串用三个双引号来表示分隔符，格式为：**""" ... """**。

实例如下：

```scala
val foo = """菜鸟教程
www.runoob.com
www.w3cschool.cc
www.runnoob.com
以上三个地址都能访问"""
```

### Null 值

空值是 scala.Null 类型。

Scala.Null和scala.Nothing是用统一的方式处理Scala面向对象类型系统的某些"边界情况"的特殊类型。

Null类是null引用对象的类型，它是每个引用类（继承自AnyRef的类）的子类。Null不兼容值类型。

### Scala 转义字符

下表列出了常见的转义字符：

| 转义字符 | Unicode | 描述                                |
| -------- | ------- | ----------------------------------- |
| \b       | \u0008  | 退格(BS) ，将当前位置移到前一列     |
| \t       | \u0009  | 水平制表(HT) （跳到下一个TAB位置）  |
| \n       | \u000a  | 换行(LF) ，将当前位置移到下一行开头 |
| \f       | \u000c  | 换页(FF)，将当前位置移到下页开头    |
| \r       | \u000d  | 回车(CR) ，将当前位置移到本行开头   |
| \"       | \u0022  | 代表一个双引号(")字符               |
| \'       | \u0027  | 代表一个单引号（'）字符             |
| \\       | \u005c  | 代表一个反斜线字符 '\'              |

## 变量

- var声明变量
- val声明常量

var VariableName : DataType [=  Initial Value]
或
val VariableName : DataType [=  Initial Value]

``` scala
var myVar : String = "Foo"
var myVar : String = "Too"
val myVal : String = "Foo"
```

在 Scala 中声明变量和常量不一定要指明数据类型，在没有指明数据类型的情况下，其数据类型是通过变量或常量的初始值推断出来的。

所以，如果在没有指明数据类型的情况下声明变量或常量必须要给出其初始值，否则将会报错。

### Scala 多个变量声明

Scala 支持多个变量的声明：

```
val xmax, ymax = 100  // xmax, ymax都声明为100
```

如果方法返回值是元组，我们可以使用 val 来声明一个元组：

```
scala> val pa = (40,"Foo")
pa: (Int, String) = (40,Foo)
```

## Scala 访问修饰符

Scala 访问修饰符基本和Java的一样，分别有：private，protected，public。

如果没有指定访问修饰符符，默认情况下，Scala对象的访问级别都是 public。

Scala 中的 private 限定符，比 Java 更严格，在嵌套类情况下，外层类甚至不能访问被嵌套类的私有成员。

### 私有(Private)成员

用private关键字修饰，带有此标记的成员仅在包含了成员定义的类或对象内部可见，同样的规则还适用内部类。

```scala
class Outer{
    class Inner{
    private def f(){println("f")}
    class InnerMost{
        f() // 正确
        }
    }
    (new Inner).f() //错误
}
```

### 保护(Protected)成员

在 scala 中，对保护（Protected）成员的访问比 java 更严格一些。因为它只允许保护成员在定义了该成员的的类的子类中被访问。而在java中，用protected关键字修饰的成员，除了定义了该成员的类的子类可以访问，同一个包里的其他类也可以进行访问。

```scala
package p{
	class Super{
    	protected def f() {println("f")}
    }
    class Sub extends Super{
        f()
    }
    class Other{
        (new Super).f() //错误
    }
}
```

### 公共(Public)成员

Scala中，如果没有指定任何的修饰符，则默认为 public。这样的成员在任何地方都可以被访问。

### 作用域保护

Scala中，访问修饰符可以通过使用限定词强调。格式为:

```
private[x] 
或 
protected[x]
```

这里的x指代某个所属的包、类或单例对象。如果写成private[x],读作"这个成员除了对[…]中的类或[…]中的包中的类及它们的伴生对像可见外，对其它所有类都是private。==包括子包==

## Scala 函数

Scala 有函数和方法，二者在语义上的区别很小。Scala 方法是类的一部分，而函数是一个对象可以赋值给一个变量。

```
def functionName ([参数列表]) : [return type]
```

如果你不写等于号和方法主体，那么方法会被隐式声明为"抽象(abstract)"，包含它的类型于是也是一个抽象类型。

### 函数定义

```scala
def functionName ([参数列表]) : [return type] = {
   function body
   return [expr]
}
```

以上代码中 **return type** 可以是任意合法的 Scala 数据类型。参数列表中的参数可以使用逗号分隔。

以下函数的功能是将两个传入的参数相加并求和：

```
object add{
   def addInt( a:Int, b:Int ) : Int = {
      var sum:Int = 0
      sum = a + b
      return sum
   }
}
```

如果函数没有返回值，可以返回为 **Unit**，这个类似于 Java 的 **void**, 实例如下：

```
object Hello{
   def printMe( ) : Unit = {
      println("Hello, Scala!")
   }
}
```

### 函数调用

Scala 提供了多种不同的函数调用方式：

以下是调用方法的标准格式：

```
functionName( 参数列表 )
```

如果函数使用了实例的对象来调用，我们可以使用类似java的格式 (使用 **.** 号)：

```
[instance.]functionName( 参数列表 )
```

### 函数类型

1. 函数传名调用(Call-by-Name)

   Scala的解释器在解析函数参数(function arguments)时有两种方式：

   - 传值调用（call-by-value）：先计算参数表达式的值，再应用到函数内部；
   - 传名调用（call-by-name）：将未计算的参数表达式直接应用到函数内部

   ```scala
   object Test {
      def main(args: Array[String]) {
           delayed(time());
      }

      def time() = {
         println("获取时间，单位为纳秒")
         System.nanoTime
      }
      // t: => Long，t为一个函数，参数为空，返回值为Long
      def delayed( t: => Long ) = {
         println("在 delayed 方法内")
         println("参数： " + t)
         t
      }
   }
   ```

2. Scala 指定函数参数名

   通过指定函数参数名，并且不需要按照顺序向函数传递参数，实例如下：

   ```scala
   object Test {
      def main(args: Array[String]) {
           printInt(b=5, a=7); // out：Value of a :  7 \n Value of b :  5
      }
      def printInt( a:Int, b:Int ) = {
         println("Value of a : " + a );
         println("Value of b : " + b );
      }
   }
   ```

3. 可变参数

   ```scala
   def printStrings( args:String* ) = {
         var i : Int = 0;
         for( arg <- args ){
            println("Arg value[" + i + "] = " + arg );
            i = i + 1;
         }
      }
   ```

   > Scala 语言中 for 循环的语法：
   >
   > `for( var x <- Range ){  statement(s);}`
   >
   > `Range` 可以是一个数字区间表示 `i to j` ，或者 `i until j`，前者包括j，后者不包括j
   >
   > 可以使用分号 (;) 来设置多个区间，它将迭代给定区间所有的可能值。`for( a <- 1 to 3; b <- 1 to 3)`

4. 默认参数

   Scala 可以为函数参数指定默认参数值，使用了默认参数，你在调用函数的过程中可以不需要传递参数

   ```scala
   def addInt( a:Int=5, b:Int=7 ) : Int = { var sum:Int = 0
   	sum = a + b
   	return sum
   }
   ```

5. 高阶函数

   高阶函数（Higher-Order Function）就是操作其他函数的函数，可以使用其他函数作为参数，或者使用函数作为输出结果。

   ```scala
   object Test {
      def main(args: Array[String]) {
         println( apply( layout, 10) )
      }
      // 函数 f 和 值 v 作为参数，而函数 f 又调用了参数 v
      def apply(f: Int => String, v: Int) = f(v)
      def layout[A](x: A) = "[" + x.toString() + "]"
   }
   ```

6. 函数嵌套

7. 匿名函数

   箭头左边是参数列表，右边是函数体。

   ```scala
   var inc = (x:Int) => x+1
   var x = inc(7)-1
   ```

8. 偏应用函数

   Scala 偏应用函数是一种表达式，你不需要提供函数需要的所有参数，只需要提供部分，或不提供所需参数。

   例如：下面代码绑定第一个 date 参数，第二个参数使用下划线(_)替换缺失的参数列表，并把这个新的函数值的索引的赋给变量。

   ```scala
   def log(date: Date, message: String)  = {
   	println(date + "----" + message)
   }
   def main(args: Array[String]) {
       val date = new Date
       val logWithDateBound = log(date, _ : String)
       logWithDateBound("message1" )
       logWithDateBound("message3" )
   }
   ```

9. 函数柯里化(Function Currying)

   柯里化(Currying)指的是将原来接受两个参数的函数变成新的接受一个参数的函数的过程。新的函数返回一个以原有第二个参数为参数的函数。

   ```scala
   def add(x:Int,y:Int)=x+y
   // to
   def add(x:Int)(y:Int) = x + y
   ```

## 闭包

闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量。

闭包通常来讲可以简单的认为是可以访问一个函数里面局部变量的另外一个函数。

```scala
var factor = 3  
val multiplier = (i:Int) => i * factor  
```

这里我们引入一个自由变量 factor，这个变量定义在函数外面。

这样定义的函数变量 multiplier 成为一个"闭包"，因为它引用到函数外面定义的变量，定义这个函数的过程是将这个自由变量捕获而构成一个封闭的函数。

## 数组

```scala
var z:Array[String] = new Array[String](3)
var z = new Array[String](3)
var z = Array("Runoob", "Baidu", "Google")

z(0) = "Runoob"; z(1) = "Baidu";

var myList = Array(1.9, 2.9, 3.4, 3.5)
// 输出所有数组元素
for ( x <- myList ) {
    println( x )
}
// 计算数组所有元素的总和
var total = 0.0;
for ( i <- 0 to (myList.length - 1)) {
    total += myList(i);
}
println("总和为 " + total);
// 查找数组中的最大元素
var max = myList(0);
for ( i <- 1 to (myList.length - 1) ) {
    if (myList(i) > max) max = myList(i);
}
println("最大值为 " + max);
```

### 多维数组

```scala
import Array._

object Test {
   def main(args: Array[String]) {
      var myMatrix = ofDim[Int](3,3)
      
      // 创建矩阵
      for (i <- 0 to 2) {
         for ( j <- 0 to 2) {
            myMatrix(i)(j) = j;
         }
      }
      
      // 打印二维阵列
      for (i <- 0 to 2) {
         for ( j <- 0 to 2) {
            print(" " + myMatrix(i)(j));
         }
         println();
      }
    
   }
}
```

### 合并数组

```
var myList3 =  concat( myList1, myList2)
```

### 创建区间数组

以下实例中，我们使用了 range() 方法来生成一个区间范围内的数组。range() 方法最后一个参数为步长，默认为 1，参考python

```
var myList1 = range(10, 20, 2)
var myList2 = range(10,20)
```

## 迭代器

Scala Iterator（迭代器）不是一个集合，它是一种用于访问集合的方法。

迭代器 it 的两个基本操作是 **next** 和 **hasNext**。

调用 **it.next()** 会返回迭代器的下一个元素，并且更新迭代器的状态。

调用 **it.hasNext()** 用于检测集合中是否还有元素。

## Scala Iterator 常用方法

下表列出了 Scala Iterator 常用的方法：

| 序号 | 方法及描述                                                   |
| ---- | ------------------------------------------------------------ |
| 1    | **def hasNext: Boolean**如果还有可返回的元素，返回true。     |
| 2    | **def next(): A**返回迭代器的下一个元素，并且更新迭代器的状态 |
| 3    | **def ++(that: => Iterator[A]): Iterator[A]**合并两个迭代器  |
| 4    | **def ++[B >: A](that :=> GenTraversableOnce[B]): Iterator[B]**合并两个迭代器 |
| 5    | **def addString(b: StringBuilder): StringBuilder**添加一个字符串到 StringBuilder b |
| 6    | **def addString(b: StringBuilder, sep: String): StringBuilder**添加一个字符串到 StringBuilder b，并指定分隔符 |
| 7    | **def buffered: BufferedIterator[A]**迭代器都转换成 BufferedIterator |
| 8    | **def contains(elem: Any): Boolean**检测迭代器中是否包含指定元素 |
| 9    | **def copyToArray(xs: Array[A], start: Int, len: Int): Unit**将迭代器中选定的值传给数组 |
| 10   | **def count(p: (A) => Boolean): Int**返回迭代器元素中满足条件p的元素总数。 |
| 11   | **def drop(n: Int): Iterator[A]**返回丢弃前n个元素新集合     |
| 12   | **def dropWhile(p: (A) => Boolean): Iterator[A]**从左向右丢弃元素，直到条件p不成立 |
| 13   | **def duplicate: (Iterator[A], Iterator[A])**生成两个能分别返回迭代器所有元素的迭代器。 |
| 14   | **def exists(p: (A) => Boolean): Boolean**返回一个布尔值，指明迭代器元素中是否存在满足p的元素。 |
| 15   | **def filter(p: (A) => Boolean): Iterator[A]**返回一个新迭代器 ，指向迭代器元素中所有满足条件p的元素。 |
| 16   | **def filterNot(p: (A) => Boolean): Iterator[A]**返回一个迭代器，指向迭代器元素中不满足条件p的元素。 |
| 17   | **def find(p: (A) => Boolean): Option[A]**返回第一个满足p的元素或None。注意：如果找到满足条件的元素，迭代器会被置于该元素之后；如果没有找到，会被置于终点。 |
| 18   | **def flatMap[B](f: (A) => GenTraversableOnce[B]): Iterator[B]**针对迭代器的序列中的每个元素应用函数f，并返回指向结果序列的迭代器。 |
| 19   | **def forall(p: (A) => Boolean): Boolean**返回一个布尔值，指明 it 所指元素是否都满足p。 |
| 20   | **def foreach(f: (A) => Unit): Unit**在迭代器返回的每个元素上执行指定的程序 f |
| 21   | **def hasDefiniteSize: Boolean**如果迭代器的元素个数有限则返回true（缺省等同于isEmpty） |
| 22   | **def indexOf(elem: B): Int**返回迭代器的元素中index等于x的第一个元素。注意：迭代器会越过这个元素。 |
| 23   | **def indexWhere(p: (A) => Boolean): Int**返回迭代器的元素中下标满足条件p的元素。注意：迭代器会越过这个元素。 |
| 24   | **def isEmpty: Boolean**检查it是否为空, 为空返回 true，否则返回false（与hasNext相反）。 |
| 25   | **def isTraversableAgain: Boolean**Tests whether this Iterator can be repeatedly traversed. |
| 26   | **def length: Int**返回迭代器元素的数量。                    |
| 27   | **def map[B](f: (A) => B): Iterator[B]**将 it 中的每个元素传入函数 f 后的结果生成新的迭代器。 |
| 28   | **def max: A**返回迭代器迭代器元素中最大的元素。             |
| 29   | **def min: A**返回迭代器迭代器元素中最小的元素。             |
| 30   | **def mkString: String**将迭代器所有元素转换成字符串。       |
| 31   | **def mkString(sep: String): String**将迭代器所有元素转换成字符串，并指定分隔符。 |
| 32   | **def nonEmpty: Boolean**检查容器中是否包含元素（相当于 hasNext）。 |
| 33   | **def padTo(len: Int, elem: A): Iterator[A]**首先返回迭代器所有元素，追加拷贝 elem 直到长度达到 len。 |
| 34   | **def patch(from: Int, patchElems: Iterator[B], replaced: Int): Iterator[B]**返回一个新迭代器，其中自第 from 个元素开始的 replaced 个元素被迭代器所指元素替换。 |
| 35   | **def product: A**返回迭代器所指数值型元素的积。             |
| 36   | **def sameElements(that: Iterator[_]): Boolean**判断迭代器和指定的迭代器参数是否依次返回相同元素 |
| 37   | **def seq: Iterator[A]**返回集合的系列视图                   |
| 38   | **def size: Int**返回迭代器的元素数量                        |
| 39   | **def slice(from: Int, until: Int): Iterator[A]**返回一个新的迭代器，指向迭代器所指向的序列中从开始于第 from 个元素、结束于第 until 个元素的片段。 |
| 40   | **def sum: A**返回迭代器所指数值型元素的和                   |
| 41   | **def take(n: Int): Iterator[A]**返回前 n 个元素的新迭代器。 |
| 42   | **def toArray: Array[A]**将迭代器指向的所有元素归入数组并返回。 |
| 43   | **def toBuffer: Buffer[B]**将迭代器指向的所有元素拷贝至缓冲区 Buffer。 |
| 44   | **def toIterable: Iterable[A]**Returns an Iterable containing all elements of this traversable or iterator. This will not terminate for infinite iterators. |
| 45   | **def toIterator: Iterator[A]**把迭代器的所有元素归入一个Iterator容器并返回。 |
| 46   | **def toList: List[A]**把迭代器的所有元素归入列表并返回      |
| 47   | **def toMap[T, U]: Map[T, U]**将迭代器的所有键值对归入一个Map并返回。 |
| 48   | **def toSeq: Seq[A]**将代器的所有元素归入一个Seq容器并返回。 |
| 49   | **def toString(): String**将迭代器转换为字符串               |
| 50   | **def zip[B](that: Iterator[B]): Iterator[(A, B)**返回一个新迭代器，指向分别由迭代器和指定的迭代器 that 元素一一对应而成的二元组序列 |

## 类和对象

类不声明为public，一个Scala源文件中可以有多个类。

## Scala 继承

Scala继承一个基类跟Java很相似, 但我们需要注意以下几点：

- 重写一个非抽象方法必须使用override修饰符。
- 只有主构造函数才可以往基类的构造函数里写参数。
- 在子类中重写超类的抽象方法时，你不需要使用override关键字。
- Scala重写一个非抽象方法，必须用override修饰符。

```scala
class Point(xc: Int, yc: Int) {
   var x: Int = xc
   var y: Int = yc

   def move(dx: Int, dy: Int) {
      x = x + dx
      y = y + dy
      println ("x 的坐标点: " + x);
      println ("y 的坐标点: " + y);
   }
}

class Location(override val xc: Int, override val yc: Int,
   val zc :Int) extends Point(xc, yc){
   var z: Int = zc

   def move(dx: Int, dy: Int, dz: Int) {
      x = x + dx
      y = y + dy
      z = z + dz
      println ("x 的坐标点 : " + x);
      println ("y 的坐标点 : " + y);
      println ("z 的坐标点 : " + z);
   }
}
```

### Scala 单例对象

在 Scala 中，是没有 static 这个东西的，但是它也为我们提供了单例模式的实现方法，那就是使用关键字 object。

Scala 中使用单例模式时，除了定义的类之外，还要定义一个同名的 object 对象，它和类的区别是，object对象不能带参数。

当单例对象与某个类共享同一个名称时，他被称作是这个类的伴生对象：companion object。你必须在同一个源文件里定义类和它的伴生对象。类被称为是这个单例对象的伴生类：companion class。类和它的伴生对象可以互相访问其私有成员。

```scala
// 私有构造方法
class Marker private(val color:String) {

  println("创建" + this)
  
  override def toString(): String = "颜色标记："+ color
  
}

// 伴生对象，与类共享名字，可以访问类的私有属性和方法
object Marker{
  
    private val markers: Map[String, Marker] = Map(
      "red" -> new Marker("red"),
      "blue" -> new Marker("blue"),
      "green" -> new Marker("green")
    )
    
    def apply(color:String) = {
      if(markers.contains(color)) markers(color) else null
    }
  
    
    def getMarker(color:String) = { 
      if(markers.contains(color)) markers(color) else null
    }
    def main(args: Array[String]) { 
        println(Marker("red"))  
        // 单例函数调用，省略了.(点)符号  
        println(Marker getMarker "blue")  
    }
}
```

## Scala Trait(特征)

Scala Trait(特征) 相当于 Java 的接口，实际上它比接口还功能强大。

与接口不同的是，它还可以定义属性和方法的实现。

所以更像抽象类，但可以实现多继承

### 特征构造顺序

特征也可以有构造器，由字段的初始化和其他特征体中的语句构成。这些语句在任何混入该特征的对象在构造时都会被执行。

构造器的执行顺序：

- 调用超类的构造器；
- 特征构造器在超类构造器之后、类构造器之前执行；
- 特征由左到右被构造；
- 每个特征当中，父特征先被构造；
- 如果多个特征共有一个父特征，父特征不会被重复构造
- 所有特征被构造完毕，子类被构造。

构造器的顺序是类的线性化的反向。线性化是描述某个类型的所有超类型的一种技术规格。

## Scala 模式匹配

Scala 提供了强大的模式匹配机制，应用也非常广泛。

一个模式匹配包含了一系列备选项，每个都开始于关键字 **case**。每个备选项都包含了一个模式及一到多个表达式。箭头符号 **=>**隔开了模式和表达式。

```scala
object Test {
   def main(args: Array[String]) {
      println(matchTest("two"))
      println(matchTest("test"))
      println(matchTest(1))
      println(matchTest(6))

   }
   def matchTest(x: Any): Any = x match {
      case 1 => "one"
      case "two" => 2
      case y: Int => "scala.Int"
      case _ => "many"
   }
}
```

### Scala 正则表达式

Scala 通过 scala.util.matching 包中的 **Regex** 类来支持正则表达式。

```scala
 def main(args: Array[String]) {
      val pattern = "Scala".r
      val str = "Scala is Scalable and cool"
      
      println(pattern findFirstIn str)
   }
   
 def main(args: Array[String]) {
      val pattern = new Regex("(S|s)cala")  // 首字母可以是大写 S 或小写 s
      val str = "Scala is scalable and cool"
      
      println((pattern findAllIn str).mkString(","))   // 使用逗号 , 连接返回结果
   }
   
  def main(args: Array[String]) {
      val pattern = "(S|s)cala".r
      val str = "Scala is scalable and cool"
      
      println(pattern replaceFirstIn(str, "Java"))
   }
   
```

## 异常处理

```scala
object Test {
   def main(args: Array[String]) {
      try {
         val f = new FileReader("input.txt")
      } catch {
         case ex: FileNotFoundException => {
            println("Missing file exception")
         }
         case ex: IOException => {
            println("IO Exception")
         }
      } finally {
         println("Exiting finally...")
      }
   }
}
```

## Scala 提取器(Extractor)

提取器是从传递给它的对象中提取出构造该对象的参数。

Scala 提取器是一个带有unapply方法的对象。unapply方法算是apply方法的反向操作：unapply接受一个对象，然后从对象中提取值，提取的值通常是用来构造该对象的值。

```scala
object Test {
   def main(args: Array[String]) {
      
      println ("Apply 方法 : " + apply("Zara", "gmail.com"));
      println ("Unapply 方法 : " + unapply("Zara@gmail.com"));
      println ("Unapply 方法 : " + unapply("Zara Ali"));

   }
   // 注入方法 (可选)
   def apply(user: String, domain: String) = {
      user +"@"+ domain
   }

   // 提取方法（必选）
   def unapply(str: String): Option[(String, String)] = {
      val parts = str split "@"
      if (parts.length == 2){
         Some(parts(0), parts(1)) 
      }else{
         None
      }
   }
}
```























