---
title: python 内置函数 数学运算
date: 2021-04-17 18:28:19
categories:
- [python, 内置函数]
- [programing_language, 比较学习]
tags:
- Python
- Java
- C++
excerpt: 本文介绍python的内置函数，并且和Java、C++进行了对比
---


## 数学运算

在python中提供了7个内置的数学运算函数
在java中标准库提供了相似的函数
在c++中标准库提供了相似的函数

1. abs

    功能：求数值的绝对值

    python中作为内置函数，参数可以是任何实现了\_\_abs\_\_()的对象，例如int、float等。
    
    函数原型：[参考文献](https://docs.python.org/zh-cn/3/library/functions.html#abs)
    
    ```python
    abs(x)
    ```

    ```python
    a = 1
    print(a, type(a), abs(a))
    b = -1.0
    print(b, type(b), abs(b))
    c = 2+0j
    print(c, type(c), abs(c))
    class X:
        def __init__(self, _x):
            self._x = _x

        def __abs__(self) :
            if self._x < 0 : return -self._x
            return self._x


    d = X(-2)
    print(d, type(d), abs(d))
    ```
    ```shell
    $ python test-abs.py
    (1, <type 'int'>, 1)
    (-1.0, <type 'float'>, 1.0)
    ((2+0j), <type 'complex'>, 2.0)
    (<__main__.X instance at 0x0000000003B38488>, <type 'instance'>, 2)
    ```

    
    java中abs位于java.lang.Math包中，参数可以是任何原生数据类型, 可以是int, float, long, double, short, byte类型。
    
    函数原型：[参考文献](https://docs.oracle.com/javase/8/docs/api/java/lang/Math.html#abs-double-)
    ```java
    public static int abs(int a)
    public static long abs(long a)
    public static float abs(float a)
    public static double abs(double a)
    ```
    ```java
    public class test_abs{
        public static void main(String args[]){
            Integer a = -1;
            double b = 2;
            long c = -9223372036854775808L;
            long d = -9223372036854775807L;
            System.out.println(Math.abs(a));
            System.out.println(Math.abs(b));
            System.out.println(Math.abs(c));
            System.out.println(Math.abs(d));
        }
    }
    ```
    注：abs(c)超过了long的表示范围，发生了溢出
    ```shell
    $ javac test_abs.java
    $ java test_abs
    1
    2.0
    -9223372036854775808
    9223372036854775807
    ```

    c++中abs位于cmath中。
    
    函数原型
    [参考](http://www.cplusplus.com/reference/cmath/abs/)
    ```c++
    double abs (double x);
    float abs (float x);
    long double abs (long double x);
    double abs (T x);   
    ```
    ```c++
    #include <iostream>
    #include <cmath>
    int main() {
        int a = -1;
        double b = 2;
        long long c = -9223372036854775807L;
        using std::cout, std::endl;
        cout << abs(a) << endl;
        cout << abs(b) << endl;
        cout << abs(c) << endl;
    }
    ```
    注：abs不支持long long类型，会发生隐式类型转换变成int类型，从而丢失精度
    ```shell
    $ g++ -g test_abs.cpp -o test_abs.exe -std=c++17
    $ test_abs.exe
    1
    2
    1
    ```


2. divmod

    功能：返回两个数值的商和余数 [参考](https://docs.python.org/zh-cn/3/library/functions.html#divmod)
    
    ```python
    import math
    a1 = divmod(5, 2)
    a2 = (5 // 2, 5 % 2)
    print(a1, type(a1), a2, type(a2))
    b1 = divmod(-5.5, 2.1)
    b2 = (math.floor(-5.5/2.1), -5.5 % 2.1)
    print(b1, type(b1), b2, type(b2))
    c1 = divmod(5.5, -2.1)
    c2 = (math.floor(5.5/-2.1), 5.5 % -2.1)
    print(c1, type(c1), c2, type(c2))
    ```

    ```shell
    $ python test_divmod.py
    ((2, 1), <type 'tuple'>, (2, 1), <type 'tuple'>)
    ((-3.0, 0.8000000000000003), <type 'tuple'>, (-3.0, 0.8000000000000003), <type 'tuple'>)
    ((-3.0, -0.8000000000000003), <type 'tuple'>, (-3.0, -0.8000000000000003), <type 'tuple'>)
    ```

    java和c++没有类似的函数

3. max

    功能：返回可迭代对象中的元素中的最大值或者所有参数的最大值

    函数原型[参考](https://docs.python.org/zh-cn/3/library/functions.html#max)
    ```python
    max(iterable, *[, key, default])
    max(arg1, arg2, *args[, key])
    ```

    python作为众所周知的弱类型语言，比较操作是会容易出现问题[参考](https://docs.python.org/3.9/reference/expressions.html#value-comparisons)

    对于float类型，float("nan")与任何浮点类型比较都会返回False，float("nan")!=float("nan")返回True

    对于set类型， Accordingly, sets are not appropriate arguments for functions which depend on total ordering (for example, min(), max(), and sorted() produce undefined results given a list of sets as inputs).

    对于1和True，type(max(1, True)) != type(max(true, 1))
    ```python
    def test(a, b) :
        print(max(a, b), max(b, a), type(max(a, b)), type(max(b, a)), max(a, b) == max(b, a))

    test(1, 2)
    test(1.0, float("nan"))
    test({1}, {2})
    test(1, True)
    print(max(1, 2, 3))
    print(max((1, 2, 3, 4)))
    print(max([1, 2, 3, 4, 5]))
    print(max({1, 2, 3, 4, 5, 6}))
    print(max({1: 6, 3: 4, 5: 2}))
    print(max(["1","2","3","4"], key = len))
    print(max({-100, 1, 10}, key = lambda x: abs(x)))
    ```
    ```shell
    $ python test_max.py
    2 2 <class 'int'> <class 'int'> True
    1.0 nan <class 'float'> <class 'float'> False
    {1} {2} <class 'set'> <class 'set'> False
    1 True <class 'int'> <class 'bool'> True
    3
    4
    5
    6
    5
    1
    -100
    ```
    
    在java中，
    java.lang.Math包中提供了max，参数可以两个原生数据类型。[参考](https://docs.oracle.com/javase/8/docs/api/java/lang/Math.html#max-int-int-)
    java.util.Collections包中提供了max, 函数原型[参考](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#max-java.util.Collection-)
    ```java
    public static <T extends Object & Comparable<? super T>> T max(Collection<? extends T> coll)
    public static <T> T max(Collection<? extends T> coll, Comparator<? super T> comp)
    ```
    ```java
    import java.util.*;
    public class test_max{
        public static void main(String[] argv) {
            List<Integer> list = new ArrayList<>();
            list.add(1);
            list.add(-2);
            list.add(3);
            list.add(-4);
            System.out.println(Collections.max(list));
            System.out.println(Collections.max(list, (a, b) -> {return Math.abs(a) - Math.abs(b);}));
        }
    }
    ```
    ```shell
    $ javac test_max.java
    $ java test_max
    3
    -4
    ```

    在c++中，algorithm提供了max函数，函数原型如下：
    ```c++
    template <class T> constexpr const T& max (const T& a, const T& b);
    template <class T, class Compare> constexpr const T& max (const T& a, const T& b, Compare comp);
    template <class T> constexpr T max (initializer_list<T> il);
    template <class T, class Compare> constexpr T max (initializer_list<T> il, Compare comp);
    ```
    还提供了max_element， 函数原型如下：
    ```c++
    template <class ForwardIterator> ForwardIterator max_element (ForwardIterator first, ForwardIterator last);
    template <class ForwardIterator, class Compare> ForwardIterator max_element (ForwardIterator first, ForwardIterator last,Compare comp);
    ```
    ```c++
    #include <iostream>
    #include <algorithm>
    #include <set>
    int main () {
        using std::cout;
        using std::endl;
        using std::max;
        cout << max(1, 2) << endl;
        struct CMP{
            bool operator()(int x, int y) const{
                return abs(x) < abs(y);
            }
        } cmp;
        cout << max(-10, 1, cmp) << endl;
        cout << max({-4, 1, 2, 3}) << endl;
        cout << max({-4, 1, 2, 3}, cmp) << endl;
        std::set<int> st{10, -20, 30, -40};
        cout << *std::max_element(st.begin(), st.end()) << endl;
        cout << *std::max_element(st.begin(), st.end(), cmp) << endl;
        return 0;
    }
    ```
    ```shell
    $ g++ -g test_max.cpp -o test_max.exe --std=c++14
    2
    -10
    3
    -4
    30
    -40
    ```

4. min

    功能：返回可迭代对象中的元素中的最小值或者所有参数的最小值
    和max极其类似

5. pow
    
    功能：返回两个数值的幂运算值或其与指定整数的模值

    函数原型[参考](https://docs.python.org/zh-cn/3/library/functions.html#pow)
    ```python
    pow(base, exp[, mod])
    ```
    在 3.8 版更改: 对于 int 操作数，三参数形式的 pow 现在允许第二个参数为负值，即可以计算倒数的余数。允许关键字参数。 之前只支持位置参数。
    ```python
    print(pow(2, 10))
    print(pow(2, -1))
    print(pow(2, 0.5))
    print(pow(2, 10, 97))
    print(pow(2, -1, 97))
    ```
    注：pow(2, -1, 97)必须在3.8或者更高版本下执行
    ```shell
    $ python test_pow.py
    1024
    0.5
    1.41421356237
    54
    23
    ```

    java中java.lang.Math提供了类似的方法
    函数原型[参考](https://docs.oracle.com/javase/8/docs/api/java/lang/Math.html#pow-double-double-)
    ```java
    static double	pow(double a, double b)
    ```

    c++中cmath提供了类似的函数
    函数原型[参考](http://www.cplusplus.com/reference/cmath/pow/)
    ```cpp
    double pow (double base, double exponent);
    float pow (float base, float exponent);
    long double pow (long double base, long double exponent);
    double pow (Type1 base, Type2 exponent);
    ```

6. round

	返回 number 舍入到小数点后 ndigits 位精度的值。 如果 ndigits 被省略或为 None，则返回最接近输入值的整数。
    
    函数原型[参考](https://docs.python.org/zh-cn/3/library/functions.html#round)

    ```python
    round(number[, ndigits])
    ```

    java函数原型[参考](https://docs.oracle.com/javase/8/docs/api/java/lang/Math.html#round-double-)
    ```java
    """
    If the argument is NaN, the result is 0.
    If the argument is negative infinity or any value less than or equal to the value of Integer.MIN_VALUE, the result is equal to the value of Integer.MIN_VALUE.
    If the argument is positive infinity or any value greater than or equal to the value of Integer.MAX_VALUE, the result is equal to the value of Integer.MAX_VALUE.
    """
    static int	round(float a)
    static long	round(double a)
    static double	scalb(double d, int scaleFactor)
    static float	scalb(float f, int scaleFactor)
    ```

    c++函数原型[参考](http://www.cplusplus.com/reference/cmath/round/)

7. sum
    对元素类型是数值的可迭代对象中的每个元素求和
    
    函数原型[参考](https://docs.python.org/zh-cn/3/library/functions.html#sum)

    ```python
    sum(iterable, /, start=0)
    ```