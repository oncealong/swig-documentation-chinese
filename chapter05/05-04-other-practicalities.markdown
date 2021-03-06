# 5.4 其他实用性建议

到目前为止，本章已经介绍了包装简单接口所需要学习的几乎所有的知识。但是，有些C程序使用了一些难以映射到脚本语言接口的习惯用法。本节将描述其中的一些问题及解决方案。

## 5.4.1 通过结构体传递值

有时C函数接受由值传递的结构参数。例如，考虑以下函数：

```c
double dot_product(Vector a, Vector b);
```

为处理这个，SWIG创建一个包装函数，将原始函数转换成使用指针的方式：

```c++
double wrap_dot_product(Vector *a, Vector *b) {
	Vector x = *a;
 	Vector y = *b;
  
	return dot_product(x,y);
}
```

在目标语言中，`dot_product()`函数接受指向`Vector`的指针而不是`Vector`。大部分情况下，这种转换是透明地，不需要去关心。

## 5.4.2 返回的是值类型

返回结构体或类数据类型值的C语言函数更难处理。考虑如下函数：

```c++
Vector cross_product(Vector v1, Vector v2);
```

这个函数像返回`Vector`，但是SWIG只支持指针。结果，SWIG会创建这样的包装代码:

```c++
Vector *wrap_cross_product(Vector *v1, Vector *v2) {
  Vector x = *v1;
  Vector y = *v2;
  Vector *result;
  result = (Vector *) malloc(sizeof(Vector));
  *(result) = cross(x,y);
  
  return result;
}
```

如果使用了-C++选项：

```c++
Vector *wrap_cross(Vector *v1, Vector *v2) {
  Vector x = *v1;
  Vector y = *v2;
  Vector *result = new Vector(cross(x,y)); // Uses default copy constructor
  
  return result;
}
```

两种情况下，SWIG都会分配新的对象，并会返回它的引用。当不再使用时，需要用户删除它们。很显然，如果你没有意识到这个隐式的内存分配的话，就不会释放对象，从而带来内存泄露的问题。应该注意到一些语言模块现在可以自动跟踪新创建的对象，并为您恢复内存。请查阅每个语言模块的文档以获得更多的详细信息。

还应该注意的是，用C++处理值的传递/返回还有一些特殊情况。例如，如果`Vector`没有定义默认构造函数，上述代码片段就不能正常工作。SWIG与C++章节有关于这个情况的更多信息。

## 5.4.3 链接结构体变量

当全局变量或类的成员变量包含结构体时，SWIG将它们处理成指针。例如，像下面这样的全局变量:

```c++
Vector unit_i;
```

映射成底层的一对set/get函数：

```c++
Vector *unit_i_get() {
	return &unit_i;
}

void unit_i_set(Vector *value) {
	unit_i = *value;
}
```

以这种方式创建的全局变量将在目标语言中视为指针。释放这样的指针是不对的。同样，C++类必须提供合适的拷贝构造函数让赋值得以正常工作。

## 5.4.4 链接到`char*`

当遇到`char*`类型的全局变量时，SWIG使用`malloc()`或`new`函数给新值分配内存。特别是当你有如下的定义:

```c
char *foo;
```

SWIG将生成如下代码：

```c++
/* C mode */
void foo_set(char *value) {
  if (foo) free(foo);
  foo = (char *) malloc(strlen(value)+1);
  strcpy(foo,value);
}
/* C++ mode. When -c++ option is used */
void foo_set(char *value) {
  if (foo) delete [] foo;
  foo = new char[strlen(value)+1];
  strcpy(foo,value);
}
```

如果这不是你想要的行为，请考虑使用`%immutable`指令将其变为只读的。或者，您可以编写一个简短的帮助函数来设置您想要的值。例如：

```c
%inline %{
void set_foo(char *value) {
	strncpy(foo,value, 50);
}
%}
```

> 注意：如果你写了像这样一个帮助函数，你需要在目标语言中调用它(表现的不像是一个变量)。例如，在Python中，你可以这样写：
>
> ```python
> >>> set_foo("Hello World")
> ```

`char *`变量的一个比较常见的错误是像这样:

```c
char *VERSION = "1.0";
```

这种情况下，变量时可读的，但是尝试更改它的值将导致段错误或通用保护异常。这是因为，SWIG通过`free`或`delete`释放旧值，但当前字符串字面量又不是通过`malloc()`或`new`申请的。为修正这样的行为，你可以标记这个变量为只读的、自定义一个typemap(第六章介绍)、或者写一个特殊的函数。还可以将它声明为一个数组:

```c
char VERSION[64] = "1.0";
```

当声明`const char *`时，SWIG依然为其生成get/set函数。但是，默认的行为不是释放先前的内容（结果可能导致内存泄露）。事实上，像这样包装代码的话你会得到警告消息：

```shell
example.i:20. Typemap warning. Setting const char * variable may leak memory
```

之所以会这样是因为`const char *`变量一般用来指向字符串字面量。例如:

```c
const char *foo = "Hello World\n";
```

因此，在这样的指针上释放指针是个坏主意。另一方面，改变指针，使其指向其他值是合法的。当设置这种类型的变量时，SWIG将会分配新的字符串(通过`malloc()`或`new`)，改变指针指向新的值。但是，重复修改值将会导致内存泄露，因为旧的值没有被释放。



## 5.4.5 数组

SWIG全面支持数组，但是它们总是被处理成指针，而不是将其映射为特殊的数组对象或目标语言的列表类型。因此，如下的声明:

```c
int foobar(int a[40]);
void grok(char *argv[]);
void transpose(double a[20][20]);
```

处理后像这样：

```c
int foobar(int *a);
void grok(char **argv);
void transpose(double (*a)[20]);
```

像C一样，SWIG不执行数组边界检查。用户需要确保指针指向合适的内存位置。

多维数组将被转换成降了一维的数组的指针。例如：

```c
int [10]; 			 	// Maps to int *
int [10][20]; 		 	// Maps to int (*)[20]
int [10][20][30]; 	 	// Maps to int (*)[20][30]
```

在C语言的类型系统中，多维数组`a[][]`与单指针`*a`或双指针`char**`**不是等价的**，注意到这个非常重要！指向数组的指针的实际值是数组起始位置的内存位置。强烈建议读者弹去C语言数据上的灰尘，重读关于数组的章节，再在SWIG中使用它们。

SWIG支持数组变量，但默认情况下是只读的。例如：

```c
int a[100][200];
```

这种情况下，读取变量`a`将放回类型`int (*)[200]`,它指向数组的第一个元素的地址`&a[0][0]`。修改`a`会导致错误。这是因为SWIG不知道如何从目标语言中拷贝数据到数组中。为解决这样的限制，你可能需要写像下面这样的帮助函数:

```c
%inline %{
void a_set(int i, int j, int val) {
	a[i][j] = val;
}
int a_get(int i, int j) {
	return a[i][j];
}
%}
```

为可变的大小和形态的数组创建动态的绑定，可在接口文件中像下面这样编写帮助函数：

```c
// Some array helpers
%inline %{
/* Create any sort of [size] array */
int *int_array(int size) {
	return (int *) malloc(size*sizeof(int));
}

/* Create a two-dimension array [size][10] */
int (*int_array_10(int size))[10] {
	return (int (*)[10]) malloc(size*10*sizeof(int));
}
%}
```

SWIG对`char`类型的数组做了特殊处理。这种情况下，目标语言的字符串可以存储到数组中。例如，有如下的声明:

```c
char pathname[256];
```

SWIG将生成set/get函数:

```c
char *pathname_get() {
	return pathname;
}
void pathname_set(char *value) {
	strncpy(pathname,value,256);
}
```

在目标语言中使用它就像访问普通的变量。

## 5.4.6 创建只读变量

使用`%immutable`指令可以创建只读变量:

```c
// File : interface.i
int a; // Can read/write
%immutable;
int b,c,d // Read only variables
%mutable;
double x,y // read/write
```

`%immutable`指令开启只读模式，直到再使用`%mutable`指令禁止。还有一种方式是单独为每个声明指定只读类型。例如：

```c
%immutable x; // Make x read-only
...
double x; // Read-only (from earlier %immutable directive)
double y; // Read-write
...
```

`%immutable`和`%mutable`指令实际上是用[`%feature`指令](#swig-feature-dicrectives)定义的：

```c++
#define %immutable %feature("immutable")
#define %mutable %feature("immutable","")
```

如果你想让所有的变量都变成只读的，只有一两个是例外的话，可以这样做:

```c
%immutable; // Make all variables read-only
%feature("immutable","0") x; // except, make x read/write
...
double x;
double y;
double z;
```

当声明变量为const类型的时候，也会创建只读变量。例如：

```c
const int foo; 					/* Read only variable */
char * const version="1.0"; 	 /* Read only variable */
```

> 兼容性注释：只读访问以前使用一对指令`%readonly`和`%readwrite`来控制。尽管这些质量依然可以工作，但会生成警告消息。可以简单的将它们替换成`%immutable`和`%mutable`指令，就可以关闭警告。不要忘记了额外的分号！

## 5.4.7 重命名与忽略声明

### 5.4.7.1 特殊标识符的简单重命名

一般情况下，经过包装后，目标语言直接使用C语言中声明的名字。但是，有时候这可能与脚本语言的关键字或函数冲突。为解决命名冲突，你可以像下面这样使用`%rename`指令:

```c
// interface.i
%rename(my_print) print;
extern void print(const char *);
%rename(foo) a_really_long_and_annoying_name;
extern int a_really_long_and_annoying_name;
```

SWIG依然会调用正确的C函数，但是这种情况下，函数`print()`在目标语言中应该使用`my_print()`进行调用。

`%rename`指令可以放在任意的位置，只要它出现在要重命名的声明之前。通用方式是像下面这样编写接口代码:

```c
// interface.i
%rename(my_print) print;
%rename(foo) a_really_long_and_annoying_name;
%include "header.h"
```

`%rename`指令将其后出现的关联名字全部重新命名。它可以应用到函数、变量、类和结构体的名字、成员函数、数据成员。例如，如果有很多C++类，都有有一个函数为`print`(在Python中时关键字)，你可以将它们统一命名为`output`：

```c
%rename(output) print; // Rename all `print' functions to `output'
```

SWIG一般不会检查，看它包装的函数是否在目标语言中定义了没。但是，如果你小心处理命名空间和模块的名字，一般情况下都可以避免这些问题。

与`%rename`指定紧密相关的指令是`%ignore`指令。`%ignore`指示SWIG忽略指定标识符的声明。例如：

```c
%ignore print; // Ignore all declarations named print
%ignore MYMACRO; // Ignore a macro
...
#define MYMACRO 123
void print(const char *);
...
```

匹配`%ignore`的任何函数、变量等都不会被包装，因此不能从目标语言中访问。一般使用`%ignore`指令是为了从声明中去掉某些声明，而不用非得在头文件中添加条件编译指令。但是，需要强调的是，这支队简单的声明有效。

如果你想移除一整段有问题的代码，应该SWIG预处理器。

> 兼容性注释：旧版本的SWIG提供了特殊的`%name`指令，用于重命名。例如：
>
> ```c
> %name(output) extern void print(const char *);
> ```
>
> 这个指令依然支持，但是过期了，应尽可能避免使用。`%rename`指令更强大，对原始头文件信息的支持更好。

### 5.4.7.2 高级重命名支持

为特定的声明编写`%rename`很简单，有时候同样的重命名规则需要应用到很多，有时是全部的SWIG输入的表示符上。例如，将所有的名字根据目标语言命名规范都做统一的命名也是非常必要的，比如在所有的包装函数前加一个统一的前缀。给每个被包装的函数重命名不实际，因此只要标示符的名字不指定的话，SWIG支持将重命名规则应用到所有声明：

```c
%rename("myprefix_%s") ""; // print -> myprefix_print
```

这也表明，`%rename`的参数可以不必是一个字符串，也可以是一个printf()一样的格式字符串。在最简单的格式中，`%s`用原始声明的名字替换。但这还不够，SWIG扩展了通常的格式字符串的语法，允许对参数应用函数(SWIG定义的)。例如，为了将所有的C函数`do_something_long()`包装的更像Java中的`doSomethingLong()`，使用`lowercamelcase`格式像下面这样表示：

```c
%rename("%(lowercamelcase)s") ""; // foo_bar -> fooBar; FooBar -> fooBar
```

一些函数课题提供参数，如_"strip"_可用来从提供的形参中剔除指定的前缀。前缀在格式字符串中指定，后面跟一个冒号:

```c
%rename("%(strip:[wx])s") ""; // wxHello -> Hello; FooBar -> FooBar
```

下面的表对目前定义的所有函数做了概要介绍，并对每个函数提供了示例。注意，它们中的部分有两个名字，一个简单点，一个更具描述性，但这两个函数是等价的:

| **函数**                    | **返回值**                             | **示例（in/out）**        |
| :------------------------ | :---------------------------------- | --------------------- |
| uppercase or upper        | 字符串大写                               | Print -> PRINT        |
| lowercase or lower        | 字符串小写                               | Print -> print        |
| title                     | 第一个字符大写，其他全部小写                      | print -> Print        |
| firstuppercase            | 第一个字符大写，其他不变                        | printIt -> PrintIt    |
| firstlowercase            | 第一个字符小写，其他不变                        | PrintIt -> printIt    |
| camelcase or ctitle       | 第一个字符及其后跟在下划线的后面的字符大写，剩下的字符小写，下划线删除 | print_it -> PrintIt   |
| lowercamelcase or lctitle | 跟在下划线后面的字符大写，剩下的包括第一个字符小写，下划线删除     | print_it -> printIt   |
| undercase or utitle       |                                     | PrintIt -> print_it   |
| schemify                  |                                     | print_it -> print-it  |
| strip:[prefix]            |                                     | wxPrint  ->Print      |
| regex:/pattern/subst/     |                                     | prefix_print -> Print |
| command:cmd               |                                     | Print -> Prnt         |

上面这些函数中最通用的是`regex`（不算上`command`，实践中它也很强大，如果考虑性能就不要使用）。这里有使用它的更多例子:

```c
// Strip the wx prefix from all identifiers except those starting with wxEVT
%rename("%(regex:/wx(?!EVT)(.*)/\\1/)s") ""; // wxSomeWidget -> SomeWidget
// wxEVT_PAINT -> wxEVT_PAINT
// Apply a rule for renaming the enum elements to avoid the common prefixes
// which are redundant in C#/Java
%rename("%(regex:/^([A-Z][a-z]+)+_(.*)/\\2/)s", %$isenumitem) ""; // Colour_Red -> Red
// Remove all "Set/Get" prefixes.
%rename("%(regex:/^(Set|Get)(.*)/\\2/)s") ""; // SetValue -> Value
// GetValue -> Value
```

像之前说的一样，所有关于`%rename`的讨论都适用于`%ignore`。事实上，事实上，后者只是前者的一个特例，忽略标识符与将其重命名为特殊的“$ignore”值一样。下面的代码片段:

```c
%ignore print;
```

和：

```c
%rename("$ignore") print;
```

是一样的，并且使用先前描述的可能匹配，`%rename`指令可以用于选择性忽略多个声明。

### 5.4.7.3 限制全局重命名规则

如前几节所解释的那样，可以重命名单个声明或将重命名规则立即应用于所有声明。然而，在实践中，后者通常是不适当的，因为一般规则都有一些例外。为了处理它们，可以使用后续的匹配参数限制未命名的`%rename`指令的范围。通过SWIG，它们可以应用到输入接口文件中声明的相关属性。例如：

```c
%rename("foo", match$name="bar") "";
```

与下面的方式是一样的:

```c
%rename("foo") bar;
```

虽然这种方式没什么意思，但`match`也能应用于声明的类型，例如：`match="class"`限制匹配只应用于类声明，`match="enumitem"`限制只应用于枚举类型。SWIG还提供了这些匹配表达式的便于使用的宏，如：

```c
%rename("%(title)s", %$isenumitem) "";
```

会将所有的枚举元素首字母大写，但不改变其他声明的。类似地，还可以使用`%$isclass`,
`%$isfunction`，`%$isconstructor`，`%$isunion`，`%$istemplate`,及 `%$isvariable`。其他的检查也是可能的，本文档不是太全，查看swig.swg文件中的“%rename predicates”部分可得到支持的匹配表达式更全的列表。

除了是使用`match`匹配字符串字面量，还可以使用`regexmatch`或`notregexmatch`去做正则表达式方式的匹配。例如，为了忽略所有以"Old"结尾的函数，可以这么做：

```c
%rename("$ignore", regexmatch$name="Old$") "";
```

对于这样的简单情况，直接指定声明名的正则表达式可以更好，可以使用`regextarget`：

```c
%rename("$ignore", regextarget=1) "Old$";
```

需要注意的是，检查只针对声明名字本省，如果你想匹配C++声明的全称，就必须指定`fullname`属性:

```c
%rename("$ignore", regextarget=1, fullname=1) "NameSpace::ClassName::.*Old$";
```

对于`notregexmatch`，它只限制那些不匹配指定正则表达式的匹配。因此，除了连续大写字母的字符创以外，所有的声明都要重命名为小写：

```c
%rename("$(lower)s", notregexmatch$name="^[A-Z]+$") "";
```

最后，`%rename`和`%ignore`的变体可用来包装C++重载函数和方法，或者使用默认参数的C++方法。C++章节的[歧义解析与重命名](#swig-ambiguity-resolution-and-renaming)部分对此有描述。

### 5.4.7.4 忽略一切后再包装几个指定的符号

使用上面描述的技术可以用来忽略头文件中的所有声明，然后再选择部分方法和类进行包装。例如，考虑头文件`myheader.h`，其中包含了很多类，只像包装一个叫`Star`的类，可以做么做：

```c
%ignore ""; // Ignore everything
// Unignore chosen class 'Star'
%rename("%s") Star;
// As the ignore everything will include the constructor, destructor, methods etc
// in the class, these have to be explicitly unignored too:
%rename("%s") Star::Star;
%rename("%s") Star::~Star;
%rename("%s") Star::shine; // named method
%include "myheader.h"
```

另外一个方法可能更合适这种情况，它不需要命名所选择类所有方法，一开始只忽略所有类。这种方法不会显式忽略任何类的方法，所以当选择的类不被忽略时，它的所有方法都会被包装：

```c
%rename($ignore, %$isclass) "";  // Only ignore all classes
%rename("%s") Star; 			// Unignore 'Star'
%include "myheader.h"
```



## 5.4.8 默认/可选参数

SWIG支持C/C++代码中的默认参数。如：

```c 
int plot(double x, double y, int color=WHITE);
```

这种情况下，SWIG会生成包装代码，而默认参数在目标语言中时可选的。如，在Tcl中可以这么调用：

```tcl
% plot -3.4 7.5 # Use default value
% plot -3.4 7.5 10 # set color to 10 instead
```

尽管ANSI C标准不允许默认参数，但在SWIG接口文件中指定C/C++的默认参数都是可以的。

> 注意：对于SWIG使用默认参数生成包装代码，这里有一个小的语义问题。当在C代码中使用默认参数时，默认值被释放到包装器中，函数用一组完整的参数调用。当包装C++生成带默认参数的重载函数时有一些不同。请参考C++章节的[默认参数](#swig-default-arguments)了解更多信息。



##5.4.9 指向函数的指针与回调

偶尔的情况下，C库中的函数参数需要接收函数指针，可能用于当做回调函数使用。SWIG全面支持函数指针，回调函数定义在C而不是目标语言中。如，考虑像这样一个函数：

```c
int binary_op(int a, int b, int (*op)(int,int));
```

当你第一次包装像这样的代码到扩展库时，你会发现这个板书不能使用。比如，在Python中：

```python
>>> def add(x,y):
... return x+y
...
>>> binary_op(3,4,add)
Traceback (most recent call last):
  File "<stdin>", line 1, in ?
TypeError: Type error. Expected _p_f_int_int__int
>>>
```

这个错误的原因是，SWIG不知道如何将脚本语言的函数映射到C中的回调函数。但是，已经存在的C函数是可以作为参数使用的（已作为常量指定了）。使用`%constant`指令可以达此目的：

```c
/* Function with a callback */
int binary_op(int a, int b, int (*op)(int,int));
/* Some callback functions */
%constant int add(int,int);
%constant int sub(int,int);
%constant int mul(int,int);
```

 这种情况下，`add`，`sub`和`mul`都在目标脚本语言中都变成了函数指针常量。可以允许你像下面这样使用：

```python
>>> binary_op(3,4,add)
7
>>> binary_op(3,4,mul)
12
>>>
```

不幸的是，将回调函数声明为常量后就不能像常规函数使用它们了。如：

```python
>>> add(3,4)
Traceback (most recent call last):
File "<stdin>", line 1, in ?
TypeError: object is not callable: '_ff020efc_p_f_int_int__int'
>>>
```

如果你想让一个函数同时作为常量和函数使用，可以使用`%callback`和`%nocallback`指令:

```c
/* Function with a callback */
int binary_op(int a, int b, int (*op)(int,int));
/* Some callback functions */
%callback("%s_cb");
int add(int,int);
int sub(int,int);
int mul(int,int);
%nocallback;
```

`callback`指令的参数也是一个printf样式的格式化字符串，可指定对调常量的命名规则(%s用函数名替换)。回调模式在用`%nocallback`指令禁止前会一直有效。当这样做后，接口像下面这样工作：

```python
>>> binary_op(3,4,add_cb)
7
>>> binary_op(3,4,mul_cb)
12
>>> add(3,4)
7
>>> mul(3,4)
12
```

>注意：当函数作为回调使用时，使用了特定的`add_cb`代替。当做常规函数调用时，使用其原始函数名`add`。

SWIG提供了大量标准C printf格式扩展可用在这种情况下。例如，下面的变体将回调变为全部大写的形式:

```c
/* Some callback functions */
%callback("%(uppercase)s");
int add(int,int);
int sub(int,int);
int mul(int,int);
%nocallback;
```

`%(lowercase)s`格式化字符串将所有字符转换成小写的。`%(title)s`将首字母大写，剩下的小写。

现在，介绍关于函数指针支持的最后一点。尽管SWIG一般不允许在目标语言中写回调，但可以使用typemap和其他SWIG高级特性达此目的。查看[Typemaps](#swig-typemaps)章节获取关于typemap的信息，查看目标语言各自章节了解关于回调和`director`的更多信息。