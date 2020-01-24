---
title:  "python调用clib的两种方法"
date: 2015-08-19 16:45:29
excerpt: "python调用clib的两种方法"
tags: [clib]
---

这篇文章介绍在python中调用clib定义的函数，分别是**ctypes/Python API**的形式。

<!--more-->

### 1. ctypes

#### 1.1 一个简单的例子

首先准备hello.c文件：

```c
#include <stdio.h>

void hello()
{
    printf("Hello, World!\n");
}
```

其次准备makefile:  

```makefile
ALL:
        gcc -c -fPIC -o hello.o hello.c
        gcc -shared -o libhello.so hello.o
```

执行`make`生成动态库  
再编写hello.py:

```python
#!/usr/bin/env python
# -*- coding=utf-8 -*-


from ctypes import cdll, CDLL
from ctypes import create_string_buffer
from ctypes import c_int, c_ulonglong

# 有两种引入方式
# hellolibc = cdll.LoadLibrary('./libhello.so')
hellolibc = CDLL('./libhello.so')
hellolibc.hello()
```

一个简单的例子就编写完成了。

#### 1.2 注意事项

ctypes在使用上有很多需要注意的地方，这里介绍下一些常见的问题。  

##### 1.2.1 传入可变字符串

更新我们的hello.c文件:

```c
void hello()
{
    printf("Hello, World!\n");
}

void modify(char* str)
{
    int l = strlen(str);
    str[l] = 'C';
    str[l + 1] = '\0';
}
```
该方法对传入的字符串末尾追加一个字符'C'，执行make生成动态库。  

更新hello.py文件调用该方法：  


```python
#!/usr/bin/env python
# -*- coding=utf-8 -*-


from ctypes import cdll, CDLL
from ctypes import create_string_buffer
from ctypes import c_int, c_ulonglong

hellolibc = CDLL('./libhello.so')

# p is not immutable
p = 'hello'
print p  # hello
hellolibc.modify(p)
# p is unchanged
print p  # hello

#hellolibc.hello()

# p is immutable
p = create_string_buffer("hello")
print p.value  # hello
hellolibc.modify(p)
# p is changed
print p.value  # helloC
```
其中第一种直接传入的`p = 'hello'`的方法并没有成功修改p，需要调用`create_string_buffer`产生一段可以修改的字符串内存空间。    
第一种方法类似于修改一段只读的字符串，没有报错，但是应该会有写错的隐患。举个例子，如果把`hellolibc.hello()`的注释放开的话，会报错:  

```
AttributeError: ./libhello.so: undefined symbol: helloC
```

可以看到我们调用的是`hello`，程序实际找的却是`helloC`这个函数名，具体原因因为还没有看过python源码没法定位，只能猜测是函数名的记录被误改写了。  

##### 1.2.2. 修改返回值

更新hello.c文件：  

```c
unsigned long long add(unsigned long long a1, unsigned long long a2)
{
    return a1 + a2;
}
```
我们定义了一个add函数，输入是两个unsigned long long ，输出是求和后的值，类型也是unsigned long long。执行`make`更新动态库文件  

看下在hello.py的调用部分:  

```python
cadd = hellolibc.add
print cadd(1, 2)
print cadd(c_int(1), c_int(2))
print cadd(2147483648, 1)  # -2147483647
cadd.restype = c_ulonglong
print cadd(2147483648, 1)  # 2147483649
```

其中第一次调用`cadd(2147483648, 1)`这句时返回了负数，是因为函数默认返回值是c_int，而这个返回值超过了int的最大值，解决办法就是重新定义`restype`这个属性。  
 
##### 1.2.3 调用cpp文件编写的动态库

我们知道c和cpp编译出来的函数名是不同的，对于cpp文件编译出来的动态库，上述方法是否适用？  
编写测试用的hello.cpp文件：  

```c
#include <iostream>

void hello()
{
    std::cout << "Hello, World!" << std::endl;
}
```

编写makefile:  

```makefile
ALL:
        g++ -c -fPIC -o hello.o hello.cpp
        g++ -shared -o libhelloplus.so hello.o
```

执行`make`生成动态库libhelloplus.so  

更新hello.py中的调用部分：  

```python
ellolibcpp = CDLL('./libhelloplus.so')
hellolibcpp.hello()
```

结果报错：  

```
AttributeError: ./libhelloplus.so: undefined symbol: hello
```

可以看到在python中导入一个动态库时，是按照c的方式查找符号名字的。因此需要修改我们的hello.cpp:  

```cpp
#include <iostream>

extern "C" {
    void hello()
    {
        std::cout << "Hello, World!" << std::endl;
    }
}
```
重新生成动态库就可以正常调用了。  

关于ctypes更多使用的细节，比如如何传递数组、指针、自定义数据格式等，可以参考[python官方文档](https://docs.python.org/2/library/ctypes.html)  

### 2. Python API(Application Programmers Interface)

#### 2.1. 一个简单的例子  

这个例子是PythonDoc里自带的    
先看下use\_spammodule.py文件  

```python
#!/usr/bin/env python
# -*- coding=utf-8 -*-


import spammodule
spammodule.system("ls -l")
```

程序很简单，导入*spammodule*模块，然后执行模块下的方法system，这个方法会执行传入的字符串，与c使用的system非常相似。  

导入的模块*spammodule*其实是个动态库，生成该动态库的makefile如下：  

```makefile
PYINC = $(这里填你系统的python路径)include/python2.7
all:
        gcc -fPIC -shared -o spammodule.so spammodule.c -I$(PYINC)
```

可以看到是由*spammodule.c*编译出的动态库，继续看下*spammodule.c*的内容：  

```c
#include <Python.h>

static PyObject *spam_system(PyObject *self, PyObject *args)
{
    const char* command;
    int sts = 0;

    if (!PyArg_ParseTuple(args, "s", &command))
        return NULL;
    sts = system(command);
    return Py_BuildValue("i", sts);
}

static PyMethodDef SpamMethods[] = {
    {"system", spam_system, METH_VARARGS, "Execute a shell command."},
    {NULL, NULL, 0, NULL} //Sentinel
};

PyMODINIT_FUNC initspammodule(void)
{
    (void)Py_InitModule("spammodule", SpamMethods);
}
```

可以看到真正使用Python API的部分在这里，到这一步这一个示例也就完成了。我们重点分析下这个c文件的各个部分。  

#### 2.2. Python.h

Python API定义了一系列的函数、宏以及变量，所有这些都被包裹进了**Python.h**这个文件，因此只要在你的c文件最开始include这个文件就可以了。因为**Python.h**含有一些预处理定义，因此最好在所有非标准头文件导入之前导入。  

#### 2.3. 导出函数

要在Python中使用C的某个函数，首先需要为其编写对应的导出函数。上述例子中的`spam_system`就是对应的导出函数。  
所有的导出函数形式为下面3种：  

```c
# 普通参数
static PyObject *MyFunction( PyObject *self, PyObject *args );
# 关键字参数
static PyObject *MyFunctionWithKeywords(PyObject *self,
                                 PyObject *args,
                                 PyObject *kw);
# 无参数
static PyObject *MyFunctionWithNoArgs( PyObject *self );
```

例子中的导出函数属于第一种，返回值固定是__PyObject__，第一个参数为self，通常为NULL（感兴趣的同学可以看下不为NULL的情况给我留言），第二个参数则是python传过来的参数。

#### 2.4. 解析参数

有了导出函数，我们还需要解析出当前传过来的参数是什么。对应不同类型的导出函数，解析参数也有两种：  

```
int PyArg_ParseTuple(PyObject *args, const char *format, ...)
int PyArg_ParseTupleAndKeywords(PyObject *args, PyObject *kw, const char *format, char *keywords[], ...)
```

其中第一个参数就python传过来的args，format用于指定如何读取这些参数，例如i表示integer, f表示float，例如`PyArg_ParseTuple(args, "ii", &a, &b)`表示从args里读取两个integer类型，分别存储到变量a b，这两个变量有了内容后我们就可以在c里使用了。  

#### 2.5. 返回结果  

这个跟解析参数是相对的，在c里的计算完成后，需要返回结果到python。  
用到的是Py\_BuildValue这个API  

```
PyObject* Py_BuildValue(const char *format, ...)
```

例如`Py_BuildValue("ii", a + b, a - b)`对于python解释器来讲，就是产生了`tuple(a + b , a - b)`这么一个元组，内容为两个整型。  

注意没有任何返回值的话，需要返回一个`Py_None`，内容如下：  

```
Py_INCREF(Py_None);
return Py_None;
```

#### 2.6. 方法列表

定义了上述的方法，还需要给出Python解释器中使用的方法，即例子中的代码：  

```
static PyMethodDef SpamMethods[] = {
    {"system", spam_system, METH_VARARGS, "Execute a shell command."},
    {NULL, NULL, 0, NULL} //Sentinel
};
```

列表里的每项有四部分组成：__方法名__, __导出函数__,__参数传递方式__,__方法描述__。法名是从Python解释器中调用该方法时所使用的名字。参数传递方式则规定了Python向C函数传递参数的具体形式，可选的两种方式是METH\_VARARGS和METH\_KEYWORDS，其中METH\_VARARGS是参数传递的标准形式，它通过Python的元组在Python解释器和C函数之间传递参数，若采用METH\_KEYWORD方式，则Python解释器和C函数之间将通过Python的字典类型在两者之间进行参数传递。方法描述则对应了python里的doc描述。  

此表需要适当的成员终止与定点NULL的组成和0值

#### 2.7. 初始化函数

扩展模块需要一个初始化函数，这也是最后一部分。需要的功能被命名为initModule，其中module是模块的名称。  

用到的函数原型为：  

```
PyObject* Py_InitModule(char *name, PyMethodDef *methods)
PyObject* Py_InitModule3(char *name, PyMethodDef *methods, char *doc)
```

其中name为模块名，methods为上面定义的列表，doc为模块文档。   

注意这也是一个唯一一个非static的函数  

#### 2.8. 为什么其他函数要定义为static

主要是为了防止名字污染，因为只需要python解释器读懂就可以了，对其他方式不可见。  
stackoverflow有一个比较简短的[答案](http://stackoverflow.com/questions/28493847/in-python-c-api-why-is-wrapper-function-static)


关于Python API扩展有很多内容，比如异常、引用计数等就不一一列举了，有兴趣的可以直接查看python[官方文档](https://docs.python.org/2/extending/extending.html)  

最后，再贴一个参考资料的小例子：  

```
// example.c
int fact(int n)
{
    if (n <= 1)
        return 1;
    else
        return n * fact(n - 1);
}
```

```
//wrapper.c
#include <Python.h>

PyObject* wrap_fact(PyObject* self, PyObject* args)
{
    int n, result;

    if (!PyArg_ParseTuple(args, "i", &n))
        return NULL;

    result = fact(n);
    printf("self = %x, args = %x\n", self, args);
    return Py_BuildValue("i", result);
}

static PyMethodDef exampleMethods[] =
{
    {"fact", wrap_fact, METH_VARARGS, "Caculate N!"},
    {NULL, NULL}
};

void initexample()
{
    PyObject* m = Py_InitModule("example", exampleMethods);
}
```

```
# makefile
# PYINC为Python.h所在目录
        gcc -fPIC -c -I$(PYINC) example.c wrapper.c
        gcc -shared -o example.so example.o wrapper.o
```

### 3. 参考资料：

1. [Extending Python with C or C++](https://docs.python.org/2/extending/extending.html)  
2. [ctypes — A foreign function library for Python](https://docs.python.org/2/library/ctypes.html)  
3. [用C语言扩展Python的功能](http://www.ibm.com/developerworks/cn/linux/l-pythc/)  

