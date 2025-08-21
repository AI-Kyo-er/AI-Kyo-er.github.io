---
title: 'ABSTRACT BASE CLASS: Comparison of the Python and Cpp'
date: 2024-09-09 11:00:11
categories: 
- CS 
tags:
- Note
---

### 抽象基类：Python 与 C++ 的比较

在编程中，抽象基类（Abstract Base Class, ABC）是一种定义接口的方式，要求所有继承类必须实现基类中定义的某些方法。Python 和 C++ 都支持抽象基类，但它们的实现方式和细节有所不同。

---

### 1. 抽象基类的定义方式

#### Python
在 Python 中，抽象基类通过继承 `ABC` 类（来自 `abc` 模块）并使用 `@abstractmethod` 装饰器来定义抽象方法。

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass
```

#### C++
在 C++ 中，抽象基类是通过定义**纯虚函数**来实现的。纯虚函数的定义形式为 `= 0`。

```cpp
class Shape {
public:
    virtual double area() const = 0;  // 纯虚函数
};
```

---

### 2. 强制实现抽象方法

#### Python
- 如果子类没有实现所有的抽象方法，Python 会在运行时抛出 `TypeError`，禁止实例化该子类。

```python
class Circle(Shape):
    pass  # 如果没有实现 area 方法，实例化时会抛出 TypeError

# c = Circle()  # TypeError: Can't instantiate abstract class Circle
```

#### C++
- 在 C++ 中，如果子类没有实现基类的纯虚函数，编译时会报错，并且无法实例化该子类。

```cpp
class Circle : public Shape {
    // 不实现 area 方法的话，编译器会报错，无法实例化 Circle
};
```

---

### 3. 实例化抽象基类

#### Python
- 抽象基类不能直接被实例化，尝试实例化会抛出 `TypeError`。

```python
shape = Shape()  # TypeError: Can't instantiate abstract class Shape
```

#### C++
- 在 C++ 中，包含纯虚函数的抽象基类也不能直接实例化，尝试实例化会导致编译时错误。

```cpp
Shape shape;  // 编译错误：无法实例化抽象类
```

---

### 4. 构造函数和析构函数

#### Python
- Python 的抽象基类不需要显示定义构造函数和析构函数，垃圾回收器自动处理内存管理。

#### C++
- C++ 中的抽象基类通常需要定义虚析构函数 (`virtual destructor`)，以确保通过基类指针删除子类对象时，能够正确调用子类的析构函数。

```cpp
class Shape {
public:
    virtual ~Shape() {}  // 虚析构函数
};
```

---

### 5. 抽象类的多重继承

#### Python
- Python 支持多重继承，你可以继承多个抽象基类并实现多个接口。

```python
class Drawable(ABC):
    @abstractmethod
    def draw(self):
        pass

class Movable(ABC):
    @abstractmethod
    def move(self):
        pass

class Circle(Shape, Drawable, Movable):
    def area(self):
        return 3.14 * radius * radius

    def draw(self):
        print("Drawing circle")

    def move(self):
        print("Moving circle")
```

#### C++
- C++ 也支持多重继承，并可以继承多个抽象基类。但 C++ 的多重继承可能导致“菱形继承”问题，需使用虚继承来解决。

```cpp
class Drawable {
public:
    virtual void draw() const = 0;
};

class Movable {
public:
    virtual void move() const = 0;
};

class Circle : public Shape, public Drawable, public Movable {
public:
    double area() const override {
        return 3.14 * radius * radius;
    }

    void draw() const override {
        std::cout << "Drawing circle" << std::endl;
    }

    void move() const override {
        std::cout << "Moving circle" << std::endl;
    }
};
```

---

### 6. 动态绑定与静态绑定

#### Python
- **所有方法都是动态绑定的**：Python 是动态语言，所有方法调用都是基于对象的实际类型进行解析的，类似于 C++ 中虚函数的动态绑定。
  
```python
class Base:
    def print(self):
        print("Base print")

class Derived(Base):
    def print(self):
        print("Derived print")

base_ref = Derived()
base_ref.print()  # 输出: "Derived print"
```

#### C++
- **虚函数和非虚函数**：在 C++ 中，只有将基类方法声明为 `virtual` 时，才会启用动态绑定。否则，即使基类指针指向子类对象，也只会调用基类中的方法（静态绑定）。

```cpp
class Base {
public:
    virtual void print() {  // 虚函数
        std::cout << "Base print" << std::endl;
    }
};

class Derived : public Base {
public:
    void print() override {
        std::cout << "Derived print" << std::endl;
    }
};

int main() {
    Base* ptr = new Derived();  // 基类指针指向子类对象
    ptr->print();  // 输出：Derived print
    return 0;
}
```

- **静态绑定的例子**：如果基类的 `print` 方法不是虚函数，基类指针将调用基类的 `print` 方法（静态绑定）。

```cpp
class Base {
public:
    void print() {  // 非虚函数
        std::cout << "Base print" << std::endl;
    }
};

class Derived : public Base {
public:
    void print() {
        std::cout << "Derived print" << std::endl;
    }
};

int main() {
    Base* ptr = new Derived();  // 基类指针指向子类对象
    ptr->print();  // 输出：Base print（静态绑定）
    return 0;
}
```

---

### 7. 总结
**All in all, 由于C++是静态编译语言，因此为了实现动态绑定不得不通过虚函数表实现动态绑定，而Python的解释器运行的特性使得动态绑定（子类指针多态地访问子类方法）
是既得的，所以虚函数、虚类就仅用于规范化接口上。**
- **C++**：必须使用 `virtual` 关键字来启用动态绑定。基类指针调用函数时，只有虚函数会根据对象的实际类型来调用相应的函数版本。
  
- **Python**：所有方法都是动态绑定的。Python 总是根据对象的实际类型来解析方法调用，因此不需要显式定义虚函数。 

C++ 和 Python 在抽象基类的处理方式上有许多相似之处，但由于语言的特性，它们在具体实现和调用机制上有所不同。