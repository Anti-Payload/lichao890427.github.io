---
layout: post
title: 重识new
categories: C/C++
description: 重识new
keywords: 
---

## 一、new primer

&emsp;&emsp;new操作符并不是c++的专属，c运行库有new.h头文件。C++中的new操作符，众所周知是用来分配动态内存的，而要能达到“动态”这种灵活性的特征，非堆区莫属，因为堆区支持手动分配。如果内存空间不够，new操作符返回空，或抛出异常（下面会讲述3种new操作符，常规new操作符、定位new操作符和禁止抛出异常的new操作符，如果禁止抛异常，那么只好返回空了）下面代码为MSDN中的，检测new分配是否成功，内存分配失败处理还可以通过_set_new_handler注册分配异常函数结果

```C++
// insufficient_memory_conditions.cpp
// compile with: /EHsc
#include <iostream>
using namespace std;
#define BIG_NUMBER 100000000
int main() {
   int *pI = new int[BIG_NUMBER];
   if( pI == 0x0 ) {
      cout << "Insufficient memory" << endl;
      return -1;
   }
}
```

&emsp;&emsp;对于对象使用new操作符的情况，生成的代码执行的流程一般为：先调用内部合适的new函数，该函数最终调用内存分配API进行堆内存分配，同时初始化一些方便内存管理的结构体和数据。成功分配后，将该地址视为this，如有必要则设置虚表指针，之后调用构造函数对该地址处对象进行构造。构造函数一般也是执行初始化功能而已，修改修改数据啥的。这些都是老生常谈不再赘述。  
&emsp;&emsp;即使请求的空间是0字节大小，new也会返回不同的地址，也就说总是在不同区域分配。这里注意和空类的区别，空类的大小是1，传给new函数（该函数在之后给出）之后，new函数接受到的参数只会>=1，而对于特殊情况：char* pch=new char[0];new函数接受的参数确实是0，不过分配的空间并不是0字节，因为存在内存管理相关的结构体也会占用内存。  
&emsp;&emsp;new操作符有2种作用域，一种是全局new，一种是类作用域new。用户在自定义类中可以重载自定义new函数。代码如下：

```C++
#include <malloc.h>
#include <memory.h>
class Blanks
{
public:
    Blanks(){}
    void *operator new( size_t stAllocateBlock, char chInit );
};
void *Blanks::operator new( size_t stAllocateBlock, char chInit )
{
    void *pvTemp = malloc( stAllocateBlock );
    if( pvTemp != 0 )
        memset( pvTemp, chInit, stAllocateBlock );
    return pvTemp;
}
// 对于Blanks对象，全局new操作符已被替换，因此下面的代码将分配sizeof(Blanks)大小的空间并把数据赋值为0xa5
int main()
{
   Blanks *a5 = new(0xa5) Blanks;
   return a5 != 0;
}
```

## 二、new primer plus

&emsp;&emsp;new和delete操作符是通过一种称为自由存储区的内存池分配内存的，而new和delete操作符本身在编译时会由编译器选择合适的new函数和delete函数进行实现。C运行库的new函数会在失败时抛出std::bad_alloc异常，如果要使用不抛出异常的new版本，则需要链接nothrownew.obj，然而一旦链接了该文件，标准C++库中的new就不起作用了。new有2种语法形式
* [::] new [placement] new-type-name [new-initializer]
* [::] new [placement] ( type-name ) [new-initializer]

2种new函数原型为：
* `void* operator new( std::size_t _Count ) throw(bad_alloc); `
* `void* operator new( std::size_t _Count, const std::nothrow_t& ) throw( ); `
* `void* operator new( std::size_t _Count, void* _Ptr ) throw( );`
* `void *operator new[]( std::size_t _Count ) throw(std::bad_alloc); `
* `void *operator new[]( std::size_t _Count, const std::nothrow_t& ) throw( ); `
* `void *operator new[]( std::size_t _Count, void* _Ptr ) throw( );`

其用法如下：
```C++
#include<new>
#include<iostream>
using namespace std;
class MyClass 
{
public: 
   MyClass( )
   {
      cout << "Construction MyClass." << this << endl;
   };
   ~MyClass( )
   {
      imember = 0; cout << "Destructing MyClass." << this << endl;
   };
   int imember;
};

int main( ) 
{
   // The first form of new delete
   MyClass* fPtr = new MyClass;
   delete fPtr;
   // The second form of new delete
   MyClass* fPtr2 = new( nothrow ) MyClass;
   delete fPtr2;
   // The third form of new delete
   char x[sizeof( MyClass )];
   MyClass* fPtr3 = new( &x[0] ) MyClass;
   fPtr3 -> ~MyClass();
   cout << "The address of x[0] is : " << ( void* )&x[0] << endl;
}


Construction MyClass.000B3F30
Destructing MyClass.000B3F30
Construction MyClass.000B3F30
Destructing MyClass.000B3F30
Construction MyClass.0023FC60
Destructing MyClass.0023FC60
The address of x[0] is : 0023FC60

#include <new>
#include <iostream>
using namespace std;

class MyClass {
public:
   MyClass() {
      cout << "Construction MyClass." << this << endl;
   };

   ~MyClass() {
      imember = 0; cout << "Destructing MyClass." << this << endl;
      };
   int imember;
};

int main() {
   // The first form of new delete
   MyClass* fPtr = new MyClass[2];
   delete[ ] fPtr;
   // The second form of new delete
   char x[2 * sizeof( MyClass ) + sizeof(int)];
   MyClass* fPtr2 = new( &x[0] ) MyClass[2];
   fPtr2[1].~MyClass();
   fPtr2[0].~MyClass();
   cout << "The address of x[0] is : " << ( void* )&x[0] << endl;
   // The third form of new delete
   MyClass* fPtr3 = new( nothrow ) MyClass[2];
   delete[ ] fPtr3;
}


Construction MyClass.00311AEC
Construction MyClass.00311AF0
Destructing MyClass.00311AF0
Destructing MyClass.00311AEC
Construction MyClass.0012FED4
Construction MyClass.0012FED8
Destructing MyClass.0012FED8
Destructing MyClass.0012FED4
The address of x[0] is : 0012FED0
Construction MyClass.00311AEC
Construction MyClass.00311AF0
Destructing MyClass.00311AF0
Destructing MyClass.00311AEC
```

## 探究第一种new形式实现
&emsp;&emsp;第一种形式为常规new，`MyClass* fPtr1 = new MyClass;`

```C++
// new.cpp
void *__CRTDECL operator new(size_t size) _THROW1(_STD bad_alloc)
{       // try to allocate size bytes
	void *p;
	while ((p = malloc(size)) == 0)
	if (_callnewh(size) == 0)
	{       // report no memory
		_THROW_NCEE(_XSTD bad_alloc, );
	}
	return (p);
}
```

&emsp;&emsp;从上面new函数实现可以看到是使用malloc函数进行分配，如果失败则调用_callnewh调用注册过的“new操作符失败回调”函数（注册用_set_new_handler），如果原先没注册new失败回调，则抛出bad_alloc异常，可见在默认情况下，该while只会执行1次，仅当自定义new失败回调函数返回true，才可能多次尝试分配。  
&emsp;&emsp;该种形式delete实现方式，单步以后vs不能定位到源码，不过我们可以换一种思路，既然知道一定执行析构函数，那么就在析构中下断点，断下后查看反汇编，并执行到上一级调用即可找到delete实现方法，因此看汇编实现，发现是一个名为“scalar deleting destructor”的内部函数：

```C++
// 
00EB3470  push        ebp 
00EB3471  mov         ebp,esp 
00EB3473  sub         esp,0CCh  
00EB3479  push        ebx 
00EB347A  push        esi 
00EB347B  push        edi 
00EB347C  push        ecx 
00EB347D  lea         edi,[ebp-0CCh]  
00EB3483  mov         ecx,33h  
00EB3488  mov         eax,0CCCCCCCCh  
00EB348D  rep stos    dword ptr es:[edi]  
00EB348F  pop         ecx  //以上部分为debug版API常见头，无需理会
00EB3490  mov         dword ptr [this],ecx 
00EB3493  mov         ecx,dword ptr [this]  
00EB3496  call        MyClass::~MyClass (0EB1023h)  //执行析构
00EB349B  mov         eax,dword ptr [ebp+8]  
00EB349E  and         eax,1  
00EB34A1  je          MyClass::`scalar deleting destructor'+3Fh (0EB34AFh)  //如果传入参数允许释放则进行调用对应delete函数(对于定位new对应的delete该参数是设置为不允许的)
00EB34A3  mov         eax,dword ptr [this]  
00EB34A6  push        eax 
00EB34A7  call        operator delete (0EB1154h)  
00EB34AC  add         esp,4  
00EB34AF  mov         eax,dword ptr [this]  //以下是无关的收尾工作
00EB34B2  pop         edi 
00EB34B3  pop         esi 
00EB34B4  pop         ebx 
00EB34B5  add         esp,0CCh  
00EB34BB  cmp         ebp,esp 
00EB34BD  call        __RTC_CheckEsp (0EB1352h)  
00EB34C2  mov         esp,ebp 
00EB34C4  pop         ebp 
00EB34C5  ret         4 
```

&emsp;&emsp;执行到call operator delete这行，步入之后转到源码，可以看到使用的是dbgdel.cpp的delete函数。实现如下

```C++
// dbgdel.cpp
void operator delete( void *pUserData )
{
	_CrtMemBlockHeader * pHead;
	RTCCALLBACK(_RTC_Free_hook, (pUserData, 0));
	if (pUserData == NULL)
		return;
	_mlock(_HEAP_LOCK);  /* 阻塞其他线程*/
	__TRY
		/* 得到用于内存块信息头指针*/
		pHead = pHdr(pUserData);
		/* 检查区块类型 */
		_ASSERTE(_BLOCK_TYPE_IS_VALID(pHead->nBlockUse));
		_free_dbg( pUserData, pHead->nBlockUse );//调用free函数释放内存
	__FINALLY
		_munlock(_HEAP_LOCK);  /* 解锁其他线程*/
	__END_TRY_FINALLY
	return;
}
```

## 探究第二种new形式实现
&emsp;&emsp;第二种方式为不抛出异常的new，`MyClass* fPtr2 = new( nothrow ) MyClass;`，可见，这里用try块捕获了异常，因此不再抛出异常，余下的就是调用常规new函数而已。

```C++
// newopnt.cpp
void * __CRTDECL operator new(size_t count, const std::nothrow_t&) _THROW0()
{   // try to allocate count bytes
    void *p;
    _TRY_BEGIN
    p = operator new(count);
    _CATCH_ALL
    p = 0;
    _CATCH_END
    return (p);
}
#define _TRY_BEGIN try {
#define _CATCH(x)  } catch (x) {
#define _CATCH_ALL } catch (...) {
#define _CATCH_END }
```

## 探究第三种new形式实现
&emsp;&emsp;第三种方式为布局new：  
`char x1[sizeof( MyClass )];MyClass* fPtr3 = new( &x1[0] ) MyClass;`  
&emsp;&emsp;这里所谓的布局就是说告诉new我们已经有一个内存位置了：

```C++
// new
inline void *__CRTDECL operator new(size_t, void *_Where) _THROW0()
{   // construct array with placement at _Where
    return (_Where);
}
```

&emsp;&emsp;可以发现该new什么都没做，那么为什么还要new呢？仔细想想可以知道，编译器对new的处理是调用new函数后，之后将该地址作为this指针进行初始化操作（比如设置虚表），再调用构造函数，而构造函数这玩意不能直接调用，不像析构函数那样，因为构造之前还没有对象和指针呢，对象和指针是构造以后才有的，而调用析构函数的时候，是已经有对象或指针的。所以这种定位new在我理解，就是可以相当于可以直接构造了。  
&emsp;&emsp;从上面可以看到new操作符先调用合适的new函数分配空间，之后调用构造函数构造，而delete函数刚好相对，先进行析构之后调用析构函数析构；同时可以看到布局new操作符的好处是可以手动指定构造和析构的时间，对于new无论哪种形式，在调用new函数分配好内存后都会调用构造函数进行构造，而定位new函数实则是直接返回，这就导致直接使用当前地址进行构造，相当于显示调用构造函数,而析构时由于没有实际分配空间，因此不能用delete，而是显示调用析构函数进行析构。  
&emsp;&emsp;上面都是对于有构造函数和析构函数对象的情况，用delete时，编译器会为该类专门生成一个scalar deleting destructor函数，该函数中先进行析构，之后调用operator delete函数。当然，如果没有析构函数，那么就不会有scalar deleting destructor函数了，此时单步是可以看到delete源码的，即dbgdel.cpp中的void operator delete(void *pUserData)函数。这一点在delete用于基本类型时显而易见。

## 探究第一种new[]形式实现
&emsp;&emsp;第一种类型new，`MyClass* fPtr4 = new MyClass[2]`
```C++
// newaop.cpp
void *__CRTDECL operator new[](size_t count) _THROW1(std::bad_alloc)
{   // try to allocate count bytes for an array
    return (operator new(count));
}
```

&emsp;&emsp;而编译器传给该new[]函数的参数count是sizeof(MyClass[2])+sizeof(int)，该sizeof(int)用于内存管理。new[]()仍然调用了new();没有本质区别，即这么多对象占用的内存是当作整体分配的。再分配好之后，就需要对每个对象this指针处进行初始化和构造了。  
&emsp;&emsp;通过逆向分析可知先调用了new[]，如果成功分配内存，则调用数组构造迭代器vector_constructor_iterator对每个对象进行构造。`void* base=new[](sizeof(int)+sizeof(MyClass[2]));`，起始4字节存储要初始化的对象个数，剩余空间为对象占用内存

```Asm
                push    0Ch             ; count
                call    j_??_U@YAPAXI@Z ; operator new[](uint)
                add     esp, 4
                mov     [ebp+var_1A0], eax
                mov     [ebp+var_4], 3
                cmp     [ebp+var_1A0], 0
                jz      short loc_41851B
                mov     eax, [ebp+var_1A0]
                mov     dword ptr [eax], 2
                push    offset j_??1MyClass@@QAE@XZ ; pDtor
                push    offset j_??0MyClass@@QAE@XZ ; pCtor
                push    2               ; count
                push    4               ; size
                mov     ecx, [ebp+var_1A0]
                add     ecx, 4
                push    ecx             ; ptr
                call    j_??_L@YGXPAXIHP6EX0@Z1@Z ; `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *))
; ---------------------------------------------------------------------------
                mov     edx, [ebp+var_1A0]
                add     edx, 4
                mov     [ebp+var_22C], edx
                jmp     short loc_418525
; ---------------------------------------------------------------------------

loc_41851B:                             ; CODE XREF: _main+1FF j
                mov     [ebp+var_22C], 0

loc_418525:                             ; CODE XREF: _main+239 j
                mov     eax, [ebp+var_22C]
                mov     [ebp+var_1AC], eax
                mov     [ebp+var_4], 0FFFFFFFFh
                mov     ecx, [ebp+var_1AC]
                mov     [ebp+fPtr4], ecx
```

```C++
if(base)
{
  *(int*)base=2;//2个对象
  vector_construtor_iterator((MyClass*)((char*)base+4),sizeof(MyClass[2]),2,&MyClass::MyClass,&MyClass::~MyClass);
}
```

vector_constructor_iterator对应代码为：

```Asm
 ; void __stdcall `eh vector constructor iterator'(void *ptr, unsigned int size, int count, void (__thiscall *pCtor)(void *), void (__thiscall *pDtor)(void *))
 ??_L@YGXPAXIHP6EX0@Z1@Z proc near       ; CODE XREF: `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *)) j

 success         = dword ptr -20h
 i               = dword ptr -1Ch
 ms_exc          = CPPEH_RECORD ptr -18h
 ptr             = dword ptr  8
 size            = dword ptr  0Ch
 count           = dword ptr  10h
 pCtor           = dword ptr  14h
 pDtor           = dword ptr  18h

                 push    ebp
                 mov     ebp, esp
                 push    0FFFFFFFEh
                 push    offset stru_41F9A0
                 push    offset j___except_handler4
                 mov     eax, large fs:0
                 push    eax
                 add     esp, 0FFFFFFF0h
                 push    ebx
                 push    esi
                 push    edi
                 mov     eax, ___security_cookie
                 xor     [ebp+ms_exc.registration.ScopeTable], eax
                 xor     eax, ebp
                 push    eax
                 lea     eax, [ebp+ms_exc.registration]
                 mov     large fs:0, eax
                 mov     [ebp+success], 0
                 mov     [ebp+ms_exc.registration.TryLevel], 0
                 mov     [ebp+i], 0
                 jmp     short loc_415730
 ; ---------------------------------------------------------------------------

 loc_415727:                             ; CODE XREF: `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *))+67 j
                 mov     eax, [ebp+i]
                 add     eax, 1
                 mov     [ebp+i], eax

 loc_415730:                             ; CODE XREF: `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *))+45 j
                 mov     ecx, [ebp+i]
                 cmp     ecx, [ebp+count]
                 jge     short loc_415749
                 mov     ecx, [ebp+ptr]
                 call    [ebp+pCtor]
                 mov     edx, [ebp+ptr]
                 add     edx, [ebp+size]
                 mov     [ebp+ptr], edx
                 jmp     short loc_415727
 ; ---------------------------------------------------------------------------

 loc_415749:                             ; CODE XREF: `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *))+56 j
                 mov     [ebp+success], 1
                 mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh
                 call    $LN9            ; Finally handler 0 for function 4156E0
 ; ---------------------------------------------------------------------------

 loc_41575C:                             ; CODE XREF: `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *)):$LN10 j
                 jmp     short $LN12
 ; ---------------------------------------------------------------------------

 $LN9:                                   ; CODE XREF: `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *))+77 j
                                         ; DATA XREF: .rdata:stru_41F9A0 o
                 cmp     [ebp+success], 0 ; Finally handler 0 for function 4156E0
                 jnz     short $LN10
                 mov     eax, [ebp+pDtor]
                 push    eax             ; pDtor
                 mov     ecx, [ebp+i]
                 push    ecx             ; count
                 mov     edx, [ebp+size]
                 push    edx             ; size
                 mov     eax, [ebp+ptr]
                 push    eax             ; ptr
                 call    j_?__ArrayUnwind@@YGXPAXIHP6EX0@Z@Z ; __ArrayUnwind(void *,uint,int,void (*)(void *))

 $LN10:                                  ; CODE XREF: `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *))+82 j
                 retn
 ; ---------------------------------------------------------------------------

 $LN12:                                  ; CODE XREF: `eh vector constructor iterator'(void *,uint,int,void (*)(void *),void (*)(void *)):loc_41575C j
                 mov     ecx, [ebp+ms_exc.registration.Next]
                 mov     large fs:0, ecx
                 pop     ecx
                 pop     edi
                 pop     esi
                 pop     ebx
                 mov     esp, ebp
                 pop     ebp
                 retn    14h
 ??_L@YGXPAXIHP6EX0@Z1@Z endp

```

逆向分析得到C++语法：
```C++
void __stdcall vector_constructor_iterator(MyClass *objs, unsigned int size, int count, void (__thiscall *pCtor)(void *), void (__thiscall *pDtor)(void *))
{
    int i=0;
    __try
    {
        for(;i<count;i++,objs++)
        {
            objs->pCtor();//用构造函数构造
        }       
    }
    __except(1)
    {
        __ArrayUnwind(objs,size,i,pDtor);//如果某个构造函数产生异常，则进行栈解退，用到析构函数 
    }
}
```

&emsp;&emsp;栈解退，这里引用C++ Primer Plus的解释：“现在假设函数由于出现异常而终止（而不是由于返回），则程序也将释放栈中的内存，但不会在释放栈的第一个返回地址后停止，而是继续释放栈，直到找到一个位于try块中的返回地址，随后控制权将转到块尾的异常处理程序，而不会函数调用后面的第一条语句。这个过程称为栈解退，引发机制的一个非常重要的特性是，和函数返回一样，对于栈中的自动类对象，而throw语句则处理try块和throw之间的整个函数调用徐丽放在栈中的对象。如果没有栈解退这种特性，则引发异常后，对于中间函数调用放在栈中的自动类对象，其析构函数将不会被调用。”。unwind就是解退的意思，现在来查看__ArrayUnwind的源码，根据函数名可知该函数用于对象数组解退：

```Asm

                 push    ebp
                 mov     ebp, esp
                 push    0FFFFFFFEh
                 push    offset stru_41F9E0
                 push    offset j___except_handler4
                 mov     eax, large fs:0
                 push    eax
                 sub     esp, 8
                 push    ebx
                 push    esi
                 push    edi
                 mov     eax, ___security_cookie
                 xor     [ebp+ms_exc.registration.ScopeTable], eax
                 xor     eax, ebp
                 push    eax
                 lea     eax, [ebp+ms_exc.registration]
                 mov     large fs:0, eax
                 mov     [ebp+ms_exc.old_esp], esp
                 mov     [ebp+ms_exc.registration.TryLevel], 0

 loc_41591A:                             ; CODE XREF: __ArrayUnwind(void *,uint,int,void (*)(void *))+54 j
                 mov     eax, [ebp+count]
                 sub     eax, 1
                 mov     [ebp+count], eax
                 js      short loc_415936
                 mov     ecx, [ebp+ptr]
                 sub     ecx, [ebp+size]
                 mov     [ebp+ptr], ecx
                 mov     ecx, [ebp+ptr]
                 call    [ebp+pDtor]
                 jmp     short loc_41591A
 ; ---------------------------------------------------------------------------

 loc_415936:                             ; CODE XREF: __ArrayUnwind(void *,uint,int,void (*)(void *))+43 j
                 mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh
                 jmp     short loc_415956
 ; ---------------------------------------------------------------------------

 $LN7:                                   ; DATA XREF: .rdata:stru_41F9E0 o
                 mov     edx, [ebp+ms_exc.exc_ptr] ; Exception filter 0 for function 4158E0
                 push    edx             ; pExPtrs
                 call    ArrayUnwindFilter
                 add     esp, 4

 $LN9_1:
                 retn
 ; ---------------------------------------------------------------------------

 $LN8_0:                                 ; DATA XREF: .rdata:stru_41F9E0 o
                 mov     esp, [ebp+ms_exc.old_esp] ; Exception handler 0 for function 4158E0
                 mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh

 loc_415956:                             ; CODE XREF: __ArrayUnwind(void *,uint,int,void (*)(void *))+5D j
                 mov     ecx, [ebp+ms_exc.registration.Next]
                 mov     large fs:0, ecx
                 pop     ecx
                 pop     edi
                 pop     esi
                 pop     ebx
                 mov     esp, ebp
                 pop     ebp
                 retn    10h

```

逆向分析得到C++代码：
```C++
void __stdcall __ArrayUnwind(MyClass* objs,unsigned size,int count,void (__thiscall *pDtor)(void*))
{//第[count]对象由于没有构造成功，因此从[count-1]个对象开始析构
    __try
    {
        while(count--)
        {
           objs--;
           objs->pDtor();
        }
    }
    __except(terminate(),0)//如果析构发生异常，则终止程序
    {
        return;
    }
}
```

## 探究第一种形式delete[]形式实现

```Asm
                mov     eax, [ebp+fPtr4]
                mov     [ebp+var_188], eax
                mov     ecx, [ebp+var_188]
                mov     [ebp+var_194], ecx
                cmp     [ebp+var_194], 0
                jz      short loc_418574//如果之前new成功，则往下执行
                push    3               ; unsigned int
                mov     ecx, [ebp+var_194] ; this
                call    j_??_EMyClass@@QAEPAXI@Z ; MyClass::`vector deleting destructor'(uint)
                mov     [ebp+var_22C], eax
                jmp     short loc_41857E
; ---------------------------------------------------------------------------

loc_418574:                             ; CODE XREF: _main+27D j
                mov     [ebp+var_22C], 0

```

&emsp;&emsp;可见vector_deleting_destructor是用来析构对象数组的，原型为`void* __thiscall MyClass::vector_deleting_destructor(usigned int flag);`，该函数是编译器内部为MyClass类加的成员函数
flag含义未知，所以需要分析该函数源码：
```Asm
                 push    ebp
                 mov     ebp, esp
                 sub     esp, 0CCh
                 push    ebx
                 push    esi
                 push    edi
                 push    ecx
                 lea     edi, [ebp+var_CC]
                 mov     ecx, 33h
                 mov     eax, 0CCCCCCCCh
                 rep stosd
                 pop     ecx
                 mov     [ebp+this], ecx
                 mov     eax, [ebp+arg_0]
                 and     eax, 2
                 jz      short loc_413431
                 push    offset j_??1MyClass@@QAE@XZ ; pDtor
                 mov     eax, [ebp+this]
                 mov     ecx, [eax-4]
                 push    ecx             ; count
                 push    4               ; size
                 mov     edx, [ebp+this]
                 push    edx             ; ptr
                 call    j_??_M@YGXPAXIHP6EX0@Z@Z ; `eh vector destructor iterator'(void *,uint,int,void (*)(void *))
 ; ---------------------------------------------------------------------------
                 mov     eax, [ebp+arg_0]
                 and     eax, 1
                 jz      short loc_413429
                 mov     eax, [ebp+this]
                 sub     eax, 4
                 push    eax             ; void *
                 call    j_??_V@YAXPAX@Z_0 ; operator delete[](void *)
                 add     esp, 4

 loc_413429:                             ; CODE XREF: MyClass::`vector deleting destructor'(uint)+48 j
                 mov     eax, [ebp+this]
                 sub     eax, 4
                 jmp     short loc_413450
 ; ---------------------------------------------------------------------------

 loc_413431:                             ; CODE XREF: MyClass::`vector deleting destructor'(uint)+29 j
                 mov     ecx, [ebp+this] ; this
                 call    j_??1MyClass@@QAE@XZ ; MyClass::~MyClass(void)
                 mov     eax, [ebp+arg_0]
                 and     eax, 1
                 jz      short loc_41344D
                 mov     eax, [ebp+this]
                 push    eax             ; void *
                 call    j_??3@YAXPAX@Z_0 ; operator delete(void *)
                 add     esp, 4

 loc_41344D:                             ; CODE XREF: MyClass::`vector deleting destructor'(uint)+6F j
                 mov     eax, [ebp+this]

 loc_413450:                             ; CODE XREF: MyClass::`vector deleting destructor'(uint)+5F j
                 pop     edi
                 pop     esi
                 pop     ebx
                 add     esp, 0CCh
                 cmp     ebp, esp
                 call    j___RTC_CheckEsp
                 mov     esp, ebp
                 pop     ebp
                 retn    4

```

经过逆向分析得到C++代码：
```C++
void* __thiscall MyClass::vector_deleting_destructor(usigned int flag)
{
    if(flag&2)//由于push的是3，因此这里成立
    {
         vector_destructor_iterator(this,sizeof(MyClass),*(int*)((char*)this-4),MyClass::~MyClass);
         if(flag&1))//由于push的是3，因此这里成立
      {
          delete[]((char*)this-4);
      }
    }
    else
    {
         this->~MyClass();
         if(flag&1)
         {
          delete(this);
      }
    }
}
```

仅从以上代码可以分析出以下几点：
* 1.this-4这个地址为之前new[]()成功分配所返回值，可以将其看成sizeof(int)+sizeof(MyClass[2])大小的结构体，第一个成员为对象个数。
* 2.该函数对数组和非数组进行了分别处理，可以分析出第2个二进制位为1时，是析构对象数组，为0时是析构普通对象。而第1个二进制位是规定是否释放内存，可以想象如果这里是定位new，那么这里是不应该释放的。
* 3.vector_destructor_iterator起实际析构作用原型`
void __stdcall vector_destructor_iterator(MyClass *objs, unsigned int size, int count, void (__thiscall *pDtor)(void *));`，下面来看该函数

```Asm
                push    ebp
                mov     ebp, esp
                push    0FFFFFFFEh
                push    offset stru_41F9C0
                push    offset j___except_handler4
                mov     eax, large fs:0
                push    eax
                add     esp, 0FFFFFFF4h
                push    ebx
                push    esi
                push    edi
                mov     eax, ___security_cookie
                xor     [ebp+ms_exc.registration.ScopeTable], eax
                xor     eax, ebp
                push    eax
                lea     eax, [ebp+ms_exc.registration]
                mov     large fs:0, eax
                mov     [ebp+success], 0
                mov     eax, [ebp+size]
                imul    eax, [ebp+count]
                add     eax, [ebp+ptr]
                mov     [ebp+ptr], eax
                mov     [ebp+ms_exc.registration.TryLevel], 0

loc_41580B:                             ; CODE XREF: `eh vector destructor iterator'(void *,uint,int,void (*)(void *))+65 j
                mov     ecx, [ebp+count]
                sub     ecx, 1
                mov     [ebp+count], ecx
                js      short loc_415827
                mov     edx, [ebp+ptr]
                sub     edx, [ebp+size]
                mov     [ebp+ptr], edx
                mov     ecx, [ebp+ptr]
                call    [ebp+pDtor]
                jmp     short loc_41580B
; ---------------------------------------------------------------------------

loc_415827:                             ; CODE XREF: `eh vector destructor iterator'(void *,uint,int,void (*)(void *))+54 j
                mov     [ebp+success], 1
                mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh
                call    $LN8            ; Finally handler 0 for function 4157C0
; ---------------------------------------------------------------------------

loc_41583A:                             ; CODE XREF: `eh vector destructor iterator'(void *,uint,int,void (*)(void *)):$LN9_0 j
                jmp     short $LN11
; ---------------------------------------------------------------------------

$LN8:                                   ; CODE XREF: `eh vector destructor iterator'(void *,uint,int,void (*)(void *))+75 j
                                        ; DATA XREF: .rdata:stru_41F9C0 o
                cmp     [ebp+success], 0 ; Finally handler 0 for function 4157C0
                jnz     short $LN9_0
                mov     eax, [ebp+pDtor]
                push    eax             ; pDtor
                mov     ecx, [ebp+count]
                push    ecx             ; count
                mov     edx, [ebp+size]
                push    edx             ; size
                mov     eax, [ebp+ptr]
                push    eax             ; ptr
                call    j_?__ArrayUnwind@@YGXPAXIHP6EX0@Z@Z ; __ArrayUnwind(void *,uint,int,void (*)(void *))

$LN9_0:                                 ; CODE XREF: `eh vector destructor iterator'(void *,uint,int,void (*)(void *))+80 j
                retn
; ---------------------------------------------------------------------------

$LN11:                                  ; CODE XREF: `eh vector destructor iterator'(void *,uint,int,void (*)(void *)):loc_41583A j
                mov     ecx, [ebp+ms_exc.registration.Next]
                mov     large fs:0, ecx
                pop     ecx
                pop     edi
                pop     esi
                pop     ebx
                mov     esp, ebp
                pop     ebp
                retn    10h

```

经过逆向分析得到C++代码：

```C++
void __stdcall vector_destructor_iterator(MyClass *objs, unsigned int size, int count, void (__thiscall *pDtor)(void *))
{
    int i=0;
    __try
    {
        MyClass* last=objs+size-1;//从最后一个对象开始析构
        while(count--)
        {
           last->pDtor();
           last--;
        }
    }
    __except(1)
    {
        __ArrayUnwind(objs,size,count,pDtor);//如果某个析构函数产生异常，则跳过该对象，继续析构之前的对象 
    }
}
```

&emsp;&emsp;鉴于__ArrayUnwind前面已经介绍过，这里就不分析了。如果仔细分析上一节开头给出main汇编代码，会发现只要new成功了，delete都会去执行析构，即使出现对象数组中某个对象构造失败导致已经进行析构，delete时所有元素仍会析构一次。

## 探究第二种情况new[]形式实现
`MyClass* fPtr5 = new( nothrow ) MyClass[2];`

```C++
// newaopnt.cpp
void * __CRTDECL operator new[](::size_t count, const std::nothrow_t& x)
    _THROW0()
    {   // try to allocate count bytes for an array
    return (operator new(count, x));
    }
```

可见调用了单对象的第二种new形式，与非数组形式类似，不再赘述

## 探究第三种情况new[]形式实现

```C++
char x2[2*sizeof( MyClass ) + sizeof(int)];
MyClass* fPtr6 = new ( &x2[0] ) MyClass[2];

inline void *__CRTDECL operator new[](size_t, void *_Where) _THROW0()
{   // construct array with placement at _Where
    return (_Where);
}
```

&emsp;&emsp;可见等同于对象第三种new形式，不再赘述。下面我们来看看其他内存分配函数

## malloca
`void* _malloca(size_t size);`
&emsp;&emsp;MSDN里是这么描述的：在栈上分配内存，是_alloca的安全性增强版本。返回指针是根据对象大小对齐，如果size是0则返回长度0的合法指针。如果地址空间无法分配会抛出一个栈溢出异常，该异常不是C++异常，需要使用SEH。_malloca和_alloca的区别在于_alloc无论大小总是在栈上分配，且无需free释放内存。而_malloca需要使用_freea释放内存，在调试模式下,_malloca总是在堆上分配。在异常处理时显式调用_malloca有一些限制，x86架构处理器异常处理例程会自动控制函数栈帧，在执行操作时并不基于当前闭合函数栈帧，这一点在Windows NT SEH和C++异常处理的catch语句中很常见。因此在以下情况显示调用_malloca，在执行异常处理例程后会产生程序崩溃。  
&emsp;&emsp;Windows NT SEH异常过滤表达式：__except(_malloca())  
&emsp;&emsp;Windows NT SEH最终执行表达式：__finally(_malloca())  
&emsp;&emsp;C++ 异常处理 catch语句  
&emsp;&emsp;然而_malloca可以从异常处理例程中除上述情况以外的情况下直接调用，或在异常处理所触发的回调函数中调用也是允许的。先来看一个例子：

```C++
#include <windows.h>
#include <stdio.h>
#include <malloc.h>
 
int main()
{
    int     size;
    int     numberRead = 0;
    int     errcode = 0;
    void    *p = NULL;
    void    *pMarker = NULL;
 
    while (numberRead == 0)
    {
        printf_s("Enter the number of bytes to allocate "
                 "using _malloca: ");
        numberRead = scanf_s("%d", &size);
    }
 
    // Do not use try/catch for _malloca,
    // use __try/__except, since _malloca throws
    // Structured Exceptions, not C++ exceptions.
 
    __try
    {
        if (size > 0)
        {
            p =  _malloca( size );
        }
        else
        {
            printf_s("Size must be a positive number.");
        }
        _freea( p );
    }
 
    // Catch any exceptions that may occur.
    __except( GetExceptionCode() == STATUS_STACK_OVERFLOW )
    {
        printf_s("_malloca failed!\n");
 
        // If the stack overflows, use this function to restore.
        errcode = _resetstkoflw();
        if (errcode)
        {
            printf("Could not reset the stack!");
            _exit(1);
        }
    };
}
```

```C++
// malloc.h
#define _ALLOCA_S_THRESHOLD     1024
#define _ALLOCA_S_STACK_MARKER  0xCCCC
#define _ALLOCA_S_HEAP_MARKER   0xDDDD
 
#if defined(_M_IX86)
#define _ALLOCA_S_MARKER_SIZE   8
#elif defined(_M_X64)
#define _ALLOCA_S_MARKER_SIZE   16
#elif defined(_M_ARM)
#define _ALLOCA_S_MARKER_SIZE   8
#elif !defined (RC_INVOKED)
#error Unsupported target platform.
#endif
 
......
 
#if !defined(__midl) && !defined(RC_INVOKED)
#pragma warning(push)
#pragma warning(disable:6540)
__inline void *_MarkAllocaS(_Out_opt_ __crt_typefix(unsigned int*) void *_Ptr, unsigned int _Marker)
{
    if (_Ptr)
    {
        *((unsigned int*)_Ptr) = _Marker;
        _Ptr = (char*)_Ptr + _ALLOCA_S_MARKER_SIZE;
    }
    return _Ptr;
}
#pragma warning(pop)
#endif
 
#if defined(_DEBUG)
#if !defined(_CRTDBG_MAP_ALLOC)
#undef _malloca
#define _malloca(size) \
__pragma(warning(suppress: 6255)) \
        _MarkAllocaS(malloc((size) + _ALLOCA_S_MARKER_SIZE), _ALLOCA_S_HEAP_MARKER)
#endif
#else
#undef _malloca
#define _malloca(size) \
__pragma(warning(suppress: 6255)) \
    ((((size) + _ALLOCA_S_MARKER_SIZE) <= _ALLOCA_S_THRESHOLD) ? \
        _MarkAllocaS(_alloca((size) + _ALLOCA_S_MARKER_SIZE), _ALLOCA_S_STACK_MARKER) : \
        _MarkAllocaS(malloc((size) + _ALLOCA_S_MARKER_SIZE), _ALLOCA_S_HEAP_MARKER))
#endif
```

&emsp;&emsp;可以分析出DEBUG版下，宏调用了malloc进行分配，之后使用_MarkAllocaS对分配内存进行一些处理（后面讨论），而RELEASE版下，宏先判断要分配的内存是否过大，该门限为_ALLOCA_S_THRESHOLD-_ALLOCA_S_MARKER_SIZE=1016，如果超过该值则调用malloc，否则调用_alloca。从字面意思上可以知道_ALLOCA_S_HEAP_MARKER这个标志位说明该内存区是在堆上分配的，而_ALLOCA_S_STACK_MARKER标志是在栈上分配的。在malloc或_alloca分配成功后，总会调用_MarkAllocaS进行调整。结合字面意思和5行C语言代码可知，在执行过内存分配后，返回的指针前sizeof(unigned int*)字节为分配内存类型标志，之后指针调整到空闲位置丢给用户操作。那么所有的问题都落在_alloca和malloc的源码上，下面会进行分析。  
&emsp;&emsp;_alloca（我第一次见栈上分配内存是在逆向一个易语言程序时，用的是sub esp，而微软这个函数是第一次见）`void* _alloca(size_t size);`，该函数只在程序栈中分配字节，而函数退出时该空间会自动释放，因此无需手动释放。用此函数的限制和_malloca相同。  

```C++
#include <windows.h>
#include <stdio.h>
#include <malloc.h>
 
int main()
{
    int     size = 1000;
    int     errcode = 0;
    void    *pData = NULL;
 
    // 注意：不要使用try/catch，而要使用__try/__except，因为_alloca抛出SEH而不是C++异常
    __try {
        // 使用_alloca分配太大的空间很容易崩溃，推荐1024字节以下的空间  
        if (size > 0 && size < 1024)
        {
            pData = _alloca( size );
            printf_s( "Allocated %d bytes of stack at 0x%p",
                      size, pData);
        }
        else
        {
            printf_s("Tried to allocate too many bytes.\n");
        }
    }
 
    // 如果溢出
    __except( GetExceptionCode() == STATUS_STACK_OVERFLOW )
    {
        printf_s("_alloca failed!\n");
 
        // 使用下面的函数恢复函数栈
        errcode = _resetstkoflw();
        if (errcode)
        {
            printf_s("Could not reset the stack!\n");
            _exit(1);
        }
    };
}
```

来看反汇编
```Asm
; int __cdecl main()
_main           proc near               ; CODE XREF: j__main j

pAllocaBase     = dword ptr -120h
cbSize          = dword ptr -11Ch
var_114         = dword ptr -114h
allocaList      = dword ptr -48h
pData           = dword ptr -3Ch
errcode         = dword ptr -30h
size            = dword ptr -24h
var_1C          = dword ptr -1Ch
ms_exc          = CPPEH_RECORD ptr -18h

                push    ebp
                mov     ebp, esp
                push    0FFFFFFFEh
                push    offset stru_416F80
                push    offset j___except_handler4
                mov     eax, large fs:0
                push    eax
                add     esp, 0FFFFFEF0h
                push    ebx
                push    esi
                push    edi
                lea     edi, [ebp+pAllocaBase]
                mov     ecx, 42h
                mov     eax, 0CCCCCCCCh
                rep stosd
                mov     eax, ___security_cookie
                xor     [ebp+ms_exc.registration.ScopeTable], eax
                xor     eax, ebp
                mov     [ebp+var_1C], eax
                push    eax
                lea     eax, [ebp+ms_exc.registration]
                mov     large fs:0, eax
                mov     [ebp+ms_exc.old_esp], esp
                mov     [ebp+allocaList], 0
                mov     [ebp+size], 3E8h
                mov     [ebp+errcode], 0
                mov     [ebp+pData], 0
                mov     [ebp+ms_exc.registration.TryLevel], 0
                cmp     [ebp+size], 0
                jle     short loc_4114D3
                cmp     [ebp+size], 400h
                jge     short loc_4114D3
                mov     eax, [ebp+size]
                add     eax, 24h
                mov     [ebp+cbSize], eax
                mov     eax, [ebp+cbSize]
                call    j___alloca_probe_16
                mov     [ebp+pAllocaBase], esp
                mov     [ebp+ms_exc.old_esp], esp
                lea     ecx, [ebp+allocaList]
                push    ecx             ; pAllocaInfoList
                mov     edx, [ebp+cbSize] ; cbSize
                mov     ecx, [ebp+pAllocaBase] ; pAllocaBase
                call    j_@_RTC_AllocaHelper@12 ; _RTC_AllocaHelper(x,x,x)
                add     [ebp+pAllocaBase], 20h
                mov     edx, [ebp+pAllocaBase]
                mov     [ebp+pData], edx
                mov     esi, esp
                mov     eax, [ebp+pData]
                push    eax
                mov     ecx, [ebp+size]
                push    ecx
                push    offset Format   ; "Allocated %d bytes of stack at 0x%p"
                call    ds:__imp__printf_s
                add     esp, 0Ch
                cmp     esi, esp
                call    j___RTC_CheckEsp
                jmp     short loc_4114EA
; ---------------------------------------------------------------------------

loc_4114D3:                             ; CODE XREF: _main+72 j
                                        ; _main+7B j
                mov     esi, esp
                push    offset aTriedToAllocat ; "Tried to allocate too many bytes.\n"
                call    ds:__imp__printf_s
                add     esp, 4
                cmp     esi, esp
                call    j___RTC_CheckEsp

loc_4114EA:                             ; CODE XREF: _main+E1 j
                mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh
                jmp     loc_41158D
; ---------------------------------------------------------------------------

$LN10:                                  ; DATA XREF: .rdata:stru_416F80 o
                mov     eax, [ebp+ms_exc.exc_ptr] ; Exception filter 0 for function 4113F0
                mov     ecx, [eax]
                mov     edx, [ecx]
                mov     [ebp+var_114], edx
                cmp     [ebp+var_114], 0C00000FDh
                jnz     short loc_41151B
                mov     [ebp+cbSize], 1
                jmp     short loc_411525
; ---------------------------------------------------------------------------

loc_41151B:                             ; CODE XREF: _main+11D j
                mov     [ebp+cbSize], 0

loc_411525:                             ; CODE XREF: _main+129 j
                mov     eax, [ebp+cbSize]

$LN12:
                retn
; ---------------------------------------------------------------------------

$LN11:                                  ; DATA XREF: .rdata:stru_416F80 o
                mov     esp, [ebp+ms_exc.old_esp] ; Exception handler 0 for function 4113F0
                mov     esi, esp
                push    offset a_allocaFailed ; "_alloca failed!\n"
                call    ds:__imp__printf_s
                add     esp, 4
                cmp     esi, esp
                call    j___RTC_CheckEsp
                mov     esi, esp
                call    ds:__imp___resetstkoflw
                cmp     esi, esp
                call    j___RTC_CheckEsp
                mov     [ebp+errcode], eax
                cmp     [ebp+errcode], 0
                jz      short loc_411586
                mov     esi, esp
                push    offset aCouldNotResetT ; "Could not reset the stack!\n"
                call    ds:__imp__printf_s
                add     esp, 4
                cmp     esi, esp
                call    j___RTC_CheckEsp
                mov     esi, esp
                push    1               ; Code
                call    ds:__imp___exit
; ---------------------------------------------------------------------------
                cmp     esi, esp
                call    j___RTC_CheckEsp

loc_411586:                             ; CODE XREF: _main+16C j
                mov     [ebp+ms_exc.registration.TryLevel], 0FFFFFFFEh

loc_41158D:                             ; CODE XREF: _main+101 j
                jmp     short loc_411591
; ---------------------------------------------------------------------------
                jmp     short loc_411593
; ---------------------------------------------------------------------------

loc_411591:                             ; CODE XREF: _main:loc_41158D j
                xor     eax, eax

loc_411593:                             ; CODE XREF: _main+19F j
                push    edx
                mov     ecx, ebp        ; frame
                push    eax
                lea     edx, v          ; v
                push    [ebp+allocaList] ; allocaList
                call    j_@_RTC_CheckStackVars2@12 ; _RTC_CheckStackVars2(x,x,x)
                pop     eax
                pop     edx
                lea     esp, [ebp-130h]
                mov     ecx, [ebp+ms_exc.registration.Next]
                mov     large fs:0, ecx
                pop     ecx
                pop     edi
                pop     esi
                pop     ebx
                mov     ecx, [ebp+var_1C]
                xor     ecx, ebp        ; cookie
                call    j_@__security_check_cookie@4 ; __security_check_cookie(x)
                mov     esp, ebp
                pop     ebp
                retn

                mov     eax, [ebp+size]
                add     eax, 24h
                mov     [ebp+cbSize], eax
                mov     eax, [ebp+cbSize]
                call    j___alloca_probe_16
                mov     [ebp+pAllocaBase], esp
                mov     [ebp+ms_exc.old_esp], esp
                lea     ecx, [ebp+allocaList]
                push    ecx             ; pAllocaInfoList
                mov     edx, [ebp+cbSize] ; cbSize
                mov     ecx, [ebp+pAllocaBase] ; pAllocaBase
                call    j_@_RTC_AllocaHelper@12 ; _RTC_AllocaHelper(x,x,x)
                add     [ebp+pAllocaBase], 20h
                mov     edx, [ebp+pAllocaBase]
                mov     [ebp+pData], edx

```
&emsp;&emsp;很疑惑地，在__alloca_probe_16调用之前，发生了add eax,24h和mov eax,[ebp+cbSize]，而在之后发生了mov [ebp+pAllocaBase], esp，那么大胆做出猜测：
* 1.add eax,24h，说明这24h字节用来实现内存管理或字节对齐之类功能
* 2.__alloca_probe_16为接受一个参数的函数，该参数通过eax传递，进行的操作是修改esp，因此esp可以看做执行结果
* 3.另一个函数原型为`void __fastcall _RTC_AllocaHelper(_RTC_ALLOCA_NODE *pAllocaBase, unsigned int cbSize, _RTC_ALLOCA_NODE **pAllocaInfoList)`，反汇编得到

```Asm
; void __fastcall _RTC_AllocaHelper(_RTC_ALLOCA_NODE *pAllocaBase, unsigned int cbSize, _RTC_ALLOCA_NODE **pAllocaInfoList)
@_RTC_AllocaHelper@12 proc near         ; CODE XREF: _RTC_AllocaHelper(x,x,x) j
pAllocaInfoList = dword ptr  8
pAllocaBase = ecx
cbSize = edx
                push    ebp
                mov     ebp, esp
                push    ebx
                push    esi
                mov     esi, pAllocaBase
                mov     ebx, cbSize
                test    esi, esi
                jz      short loc_4116CC
                test    ebx, ebx
                jz      short loc_4116CC
                mov     cbSize, [ebp+pAllocaInfoList]
                test    cbSize, cbSize
                jz      short loc_4116CC
                push    edi
                mov     al, 0CCh
                mov     edi, esi
                mov     pAllocaBase, ebx
                rep stosb
                mov     eax, [cbSize]
                mov     [esi+4], eax
                mov     [esi+0Ch], ebx
                mov     [cbSize], esi
                pop     edi
loc_4116CC:                             ; CODE XREF: _RTC_AllocaHelper(x,x,x)+B j
                                        ; _RTC_AllocaHelper(x,x,x)+F j ...
                pop     esi
                pop     ebx
                pop     ebp
                retn    4
@_RTC_AllocaHelper@12 endp
```

逆向分析得道C++代码：

```C++
void main()
{
    ......
    int size=1024,cbsize=size+sizeof(_RTC_ALLOCA_NODE)+4;
    _RTC_ALLOCA_NODE* pAllocaBase=__alloca_probe_16(cbsize);
    _RTC_AllocaHelper(pAllocaBase,cbsize,NULL);
    void* pData=(void*)(pAllocaBase+1);
    ......
}
 
//这个结构体来自于reactos
#pragma pack(push,1)
typedef struct _RTC_ALLOCA_NODE
{
     __int32 guard1;
    struct _RTC_ALLOCA_NODE *next;
    #if (defined(_X86_) && !defined(__x86_64))
        __int32 dummypad;
    #endif
        size_t allocaSize;
    #if (defined(_X86_) && !defined(__x86_64))
        __int32 dummypad2;
    #endif
        __int32 guard2[3];
}_RTC_ALLOCA_NODE;
#pragma pack(pop)
 
void __fastcall _RTC_AllocaHelper(_RTC_ALLOCA_NODE *pAllocaBase, unsigned int cbSize, _RTC_ALLOCA_NODE **pAllocaInfoList)
{//初始化已分配空间，可以用于维护调试版函数栈
  if(pAllocaBase && cbSize && pAllocaInfoList)//由于最后一个参数在本例中为0，这个函数实际相当于没有执行
  {
     memset(pAllocaBase,0xCC,cbSize);//经常在调试版程序栈空间看到0xCC  "烫烫烫烫烫"  对吧，就是这样的。。。
     pAllocaBase->next=*pAllocaInfoList;//链接到前一个结构；
     pAllocaBase->allocaSize=cbSize;
     *pAllocaInfoList=pAllocaBase;//自此可知，上述结构形成链表，pAllocaInfoList指向当前结构
  }
}
//__alloca_probe_16代码下面会进行分析
```

&emsp;&emsp;以上是debug版的情况，如果尝试用release版查看反汇编代码，会发现只有push和call __alloca_probe_16部分，可知add eax,24和AllocaHelper只是调试版本用于内存管理的。所以重点落在该函数的解析上。进入源代码查看，__alloca_probe_16用来按16字节对齐内存，而chkstk子例程进行实际分配操作：

```Asm
// alloca16.asm

; _alloca_probe_16, _alloca_probe_8 - 按8/16字节对齐例程
;输入:EAX = 栈帧大小
;输出:调整EAX，修改esp.
 
public  _alloca_probe_8
_alloca_probe_16 proc                   ; 16 byte aligned alloca
 
        push    ecx
        lea     ecx, [esp] + 8          ; 父函数栈顶（call _alloca_probe_16和push ecx）
        sub     ecx, eax                ; 
        and     ecx, (16 - 1)           ; 计算地址低4位未对齐偏移
        add     eax, ecx                ; 增加cbSize使其对齐
        sbb     ecx, ecx                ; 如果cbSize溢出，ecx = 0xFFFFFFFF，否则ecx = 0
        or      eax, ecx                ; 如果溢出，则eax = 0xFFFFFFFF
        pop     ecx                     ; 还原ecx
        jmp     _chkstk         ; eax存储修正cbSize，并交给_chkstk处理
_alloca_probe_16 endp
 
        end
 
 
public  _alloca_probe
_chkstk proc
_alloca_probe    =  _chkstk
        push    ecx
        lea     ecx, [esp] + 8 - 4      ; 考虑到之后的ret指令对未来esp的修改
        sub     ecx, eax                ; 分配栈空间，ecx存储更新后的栈位置
        sbb     eax, eax                ; 如果申请空间过大，eax = 0xFFFFFFFF，否则eax = 0
        not     eax                     ; 
        and     ecx, eax                ; ecx = 0 | ecx = ecx
        mov     eax, esp                ;
        and     eax, not ( _PAGESIZE_ - 1) ; 得到当前栈位置所处页面地址
cs10:
        cmp     ecx, eax                ; 
        jb      short cs20              ; 如果新的栈位置小于页面地址
        mov     eax, ecx                ; 
        pop     ecx
        xchg    esp, eax                ; 更新esp，原始esp存储在eax中
        mov     eax, dword ptr [eax]    ; 当前esp指向返回地址
        mov     dword ptr [esp], eax    ; 修正函数栈帧，使其可以正确返回
        ret
cs20:
        sub     eax, _PAGESIZE_         ; 获取上一个页面
        test    dword ptr [eax],eax     ; 探测页面权限
        jmp     short cs10      ; 如果没有产生异常则跳转，如果出现异常，则直接进入父函数的异常处理中
 
_chkstk endp
        end
```

## calloc
`void* calloc(size_t num,size_t size);`  
&emsp;&emsp;calloc用来分配数组空间，同样返回指针是根据对象类型对齐的，每个对象都被初始化为0，如果待分配内存超过_HEAP_MAXREQ或分配失败则设置errno为ENOMEM，calloc内部调用了malloc函数使用_set_new_mode函数设置回调模式，该回调用于处理分配失败情况，默认情况下，分配失败后malloc不会调用新回调分配内存，然后我们可以通过提前调用_set_new_mode(1)或者链接NEWMODE.OBJ修改这种默认行为.calloc用法如下：
```C++
#include <stdio.h>
#include <malloc.h>
 
int main( void )
{
   long *buffer;
 
   buffer = (long *)calloc( 40, sizeof( long ) );
   if( buffer != NULL )
      printf( "Allocated 40 long integers\n" );
   else
      printf( "Can't allocate memory\n" );
   free( buffer );
}
```

Calloc源码：

```C++
// calloc.c和calloc_impl.c
void * __cdecl _calloc_base (size_t num, size_t size)
{
    int errno_tmp = 0;
    void * pv = _calloc_impl(num, size, &errno_tmp);
 
    if ( pv == NULL && errno_tmp != 0 && _errno())
    {
        errno = errno_tmp; // recall, #define errno *_errno()
    }
    return pv;
}
void * __cdecl _calloc_impl (size_t num, size_t size, int * errno_tmp)
{
        size_t  size_orig;
        void *  pvReturn;
 
        /* ensure that (size * num) does not overflow */
        if (num > 0)
        {
            _VALIDATE_RETURN_NOEXC((_HEAP_MAXREQ / num) >= size, ENOMEM, NULL);
        }
        size_orig = size = size * num;
 
 
        /* force nonzero size */
        if (size == 0)
            size = 1;
 
        for (;;)
        {
            pvReturn = NULL;
 
            if (size <= _HEAP_MAXREQ)
            {
                if (pvReturn == NULL)
                    pvReturn = HeapAlloc(_crtheap, HEAP_ZERO_MEMORY, size);
            }
 
            if (pvReturn || _newmode == 0)
            {
                RTCCALLBACK(_RTC_Allocate_hook, (pvReturn, size_orig, 0));
                if (pvReturn == NULL)
                {
                    if ( errno_tmp )
                        *errno_tmp = ENOMEM;
                }
                return pvReturn;
            }
 
            /* call installed new handler */
            if (!_callnewh(size))
            {
                if ( errno_tmp )
                    *errno_tmp = ENOMEM;
                return NULL;
            }
 
            /* new handler was successful -- try to allocate again */
        }
}
```

&emsp;&emsp;现在来分析_calloc_impl执行流程：
* 1.先检查申请大小是否超出门限，若申请大小为0则强制为1
* 2.使用HeapAlloc分配内存并清零。如果成功则返回，否则执行_callnewh，即定义的失败处理函数，如果该回调函数返回0则原函数返回0退出，如果该回调函数返回非0，则原函数重复执行2直到成功。（_callnewh最终调用了NtQueryInformationProcess 0x24）  

&emsp;&emsp;可见calloc并没有像MSDN说的那样调用了malloc。。。另外，没看到有异常处理机制。

## _expand
&emsp;&emsp;用于扩展或缩小已分配内存，用于改变已分配内存区大小。`void* _expand(void* memblock,size_t newsize);`  
&emsp;&emsp;该函数会检测地址的内存权限，如果不通过移动内存无法得到足够的空间，该函数会返回空，该函数不会分配小于请求大小的内存区。该函数不通过移动内存块的方式增缩已分配堆内存，在64位下该函数不会缩小内存区，大小小于16k的内存块都是在低碎片堆中分配的，在这种情况下_expand不会对内存块做任何变动直接返回memblock。同样该函数会检测参数合法性，且size不能超过门限值_HEAP_MAXREQ。

```C++
#include <stdio.h>
#include <malloc.h>
#include <stdlib.h>
 
int main( void )
{
   char *bufchar;
   printf( "Allocate a 512 element buffer\n" );
   if( (bufchar = (char *)calloc( 512, sizeof( char ) )) == NULL )
      exit( 1 );
   printf( "Allocated %d bytes at %Fp\n", 
         _msize( bufchar ), (void *)bufchar );
   if( (bufchar = (char *)_expand( bufchar, 1024 )) == NULL )
      printf( "Can't expand" );
   else
      printf( "Expanded block to %d bytes at %Fp\n", 
            _msize( bufchar ), (void *)bufchar );
   // Free memory 
   free( bufchar );
   exit( 0 );
}
```

expand源码为：

```C++
void * __cdecl _expand_base (void * pBlock, size_t newsize)
{
        void *      pvReturn;
 
        size_t oldsize;
 
        /* validation section */
        _VALIDATE_RETURN(pBlock != NULL, EINVAL, NULL);
        if (newsize > _HEAP_MAXREQ) {
            errno = ENOMEM;
            return NULL;
        }
 
        if (newsize == 0)
        {
            newsize = 1;
        }
 
        oldsize = (size_t)HeapSize(_crtheap, 0, pBlock);
 
        pvReturn = HeapReAlloc(_crtheap, HEAP_REALLOC_IN_PLACE_ONLY, pBlock, newsize);
 
        if (pvReturn == NULL)
        {
            /* 如果使用了低碎片堆则返回原指针. */
            if (oldsize <= 0x4000 /* 低碎片堆最多申请16KB内存 */
                    && newsize <= oldsize && _is_LFH_enabled())
                pvReturn = pBlock;
            else
                errno = _get_errno_from_oserr(GetLastError());
        }
 
        if (pvReturn)
        {
            RTCCALLBACK(_RTC_Free_hook, (pBlock, 0));
            RTCCALLBACK(_RTC_Allocate_hook, (pvReturn, newsize, 0));
        }
 
        return pvReturn;
}
```

&emsp;&emsp;可见_expand函数最终调用了HeapReAlloc函数。  
&emsp;&emsp;一个小插曲：其实这些内存管理的库函数普遍使用debug和release两个版本，debug源码在dbg*.c(pp)中可以找到，同时调试的时候是可以直接定位进去的，而release版函数通常在函数名所在文件，比如delete在delete.cpp中；而debug版源码内存管理函数通常会有一种_CrtMemBlockHeader的结构体，这些都需要自己摸索。

## realloc
源码：
```C++
void * __cdecl _realloc_base (void * pBlock, size_t newsize)
{
        void *      pvReturn;
        size_t      origSize = newsize;
 
        //  if ptr is NULL, call malloc
        if (pBlock == NULL)
            return(_malloc_base(newsize));
 
        //  if ptr is nonNULL and size is zero, call free and return NULL
        if (newsize == 0)
        {
            _free_base(pBlock);
            return(NULL);
        }
 
 
        for (;;) {
 
            pvReturn = NULL;
            if (newsize <= _HEAP_MAXREQ)
            {
                if (newsize == 0)
                    newsize = 1;
                pvReturn = HeapReAlloc(_crtheap, 0, pBlock, newsize);
            }
            else
            {
                _callnewh(newsize);
                errno = ENOMEM;
                return NULL;
            }
 
            if ( pvReturn || _newmode == 0)
            {
                if (pvReturn)
                {
                    RTCCALLBACK(_RTC_Free_hook, (pBlock, 0));
                    RTCCALLBACK(_RTC_Allocate_hook, (pvReturn, newsize, 0));
                }
                else
                {
                    errno = _get_errno_from_oserr(GetLastError());
                }
                return pvReturn;
            }
 
            //  call installed new handler
            if (!_callnewh(newsize))
            {
                errno = _get_errno_from_oserr(GetLastError());
                return NULL;
            }
 
            //  new handler was successful -- try to allocate again
        }
}
```

得到realloc->HeapReAlloc

## malloc
源码：
```C++
void * __cdecl _malloc_base (size_t size)
{
    void *res = NULL;
    //  validate size
    if (size <= _HEAP_MAXREQ) {
        for (;;) {
            //  allocate memory block
            res = _heap_alloc(size);
            //  if successful allocation, return pointer to memory
            //  if new handling turned off altogether, return NULL
            if (res != NULL)
            {
                break;
            }
            if (_newmode == 0)
            {
                errno = ENOMEM;
                break;
            }
            //  call installed new handler
            if (!_callnewh(size))
                break;
            //  new handler was successful -- try to allocate again
        }
    } else {
        _callnewh(size);
        errno = ENOMEM;
        return NULL;
    }
    RTCCALLBACK(_RTC_Allocate_hook, (res, size, 0));
    if (res == NULL)
    {
        errno = ENOMEM;
    }
    return res;
}
__forceinline void * __cdecl _heap_alloc (size_t size)
{
    if (_crtheap == 0) {
        _FF_MSGBANNER();    /* write run-time error banner */
        _NMSG_WRITE(_RT_CRT_NOTINIT);  /* write message */
        __crtExitProcess(255);  /* normally _exit(255) */
    }
    return HeapAlloc(_crtheap, 0, size ? size : 1);
}
```

得到malloc->_heap_alloc->HeapAlloc

## free
源码：

```C++
void __cdecl _free_base (void * pBlock)
{
        int retval = 0;
        if (pBlock == NULL)
            return;
        RTCCALLBACK(_RTC_Free_hook, (pBlock, 0));
        retval = HeapFree(_crtheap, 0, pBlock);
        if (retval == 0)
        {
            errno = _get_errno_from_oserr(GetLastError());
        }
}
```
得到free->HeapFree

现在所有问题都集中在了HeapAlloc HeapFree HeapReAlloc上

