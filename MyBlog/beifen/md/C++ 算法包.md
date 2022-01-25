C++ 算法包



## C++算法包内置源码全面研究  

1.for_each，自定义谓词

```c++
//stl_algo.h#for_each
template<typename _InputIterator, typename _Function>
_Function
for_each(_InputIterator __first, _InputIterator __last, _Function __f)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_requires_valid_range(__first, __last);
    for (; __first != __last; ++__first)
        __f(*__first);
    return __f; // N.B. [alg.foreach] says std::move(f) but it's redundant.
}
```

![image-20220108224747729](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108224747729.png)



2.transform

遍历打印和变换修改值

```c++
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

class unary_op{
public:
    int operator()(const int first){
        return first + 9;//将每个元素都增加9
    }
};

int main(){
    vector<int> vectorVar;
    vectorVar.insert(vectorVar.begin(), 10000);
    vectorVar.insert(vectorVar.begin(), 20000);
    vectorVar.insert(vectorVar.begin(), 30000);
    vectorVar.insert(vectorVar.begin(), 40000);
    //第三个参数可以是本身的容器，覆盖之前的值
    transform(vectorVar.begin(), vectorVar.end(), vectorVar.begin(), unary_op());//自定义仿函数unary_op
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++){
        cout << *it << endl;
    }
    //第三个参数可以是其他容器，新的一个容器
    vector<int> result;
    result.resize(vectorVar.size());//必须设置容器大小
    transform(vectorVar.begin(), vectorVar.end(), result.begin(), unary_op());
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++){
        cout << *it << endl;
    }
    
    return 0;
}
```



```c++
//stl_algo.h#transform
template<typename _InputIterator, typename _OutputIterator,
typename _UnaryOperation>
_OutputIterator
transform(_InputIterator __first, _InputIterator __last,
          _OutputIterator __result, _UnaryOperation __unary_op)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
                                    // "the type returned by a _UnaryOperation"
                                    __typeof__(__unary_op(*__first))>)
        __glibcxx_requires_valid_range(__first, __last);

    for (; __first != __last; ++__first, (void)++__result)
        *__result = __unary_op(*__first);
    return __result;
}
```

![image-20220108231101311](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20220108231101311.png)



3.find和find_if

find没有自定义仿函数，find_if有

find和find_if其实都是对__find_if的封装

```c++
//stl_algo.h#find
//开始的迭代器_InputIterator __first
//结尾的迭代器_InputIterator __last
//需要查找的元素const _Tp& __val
template<typename _InputIterator, typename _Tp>
inline _InputIterator
find(_InputIterator __first, _InputIterator __last,
     const _Tp& __val)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_EqualOpConcept<
                                    typename iterator_traits<_InputIterator>::value_type, _Tp>)
        __glibcxx_requires_valid_range(__first, __last);
    return std::__find_if(__first, __last,
                          __gnu_cxx::__ops::__iter_equals_val(__val));
}
```



```c++
//stl_algo.h#find_if
//开始的迭代器_InputIterator __first
//结尾的迭代器_InputIterator __last
//自定义仿函数_Predicate __pred
template<typename _InputIterator, typename _Predicate>
inline _InputIterator
find_if(_InputIterator __first, _InputIterator __last,
        _Predicate __pred)
{
    // concept requirements
    //用于检测工作
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_UnaryPredicateConcept<_Predicate,
                                    typename iterator_traits<_InputIterator>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);
	//这里传的第三个入参是predefined_ops.h中的__pred_iter，也是一个仿函数，仿函数参数为外部自定义的仿函数__pred
    return std::__find_if(__first, __last,
                          __gnu_cxx::__ops::__pred_iter(__pred));
}
```



```c++
//stl_algo.h#__find_if
template<typename _InputIterator, typename _Predicate>
inline _InputIterator
__find_if(_InputIterator __first, _InputIterator __last,
          _Predicate __pred, input_iterator_tag)
{
    //这里的__pred(__first)会被迷惑到，但实际上是内部仿函数的封装，实际传的值是*__first
    while (__first != __last && !__pred(__first))
        ++__first;
    return __first;
}
```

这里笔者用得是MinGW，实际上内部封装了仿函数传入迭代器的操作，所以我们外部定义仿函数操作的时候直接传入`*__first`值即可，这里传入的就是int型，而不是`int*`型。

```c++
//predefined_ops.h#_Iter_pred
//这个就是内部封装的仿函数，__find_if(__first)会调用到这里
template<typename _Predicate>
inline _Iter_pred<_Predicate>
__pred_iter(_Predicate __pred)
{ 
    return _Iter_pred<_Predicate>(_GLIBCXX_MOVE(__pred)); 
}

//predefined_ops.h#_Iter_pred
//从find_if函数中可以发现，最终的内部仿函数为_Iter_pred，再去调用外部定义的仿函数_M_pred
template<typename _Predicate>
struct _Iter_pred
{
    _Predicate _M_pred;

    explicit
        _Iter_pred(_Predicate __pred)
        : _M_pred(_GLIBCXX_MOVE(__pred))
        { }

    template<typename _Iterator>
    bool
    operator()(_Iterator __it)
    { return bool(_M_pred(*__it)); }
};
```

外部自定义仿函数

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

class __pred {
public:
    int number;
    __pred(int number) : number(number) {}
    bool operator() (const int value) {
        cout << "value = " << value << endl;
        return number == value;
    }
};

int main() {
    vector<int> vectorVar;
    vectorVar.insert(vectorVar.begin(), 10000);
    vectorVar.insert(vectorVar.begin(), 20000);
    vectorVar.insert(vectorVar.begin(), 30000);
    vectorVar.insert(vectorVar.begin(), 40000);

    auto it = find_if(vectorVar.begin(), vectorVar.end(), __pred(30000));

    if (it != vectorVar.end()) {
        cout << "查找到了" << endl;
    } else {
        cout << "没有找到" << endl;
    }
    return 0;
}
```



4.count和count_if

用于统计元素的个数

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    //统计2的个数
    int number = count(vectorVar.begin(), vectorVar.end(), 2);
    //count_if比较灵活可以统计比某个数大的，比某个数小的，和某个数相等的
    //count_if同时也需要用到函数适配器bind2nd
    int number = cout_if(vectorVar.begin(), vectorVar.end(), bind2nd(less<int>(), 2));
    int number = cout_if(vectorVar.begin(), vectorVar.end(), bind2nd(greater<int>(), 2));
    int number = cout_if(vectorVar.begin(), vectorVar.end(), bind2nd(equal_to<int>(), 2));
    return 0;
}
```

和上述的find和find_if类似，这里就不重复贴内部封装的仿函数

```c++
//stl_algo.h#count
template<typename _InputIterator, typename _Tp>
inline typename iterator_traits<_InputIterator>::difference_type
count(_InputIterator __first, _InputIterator __last, const _Tp& __value)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_EqualOpConcept<
                                    typename iterator_traits<_InputIterator>::value_type, _Tp>)
        __glibcxx_requires_valid_range(__first, __last);

    return std::__count_if(__first, __last,
                           __gnu_cxx::__ops::__iter_equals_val(__value));
}

//stl_algo.h#count_if
template<typename _InputIterator, typename _Predicate>
inline typename iterator_traits<_InputIterator>::difference_type
count_if(_InputIterator __first, _InputIterator __last, _Predicate __pred)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
        __glibcxx_function_requires(_UnaryPredicateConcept<_Predicate,
                                    typename iterator_traits<_InputIterator>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);

    return std::__count_if(__first, __last,
                           __gnu_cxx::__ops::__pred_iter(__pred));
}

//stl_algo.h#__count_if
template<typename _InputIterator, typename _Predicate>
typename iterator_traits<_InputIterator>::difference_type
    __count_if(_InputIterator __first, _InputIterator __last, _Predicate __pred)
{
    typename iterator_traits<_InputIterator>::difference_type __n = 0;
    for (; __first != __last; ++__first)
        if (__pred(__first))
            ++__n;
    return __n;
}
```



5.merge

用于容器的合并

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    vector<int> vectorVar1;
    vectorVar1.push_back(1);
    vectorVar1.push_back(2);
    vectorVar1.push_back(3);
    
    vector<int> result;
    result.resize(vectorVar.size() + vectorVar1.size());
    merge(vectorVar.begin(), vectorVar.end(), vectorVar1.begin(), vectorVar.end(), result.begin());
    
    for(auto it = result.begin(); it != result.end(); it++){
        cout << *it << endl;
    }
    return 0;
}
```

源码如下

```c++
//stl_algo.h#merge
template<typename _InputIterator1, typename _InputIterator2,
typename _OutputIterator>
inline _OutputIterator
merge(_InputIterator1 __first1, _InputIterator1 __last1,
          _InputIterator2 __first2, _InputIterator2 __last2,
          _OutputIterator __result)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator1>)
        __glibcxx_function_requires(_InputIteratorConcept<_InputIterator2>)
        __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
                                    typename iterator_traits<_InputIterator1>::value_type>)
        __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
                                    typename iterator_traits<_InputIterator2>::value_type>)
        __glibcxx_function_requires(_LessThanOpConcept<
                                    typename iterator_traits<_InputIterator2>::value_type,
                                    typename iterator_traits<_InputIterator1>::value_type>)	
        __glibcxx_requires_sorted_set(__first1, __last1, __first2);
    __glibcxx_requires_sorted_set(__first2, __last2, __first1);
    __glibcxx_requires_irreflexive2(__first1, __last1);
    __glibcxx_requires_irreflexive2(__first2, __last2);

    return _GLIBCXX_STD_A::__merge(__first1, __last1,
                                   __first2, __last2, __result,
                                   __gnu_cxx::__ops::__iter_less_iter());
}

//stl_algo.h#__merge
//前面先做一个排序工作，后面再进行一个copy工作
template<typename _InputIterator1, typename _InputIterator2,
typename _OutputIterator, typename _Compare>
_OutputIterator
__merge(_InputIterator1 __first1, _InputIterator1 __last1,
            _InputIterator2 __first2, _InputIterator2 __last2,
            _OutputIterator __result, _Compare __comp)
{
    while (__first1 != __last1 && __first2 != __last2)
    {
        if (__comp(__first2, __first1))
        {
            *__result = *__first2;
            ++__first2;
        }
        else
        {
            *__result = *__first1;
            ++__first1;
        }
        ++__result;
    }
    return std::copy(__first2, __last2,
                     std::copy(__first1, __last1, __result));
}
```



6.sort

排序

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    //从小到大
    sort(vectorVar.begin(), vectorVar.end(), less<int>());
    //从大到小
    //sort(vectorVar.begin(), vectorVar.end(), greater<int>());
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```

sort源码，通过一层层剥离可以看到调用到`__comp(__i, __first)`可以看出仿函数是bool类型的

```c++
//stl_algo.h#sort
template<typename _RandomAccessIterator, typename _Compare>
inline void
sort(_RandomAccessIterator __first, _RandomAccessIterator __last,
     _Compare __comp)
{
    // concept requirements
    __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
                                _RandomAccessIterator>)
        __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
                                    typename iterator_traits<_RandomAccessIterator>::value_type,
                                    typename iterator_traits<_RandomAccessIterator>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);
    __glibcxx_requires_irreflexive_pred(__first, __last, __comp);

    std::__sort(__first, __last, __gnu_cxx::__ops::__iter_comp_iter(__comp));
}

//stl_algo.h#__sort
template<typename _RandomAccessIterator, typename _Compare>
inline void
__sort(_RandomAccessIterator __first, _RandomAccessIterator __last,
       _Compare __comp)
{
    if (__first != __last)
    {
        std::__introsort_loop(__first, __last,
                              std::__lg(__last - __first) * 2,
                              __comp);
        std::__final_insertion_sort(__first, __last, __comp);
    }
}

//stl_algo.h#__final_insertion_sort
template<typename _RandomAccessIterator, typename _Compare>
void
__final_insertion_sort(_RandomAccessIterator __first,
                       _RandomAccessIterator __last, _Compare __comp)
{
    if (__last - __first > int(_S_threshold))
    {
        std::__insertion_sort(__first, __first + int(_S_threshold), __comp);
        std::__unguarded_insertion_sort(__first + int(_S_threshold), __last,
                                        __comp);
    }
    else
        std::__insertion_sort(__first, __last, __comp);
}

//stl_algo.h#__insertion_sort
template<typename _RandomAccessIterator, typename _Compare>
void
__insertion_sort(_RandomAccessIterator __first,
                 _RandomAccessIterator __last, _Compare __comp)
{
    if (__first == __last) return;

    for (_RandomAccessIterator __i = __first + 1; __i != __last; ++__i)
    {
        if (__comp(__i, __first))
        {
            typename iterator_traits<_RandomAccessIterator>::value_type
                __val = _GLIBCXX_MOVE(*__i);
            _GLIBCXX_MOVE_BACKWARD3(__first, __i, __i + 1);
            *__first = _GLIBCXX_MOVE(__val);
        }
        else
            std::__unguarded_linear_insert(__i,
                                           __gnu_cxx::__ops::__val_comp_iter(__comp));
    }
}
```



7.random_shuffle

随记打乱

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    //从小到大
    sort(vectorVar.begin(), vectorVar.end(), less<int>());
    //从大到小
    //sort(vectorVar.begin(), vectorVar.end(), greater<int>());
    //随机打乱
    random_shuffle(vectorVar.begin(), vectorVar.end());
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```

源码分析，可以看到通过除以一个rand()数来计算

```c++
//stl_algo.h#random_shuffle
template<typename _RandomAccessIterator>
inline void
random_shuffle(_RandomAccessIterator __first, _RandomAccessIterator __last)
{
    // concept requirements
    __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
                                _RandomAccessIterator>)
        __glibcxx_requires_valid_range(__first, __last);

    if (__first != __last)
        for (_RandomAccessIterator __i = __first + 1; __i != __last; ++__i)
        {
            // XXX rand() % N is not uniformly distributed
            _RandomAccessIterator __j = __first
                + std::rand() % ((__i - __first) + 1);
            if (__i != __j)
                std::iter_swap(__i, __j);
        }
}
```



8.copy

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    vector<int> vectorVar1;
    vector1.resize(vectorVar.size());
    copy(vectorVar.begin(), vectorVar.end(), vectorVar1.end());
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```

源码分析，实际上是遍历原来的数组放进去

```c++
//stl_algo.h#copy
template<typename _II, typename _OI>
inline _OI
copy(_II __first, _II __last, _OI __result)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_II>)
        __glibcxx_function_requires(_OutputIteratorConcept<_OI,
                                    typename iterator_traits<_II>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);

    return (std::__copy_move_a2<__is_move_iterator<_II>::__value>
            (std::__miter_base(__first), std::__miter_base(__last),
             __result));
}

//stl_algo.h#__copy_move_a2
template<bool _IsMove, typename _CharT>
typename __gnu_cxx::__enable_if<__is_char<_CharT>::__value,
ostreambuf_iterator<_CharT> >::__type
    __copy_move_a2(_CharT* __first, _CharT* __last,
                   ostreambuf_iterator<_CharT> __result)
{
    const streamsize __num = __last - __first;
    if (__num > 0)
        __result._M_put(__first, __num);
    return __result;
}
```



9.replace

替换容器元素

```c++
#include <isotream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> vectorVar;
    vectorVar.push_back(1);
    vectorVar.push_back(2);
    vectorVar.push_back(3);
    vectorVar.push_back(4);
    vectorVar.push_back(5);
    
    //第三个参数为原来的值，最后一个参数为新值
    replace(vectorVar.begin(), vectorVar.end(), 5, 6);
    
    for(auto it = vectorVar.begin(); it != vectorVar.end(); it++)
    {
        cout << *it << endl;
    }
    return 0;
}
```

源码分析，可以看到这里用到了引用值，所以没有副本存在，这里面直接修改了迭代器的值，本身迭代器就跟智能指针类似，传入的迭代器修改会同步迭代器的本身。

```c++
template<typename _ForwardIterator, typename _Tp>
void
replace(_ForwardIterator __first, _ForwardIterator __last,
        const _Tp& __old_value, const _Tp& __new_value)
{
    // concept requirements
    __glibcxx_function_requires(_Mutable_ForwardIteratorConcept<
                                _ForwardIterator>)
        __glibcxx_function_requires(_EqualOpConcept<
                                    typename iterator_traits<_ForwardIterator>::value_type, _Tp>)
        __glibcxx_function_requires(_ConvertibleConcept<_Tp,
                                    typename iterator_traits<_ForwardIterator>::value_type>)
        __glibcxx_requires_valid_range(__first, __last);

    for (; __first != __last; ++__first)
        if (*__first == __old_value)
            *__first = __new_value;
}
```

