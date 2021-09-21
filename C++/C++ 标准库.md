### 3.3.1 标准异常类别

语言本身或标准库所抛出的**所有异常**都**派生自基类**`exception`

* 语言本身支持的异常
* C++标准程序库发出的异常
* 程序作用域之外发出的异常



**语言本身支持的异常**

1. bad_alloc

   全局操作符new操作失败时

2. bad_cast

   对象引用进行动态型别转换操作失败时

```c++
// bad_cast example
#include <iostream>       // std::cout
#include <typeinfo>       // std::bad_cast

class Base {virtual void member(){}};
class Derived : Base {};

int main () {
  try
  {
    Base b;
    Derived& rd = dynamic_cast<Derived&>(b);
  }
  catch (std::bad_cast& bc)
  {
     std::cerr << "bad_cast caught: " << bc.what() << '\n';
  }
  return 0;
}
```



1. bad_typeld

   typeid参数为0或者空指针时

2. bad_exception

   在异常列表中没有的异常，如果列表中有bad_exception，会作为bad_exception抛出

**C++ 标准库发出的异常**

1. invalid_argument

   无效参数

2. length_error

   可能超越最大极限

3. out_of_range

   不再预期范围内，数组越界访问之类的

4. domain_error

   专业领域范畴内错误



**程序作用域之外发出的异常**

1. range_error

   内部计算时的区间错误

2. overflow_error

   算数运算发生上溢位

3. underflow_error

   算数运算发生下溢位
   
   


## 3.4 配置器



# 4 通用工具

## 4.1 Pairs（对组）

