# static的作用--2019.3.4

## 概述
>static关键字在c语言中比较常用，使用恰当能够大大提高程序的模块化特性，有利于扩展和维护。
>
>在程序中使用static

## 变量
### 1. 局部变量
普通局部变量是再熟悉不过的变量了，在任何一个函数内部定义的变量（不加static修饰符）都属于这个范畴。编译器一般不对普通局部变量进行初始化，也就是说它的值在初始时是不确定的，除非对其显式赋值。   
```
普通局部变量存储于进程栈空间，使用完毕会立即释放。
```
静态局部变量使用static修饰符定义，即使在声明时未赋初值，编译器也会把它初始化为0。且静态局部变量存储于进程的全局数据区，即使函数返回，它的值也会保持不变。    
```
变量在全局数据区分配内存空间；编译器自动对其初始化      
其作用域为局部作用域，当定义它的函数结束时，其作用域随之结束     
```

小程序体会一下静态局部变量的威力：
```C
#include <stdio.h>
void fn(void)
{
	int n = 10;
	printf("n=%d\n", n);
	n++;
	printf("n++=%d\n", n);

}
void fn_static(void)
{
	static int n = 10;

	printf("static n=%d\n", n);
	n++;
	printf("n++=%d\n", n);
}
int main(void)
{
	fn(); 
	printf("--------------------\n"); 
	fn_static(); 
	printf("--------------------\n"); 
	fn(); 
	printf("--------------------\n"); 
	fn_static(); 
	
	return 0; 
}
```
运行结果如下：
```C
-> % ./a.out 
n=10
n++=11
--------------------
static n=10
n++=11
--------------------
n=10
n++=11
--------------------
static n=11
n++=12
```

可见，静态局部变量的效果跟全局变量有一拼，但是位于函数体内部，就极有利于程序的模块化了。

### 2. 全局变量
全局变量定义在函数体外部，在全局数据区分配存储空间，且编译器会自动对其初始化。

普通全局变量对整个工程可见，其他文件可以使用extern外部声明后直接使用。也就是说其他文件不能再定义一个与其相同名字的变量了（否则编译器会认为它们是同一个变量）。

静态全局变量仅对当前文件可见，其他文件不可访问，其他文件可以定义与其同名的变量，两者互不影响。

	在定义不需要与其他文件共享的全局变量时，加上static关键字能够有效地降低程序模块之间的
	耦合,避免不同文件同名变量的冲突，且不会误使用。

## 函数

函数的使用方式与全局变量类似，在函数的返回类型前加上static，就是静态函数。其特性如下：

* 其他文件中可以定义相同名字的函数，不会发生冲突
* 静态函数不能被其他文件所用，而非静态函数可以在另一个文件中直接引用，甚至不必使用extern声明

下面两个文件的例子说明使用static声明的函数不能被另一个文件引用：
	
```c
/* file1.c */
#include <stdio.h>
static void fun(void)
{
    printf("hello from fun.\n");
}

int main(void)
{
    fun();
    fun1();
    return 0;
}

/* file2.c */
#include <stdio.h>

static void fun1(void)
{
    printf("hello from static fun1.\n");
}
```

使用 `gcc file1.c file2.c ` 编译时，错误报告如下：
```c
/tmp/cc2VMzGR.o：在函数‘main’中：
static_fun.c:(.text+0x20)：对‘fun1’未定义的引用
collect2: error: ld returned 1 exit status
	
修改文件，不使用static修饰符，可在另一文件中引用该函数： 
```

```C
/* file1.c */
#include <stdio.h>

void fun(void)
{
    printf("hello from fun.\n");
}

/* file2.c */
int main(void)
{
    fun();

    return 0;
}
```

同样使用 `gcc file1.c file2.c` 编译，编译通过，运行结果如下：

```C
-> % ./a.out 
hello from fun.
```

## 总结
static是一个很有用的关键字，使用得当可以使程序锦上添花。当然，有的公司编码规范明确规定只用于本文件的函数要全部使用static关键字声明，这是一个良好的编码风格。<\br>
无论如何，要在实际编码时注意自己的编码习惯，尽量体现出语言本身的优雅和编码者的编码素质。

--------------------- 
作者：guotianqing 
来源：CSDN 
原文：https://blog.csdn.net/guotianqing/article/details/79828100 
版权声明：本文为博主原创文章，转载请附上博文链接！
