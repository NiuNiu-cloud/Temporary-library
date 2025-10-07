# 重载函数

在C++中，函数重载（Function Overloading）是指在同一个作用域内，可以可以定义多个同名函数，只要它们的参数列表不同（参数个数、参数类型或参数顺序不同）。编译器会根据调用时提供的实参自动匹配到对应的函数。

> my understand: ==函数名可以一样，但是输入参数不能一样。==

```C++
#include <iostream>
using namespace std;
class printData
{
   public:
      void print(int i) {
        cout << "整数为: " << i << endl;
      }
 
      void print(double  f) {
        cout << "浮点数为: " << f << endl;
      }
 
      void print(char c[]) {
        cout << "字符串为: " << c << endl;
      }
};
 
int main(void)
{
   printData pd;
   // 输出整数
   pd.print(5);
   // 输出浮点数
   pd.print(500.263);
   // 输出字符串
   char c[] = "Hello C++";
   pd.print(c);
   return 0;
}
```

# 重载运算符

> 不好理解
