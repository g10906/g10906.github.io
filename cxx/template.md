# Template

## 变量模板(c++14)
```c++
#include<iostream>
using namespace std;
template<class T>
constexpr T pi=T(3.141592653589L);
```
 
## ... 解包
``` c++
#include<iostream>
#include <utility>

using namespace std;

/** 递归解包 */
void print(){
    cout<<"hello world"<<endl;

}
template<Class T, class ...arg>
void print(constT& a, const arg&... b){
    cout<< a << ", ";
    print(b...);
}


/** 配合integer_sequence */
template<typename T, T... ints>
void print_sequence(std::integer_sequence<T, ints...> int_seq)
{
    std::cout << "The sequence of size " << int_seq.size() << ": ";
    ((std::cout << ints << ' '), ...); 
    // pack fold expression (in c++17 折叠表达式)
    // (std::cout << 9 << ' '), (std::cout << 2 << ' '), ...
    std::cout << '\n';
}

int main() {
    print_sequence(std::integer_sequence<int, 9, 2, 5, 1, 9, 1, 6>{});
    return 0;
}

// 输出：7 9 2 5 1 9 1 6


/** 配合 tuple */
template <std::size_t... Is, typename F, typename T>
auto map_filter_tuple(F f, T& t) {
    return std::make_tuple(f(std::get<Is>(t))...);
}

template <std::size_t... Is, typename F, typename T>
auto map_filter_tuple(std::index_sequence<Is...>, F f, T& t) {
    return std::make_tuple(f(std::get<Is>(t))...);
}

template <typename S, typename F, typename T>
auto map_filter_tuple(F&& f, T& t) {
    return map_filter_tuple(S{}, std::forward<F>(f), t);
}


```
## 折叠表达式(fold expression)
refer [C++17尝鲜：fold expression（折叠表达式](https://blog.csdn.net/zwvista/article/details/53981696)
折叠表达式是C++17新引进的语法特性。使用折叠表达式可以简化对C++11中引入的参数包的处理，从而在某些情况下避免使用递归。

### 语法形式

折叠表达式共有四种语法形式。分别为一元的左折叠和右折叠，以及二元的左折叠和右折叠。

1. 一元右折叠(unary right fold)
    ( pack op ... )
    一元右折叠(E op ...)展开之后变为 E1 op (... op (EN-1 op EN))
2. 一元左折叠(unary left fold)
    ( ... op pack )
    一元左折叠(... op E)展开之后变为 ((E1 op E2) op ...) op EN
3. 二元右折叠(binary right fold)
    ( pack op ... op init )
    二元右折叠(E op ... op I)展开之后变为 E1 op (... op (EN−1 op (EN op I)))
4. 二元左折叠(binary left fold)
    ( init op ... op pack )
    二元左折叠(I op ... op E)展开之后变为 (((I op E1) op E2) op ...) op EN

语法形式中的op代表运算符，pack代表参数包，init代表初始值。
初始值在右边的为右折叠，展开之后从右边开始折叠。而初始值在左边的为左折叠，展开之后从左边开始折叠。
不指定初始值的为一元折叠表达式，而指定初始值的为二元折叠表达式。
当一元折叠表达式中的参数包为空时，只有三个运算符（&& || 以及逗号）有缺省值，其中&&的缺省值为true,||的缺省值为false，逗号的缺省值为void()。
  
下面看看具体应用
### 求和
``` c++
#include <iostream>
using namespace std;
 
template<typename First>
First sum1(First&& value)
{
    return value;
}
 
template<typename First, typename Second, typename... Rest>
First sum1(First&& first, Second&& second, Rest&&... rest)
{
    return sum1(first + second, forward<Rest>(rest)...);
}
 
template<typename First, typename... Rest>
First sum2(First&& first, Rest&&... rest)
{
    return (first + ... + rest);
}
 
int main()
{
    cout << sum1(1) << sum1(1, 2) << sum1(1, 2, 3) << endl; // 136
    cout << sum2(1) << sum2(1, 2) << sum2(1, 2, 3) << endl; // 136
}
```
- 在C++17之前，求和函数sum1的实现必须分成两个部分。  
    其中4到8行的sum1函数用于处理一个参数的情况。而10到14行的sum1函数用于处理两个及以上参数的情况。  
    当参数个数大于一个时，10到14行的sum1函数将前两个参数相加，然后递归调用自身。  
    当参数个数只有一个时，4到8行的sum1函数将此参数返回，完成求和。  
    sum1(1, 2, 3) = sum1(1+2, 3) = sum1(3, 3) = sum1(3+3) = sum1(6) = 6
- 而在C++17之后，由于有了折叠表达式这个新特性，求和函数sum2不再需要处理特殊情况，实现大为简化。  
    sum2(1, 2, 3) = (1 + ... + pack(2, 3)) = (1+2) + 3 = 6  
    这里sum2的实现所采用的是二元左折叠。
### “与”和“或”
``` c++
#include <iostream>
using namespace std;
 
template<typename... Args>
bool all(Args... args) {return (... && args);}
template<typename... Args>
bool any(Args... args) {return (... || args);}
 
int main() 
{
    cout << boolalpha << all(true, false, true) << endl; // false
    cout << boolalpha << any(true, false, true) << endl; // true
    cout << boolalpha << all() << endl; // true
    cout << boolalpha << any() << endl; // false
}
```

- 在这里all和any函数分别实现了不特定多数布尔值的与和或的运算。这两个函数的实现均采用了一元左折叠。
    all(true, false, true) = (... && pack(true, false, true)) = (true && false) && true = false
    any(true, false, true) = (... || pack(true, false, true)) = (true || false) || true = true
- 当一元折叠表达式中的参数包为空时，&&的缺省值为true，而||的缺省值为false。
    all() = (... && pack()) = true
    any() = (... || pack()) = false
### “打印”和“调用”
``` c++
#include <iostream>
using namespace std;
 
template<typename... Ts>
void printAll(Ts&&... mXs)
{
    (cout << ... << mXs) << endl;
}
 
template<typename TF, typename... Ts>
void forArgs(TF&& mFn, Ts&&... mXs)
{
    (mFn(mXs), ...);
}
 
int main() 
{
    printAll(3, 4.0, "5"); // 345
    printAll(); // 空行
    forArgs([](auto a){cout << a;}, 3, 4.0, "5"); // 345
    forArgs([](auto a){cout << a;}); // 空操作
}
```
- printAll函数实现了对不特定多数值的打印输出。该函数的实现采用了二元左折叠。
    printAll(3, 4.0, "5")
    \= (cout << ... << pack(3, 4.0, "5")) << endl
    \= ((cout << 3) << 4.0) << "5" << endl
    \= 打印345并换行
- 当二元折叠表达式的参数包为空时，其计算结果为该二元折叠表达式中所预设的初始值。
    printAll()
    \= (cout << ... << pack()) << endl
    \= cout << endl
    \= 空行
- forArgs函数实现了依次使用多个参数调用某个单参数函数的功能。该函数的实现采用了一元右折叠。
    forArgs(\[\](auto a){cout << a;}, 3, 4.0, "5")
    \= (\[\](auto a){cout << a;}(pack(3, 4.0, "5")), ...)
    \= \[\](auto a){cout << a;}(3), (\[\](auto a){cout << a;}(4.0), (\[\](auto a){cout << a;}("5")))
    \= 打印345
- 当使用逗号的一元折叠表达式中的参数包为空时，其计算结果为标准规定的缺省值void()。
    forArgs(\[\](auto a){cout << a;})
    \= (\[\](auto a){cout << a;}(pack()), ...)
    \= void()
