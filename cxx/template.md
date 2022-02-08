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
    ((std::cout << ints << ' '), ...); // pack fold expression (in c++17 折叠表达式)
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
