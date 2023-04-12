---
title: C++模板元编程的初步介绍和应用
date: 2021-05-10T10:43:00+08:00
summary: 本文的主要内容是对C++模板元编程的初步介绍和应用，将围绕luna库进行展开
categories: [编程]
tags: [C++, 模板元编程, Lua]
---

## 一、序
本文的主要内容是对C++模板元编程的初步介绍和应用，将围绕luna库进行展开

### （一）luna库简要介绍
[luna](https://github.com/trumanzhao/luna)是一个C++的lua binding库，在我们游戏服务器中有用到，它的核心代码只有一个头文件和一个源文件，实现得非常简洁和优雅。它主要应用了模板元编程的相关技巧，分析待绑定函数的签名信息，自动生成对应的lua绑定函数。阅读本文不需要提前熟悉luna库，只需对lua有初步认识即可

luna库的简单应用如下所示：

```cpp
static int add(int a, int b) {
  std::cout << "add is called!" << std::endl;
  return a + b;
}

void main() {
  lua_State* L = luaL_newstate();
  luaL_openlibs(L);

  lua_register_function(L, "add", &add);
  luaL_dostring(L, "print(add(1, 2))");

  lua_close(L);
}
// 程序输出:
// add is called!
// 3
```

### （二）总览
本文主要分为两大部分

第一部分会先介绍模板元编程的一些基础概念，基本只会涉及luna库里出现过的模板元编程技巧，不会过度深入

第二部分则是对luna库进行扩展，这也是本文的重点。一直以来模板元编程给人的印象只是炫技，不实用。在这一部分我会应用模板元编程的相关技巧，用60行左右的代码实现一个非常有用的功能：将任意lua函数封装到std::function里，简化C++注册lua回调的方式。这很难用代码生成的方式实现，目前并没有发现其他lua binding库实现了类似功能（老版本的cocos2d-x实现了类似功能，但需要特意用clang插件对C++源码进行语法分析来生成）。本文的最终目标，是扩展luna库实现这样的功能：

```cpp
using Callback = std::function<std::string(int, float)>;

static Callback callback;

static void set_callback(Callback value) {
  callback = std::move(value);
}

void main() {
  lua_State* L = luaL_newstate();
  luaL_openlibs(L);

  lua_register_function(L, "set_callback", &set_callback);

  luaL_dostring(L, R"(
    set_callback(function(i, f)
      print(string.format("Print in Lua, args: %d, %f", i, f))
      return "I'm Lua callback"
    end)
  )");
  std::string ret = callback(123, 0.125f);
  std::cout << "Callback result: " << ret << std::endl;

  callback = nullptr;
  lua_close(L);
}

// 程序输出：
// Print in Lua, args: 123, 0.125000
// Callback result : I'm Lua callback
```
由于我们游戏服务器用到的luna库版本比较老，所以本文对luna库的分析都是基于这个比较老的版本

## 二、基础概念

### （一）什么是模板元编程
模板元编程（节选自wikibook）：
```
Template meta-programming (TMP) refers to uses of the C++ template system to perform computation at compile-time within the code. It can, for the most part, be considered to be "programming with types" — in that, largely, the "values" that TMP works with are specific C++ types.

TMP is much closer to functional programming than ordinary idiomatic C++ is. This is because 'variables' are all immutable, and hence it is necessary to use recursion rather than iteration to process elements of a set.

Historically TMP is something of an accident; it was discovered during the process of standardizing the C++ language that its template system happens to be Turing-complete, i.e., capable in principle of computing anything that is computable.
```
关键字：
- 是编译期的运算
- 主要操作的“值”是类型，类型在模板元编程里是First class citizen
- 类似于函数式编程（“变量”不可变、递归）
- 是意外被发现的，而不是被发明出来
- 图灵完全

### （二）函数与值

#### 1、模板元函数
模板元函数（Template meta function）是模板元编程的核心组成部分之一，其本质其实就是模板类，只是看待模板类的角度不一样。所以：
```cpp
std::vector
std::shared_ptr
std::function
```
理论上都可以看作是模板元函数（下面简称元函数）

我可以将模板类看作一个“函数”，将模板参数看作是“函数参数”，将模板类生成的类看作是“函数返回值”，即：

```cpp
// 函数：std::vector
// 输入：int
// 输出：std::vector<int>
std::vector<int>
```
在模板元编程里，主要的输入“数据”是类型、编译期常量和模板类，也就是说，模板元函数：
- 是一个模板类
- 输入：类型、编译期常量、模板类
- 输出：类型

一般来说，我们不会直接使用输出的类型，而是从输出的类型中提取有用的“数据”，这些有用“数据”也是模板元编程的核心组成部分之一

#### 2、从输出类型中提取静态常量
这里我们来实现std::is_same元函数，它的作用是判断输入的两个类型是否一样

```cpp
// 定义is_same模板类，它接受两个模板参数T和U，它在类内定义了一个叫value的bool静态常量字段，值总是false
template<class T, class U>
struct is_same {
  static constexpr bool value = false;
};
// 对is_same偏特化，当T和U这两个类型一样时，它在类内定义了一个叫value的bool静态常量字段，值总是true
template<class T>
struct is_same<T, T> {
  static constexpr bool value = true;
};

// 定义一个类型别名，将is_same<int, int>绑定到Result1
// 从模板元编程的角度来看，这里可以看作是调用了元函数is_same，输入类型int和类型int，输出类型is_same<int, int>，赋值给Result1
using Result1 = is_same<int, int>;

// Result1就是类型is_same<int, int>，它有一个叫value的bool静态常量字段
// 由于is_same<int, int>的两个模板参数都是int，所以这里是偏特化后的版本，value的值是true
static_assert(Result1::value);

// 调用元函数is_same，输入类型int和类型float，输出类型is_same<int, float>，赋值给Result2
using Result2 = is_same<int, float>;

// 由于is_same<int, float>的两个模板参数不一样，所以这里没有偏特化，value的值是false
static_assert(!Result2::value);
```

#### 3、从输出类型中提取类型别名
这里我们来实现std::add_pointer元函数，它的作用是为输入的类型T增加一级指针（以下实现不完善，例如没有考虑T为引用的情况）

```cpp
// 定义add_pointer模板类，它接受一个模板参数T，它在类内定义了一个叫type的类型别名，值为T*
template<class T>
struct add_pointer {
  using type = T*;
};

// 调用元函数add_pointer，输入类型int，输出类型add_pointer<int>，赋值给Result
using Result = add_pointer<int>;

// Result就是类型add_pointer<int>，它里面有一个叫type的类型别名，值是int*
static_assert(is_same<int*, Result::type>::value);
```

#### 4、模板元编程与普通编程的差异

### （三）条件分支

#### 1、模板特化
在模板元编程里，我们可以将模板特化看作是一种“条件分支”。上面定义的is_same就已经用了模板特化：
```cpp
// 默认情况是false，相当于else分支
template<class T, class U>
struct is_same {
  static constexpr bool value = false;
};
// 当两个模板参数对应的类型相同时，value的值是true，相当于if分支
template<class T>
struct is_same<T, T> {
  static constexpr bool value = true;
};
下面的lua_to_native是从luna库抽出来的代码（未修改）：

// lua_to_native函数的作用是将lua对象转换成C++对象
// 默认情况下，会转发给lua_to_object函数进行处理
template <typename T>
T lua_to_native(lua_State* L, int i) { return lua_to_object<T>(L, i); }

// 根据C++的各种基本类型进行特化
template <> inline bool lua_to_native<bool>(lua_State* L, int i) { return lua_toboolean(L, i) != 0; }
template <> inline char lua_to_native<char>(lua_State* L, int i) { return (char)lua_tointeger(L, i); }
template <> inline unsigned char lua_to_native<unsigned char>(lua_State* L, int i) { return (unsigned char)lua_tointeger(L, i); }
template <> inline short lua_to_native<short>(lua_State* L, int i) { return (short)lua_tointeger(L, i); }
template <> inline unsigned short lua_to_native<unsigned short>(lua_State* L, int i) { return (unsigned short)lua_tointeger(L, i); }
template <> inline int lua_to_native<int>(lua_State* L, int i) { return (int)lua_tointeger(L, i); }
template <> inline unsigned int lua_to_native<unsigned int>(lua_State* L, int i) { return (unsigned int)lua_tointeger(L, i); }
template <> inline long lua_to_native<long>(lua_State* L, int i) { return (long)lua_tointeger(L, i); }
template <> inline unsigned long lua_to_native<unsigned long>(lua_State* L, int i) { return (unsigned long)lua_tointeger(L, i); }
template <> inline long long lua_to_native<long long>(lua_State* L, int i) { return lua_tointeger(L, i); }
template <> inline unsigned long long lua_to_native<unsigned long long>(lua_State* L, int i) { return (unsigned long long)lua_tointeger(L, i); }
template <> inline float lua_to_native<float>(lua_State* L, int i) { return (float)lua_tonumber(L, i); }
template <> inline double lua_to_native<double>(lua_State* L, int i) { return lua_tonumber(L, i); }
template <> inline const char* lua_to_native<const char*>(lua_State* L, int i) { return lua_tostring(L, i); }
template <> inline std::string lua_to_native<std::string>(lua_State* L, int i) {
  const char* str = lua_tostring(L, i);
  return str == nullptr ? "" : str;
}
```
- 是C++模板的基础功能
- 只能指定特定类型进行特化，不能进行范围选择以及进一步的条件判断
- 需要注意：只有模板类（以及C++14的模板变量）才有偏特化，而模板函数只有全特化。由于全特化不够灵活，所以一般来说模板函数不会选择使用模板特化来做条件判断，常用的方案有：tag dispatch、模板类静态函数、std::enable_if和if constexpr

#### 2、SFINAE
```
Substitution Failure Is Not An Error（替换失败并非错误）
```
所谓替换，是指将模板参数代入到模板的过程。在某些情况下，如果替换后会生成无效代码，SFINAE规则规定编译器此时不应该抛出错误，而应该继续尝试其他可用的重载版本

下面的has_member_gc是从luna库抽出来的代码（经过修改，将一些高版本C++才支持的模板元编程技巧替换成更基础的技巧）：
```cpp
// has_member_gc用来判断一个类T是否定义了成员函数void __gc()
template<class T>
struct has_member_gc {
  // 声明辅助模板类sfinae，它接受两个模板参数：U以及U的成员函数指针常量（签名为void()）
  template<class U, void (U::*)()>
  struct sfinae;

  // 声明辅助模板函数test，它有一个模板参数U，它有一个函数参数叫unused，unused的类型是sfinae<U, &U::__gc>*，返回类型char
  template<class U>
  static char test(sfinae<U, &U::__gc>* unused);

  // 声明辅助模板函数test的一个重载版本，它有一个模板参数，函数参数是可变参数，返回类型int
  template<class>
  static int test(...);

  // 核心的判断，考虑函数的调用：test<T>(nullptr)
  // test函数有两个重载，根据重载规则，可变参数的重载优先级是最低的，所以会优先考虑第一个重载版本
  // 编译器会先试着实例化sfinae<T, &T::__gc>，这里分两种情况考虑：
  // 1、如果T定义了成员函数void __gc()
  //   则sfinae<T, &T::__gc>是一个合法的类型，最终会调用test的第一个版本，所以test的返回类型是char
  //   这个时候，sizeof(返回类型) == sizeof(char)，value的值为true
  // 2、否则
  //   sfinae<T, &T::__gc>不是一个合法的类型，根据SFINAE规则，编译器不会报错，继续去找下一个重载版本
  //   此时会调用到test(...)，返回类型是int
  //   这个时候，sizeof(返回类型) != sizeof(char)，value的值为false
  static constexpr bool value = sizeof(test<T>(nullptr)) == sizeof(char);
};

struct foo {
  void __gc() {}
};

static_assert(has_member_gc<foo>::value);
static_assert(!has_member_gc<int>::value);
```
- 是模板元编程最重要的组成部分之一
- 当需要判断某个类有没有某个成员变量、成员函数、内部类、内部类型别名等等时，几乎只能用SFINAE（或者用C++20的Concept）
- 对C++标准的要求很低，C++98也能使用
- 技巧奇多，代码晦涩难懂，不建议直接使用

#### 3、std::enable_if
std::enable_if是用SFINAE的方法，根据类型特性有条件地选择对应的重载函数

下面的lua_handle_gc是从luna库抽出来的代码（未修改）：
```cpp
// 定义enable_if模板类，它接受两个模板参数Pred和T，Pred是bool常量，T是一个默认为void的类型，它是空类
template<bool Pred, class T = void>
struct enable_if {
};

// 当Pred == true时的偏特化，此时它有一个叫type的类型别名，值为模板参数T
template<class T>
struct enable_if<true, T> {
  using type = T;
};

// 定义lua_handle_gc函数，我们希望：
// 1、默认情况，调用lua_handle_gc(obj)相当于调用delete obj
// 2、如果T定义了成员函数__gc()，则调用lua_handle_gc(obj)相当于调用obj->__gc()
template <class T>
typename enable_if<!has_member_gc<T>::value>::type lua_handle_gc(T* obj) {
  delete obj;
}
template <class T>
typename enable_if<has_member_gc<T>::value>::type lua_handle_gc(T* obj) {
  obj->__gc();
}

struct foo {
  void __gc() {}
};

void main() {
  // 由于foo定义了成员函数__gc()，has_member_gc<foo>::value == true
  // 考虑lua_handle_gc的两个重载，将has_member_gc<foo>::value == true代入
  // 重载1：typename enable_if<false>::type lua_handle_gc(foo* obj);
  //   这里enable_if<false>用的是默认版本，它是一个空类，没有叫type的类型别名，替换失败
  //   根据SFINAE规则，重载1会被抛弃
  // 重载2：typename enable_if<true>::type lua_handle_gc(foo* obj);
  //   这里enable_if<true>用的是偏特化的版本，它有一个叫type的类型别名，值为void，替换成功
  // 所以，只有重载2是合法的，这里最终会调用到重载2的版本
  lua_handle_gc(new foo);

  // 类似地，这里只有重载1是合法的，最终会调用到重载2的版本
  lua_handle_gc(new int);
}
```
- 主要用于函数重载
- C++选择重载函数的规则非常复杂（还需考虑模板函数的特化），enable_if可以指导应该怎么选择
- 对C++标准的要求很低，C++98也能使用
- 因为一般来说重载函数的优先级很难控制，所以多个enable_if限定的范围不能重叠，写上去非常繁琐

#### 4、if constexpr
if constexpr可以理解成是强化版的std::enable_if。在编译期计算if语句的条件值常量，进行选择性编译

用if constexpr修改一下std::enable_if的例子：
```cpp
template <class T>
void lua_handle_gc(T* obj) {
  if constexpr (has_member_gc<T>::value) {
    obj->__gc();
  } else {
    // 假设T没有定义成员函数__gc()，则会进入else分支
    // 如果是普通的if语句，则两个分支都会被编译器编译，但是由于T没有定义成员函数__gc()，会编译出错
    // 而使用if constexpr语句的话，编译器会丢弃其他分支，编译正常通过
    delete obj;
  }
}
```
- 可以用类似if语句的方式控制哪部分需要编译以及哪部分丢弃，非常灵活
- 可以覆盖大部分enable_if的应用场景，甚至比enable_if做得更好
- 需要C++17的支持

### （四）模板元编程的相关技术总览
- C++98
  - sizeof
  - SFINAE
  - tag dispatch
  - boost::mpl (Meta Programming Library)
  - Meta functions
- C++11
  - constexpr
  - decltype
  - Expression SFINAE
  - Template aliases
  - Trailing function return types
  - Type triats
  - std::enable_if
  - Variadic template
- C++14
  - decltype(auto)
  - Extended constexpr
  - Variable templates
  - boost::hana
- C++17
  - if constexpr
  - Fold expressions
- C++20
  - Concepts

## 三、luna库扩展
### （一）支持enum
luna不支持enum，当待绑定的C++函数签名包含enum类型时，会编译出错：
```cpp
enum Color { Red, Green, Blue };

static void set_color(Color color) {}

void main() {
  lua_State* L = luaL_newstate();
  luaL_openlibs(L);

  lua_register_function(L, "set_color", &set_color);
  luaL_dostring(L, "set_color(1)");

  lua_close(L);
}

// oops！编译出错：
// error: 'lua_to_object': no matching overloaded function found
```
这是因为lua_to_object只对T为指针或引用时做了处理

增加对enum的支持很简单，只需在lua_to_native里加几行代码：
```cpp
template <class T>
T lua_to_native(lua_State* L, int i) {
  // std::is_enum元函数用来判断T是不是enum
  if constexpr (std::is_enum<T>::value) {
    // std::underlying_type元函数用来获得enum对应的整数类型
    using IntegerType = typename std::underlying_type<T>::type;
    return static_cast<T>(lua_to_native<IntegerType>(L, i));
  } else {
    return lua_to_object<T>(L, i);
  }
}
```

### （二）支持注册lua callback
参照对enum的处理，这里实现的关键是在lua_to_native函数里增加对std::function的判断和处理

#### 1、lua_to_native里增加对std::function的判断和处理：
```cpp
template <class T>
T lua_to_native(lua_State* L, int i) {
  if constexpr (is_std_function<T>::value) {
    // 如果T是std::function，则构建一个lua_function_adapter函数对象放到std::function里返回
    // 当lua_function_adapter被调用时，会调用到对应的lua函数
    return lua_function_adapter<T>(L, i);
  } else if constexpr (std::is_enum<T>::value) {
    using IntegerType = typename std::underlying_type<T>::type;
    return static_cast<T>(lua_to_native<IntegerType>(L, i));
  } else {
    return lua_to_object<T>(L, i);
  }
}
```

#### 2、实现is_std_function元函数
is_std_function元函数用来判断T是不是std::function
```cpp
// 默认情况是false
template <class T>
struct is_std_function {
  static constexpr bool value = false;
};

// std::function<Ret(Args...)>偏特化，值是true
template <class Ret, class... Args>
struct is_std_function<std::function<Ret(Args...)>> {
  static constexpr bool value = true;
};
```

#### 3、实现lua_function_adapter函数对象
lua_function_adapter是一个函数对象，当它被调用时，会调用到预先注册好的lua函数里
```cpp
// 这个类用来保存lua主线程lua_State以及lua callback在注册表的位置
struct lua_function_holder {
  lua_State* main_state;
  int reg_index;

  lua_function_holder(lua_State* L, int i) {
    // 保存和恢复L的栈顶位置
    lua_guard _(L);

    // 取出lua主线程的lua_State压入栈，因为传进来的L有可能是一个子协程
    lua_rawgeti(L, LUA_REGISTRYINDEX, LUA_RIDX_MAINTHREAD);

    // 保存下主线程的lua_State
    main_state = lua_tothread(L, -1);

    // 将lua callback复制一份压入栈
    lua_pushvalue(L, i);

    // 将lua callback注册到注册表，位置保存在reg_index
    reg_index = luaL_ref(main_state, LUA_REGISTRYINDEX);
  }

  ~lua_function_holder() {
    // 注销注册表里的lua callback
    luaL_unref(main_state, LUA_REGISTRYINDEX, reg_index);
  }
};

// 适配器类的声明，没必要实现
template <class T>
struct lua_function_adapter;

// std::function<Ret(Args...)>偏特化
template <class Ret, class... Args>
struct lua_function_adapter<std::function<Ret(Args...)>> {
  // lua_function_adapter最终会放到std::function里
  // 而std::function要求函数对象必须CopyConstructible
  // 所以这里用了std::shared_ptr保证资源释放
  std::shared_ptr<lua_function_holder> holder;

  lua_function_adapter(lua_State* L, int i)
    : holder(std::make_shared<lua_function_holder>(L, i)) {}

  Ret operator()(Args... args) {
    lua_State* L = holder->main_state;

    // 保存和恢复L的栈顶位置
    lua_guard _(L);

    // 从注册表取出lua callback
    lua_rawgeti(L, LUA_REGISTRYINDEX, holder->reg_index);

    // 将C++参数转换成lua对象，这里最终会展开成：
    // int unused[] = { 0, (native_to_lua(L, arg0), 0), (native_to_lua(L, arg1), 0), (native_to_lua(L, arg2), 0), ... };
    // 作用相当于依次调用：native_to_lua(L, arg0), native_to_lua(L, arg1), native_to_lua(L, arg2), ...
    // 这种用法在luna库源码的其他地方也有用到
    // 使用C++17 Fold expressions可以简化为以下形式：
    // (native_to_lua(L, args), ...);
    int unused[] = { 0, (native_to_lua(L, args), 0)... };

    if constexpr (!std::is_same<Ret, void>::value) {
      // 调用lua callback，返回一个值
      lua_call(L, sizeof...(Args), 1);

      // 将返回值转换成C++对象
      return lua_to_native<Ret>(L, 1);
    }
    else {
      // 调用lua callback，没有返回值
      lua_call(L, sizeof...(Args), 0);
    }
  }
};
```

#### 4、应用示例
```cpp
using Callback = std::function<std::string(int, float)>;

static Callback callback;

static void set_callback(Callback value) {
  callback = std::move(value);
}

void main() {
  lua_State* L = luaL_newstate();
  luaL_openlibs(L);

  lua_register_function(L, "set_callback", &set_callback);

  // 设置lua callback
  luaL_dostring(L, R"(
    set_callback(function(i, f)
      print(string.format("Print in Lua, args: %d, %f", i, f))
      return "I'm Lua callback"
    end)
  )");
  std::string ret = callback(123, 0.125f);
  std::cout << "Callback result: " << ret << std::endl;

  // 设置C++ callback
  set_callback([](int i, float f) {
    std::cout << "Print in C++, args: " << i << ", " << f << std::endl;
    return "I'm C++ callback";
  });
  ret = callback(456, -8.f);
  std::cout << "Callback result: " << ret << std::endl;

  callback = nullptr;
  lua_close(L);
}

// 程序输出：
// Print in Lua, args: 123, 0.125000
// Callback result : I'm Lua callback
// Print in C++, args : 456, -8
// Callback result : I'm C++ callback
```

## 四、进阶
### (一) lua callback支持返回多个值
应用示例
```cpp
using Callback = std::function<std::tuple<int, float, std::string>(int, float)>;

static Callback callback;

static void set_callback(Callback value) {
  callback = std::move(value);
}

void main() {
  lua_State* L = luaL_newstate();
  luaL_openlibs(L);

  lua_register_function(L, "set_callback", &set_callback);

  luaL_dostring(L, R"(
    set_callback(function(i, f)
      print(string.format("Print in Lua, args: %d, %f", i, f))
      return i + 100, f * 8, "I'm Lua callback"
    end)
  )");
  auto [res0, res1, res2] = callback(123, 0.125f);
  std::cout << "Callback result: " << res0 << ", " << res1 << ", " << res2 << std::endl;

  callback = nullptr;
  lua_close(L);
}

// 程序输出：
// Print in Lua, args: 123, 0.125000
// Callback result : 223, 1, I'm Lua callback
```

核心实现代码：
```cpp
template <class T>
struct is_tuple {
  static constexpr bool value = false;
};
template <class... Ts>
struct is_tuple<std::tuple<Ts...>> {
  static constexpr bool value = true;
};

template <class T>
struct tag {};

template <class... Ts, size_t... Is>
std::tuple<Ts...> lua_to_tuple(lua_State* L, int i, tag<std::tuple<Ts...>>, luna_sequence<Is...>) {
  return { lua_to_native<Ts>(L, i + Is)... };
};

template <class Ret, class... Args>
struct lua_function_adapter<std::function<Ret(Args...)>> {
  std::shared_ptr<lua_function_holder> holder;

  lua_function_adapter(lua_State* L, int i)
    : holder(std::make_shared<lua_function_holder>(L, i)) {}

  Ret operator()(Args... args) {
    lua_State* L = holder->main_state;
    lua_guard _(L);
    lua_rawgeti(L, LUA_REGISTRYINDEX, holder->reg_index);
    int unused[] = { 0, (native_to_lua(L, args), 0)... };

    if constexpr (is_tuple<Ret>::value) {
      // tuple的元素个数
      constexpr std::size_t TUPLE_SIZE = std::tuple_size<Ret>::value;

      // 调用lua callback，返回TUPLE_SIZE个值
      lua_call(L, sizeof...(Args), TUPLE_SIZE);

      // 将返回的多个lua对象转换成std::tuple
      return lua_to_tuple(L, 1, tag<Ret>{}, make_luna_sequence<TUPLE_SIZE>{});
    }else if constexpr (!std::is_same<Ret, void>::value) {
      lua_call(L, sizeof...(Args), 1);
      return lua_to_native<Ret>(L, 1);
    }else {
      lua_call(L, sizeof...(Args), 0);
    }
  }
};
```

### (二) 编译器只支持C++11时的替换方案：不使用if constexpr，改用tag dispatch
```cpp
template <class T>
struct tag {};

// 定义lua_to_native_impl函数的多个重载，使用tag dispatch做条件判断
// 注意这里没有使用模板特化
template <class T>
T lua_to_native_impl(lua_State* L, int i, tag<T>, ...) {
    return lua_to_object<T>(L, i);
}
template <class Ret, class... Args>
std::function<Ret(Args...)> lua_to_native_impl(lua_State* L, int i, tag<std::function<Ret(Args...)>>) {
    return lua_function_adapter<std::function<Ret(Args...)>>(L, i);
}
template <class T>
typename std::enable_if<std::is_enum<T>::value, T>::type lua_to_native_impl(lua_State* L, int i, tag<T>) {
    using IntegerType = typename std::underlying_type<T>::type;
    return static_cast<T>(lua_to_native<IntegerType>(L, i));
}
inline bool lua_to_native_impl(lua_State* L, int i, tag<bool>) {
    return lua_toboolean(L, i) != 0;
}
inline char lua_to_native_impl(lua_State* L, int i, tag<char>) {
    return (char)lua_tointeger(L, i);
}
inline unsigned char lua_to_native_impl(lua_State* L, int i, tag<unsigned char>) {
    return (unsigned char)lua_tointeger(L, i);
}
inline short lua_to_native_impl(lua_State* L, int i, tag<short>) {
    return (short)lua_tointeger(L, i);
}
inline unsigned short lua_to_native_impl(lua_State* L, int i, tag<unsigned short>) {
    return (unsigned short)lua_tointeger(L, i);
}
inline int lua_to_native_impl(lua_State* L, int i, tag<int>) {
    return (int)lua_tointeger(L, i);
}
inline unsigned int lua_to_native_impl(lua_State* L, int i, tag<unsigned int>) {
    return (unsigned int)lua_tointeger(L, i);
}
inline long lua_to_native_impl(lua_State* L, int i, tag<long>) {
    return (long)lua_tointeger(L, i);
}
inline unsigned long lua_to_native_impl(lua_State* L, int i, tag<unsigned long>) {
    return (unsigned long)lua_tointeger(L, i);
}
inline long long lua_to_native_impl(lua_State* L, int i, tag<long long>) {
    return lua_tointeger(L, i);
}
inline unsigned long long lua_to_native_impl(lua_State* L, int i, tag<unsigned long long>) {
    return (unsigned long long)lua_tointeger(L, i);
}
inline float lua_to_native_impl(lua_State* L, int i, tag<float>) {
    return (float)lua_tonumber(L, i);
}
inline double lua_to_native_impl(lua_State* L, int i, tag<double>) {
    return lua_tonumber(L, i);
}
inline const char* lua_to_native_impl(lua_State* L, int i, tag<const char*>) {
    return lua_tostring(L, i);
}
inline std::string lua_to_native_impl(lua_State* L, int i, tag<std::string>) {
    const char* str = lua_tostring(L, i);
    return str == nullptr ? "" : str;
}

// lua_to_native函数，用tag将T的类型传递给lua_to_native_impl
template <class T>
T lua_to_native(lua_State* L, int i) {
    return lua_to_native_impl(L, i, tag<T>{});
}
```
