---
title: python面向对象
date: 2018/3/16 08:28:25
category:
- python
- 基础
tag:
- python
comments: true  
---

# 1 面向对象编程

- 类和实例

  ```
  class Student(object):

      def __init__(self, name, score):
          self.name = name
          self.score = scor
  ```

  有了`__init__`方法，在创建实例的时候，就不能传入空的参数了，必须传入与`__init__`方法匹配的参数，但`self`不需要传，Python解释器自己会把实例变量传进去

- 数据封装

- 访问限制

  实例的变量名如果以`__`开头，就变成了一个私有变量（private）

  需要注意的是，在Python中，变量名类似`__xxx__`的，也就是以双下划线开头，并且以双下划线结尾的，是特殊变量，特殊变量是可以直接访问的，不是private变量，所以，不能用`__name__`、`__score__`这样的变量名。

  双下划线开头的实例变量是不是一定不能从外部访问呢？其实也不是。不能直接访问`__name`是因为Python解释器对外把`__name`变量改成了`_Student__name`，所以，仍然可以通过`_Student__name`来访问`__name`变量

  > `python _、__和__xx__的区别`
  >
  > - `"_"单下划线`
  >
  >   1. **在解释器中**：在这种情况下，“_”代表交互式解释器会话中上一条执行的语句的结果。
  >   2. 以下划线“\_”为前缀的名称（如_spam）应该被视为API中非公开的部分（不管是函数、方法还是数据成员）
  >
  > - `"__"双下划线`
  >
  >   \_\_spam这种形式（至少两个前导下划线，最多一个后续下划线）的任何标识符将会被“_classname__spam”这种形式原文取代，在这里“classname”是去掉前导下划线的当前类名。
  >
  > - `名称前后的双下划线`
  >
  >   变量名类似`__xxx__`的，也就是以双下划线开头，并且以双下划线结尾的，是特殊变量，特殊变量是可以直接访问的，不是private变量

- 继承和多态

  ``` python
  class Class(superClass):
    	pass
  ```

- 获取对象信息

  - type()

    ```
    >>> type(abs)==types.BuiltinFunctionType
    True
    >>> type(lambda x: x)==types.LambdaType
    True
    >>> type((x for x in range(10)))==types.GeneratorType
    True
    >>> type(123)==int
    True
    >>> type('abc')==type('123')
    True
    >>> type(abs)
    <class 'builtin_function_or_method'>
    ```

  - isinstance()

    ```
    >>> isinstance(h, Dog)
    True
    ```

  - dir()

    ```
    >>> dir('ABC')
    ['__add__', '__class__',..., '__subclasshook__', 'capitalize', 'casefold',..., 'zfill']
    ```

    仅仅把属性和方法列出来是不够的，配合`getattr()`、`setattr()`以及`hasattr()`，我们可以直接操作一个对象的状态：

    ```
    >>> class MyObject(object):
    ...     def __init__(self):
    ...         self.x = 9
    ...     def power(self):
    ...         return self.x * self.x
    ...
    >>> obj = MyObject()

    >>> hasattr(obj, 'x') # 有属性'x'吗？
    True
    >>> obj.x
    9
    >>> hasattr(obj, 'y') # 有属性'y'吗？
    False
    >>> setattr(obj, 'y', 19) # 设置一个属性'y'
    ```

- 实例属性和类属性

  给实例绑定属性的方法是通过实例变量，或者通过`self`变量：

  ```
  class Student(object):
      def __init__(self, name):
          self.name = name

  s = Student('Bob')
  s.score = 90

  ```

  `Student`类本身绑定属性直接在class中定义属性

  ```
  class Student(object):
      name = 'Student'

  ```

  当我们定义了一个类属性后，这个属性虽然归类所有，但类的所有实例都可以访问到。

  ```
  >>> class Student(object):
  ...     name = 'Student'
  ...
  >>> s = Student() # 创建实例s
  >>> print(s.name) # 打印name属性，因为实例并没有name属性，所以会继续查找class的name属性
  Student
  >>> print(Student.name) # 打印类的name属性
  Student
  >>> s.name = 'Michael' # 给实例绑定name属性
  >>> print(s.name) # 由于实例属性优先级比类属性高，因此，它会屏蔽掉类的name属性
  Michael
  >>> print(Student.name) # 但是类属性并未消失，用Student.name仍然可以访问
  Student
  >>> del s.name # 如果删除实例的name属性
  >>> print(s.name) # 再次调用s.name，由于实例的name属性没有找到，类的name属性就显示出来了
  Student
  ```


# 2 面向对象高级

# 2.1 使用\_\_slots\_\_

正常情况下，当我们定义了一个class，创建了一个class的实例后，我们可以给该实例绑定任何属性和方法，这就是动态语言的灵活性

```
class Student(object):
    pass

```

然后，尝试给实例绑定一个属性：

```
>>> s = Student()
>>> s.name = 'Michael' # 动态给实例绑定一个属性
>>> print(s.name)
Michael
```

还可以尝试给实例绑定一个方法：

```
>>> def set_age(self, age): # 定义一个函数作为实例方法
...     self.age = age
...
>>> from types import MethodType
>>> s.set_age = MethodType(set_age, s) # 给实例绑定一个方法
>>> s.set_age(25) # 调用实例方法
>>> s.age # 测试结果
25
```

> **注意，以上都是给实例绑定的方法**

为了给所有实例都绑定方法，可以给class绑定方法：

```
>>> def set_score(self, score):
...     self.score = score
...
>>> Student.set_score = set_score
```

### 使用`__slots__`

`__slots__`变量，来限制该class实例能添加的属性，`__slots__`定义的属性仅对当前类实例起作用，对继承的子类是不起作用的：

```
class Student(object):
    __slots__ = ('name', 'age') # 用tuple定义允许绑定的属性名称
```

在子类中也定义`__slots__`，这样，子类实例允许定义的属性就是自身的`__slots__`加上父类的`__slots__`。

#### 2.2 @property

Python内置的`@property`装饰器就是负责把一个方法变成属性调用的：

```python
class Student(object):

    @property
    def score(self):
        return self._score

    @score.setter
    def score(self, value):
        if not isinstance(value, int):
            raise ValueError('。。。')
        if value < 0 or value > 100:
            raise ValueError('。。。')
        self._score = value
```

getter方法变成属性，只需要加上`@property`此时，`@property`本身又创建了另一个装饰器`@score.setter`，负责把一个setter方法变成属性赋值

## 2.3 多重继承

```
class Bat(Mammal, Flyable):
    pass
```

## 2.4 定制类

`__slots__`这种形如`__xxx__`的变量或者函数名就要注意，这些在Python中是有特殊用途的

- `__slots__`


- `__len__()`方法我们也知道是为了能让class作用于`len()`函数。

- `__str__` 类似toString()

- `__repr__()` 返回程序开发者看到的字符串,`<__main__.Student object at 0x109afb310>`,`__repr__ = __str__`

- `__iter__`用于`for ... in`循环

  ```
  class Fib(object):
      def __init__(self):
          self.a, self.b = 0, 1 # 初始化两个计数器a，b

      def __iter__(self):
          return self # 实例本身就是迭代对象，故返回自己

      def __next__(self):
          self.a, self.b = self.b, self.a + self.b # 计算下一个值
          if self.a > 100000: # 退出循环的条件
              raise StopIteration()
          return self.a # 返回下一个值
  ```

- `__getitem__`由于类list访问，通过[]下标

  ```
  class Fib(object):
      def __getitem__(self, n):
          if isinstance(n, int): # n是索引
              a, b = 1, 1
              for x in range(n):
                  a, b = b, a + b
              return a
          if isinstance(n, slice): # n是切片
              start = n.start
              stop = n.stop
              if start is None:
                  start = 0
              a, b = 1, 1
              L = []
              for x in range(stop):
                  if x >= start:
                      L.append(a)
                  a, b = b, a + b
              return L
  ```

- `__getattr__`避免调用不存在的`score`属性

  ```
  def __getattr__(self, attr):
          if attr=='score':
              return 99
  ```

- `__call__` 对象看成函数

  ```
  class Student(object):
      def __init__(self, name):
          self.name = name

      def __call__(self):
          print('My name is %s.' % self.name)
  ```

  调用方式如下：

  ```
  >>> s = Student('Michael')
  >>> s() # self参数不要传入
  My name is Michael.
  ```

  通过`callable()`函数，我们就可以判断一个对象是否是“可调用”对象。

## 2.3 枚举

```python
from enum import Enum

Month = Enum('Month', ('Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'))
```

```
for name, member in Month.__members__.items():
    print(name, '=>', member, ',', member.value)
```

`value`属性则是自动赋给成员的`int`常量，默认从`1`开始计数。

如果需要更精确地控制枚举类型，可以从`Enum`派生出自定义类：

```python
from enum import Enum, unique

@unique
class Weekday(Enum):
    Sun = 0 # Sun的value被设定为0
    Mon = 1
    Tue = 2
    Wed = 3
    Thu = 4
    Fri = 5
    Sat = 6
```

`@unique`装饰器可以帮助我们检查保证没有重复值。

## 2.4 使用元类

### 2.4.1 type()

动态语言和静态语言最大的不同，就是函数和类的定义，不是编译时定义的，而是运行时动态创建的。

`type()`函数可以查看一个类型或变量的类型，`Hello`是一个class，它的类型就是`type`，而`h`是一个实例，它的类型就是class `Hello`。



`type()`函数既可以返回一个对象的类型，又可以创建出新的类型，要创建一个class对象，`type()`函数依次传入3个参数：

1. class的名称；
2. 继承的父类集合，注意Python支持多重继承，如果只有一个父类，别忘了tuple的单元素写法；
3. class的方法名称与函数绑定，这里我们把函数`fn`绑定到方法名`hello`上。

### 2.4.2 metaclass

metaclass，直译为元类，metaclass允许你创建类或者修改类。

当我们定义了类以后，就可以根据这个类创建出实例，所以：先定义类，然后创建实例。但是如果我们想创建出类呢？那就必须根据metaclass创建出类，所以：先定义metaclass，然后创建类。