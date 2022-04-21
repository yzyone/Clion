# 超级炫酷的C语言技巧！



C语言常常让人觉得它所能表达的东西非常有限。它不具有类似第一级函数和模式匹配这样的高级功能。但是C非常简单，并且仍然有一些非常有用的语法技巧和功能，只是没有多少人知道罢了。

**一、指定的初始化**

很多人都知道像这样来静态地初始化数组：

```
int fibs[] = {1, 1, 2, 3, 5};
```

C99标准实际上支持一种更为直观简单的方式来初始化各种不同的集合类数据（如：结构体，联合体和数组）。

**二、数组**

我们可以指定数组的元素来进行初始化。这非常有用，特别是当我们需要根据一组#define来保持某种映射关系的同步更新时。来看看一组错误码的定义，如：

```
/* Entries may not correspond to actual numbers. Some entries omitted. */
#define EINVAL 1
#define ENOMEM 2
#define EFAULT 3
/* ... */
#define E2BIG 7
#define EBUSY 8
/* ... */
#define ECHILD 12
/* ... */
```

现在，假设我们想为每个错误码提供一个错误描述的字符串。为了确保数组保持了最新的定义，无论头文件做了任何修改或增补，我们都可以用这个数组指定的语法。  

```
char *err_strings[] = {
[0] = "Success",
[EINVAL] = "Invalid argument",
[ENOMEM] = "Not enough memory",
[EFAULT] = "Bad address",
/* ... */
[E2BIG ] = "Argument list too long",
[EBUSY ] = "Device or resource busy",
/* ... */
[ECHILD] = "No child processes"
/* ... */
};
```

这样就可以静态分配足够的空间，且保证最大的索引是合法的，同时将特殊的索引初始化为指定的值，并将剩下的索引初始化为0。

**三、结构体与联合体**

用结构体与联合体的字段名称来初始化数据是非常有用的。假设我们定义：

```
struct point {
int x;
int y;
int z;
}
```

然后我们这样初始化struct point：

```
struct point p = {.x = 3, .y = 4, .z = 5};
```

当我们不想将所有字段都初始化为0时，这种作法可以很容易的在编译时就生成结构体，而不需要专门调用一个初始化函数。  

对联合体来说，我们可以使用相同的办法，只是我们只用初始化一个字段。

**四、宏列表**

C中的一个惯用方法，是说有一个已命名的实体列表，需要为它们中的每一个建立函数，将它们中的每一个初始化，并在不同的代码模块中扩展它们的名字。这在Mozilla的源码中经常用到，我就是在那时学到这个技巧的。例如，在我去年夏天工作的那个项目中，我们有一个针对每个命令进行标记的宏列表。其工 作方式如下：

```
#define FLAG_LIST(_) \
_(InWorklist) \
_(EmittedAtUses) \
_(LoopInvariant) \
_(Commutative) \
_(Movable) \
_(Lowered) \
_(Guard)
```

它定义了一个FLAG_LIST宏，这个宏有一个参数称之为 _ ，这个参数本身是一个宏，它能够调用列表中的每个参数。举一个实际使用的例子可能更能直观地说明问题。假设我们定义了一个宏DEFINE_FLAG，如：

```
#define DEFINE_FLAG(flag) flag,
enum Flag {
None = 0,
FLAG_LIST(DEFINE_FLAG)
Total
};
#undef DEFINE_FLAG
```

对FLAG_LIST(DEFINE_FLAG)做扩展能够得到如下代码：

```
enum Flag {
None = 0,
DEFINE_FLAG(InWorklist)
DEFINE_FLAG(EmittedAtUses)
DEFINE_FLAG(LoopInvariant)
DEFINE_FLAG(Commutative)
DEFINE_FLAG(Movable)
DEFINE_FLAG(Lowered)
DEFINE_FLAG(Guard)
Total
};
```

接着，对每个参数都扩展DEFINE_FLAG宏，这样我们就得到了enum如下：

```
enum Flag {
None = 0,
InWorklist,
EmittedAtUses,
LoopInvariant,
Commutative,
Movable,
Lowered,
Guard,
Total
};
```

接着，我们可能要定义一些访问函数，这样才能更好的使用flag列表：

```
#define FLAG_ACCESSOR(flag) \
bool is##flag() const {\
return hasFlags(1 << flag);\
}\
void set##flag() {\
JS_ASSERT(!hasFlags(1 << flag));\
setFlags(1 << flag);\
}\
void setNot##flag() {\
JS_ASSERT(hasFlags(1 << flag));\
removeFlags(1 << flag);\
}
FLAG_LIST(FLAG_ACCESSOR)
#undef FLAG_ACCESSOR
```

一步步的展示其过程是非常有启发性的，如果对它的使用还有不解，可以花一些时间在gcc –E上。

**五、编译时断言**

这其实是使用C语言的宏来实现的非常有“创意”的一个功能。有些时候，特别是在进行内核编程时，在编译时就能够进行条件检查的断言，而不是在运行时进行，这非常有用。不幸的是，C99标准还不支持任何编译时的断言。

但是，我们可以利用预处理来生成代码，这些代码只有在某些条件成立时才会通过编译（最好是那种不做实际功能的命令）。有各种各样不同的方式都可以做到这一点，通常都是建立一个大小为负的数组或结构体。最常用的方式如下：

```
/* Force a compilation error if condition is false, but also produce a result
* (of value 0 and type size_t), so it can be used e.g. in a structure
* initializer (or wherever else comma expressions aren't permitted). */
/* Linux calls these BUILD_BUG_ON_ZERO/_NULL, which is rather misleading. */
#define STATIC_ZERO_ASSERT(condition) (sizeof(struct { int:-!(condition); }) )
#define STATIC_NULL_ASSERT(condition) ((void *)STATIC_ZERO_ASSERT(condition) )
/* Force a compilation error if condition is false */
#define STATIC_ASSERT(condition) ((void)STATIC_ZERO_ASSERT(condition))
```

如果(condition)计算结果为一个非零值（即C中的真值），即! (condition)为零值，那么代码将能顺利地编译，并生成一个大小为零的结构体。如果(condition)结果为0（在C真为假），那么在试图生成一个负大小的结构体时，就会产生编译错误。

它的使用非常简单，如果任何某假设条件能够静态地检查，那么它就可以在编译时断言。例如，在上面提到的标志列表中，标志集合的类型为uint32_t，所以，我们可以做以下断言：

```
STATIC_ASSERT(Total <= 32)
```

它扩展为：

```
(void)sizeof(struct { int:-!(Total <= 32) })
```

现在，假设Total<=32。那么-!(Total <= 32)等于0，所以这行代码相当于：

```
(void)sizeof(struct { int: 0 })
```

这是一个合法的C代码。现在假设标志不止32个，那么-!(Total <= 32)等于-1，所以这时代码就相当于：

```
(void)sizeof(struct { int: -1 } )
```

因为位宽为负，所以可以确定，如果标志的数量超过了我们指派的空间，那么编译将会失败。
