---
title: Python02_面向对象
id: d5fe0d8e-4756-4b7f-a4bd-9058dff3ec2d
date: 2024-03-30 00:07:19
auther: xmy
cover: null
excerpt: 类和实例 通过关键字class定义类 class className(object)    pass className是类名，(object)表示从object类继承，默认所有类都会继承的类，可
permalink: /archives/python02_mian-xiang-dui-xiang
categories:
 - coding
tags: 
 - python
---




## 类和实例
通过关键字`class`定义类
```Python
class className(object):
    pass
```
className是类名，(object)表示从object类继承，默认所有类都会继承的类，可以替换为其它类，表示从该类继承。


```Pytnon
class Student(object):

    # 构造函数
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def study(self, course_name):
        print('%s正在学习%s'%(self.name, course_name))

    def watch_movie(self):
        if self.age>=18:
            print('%s可以观看爱情大电影.'% self.name)
        else:
            print('%s只能观看熊出没.'% self.name)

def main():
    stu1 = Student('jerry',18)
    stu1.study('English')
    stu1.watch_movie()
    stu2 = Student('tom',15)
    stu2.study('Math')
    stu2.watch_movie()

if __name__ == '__main__':
    main()
```


## 访问限制

在属性或方法前加`__`
```Python
class Test:
    def __init__(self, foo):
        self.__foo = foo

    def __bar(self):
        print(self.__foo)
        print('__bar')

def main():
    test = Test('hello')
    test.__bar()
    print(test.__foo)

if __name__ == '__main__':
    main()
```

但是python中并没有从语法上严格保证私有属性或方法的私密性，通过一下方法仍然可以访问到
```
class Test:
    def __init__(self, foo):
        self.__foo = foo

    def __bar(self):
        print(self.__foo)
        print('__bar')

def main():
    test = Test('hello')
    test._Test__bar()
    print(test._Test__foo)

if __name__ == '__main__':
    main()
```

## 继承和多态
### 继承
定义一个Animal类作为父类
```Python
class Animal(object):
    def run(self):
        print('Animal is running...')
```

其他类就可以继承该类：
```Python
class Dog(Animal):
    pass

class Cat(Animal):
    pass
```
继承了类就获得了该类的方法，可以直接实例化后调用



### 多态
有时候不同的子类需要重写父类的方法，实现自身的功能，这样的特性就是多态，子类重写的方法后，程序运行时传入子类就会调用子类的方法
```Python
class Dog(Animal):

    def run(self):
        print('Dog is running...')

class Cat(Animal):

    def run(self):
        print('Cat is running...')
```

### 鸭子类型 Duck Type
> 对于静态语言（例如Java）来说，如果需要传入Animal类型，则传入的对象必须是Animal类型或者它的子类，否则，将无法调用run()方法。

> 对于Python这样的动态语言来说，则不一定需要传入Animal类型。我们只需要保证传入的对象有一个run()方法就可以了：

```Python
class Timer(object):
    def run(self):
        print('Start...')
```
> 这就是动态语言的“鸭子类型”，它并不要求严格的继承体系，一个对象只要“看起来像鸭子，走起路来像鸭子”，那它就可以被看做是鸭子。

> Python的“file-like object“就是一种鸭子类型。对真正的文件对象，它有一个read()方法，返回其内容。但是，许多对象，只要有read()方法，都被视为“file-like object“。许多函数接收的参数就是“file-like object“，你不一定要传入真正的文件对象，完全可以传入任何实现了read()方法的对象。


## 装饰器

* `@property`
* `setter`  

示例代码
```
class Person(object):
    def __init__(self, name, age):
        self._name = name
        self._age = age

    @property
    def name(self):
        return self._name

    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, age):
        self._age = age

    def play(self):
        if self._age <= 16:
            print('{}正在玩飞行棋.'.format(self._name))
        else:
            print('{}正在玩斗地主.'.format(self._name))

def main():
    person = Person('王大锤', 12)
    person.play()
    person.age = 22
    person.play()
    print(person.name)
    #person.name = '元芳'

if __name__ == '__main__':
    main()
```


## 静态方法和类
* `staticmethod`
* `classmethod`：类方法第一个参数名为`cls`

## 实例属性和类属性
给实例绑定属性：
```python
class Student(object):
    def __init__(self, name):
        self.name = name

s = Student('Bob')
s.score = 90
```
给类绑定属性：
```python
class Student(object):
    name = 'Student'
```
* 给类绑定属性以后，实例化出来的对象均可以获得该属性
* 实例属性优先级高于类属性，均赋值的情况下，使用实例属性

同样也可以给实例和类绑定方法：
```python
>>> def set_age(self, age): # 定义一个函数作为实例方法
...     self.age = age
...
>>> from types import MethodType
>>> s.set_age = MethodType(set_age, s) # 给实例绑定一个方法
>>> s.set_age(25) # 调用实例方法
>>> s.age # 测试结果
25

>>> def set_score(self, score):
...     self.score = score
...
>>> Student.set_score = set_score
```


## \_\_slots\_\_魔法
默认可以给实例对象添加任何类型的属性  
> 如果我们需要限定自定义类型的对象只能绑定某些属性，可以通过在类中定义__slots__变量来进行限定。需要注意的是__slots__的限定只对当前类的对象生效，对子类并不起任何作用。

```Python
class Person(object):

    __slots__ = ('_name', '_age', '_gender')

    def __init__(self, name, age):
        self._name = name
        self._age = age

    @property
    def name(self):
        return self._name

    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, age):
        self._age = age

    def play(self):
        if self._age <= 16:
            print('{}正在玩飞行棋.'.format(self._name))
        else:
            print('{}正在玩斗地主.'.format(self._name))

def main():
    person = Person('王大锤', 12)
    person.play()
    person.age = 22
    person.play()
    print(person.name)
    #person.name = '元芳'

if __name__ == '__main__':
    main()
```


## 多重继承
python支持多重继承，通常称为MixIn。  
例如设计一个多线程模式的TCP服务，可以继承TCPServer和ForkingMixIn：
```python
class MyTCPServer(TCPServer, ForkingMixIn):
    pass
```


## 抽象类
> Python从语法层面并没有像Java或C#那样提供对抽象类的支持，但是我们可以通过abc模块的ABCMeta元类和abstractmethod包装器来达到抽象类的效果，如果一个类中存在抽象方法那么这个类就不能够实例化（创建对象）。

```
from abc import ABCMeta, abstractmethod

@abstractmethod


```