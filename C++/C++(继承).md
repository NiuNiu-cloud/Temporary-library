# 面向对象编程（二）
## 继承
> 继承允许我们依据另一个类来定义一个类,已有的类称为基类，新建的类称为派生类。
1. 继承语法：class 派生类名: 继承方式 基类名
2. 继承方式：
    - public
        - 基类的public成员 → 派生类中仍为public
        - 基类的protected成员 → 派生类中仍为protected
        - 基类的private成员 → 派生类中不可访问（无论哪种继承方式都不可访问）
    - protected
        - 基类的public和protected成员 → 派生类中均为protected
    - private
        - 基类的public和protected成员 → 派生类中均为private

## 多继承
class <派生类名>: <继承方式1><基类名1>, <继承方式2><基类名2>,…
{
    <派生类类体>
};