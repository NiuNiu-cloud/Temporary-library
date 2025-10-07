# 面向对象编程（一）
## 类
```C++
class 类名 
{
    // 访问控制符：public, private, protected
    public:
        // 公有成员（可以被类外部访问）
        返回类型 成员函数名(参数列表);
        
    private:
        // 私有成员（只能被类内部访问）
        数据类型 成员变量名;
        
    protected:
        // 保护成员（类内部和派生类可访问）
        数据类型 成员变量名;
};      //注意：要以分号结束

// 成员函数的实现
返回类型 类名::成员函数名(参数列表) 
{
    // 函数体
}

```
- public（公有）：类的外部可以直接访问，通常用于定义接口函数。
- private（私有）：只能被类的成员函数访问，通常用于存储数据。
- protected（保护）：类似 private，但派生类可以访问。

## 类的对象
>   类名 类的对象名；   //Person p1;

## 构造函数
> 构造函数的命名和类是一样的，这就使得创创建对象的同时完成变量初始化。

## 成员函数
> 在类的内部声明和定义（或仅声明，在类外定义）的函数。

```C+
class Circle 
{
    private:
        double radius; // 私有成员
    public:
        // 成员函数：在类内声明并定义
        void setRadius(double r) 
        {
            radius = r; // 直接访问私有成员
        }

        // 成员函数：仅声明，类外定义
        double getArea();
};

// 类外定义成员函数（需加类名限定符::）
double Circle::getArea() 
{
    return 3.14 * radius * radius;
    // 通过this指针访问当前对象的radius
}
```