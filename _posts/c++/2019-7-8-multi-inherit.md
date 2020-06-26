---
layout: post
title: c++中的多继承
category: c++
tags: c++ inherit
description: c++中的多继承
---

C++中是支持多继承的，之前刚学习的时候仅仅是了解，现在学习python(python 3)中的多继承的时候，忽然想起c++的多继承一些不完美的地方。
比如在python中
我们先定义一个Person类
定义了吃饭睡觉等方法，
然后定义几个它的子类小王、小张、小军。
他们吃饭睡觉的方式都不同。
然后再定义一个类小红，继承小王，小张、小军，就继承了他们的所有特性。

```python
class Person:
    def __init__(self):
        self.ability = 1

    def eat(self):
        print("Eat: ", self.ability)
    
    def slep(self):
        print("sleep: ", self.ability)

    def save_life(self):
        print("+ : ", self.ability, ' s')

class Wang(Person):
    def eat(self):
        print('Eat ', self.ability*2)

class Zhang(Person):
    def sleep(self):
        print('Sleep ', self.ability*2)

class Jun(Person):
    def save_life(self):
        print('+ : ', self.ability*2, ' ')

class Hong(Wang, Zhang, Jun):
    pass

def test(person):
    person.eat()
    person.sleep()
    person.save_life()

if __name__ == '__main__':
    h = Hong()
    test(h)
```

代码输出为：

```sh
Eat  2
Sleep  2
+ :  2
```

那如果是c++会怎样呢？因为这里Hong的三个父类继承自同一个基类，
所以需要使用虚拟继承来解决二义性问题。所以代码如下：
```c++
#include <iostream>
using namespace std;

class Person {

public:
    int ablility = 1;

public:
    virtual void eat() {
        cout << "Eat: " << ablility << endl;
    }

    virtual void sleep() {
        cout << "Sleep: " << ablility << endl;
    }

    virtual void save_life() {
        cout << "+ " << ablility << endl;
    }
};

class Wang: virtual public Person {
public:
    void eat() {
        cout << "Eat: " << ablility *2 << endl;
    }
};

class Zhang: virtual public Person {
public:
    void sleep() {
        cout << "Sleep: " << ablility * 2 << endl;
    }
};

class Jun: virtual public Person {
public:
    void save_life() {
        cout << "+ " << ablility * 2 << endl;
    }
};

class Hong: public Wang, Zhang, Jun {

};

void test(Person *p) {
    p->eat();
    p->sleep();
    p->save_life();
}

int main() {
    Hong h;
    test(&h);
    return 0;
}
```

这和person 的结果是一样的，但如果Wang除了eat函数sleep函数也覆盖了父类呢？这会和Zhang发生一个二义性的冲突。比如将Wang类的定义修改如下：

```c++
class Wang: virtual public Person {
public:
    void eat() {
        cout << "Eat: " << ablility *2 << endl;
    }
    void sleep() {
        cout << "Sleep: " << ablility * 3 << endl;
    }
};
```

G++编译的时候会报错：

```sh
PS D:\Code\c++\pratice> g++ test.cpp
test.cpp:47:7: error: no unique final overrider for 'virtual void Person::sleep()' in 'Hong'
 class Hong: public Wang, Zhang, Jun {
```

提示必须再Hong中定义一个sleep函数来覆盖。也就是c++不允许这样的定义。
那python又如何呢？
修改Wang类

```python
class Wang(Person):
    def eat(self):
        print('Eat ', self.ability*2)

    def sleep(self):
        print('Sleep ', self.ability*3)
```

输出如下：

```sh
Eat  2
Sleep  3
+ :  2
```

这是因为python采用MRO(Method Resolution Order)来解决多继承中的二义性问题，而在python3中采用的是广度优先算法。所以有限会调用Wang中的sleep方法。如果再Hong类的继承上改变下顺序：

```python
class Hong(Zhang, Wang, Jun):
    pass
```

则输出如下：

```sh
Eat  2
Sleep  2
+ :  2
```

在这里我们可以明显发现c++和Python中对待多继承的不同，c++为了避免菱形继承的二义性问题，引入了虚继承以及各种规则，使得多继承的内存布局十分的复杂，又为了使其对用户透明，又在指针上加入了默认的指针转换。也许是c++过于底层，以及静态语言的特性，但如此复杂的东西自然是让人敬而远之了。而python以其动态语言的特性，使用MRO简单的解决了多继承的问题，使用起来比c++要方便太多了。
