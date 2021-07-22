---
title: 标准库-容器浅析-3-顺序容器-C++-vector
date: 2021-04-25 15:23:54
categories:
- [C++, 标准库]
- [programing_language, 比较学习]
tags:
- C++
excerpt: d
---
# C++顺序容器浅析

## Vector 容器

### vector介绍

标准库类型vector表示对象的集合，其中所有对象的类型都相同。集合中的每个对象都有一个与之对应的索引，索引用于访问对象。

从实现上看，vector是一个类模板。模板可以看作编译器生成类或者函数的一份说明，编译器根据模板创建类或者函数的过程被称作实例化，使用模板的时候，需要指出编译器应该把类或者函数实例化为何种类型。

在C++中容器是使用模板编写的，vector、deque、list、forward_list等顺序容器的模板参数为
```c++
template < class T, class Alloc = allocator<T> >
```

STL容器采用分配器来完成对内存空间的动态分配，迭代器可以访问容器的底层实现，即迭代器可以访问分配器。

分析包括容器的常见接口、容器迭代器

LLVM是C++标准的一个实现。
源码的github地址: [https://github.com/llvm/llvm-project/tree/main/libcxx/include](https://github.com/llvm/llvm-project/tree/main/libcxx/include)

GNU G++ 是C++标准的一个实现,
源码的github地址: [https://github.com/gcc-mirror/gcc/tree/releases/gcc-11/libstdc%2B%2B-v3/include](https://github.com/gcc-mirror/gcc/tree/releases/gcc-11/libstdc%2B%2B-v3/include)
### vector的定义和初始化

| 例子| 说明 |
| ---|--- |
| vector<T> v1 | v1是一个空的vector, 他潜在的元素是T类型的，执行默认初始化 |
| vector<T> v2(v1) | v2中包含有v1的全部副本 |
| vector<T> v2 = v1 | 等价于v2(v1), v2中包含v1的全部副本 |
| vecior<T> v3(n, val) | v3包含了n个重复元素，每个元素的值都是val |
| vector<T> v4(n) | v4包含了n个重复地执行了值初始化的对象 |
| vector<T> v5{a,b,c,...} | v5包含了初始值个数的元素，每个元素被赋予相应的初始值 |
| vector<T> v6={a,b,c,...} | 等价于v5{a,b,c,...} |

如果使用花括号，初始化过程会尽可能的将花括号里的值当成元素初始值的列表来处理，只有当无法执行列表初始化才会考虑其他初始化方式。如果使用了花括号的方式又不能用来初始化列表就会考虑用这样的值来构造对象了。
```cpp
vector<string> v1{"hi"}; // 列表初始化，v1有一个元素
//vector<string> v2("hi");Error
vector<string> v3{10} //等价于vector<string> v3(10), v3有10个元素。
vector<string> v4{10, "hi"}; //等价于vector<string> v4(10, "hi"), v4由10个元素。
```

### vector的设计

普通数组的一个缺陷是，数组大小是固定的，预先申请空间很难满足实际需要，而利用指针动态申请与管理空间容易引发各种问题。而vector应该能够自动管理内存空间，我们很有可能想到使用内存池，从降低耦合的角度来讲，把内存的管理抽象为独立对象是可行的解决方案。C++ STL组件分配器就是用于分配管理内存空间的，默认情况下，vector使用的是allocator<_Tp>。

采用分配器去管理内存，在动态管理内存的思路下，预分配空间是一个很常见的策略。vector采用了预分配空间的做法。

vector的职责是高效存储数据，从单一职责的角度来看将遍历所有元素的行为抽象成独立对象时有更好的，因此引入了迭代器。迭代器要能够访问vector的底层结构，也就是迭代器要能访问分配器，对于具体的分配器，要实现具体的迭代器。vector需要提供方法去返回迭代器，主要提供begin()、end()、cbegin()、cend()、rbegin()、rend()、crbegin()、crend()等方法。而内部就要保存begin、end。

考虑普通数组的不足，数组可以使用下标访问元素，不可避免的，这会产生越界，就是所谓的缓冲区溢出，数组越界可能引发各种意想不到的异常。而vector应该采用更加的安全方法，在vector中，使用at()去访问元素，并且发生越界的时候抛出异常。

#### LLVM的实现

以下是vector的部分类图

github源码：https://github.com/llvm/llvm-project/blob/main/libcxx/include/vector

{% plantuml %}
class __vector_base_common < template< bool > > {
    # __vector_base_common()
    # __throw_length_error() const
    # __throw_out_of_range() const
}
class __vector_base< template< typename _Tp,typename _Allocator> > << protected __vector_base_common< true > >>{
    # __begin_ : allocator_traits<_Allocator>::pointer
    # __end_: allocator_traits<_Allocator>::pointer
    # __end_cap_: __compressed_pair< allocator_traits<_Allocator>::pointer, _Allocator>

    # __alloc() : _Allocator& || const _Allocator&
    # __end_cap() : allocator_traits<_Allocator>::pointer || const allocator_traits<_Allocator>::pointer
    # __vector_base()
    # __vector_base(const allocator_type& __a);
    # __vector_base(allocator_type&& __a);
    # ~__vector_base()
    # clear(): void
    # capacity(): size_type
    # __destruct_at_end(pointer __new_last): void
    # __copy_assign_alloc(const __vector_base& __c): void
    # __move_assign_alloc(__vector_base& __c): void
}
__vector_base_common <|-- __vector_base
class vector< template< typename _Tp, typename _Allocator> > << private __vector_base<_Tp, _Allocator> >> {
    + vector(...) 各种构造函数-重载
    + ~vector()
    + swap(vector&) : void
    + size():  allocator_traits<_Allocator>::size_type 
    + max_size(): allocator_traits<_Allocator>::size_type 
    + empty(): bool
    + begin() : __wrap_iter< allocator_traits<_Allocator>::pointer> || __wrap_iter< allocator_traits<_Allocator>::const_pointer> 
    + end() : __wrap_iter< allocator_traits<_Allocator>::pointer> || __wrap_iter< allocator_traits<_Allocator>::const_pointer> 
    + cbegin() :  __wrap_iter< allocator_traits<_Allocator>::const_pointer> 
    + cend() : __wrap_iter< allocator_traits<_Allocator>::const_pointer> 

    + get_allocator() : _Allocator
    + at(): _Tp& || const _Tp&
    + data()： _Tp* || const _Tp*
    + front(): _Tp& || const _Tp&
    + back(): _Tp& || const _Tp&
    + push_back(const _Tp& __x): void
    + push_back(_Tp&& __x): void
    + emplace_back(_Args&&... __args) : _Tp&
    + pop_back(): void
    + insert(...): __wrap_iter< allocator_traits<_Allocator>::const_pointer> /各种插入
    + emplace(...): __wrap_iter< allocator_traits<_Allocator>::const_pointer> 各种插入
    + erase(const_iterator __position): __wrap_iter< allocator_traits<_Allocator>::const_pointer>
    + erase(const_iterator __first, const_iterator __last): __wrap_iter< allocator_traits<_Allocator>::const_pointer>
    + clear(): void
    + resize(): void
    + reserve(): void
    + shrink_to_fit(): void
    + assign(...) 各类assign函数-重载
    + rbegin(): _VSTD::reverse_iterator<__wrap_iter< allocator_traits<_Allocator>::pointer>> 
    + rend(): _VSTD::reverse_iterator<__wrap_iter< allocator_traits<_Allocator>::pointer>>
    + crbegin() : _VSTD::reverse_iterator<__wrap_iter< allocator_traits<_Allocator>::const_pointer>>
    + crend() : _VSTD::reverse_iterator<__wrap_iter< allocator_traits<_Allocator>::const_pointer>>
    {method}...
}
__vector_base <|-- vector
{% endplantuml %}


### vector的迭代器类型

#### 探究LLVM的iterator类型

探究vector的iterator具体类型，依据LLVM的源码进行分析。
分析iterator, github源码：https://github.com/llvm/llvm-project/blob/release/12.x/libcxx/include/vector#L488
```cpp
template <class _Tp, class _Allocator /* = allocator<_Tp> */>
class vector : private __vector_base<_Tp, _Allocator> {
    ...
    typedef __vector_base<_Tp, _Allocator> __base;
    typedef typename __base::pointer pointer;
    typedef __wrap_iter<pointer> iterator;
    ...
}
```
_wrap_iter是对pointer进行包装，然后关注这个pointer，这里的pointer是基类pointer的别名。
也就是vector的iterator类型为__wrap_iter<__vector_base<_Tp, _Allocator>::pointer>, 其中__Tp是容器元素类型，_Allocator是分配器。

分析__vector_base<_Tp, _Allocator>::pointer, github源码：https://github.com/llvm/llvm-project/blob/release/12.x/libcxx/include/vector#L339
```cpp
template <class _Tp, class _Allocator>
class __vector_base : protected __vector_base_common<true> {
    ...
    typedef _Allocator allocator_type;
    typedef allocator_traits<allocator_type> __alloc_traits;
    typedef typename __alloc_traits::pointer pointer;
    ...
}
```
allocator_traits<_Allocator>对于分配器_Allocator进行封装, 因此iterator的类型为:__wrap_iter< allocator_traits<_Allocator>::pointer>
分析allocator_traits<_Allocator>::pointer，github源码：https://github.com/llvm/llvm-project/blob/release/12.x/libcxx/include/__memory/allocator_traits.h#L231
```cpp
template <class _Alloc>
struct allocator_traits{
    using allocator_type = _Alloc;
    using value_type = typename allocator_type::value_type;
    using pointer = typename __pointer<value_type, allocator_type>::type;
    ...
}
```
这里又是一个别名，展开 __pointer< _Alloc::value_type, _Alloc>::type，而_Alloc就是模板参数_Allocator, 因此iterator的类型为 __wrap_iter< __pointer<_Alloc::value_type, _Alloc>::type>
分析__pointer<_Alloc::value_type, _Alloc>::type github源码地址：https://github.com/llvm/llvm-project/blob/release/12.x/libcxx/include/__memory/allocator_traits.h#L33
```cpp
template <class _Tp, class _Alloc, class _RawAlloc = typename remove_reference<_Alloc>::type, bool = __has_pointer<_RawAlloc>::value>
struct __pointer {
    using type = typename _RawAlloc::pointer;
};
template <class _Tp, class _Alloc, class _RawAlloc>
struct __pointer<_Tp, _Alloc, _RawAlloc, false> {
    using type = _Tp*;
};
```
关注第一个__pointer结构， 本处type作为_RawAlloc::pointer的引用，而_RawAlloc的默认参数为typename remove_reference<_Alloc>::type，而_Alloc就是模板参数_Allocator,
因此iterator的类型为__wrap_iter< remove_reference<_Allocator>::type::pointer >

分析remove_reference<_Allocator>[^3]
```cpp
template <typename T>
class remove_reference
{
public:
   typedef T type;
};
template<typename T>
class remove_reference<T&>
{
public:
   typedef T type;
};
```
remove_reference就是去除引用。因此iterator的类型为__wrap_iter< _Allocator::pointer >, 不过这是去除引用的_Alloctor

然后探究_Alloctor，默认分配器为alloctor<_Tp>
```cpp
template <class _Tp>
class _LIBCPP_TEMPLATE_VIS allocator{
    typedef _Tp* pointer;
}
```
因此，vector的迭代器类型就是__wrap_iter< _Tp* >.

__wrap_iter是一个ConcreteIterator，拿到了_Tp* 就可以对vector进行遍历了。对于_Tp*进行了封装，并提供了各种迭代器功能。
github源码：https://github.com/llvm/llvm-project/blob/main/libcxx/include/iterator#L1265

#### 探究GNU C++的iterator类型

分析iterator，github源码：https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/stl_vector.h#L419
```cpp
template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
class vector : protected _Vector_base<_Tp, _Alloc> {
    ...
    typedef _Vector_base<_Tp, _Alloc> _Base;
    typedef typename _Base::pointer pointer;
    typedef __gnu_cxx::__normal_iterator<pointer, vector> iterator;
    ...
}
```
iterator的类型为:__gnu_cxx::__normal_iterator<_Vector_base<_Tp, _Alloc>::pointer, vector>

继续分析_Vector_base<_Tp, _Alloc>::pointer，github源码：https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/stl_vector.h#L83
```cpp
template<typename _Tp, typename _Alloc>
struct _Vector_base{
    ...
    typedef typename __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer pointer;
    ...
}
```
iterator的类型为:__gnu_cxx::__normal_iterator< __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer, vector>

继续分析 __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer，github源码：https://github.com/gcc-mirror/gcc/blob/releases/gcc-11/libstdc%2B%2B-v3/include/ext/alloc_traits.h#L47
```cpp
template<typename _Alloc, typename = typename _Alloc::value_type>
struct __alloc_traits: std::allocator_traits<_Alloc>{
    ...
    typedef std::allocator_traits<_Alloc> _Base_type;
    typedef typename _Base_type::pointer pointer;
    ...
}
```
iterator的类型为:__gnu_cxx::__normal_iterator< std::allocator_traits<_Alloc>::pointer, vector>

继续分析std::allocator_traits<_Alloc>::pointer，由于_Alloc实际上是allocator<_Tp>， 采用部分实例化的版本。
github源码：https://github.com/gcc-mirror/gcc/blob/releases/gcc-11/libstdc%2B%2B-v3/include/bits/alloc_traits.h#L416
```cpp
template<typename _Tp>
struct allocator_traits<allocator<_Tp>> {
    ...
    /// The allocator's pointer type.
    using pointer = _Tp*;
    ...
}
```
iterator的类型为:__gnu_cxx::__normal_iterator<_Tp*, std::vector< _Tp , std::allocator<_Tp> > >
拿到了_Tp*，就可以对vector进行遍历了。


[^1]: C++容器详解 https://www.cnblogs.com/sea520/p/12711554.html  

Primer c++ 第5版 page375 

[^2]: basic footnote content
[^3]: https://blog.csdn.net/slslslyxz/article/details/105778734