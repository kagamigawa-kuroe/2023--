#### const

- 修饰变量，说明该变量不可以被改变；

- 指向常量的指针（pointer to const）

- 自身是常量的指针（常量指针，const pointer）
- 修饰引用，指向常量的引用（reference to const）
- 修饰成员函数，说明该成员函数内不能修改成员变量

```c++
const char* p2 = greeting;          // 指针变量，指向字符数组常量（const 后面是 char，说明指向的字符（char）不可改变）
char* const p3 = greeting;          // 自身是常量的指针，指向字符数组变量（const 后面是 p3，说明 p3 指针自身不可改变
```

```c++
A b;                        // 普通对象，可以调用全部成员函数
const A a;                  // 常对象，只能调用常成员函数

// 类中的常量成员变量必须直接初始化
```

总计一下 const在后面的就是常指针，在前面的就是常量指针



---



#### static

修饰普通变量，修改变量的存储区域和生命周期，使变量存储在静态区

修饰普通函数，表明函数的作用范围，仅在定义该函数的文件内才能使用

修饰成员变量，修饰成员变量使所有的对象只保存一个该变量

修饰成员函数，修饰成员函数使得不需要生成对象就可以访问该函数



---



#### 内联函数

对于一些比较短的函数，用内联函数可以避免函数调用的过程，节约开销

类内定义，或者直接写在头文件中的函数 都被默认为内联

相当于宏，却比宏多了类型检查，真正具有函数特性；



1. 将 inline 函数体复制到 inline 函数调用点处；
2. 为所用 inline 函数中的局部变量分配内存空间；
3. 将 inline 函数的的输入参数和返回值映射到调用方法的局部变量空间中；
4. 如果 inline 函数有多个返回点，将其转变为 inline 函数代码块末尾的分支（使用 GOTO）。



内联是在编译期建议编译器内联，而虚函数的多态性在运行期，编译器无法知道运行期调用哪个代码，因此虚函数表现为多态性时（运行期）不可以内联。

只有在不用表现多态性的时候，虚函数才可以是内联函数



---



#### #pragma pack(n)

设置结构体的对齐方式

```c++
#pragma pack(push)  // 保存对齐状态
#pragma pack(4)     // 设定为 4 字节对齐

#pragma pack(pop)   // 恢复对齐状态
```



---



#### 位域

用于制定数据所占用的位数

位域的类型必须是整型或枚举类型

取地址运算符（&）不能作用于位域，任何指针都无法指向类的位域



---



#### 函数和类名的优先级问题

```c++
typedef struct Student { 
    int age; 
} S;

void Student() {}           // 正确，定义后 "Student" 只代表此函数

//void S() {}               // 错误，符号 "S" 已经被定义为一个 "struct Student" 的别名

int main() {
    Student(); 
    struct Student me;      // 或者 "S me";
    return 0;
}
```



---



#### explicit

- explicit 修饰构造函数时，可以防止**隐式转换和复制初始化**
- explicit 修饰转换函数时，可以防止隐式转换， 语境转换除外

```c++
B b2 = 1;		// 错误：被 explicit 修饰构造函数的对象不可以复制初始化
B b4 = { 1 };		// 错误：被 explicit 修饰构造函数的对象不可以复制列表初始化
doB(1);			// 错误：被 explicit 修饰构造函数的对象不可以从 int 到 B 的隐式转换
```



---



#### decltype

```c++
#include<iostream>

using namespace std;
int x = 0;
int y = 0;
// 判断表达式的类型
decltype(x+y) z;

// 尾返回类型
int func1(int a,int b) {
    return a + b;
}

auto func2(int a,int b) -> int {
    return a + b;
}

// 尾返回类型必须要加上auto
// int func3(int a,int b) -> int {
//     return a + b;
// }

// 尾返回类型可以与decltype结合 用于模版

template<typename T>
T func4(T t1,T t2){
    return t1+t2;
} // ok

// template<typename T>
// decltype(t1 + t2) func4(T t1,T t2){
//     return t1+t2;
// } // error 因为在编译的时候，无法识别t1 t2

template<typename T>
auto func4(T t1,T t2) -> decltype(t1+t2) {
    return t1+t2;
} // ok 用尾返回类型 + decltype 就可以完美解决这个问题

template<typename T>
auto func4(T t1,T t2){
    return t1+t2;
} // c++ 14开始可以直接用auto

// decltype(auto) 参数转发 用于函数多层调用的类型自动推导


/// 定义类型
using my_int = int;

template<typename T>
using TrueDarkMagic = MagicType<std::vector<T>, std::string>;
/// 上面的内容用decltype无法实现

int main(){
    z = 3;
    cout << z << endl;
    cout << is_same<decltype(x+y),int>::value << endl;
    return 0;
}
```



---



#### 引用折叠

- `X& &`、`X& &&`、`X&& &` 可折叠成 `X&`
- `X&& &&` 可折叠成 `X&&`



---



### initializer_list 列表初始化

新的构造函数的方式 出入的参数为一个初始化列表类型

```c++
template <class T>
struct S {
    std::vector<T> v;
    S(std::initializer_list<T> l) : v(l) {
         std::cout << "constructed with a " << l.size() << "-element list\n";
    }
    void append(std::initializer_list<T> l) {
        v.insert(v.end(), l.begin(), l.end());
    }
    std::pair<const T*, std::size_t> c_arr() const {
        return {&v[0], v.size()};  // 在 return 语句中复制列表初始化
                                   // 这不使用 std::initializer_list
    }
};

S<int> s = {1, 2, 3, 4, 5}; // 复制初始化
s.append({6, 7, 8});      // 函数调用中的列表初始化
```



---



#### 虚函数

- 可以将派生类的对象赋值给基类的指针或引用，反之不可
- 静态函数（static）不能是虚函数\
- 构造函数不能是虚函数（因为在调用构造函数时，虚表指针并没有在对象的内存空间中，必须要构造函数调用完成后才会形成虚表指针）
- 内联函数不能是表现多态性时的虚函数



---



编译器在某些情况下会禁止inline：

1：编译器禁止虚函数inline

2：带有循环或者递归的调用

3: 通过函数指针调用inline函数
