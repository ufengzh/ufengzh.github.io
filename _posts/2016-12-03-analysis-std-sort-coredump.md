---
layout: post
title:  "源码分析STL Sort Coredump"
date:   2016-12-03 14:32:00 +0800
categories: C++
---
## 一、概述
在查看后台线上服务日志时，无意中发现负责的服务MQQ_UserCouponServer偶尔有Coredump发生，由于发生的频率相对较小，没有触发SPP服务针对Coredump提供的告警机制，所以之前一直都没收到告警。另外，MQQ_UserCouponServer这个服务本身只提供查询接口没有修改用户数据的行为，加上前端调用者有重试以及终端用户的手动重试，所以对用户的影响就特别的小，没有收到用户投诉也比较正常了。

果断调整Core文件生成大小，以便记录进程异常退出时的现场，修改init.xml并重启服务。
```bash
<!--程序启动方式，请使用相对bin目录的路径-->
<start>
# 限制core文件大小为4k,用于进程coredump监控
# ulimit -c 8 -S
ulimit -c unlimited
```

## 二、Core文件分析

通过gdb backtrace分析Core文件，发现每次都Core在STL的排序sort函数上，莫非提供的比较器Comparator有问题？可是比较器内没有任何指针的操作啊，只会返回true、false。好奇怪，这是为什么呢？只有找到sort的实现逻辑，才能一了LZ的好奇心啊。

MQQ_UserCouponServer服务内的Comparator比较器，看不到任何内存越界的操作。
```cpp
bool UserCouponImp::cmpByCardType(const UCouponItem &first, const UCouponItem &second)
{
    if (first.filter != second.filter)
    {   
        return first.filter < second.filter;
    }   
    else if (first.status != second.status)
    {   
        CodeStatus status1 = CODE_STATUS_UNGETTED;
        CodeStatus status2 = CODE_STATUS_UNGETTED;

        stoe(first.status, status1);
        stoe(second.status, status2);

        // 调整已过期/已使用状态的用户券排序
        if (status1 == CODE_STATUS_OVERDUE ||
            status1 == CODE_STATUS_USED)
        {   
            return true;
        }
        else if (status2 == CODE_STATUS_OVERDUE ||
                 status2 == CODE_STATUS_USED)
        {   
            return false;
        }  
    }   

    return first.getTime < second.getTime;
}
```

## 三、Sort源码分析
考虑到GCC实现的STL源码阅读性不那么友好，这次就选用SGI实现的STL源码，据说很多STL实现都是参考SGI实现的，GCC也不例外。SGI STL的源码可在这儿下载https://www.sgi.com/tech/stl/download.html。

### 3.1. sort函数入口

`std::sort`的实现都在stl_algo.h文件当中。
首先看下std::sort入口函数，当传入的`__first`、`__last`参数不相等时（C++内部vector实现的iterator其实就是C语言的指针），就调用`__introsort_loop`、`__final_insertion_sort`函数，否则就直接返回了。
```cpp
template <class _RandomAccessIter, class _Compare>
inline void sort(_RandomAccessIter __first, _RandomAccessIter __last,
                 _Compare __comp) {
  __STL_REQUIRES(_RandomAccessIter, _Mutable_RandomAccessIterator);
  __STL_BINARY_FUNCTION_CHECK(_Compare, bool,
       typename iterator_traits<_RandomAccessIter>::value_type,
       typename iterator_traits<_RandomAccessIter>::value_type);
  if (__first != __last) {
    __introsort_loop(__first, __last,
                     __VALUE_TYPE(__first),
                     __lg(__last - __first) * 2,
                     __comp);
    __final_insertion_sort(__first, __last, __comp);
  }
}
```

> C++内部vector实现的iterator就是C语言的指针，下面是vector::iterator、vector::const_iterator的定义。
``` cpp
template <class _Tp, class _Alloc = __STL_DEFAULT_ALLOCATOR(_Tp) >
class vector : protected _Vector_base<_Tp, _Alloc> 
{
public:
  typedef _Tp value_type;
  typedef value_type* iterator;  
  typedef const value_type* const_iterator;
```

### 3.2. __introsort_loop实现

`__introsort_loop`大概的流程：
1. 如果数组长度 > 16：
	1.1. 通过`__median`函数调用返回`__first`、`(__first + (_last - __first)/2)`、`(__last - 1)`三个指针所指数据的中间值。
	1.2. 调用`__unguarded_partition`，把`__first`、`__last`通过中间值拆分成两段数据，该两段间数据保持有序，但各段内数据并不一定有序。
	1.3. 把刚刚拆分的两段数据，分别递归调用`__introsort_loop`，再次拆分，直到各个段内元素数量 <= 16。
2. 如果数组长度 <= 16，直接返回。

简而言之，`__introsort_loop`就是做了做了这样一件事件，把长度大于16的数组拆分成各个小于或等于16个元素的小段，而且各个段间是通过用户传入的`__comp`保持有序，但段内的各个元素并不一定有序。其实就是个`分段排序`嘛。

```cpp
const int __stl_threshold = 16;

template <class _RandomAccessIter, class _Tp, class _Size, class _Compare>
void __introsort_loop(_RandomAccessIter __first,
                      _RandomAccessIter __last, _Tp*,
                      _Size __depth_limit, _Compare __comp)
{   
  while (__last - __first > __stl_threshold) { 
    if (__depth_limit == 0) {
      partial_sort(__first, __last, __last, __comp);
      return;
    }
    --__depth_limit;
    _RandomAccessIter __cut =
      __unguarded_partition(__first, __last,
                            _Tp(__median(*__first,         
                                         *(__first + (__last - __first)/2),
                                         *(__last - 1), __comp)),       
       __comp);
    __introsort_loop(__cut, __last, (_Tp*) 0, __depth_limit, __comp);
    __last = __cut;
  } 
}
```

```cpp
template <class _RandomAccessIter, class _Tp, class _Compare>
_RandomAccessIter __unguarded_partition(_RandomAccessIter __first,
                                        _RandomAccessIter __last,
                                        _Tp __pivot, _Compare __comp) 
{ 
  while (true) {
    while (__comp(*__first, __pivot))
      ++__first; 
    --__last;
    while (__comp(__pivot, *__last))
      --__last;      
    if (!(__first < __last))
      return __first;
    iter_swap(__first, __last);
    ++__first;
  }
}
```

### 3.3. __final_insertion_sort实现

`__final_insertion_sort`的大概流程：
1. 数组长度 > 16：
	1.1. 调用`__insertion_sort`，把数组前16个元素通过插入排序算法排好序。
	1.2. 调用`__unguarded_insertion_sort`，把数组从第16个及以后的元素调用一个`非安全的插入排序算法`进行排序。
	1.3. 最终调用了`__unguarded_linear_insert`，显然这个函数的实现完全依赖用户提供的比较器函数`__comp`，没有任何的边界检查，如果`__comp`一直返回true，在while循环以后进行数据交换的时候`__last`可能已经指向一个非法的内存地址了，这个时候就会出现`内存访问越界`的问题。
2. 数组长度 <= 16，调用`__insertion_sort`。

```cpp
template <class _RandomAccessIter, class _Compare>
void __final_insertion_sort(_RandomAccessIter __first, 
                            _RandomAccessIter __last, _Compare __comp) {
  if (__last - __first > __stl_threshold) {
    __insertion_sort(__first, __first + __stl_threshold, __comp);
    __unguarded_insertion_sort(__first + __stl_threshold, __last, __comp);
  }                  
  else
    __insertion_sort(__first, __last, __comp);
}
```

```cpp
template <class _RandomAccessIter, class _Tp, class _Compare>
void __unguarded_linear_insert(_RandomAccessIter __last, _Tp __val, 
                               _Compare __comp) {             
  _RandomAccessIter __next = __last;
  --__next;  
  while (__comp(__val, *__next)) {
    *__last = *__next;
    __last = __next;
    --__next;
  } 
  *__last = __val;
}

template <class _RandomAccessIter, class _Tp, class _Compare>
void __unguarded_insertion_sort_aux(_RandomAccessIter __first, 
                                    _RandomAccessIter __last,      
                                    _Tp*, _Compare __comp) {       
  for (_RandomAccessIter __i = __first; __i != __last; ++__i) 
    __unguarded_linear_insert(__i, _Tp(*__i), __comp);
}

template <class _RandomAccessIter, class _Compare>
inline void __unguarded_insertion_sort(_RandomAccessIter __first,
                                       _RandomAccessIter __last,
                                       _Compare __comp) {
  __unguarded_insertion_sort_aux(__first, __last, __VALUE_TYPE(__first),
                                 __comp);
}
```


## 四、Corecump必现验证
通过刚刚分析`__final_insertion_sort`的实现，得到一个明确的逻辑：
> 只要sort使用者提供的vector.size() > 16，而且提供的Comparator一直返回truee，必然就会造成内存访问越界问题。

下面就写个简单的小程序来验证下这个逻辑，`sort_test.cpp`。
```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

bool comp(int a, int b)
{
    return a <= b;
}

int main(int argc, char *[])
{
    vector<int> vec;

    for (int i = 0; i < 17; i++)
    {
        vec.push_back(10);
    }
    // std::sort(vec.begin(), vec.end(), comp);
    std::sort(vec.rbegin(), vec.rend(), comp);

    typeof(vec.begin()) iter;
    for (iter = vec.begin(); iter != vec.end(); iter++)
    {
        cout << *iter << " ";
    }
    cout << endl;

    return 0;
}
```

编译并调整Core文件大小，然后执行就出现Coredump了，果断验证了之前的`std::sort`源码的分析逻辑。
```bash
ufeng@SWEBMYVMM002140 ~/p/cpp> g++ -ggdb sort_test.cpp 
ufeng@SWEBMYVMM002140 ~/p/cpp> ulimit -c unlimited 
ufeng@SWEBMYVMM002140 ~/p/cpp> ./a.out 
10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 0 
*** glibc detected *** ./a.out: double free or corruption (out): 0x00000000018c10d0 ***
======= Backtrace: =========
/lib64/libc.so.6(+0x75e66)[0x7f501be5ee66]
/lib64/libc.so.6(+0x789b3)[0x7f501be619b3]
./a.out[0x4020fa]
./a.out[0x40195e]
./a.out[0x401153]
./a.out[0x400ddb]
./a.out[0x400ca4]
/lib64/libc.so.6(__libc_start_main+0xfd)[0x7f501be07d5d]
./a.out[0x400aa9]
======= Memory map: ========
00400000-00405000 r-xp 00000000 fd:11 12247044                           /home/ufeng/projects/cpp/a.out
00604000-00605000 rw-p 00004000 fd:11 12247044                           /home/ufeng/projects/cpp/a.out
018c1000-018e2000 rw-p 00000000 00:00 0                                  [heap]
7f501b9c8000-7f501b9df000 r-xp 00000000 fd:01 309668                     /lib64/libpthread-2.12.so
7f501b9df000-7f501bbdf000 ---p 00017000 fd:01 309668                     /lib64/libpthread-2.12.so
7f501bbdf000-7f501bbe0000 r--p 00017000 fd:01 309668                     /lib64/libpthread-2.12.so
7f501bbe0000-7f501bbe1000 rw-p 00018000 fd:01 309668                     /lib64/libpthread-2.12.so
7f501bbe1000-7f501bbe5000 rw-p 00000000 00:00 0 
7f501bbe5000-7f501bbe7000 r-xp 00000000 fd:01 309538                     /lib64/libdl-2.12.so
7f501bbe7000-7f501bde7000 ---p 00002000 fd:01 309538                     /lib64/libdl-2.12.so
7f501bde7000-7f501bde8000 r--p 00002000 fd:01 309538                     /lib64/libdl-2.12.so
7f501bde8000-7f501bde9000 rw-p 00003000 fd:01 309538                     /lib64/libdl-2.12.so
7f501bde9000-7f501bf73000 r-xp 00000000 fd:01 309509                     /lib64/libc-2.12.so
7f501bf73000-7f501c173000 ---p 0018a000 fd:01 309509                     /lib64/libc-2.12.so
7f501c173000-7f501c177000 r--p 0018a000 fd:01 309509                     /lib64/libc-2.12.so
7f501c177000-7f501c178000 rw-p 0018e000 fd:01 309509                     /lib64/libc-2.12.so
7f501c178000-7f501c17d000 rw-p 00000000 00:00 0 
7f501c17d000-7f501c193000 r-xp 00000000 fd:01 309558                     /lib64/libgcc_s-4.4.6-20110824.so.1
7f501c193000-7f501c392000 ---p 00016000 fd:01 309558                     /lib64/libgcc_s-4.4.6-20110824.so.1
7f501c392000-7f501c393000 rw-p 00015000 fd:01 309558                     /lib64/libgcc_s-4.4.6-20110824.so.1
7f501c393000-7f501c416000 r-xp 00000000 fd:01 309608                     /lib64/libm-2.12.so
7f501c416000-7f501c615000 ---p 00083000 fd:01 309608                     /lib64/libm-2.12.so
7f501c615000-7f501c616000 r--p 00082000 fd:01 309608                     /lib64/libm-2.12.so
7f501c616000-7f501c617000 rw-p 00083000 fd:01 309608                     /lib64/libm-2.12.so
7f501c617000-7f501c6ff000 r-xp 00000000 fd:01 171070                     /usr/lib64/libstdc++.so.6.0.13
7f501c6ff000-7f501c8ff000 ---p 000e8000 fd:01 171070                     /usr/lib64/libstdc++.so.6.0.13
7f501c8ff000-7f501c906000 r--p 000e8000 fd:01 171070                     /usr/lib64/libstdc++.so.6.0.13
7f501c906000-7f501c908000 rw-p 000ef000 fd:01 171070                     /usr/lib64/libstdc++.so.6.0.13
7f501c908000-7f501c91d000 rw-p 00000000 00:00 0 
7f501c91d000-7f501c93d000 r-xp 00000000 fd:01 309485                     /lib64/ld-2.12.so
7f501ca17000-7f501ca1c000 rw-p 00000000 00:00 0 
7f501ca2a000-7f501ca2c000 rw-p 00000000 00:00 0 
7f501ca2c000-7f501ca2f000 r-xp 00000000 fd:01 309904                     /lib64/libonion_security.so.1.0.13
7f501ca2f000-7f501cb2f000 ---p 00003000 fd:01 309904                     /lib64/libonion_security.so.1.0.13
7f501cb2f000-7f501cb30000 rw-p 00003000 fd:01 309904                     /lib64/libonion_security.so.1.0.13
7f501cb30000-7f501cb3c000 rw-p 00000000 00:00 0 
7f501cb3c000-7f501cb3d000 r--p 0001f000 fd:01 309485                     /lib64/ld-2.12.so
7f501cb3d000-7f501cb3e000 rw-p 00020000 fd:01 309485                     /lib64/ld-2.12.so
7f501cb3e000-7f501cb3f000 rw-p 00000000 00:00 0 
7fff77338000-7fff77359000 rw-p 00000000 00:00 0                          [stack]
7fff773fe000-7fff77400000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
[1]    25242 abort      ./a.out
```

打开Core文件，发现vector对象在析构的时候出现内存访问越界问题，这当然是vector内存储元素的地址在通过排序操作后已经存在非法指针了。
```bash
ufeng@SWEBMYVMM002140 ~/p/cpp> gdb -c core.25778 a.out
(gdb) bt
#0  0x00007fa967fa4625 in raise () from /lib64/libc.so.6
#1  0x00007fa967fa5e05 in abort () from /lib64/libc.so.6
#2  0x00007fa967fe2537 in __libc_message () from /lib64/libc.so.6
#3  0x00007fa967fe7e66 in malloc_printerr () from /lib64/libc.so.6
#4  0x00007fa967fea9b3 in _int_free () from /lib64/libc.so.6
#5  0x00000000004020fa in __gnu_cxx::new_allocator<int>::deallocate (this=0x7fffc7bbdc40, __p=0x11200d0) at /usr/lib/gcc/x86_64-redhat-linux/4.4.6/../../../../include/c++/4.4.6/ext/new_allocator.h:95
#6  0x000000000040195e in std::_Vector_base<int, std::allocator<int> >::_M_deallocate (this=0x7fffc7bbdc40, __p=0x11200d0, __n=32)
    at /usr/lib/gcc/x86_64-redhat-linux/4.4.6/../../../../include/c++/4.4.6/bits/stl_vector.h:146
#7  0x0000000000401153 in std::_Vector_base<int, std::allocator<int> >::~_Vector_base (this=0x7fffc7bbdc40, __in_chrg=<value optimized out>)
    at /usr/lib/gcc/x86_64-redhat-linux/4.4.6/../../../../include/c++/4.4.6/bits/stl_vector.h:132
#8  0x0000000000400ddb in std::vector<int, std::allocator<int> >::~vector (this=0x7fffc7bbdc40, __in_chrg=<value optimized out>)
    at /usr/lib/gcc/x86_64-redhat-linux/4.4.6/../../../../include/c++/4.4.6/bits/stl_vector.h:313
#9  0x0000000000400ca4 in main (argc=1) at sort_test.cpp:47
(gdb) 
```

## 五、Sort使用要求总结

总结下C++ STL提供的std::sort接口，要求用户提供的比较器Comparator满足以下几点：
1. 反自反性：a、a在比较时必须是false，即`cmp(a, a) == false`。
2. 非对称性：a、b在交换位置后比较值`必须不能都是true`，否则在vector元素超过16的情况下可能会出现内存访问越界问题。即如果`cmp(a, b) == true`，那么`cmp(b, a) == false`。
3. 可传递性：a、b、c比较结果具有可传递性。
```cpp
if cmp(a, b) == true && cmp(b, c) == true then
    cmp(a, c) == true
end
```

再回到MQQ_UserCouponServer提供的Comparator，只要做这样的代码变更就解决了Coredump问题。
```diff
 bool UserCouponImp::cmpByCardType(const UCouponItem &first, const UCouponItem &second)
 {
     if (first.filter != second.filter)
     {   
         return first.filter < second.filter;
     }   
     else if (first.status != second.status)
     {   
         CodeStatus status1 = CODE_STATUS_UNGETTED;
         CodeStatus status2 = CODE_STATUS_UNGETTED;
 
         stoe(first.status, status1);
         stoe(second.status, status2);
 
         // 调整已过期/已使用状态的用户券排序
-        if (status1 == CODE_STATUS_OVERDUE ||
-            status1 == CODE_STATUS_USED)
+        if (status2 == CODE_STATUS_OVERDUE ||
+            status2 == CODE_STATUS_USED)
         {
-            return true;
+            return false;
         }
-        else if (status2 == CODE_STATUS_OVERDUE ||
-                 status2 == CODE_STATUS_USED)
+        else if (status1 == CODE_STATUS_OVERDUE ||
+                 status1 == CODE_STATUS_USED)
         {
-            return false;
+            return true;
         }
     }   
 
     return first.getTime < second.getTime;
 }
```

> 注意：在编写Comparator比较器时，`如果排序较复杂，两个元素的前后排序无关紧要，请务必返回false`。

