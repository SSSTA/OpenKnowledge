C-LOLI
======

-------------------------

> **C-LOLI**, <b>C-L</b>ightweight <b>O</b>bject-<b>L</b>ike <b>I</b>nterface

> 本文尝试讨论如何在`C`中实现一个原始而简陋的伪对象系统。

> 这个伪对象系统原定名为**T56L**，因为我就是在晚点到心碎的**T56**次列车上想出要实现这个东西的。

> 考虑到这个名字不能有效的激起众基佬的阅读兴趣，遂大幅简化设计，并将精简版改名**C-LOLI**。

-------------------------

# Hello, Loli #

有人曾经问起，如何在C下实现一个采用消息传递模型的对象系统。实现一个完整的对象系统是一件十分困难的事情，不过……我想我可以从一个比较简单的例子着手试试。

## 设计 ##

Google开发的Go语言是一个很有意思的语言。比如下面这个例子：

```Go
package main

import "fmt"

type Color struct {
    R, G, B uint8
}

func (c Color) gray_level() uint8 {
    return uint8((uint16(c.R) + uint16(c.G) + uint16(c.B)) / 3)
}

func main() {
    c := Color{0x66, 0xCC, 0xFF}
    fmt.Println(c, c.gray_level())
    // will print {102 204 255} 187
}
```

除了略显奇怪的语法，还有一个很有意思的地方：`gray_level`函数被绑定到了`Color`这个结构上。这就是利用Go实现无类OOP的一个简单的例子。

那么能不能在C里作出类似的事情呢？自然是可以的。`LOLI`的核心目的就是实现一个**建立在结构体之上的、方法隶属于类型的伪对象系统**。

简但来说，就是利用**结构体**保存实例中的属性来实现对象`LoliObject`，利用**结构体**保存类型信息和继承信息来实现类`LoliType`，利用**结构体**来记录函数指针和隶属类型来实现类方法`LoliMethod`。看上去很简单对吧？

> 称之为`LoliType`而不是`LoliClass`，是因为`LOLI`只是利用一系列的`typedef`来模拟了一个伪对象系统。

## 从零开始 ##

如何做出一个伪对象系统呢？首先，我们需要一个最基础的结构，然后设法通过一个附加的系统来封装、操纵这个结构，并以此模拟“类”与“对象”的概念。C语言中的结构体可能是个不错的选择：

```C
typedef struct {
    int R, G, B;
    char *desc;
} RGBColor;
```

但是这个选择会有一个问题：结构体在编译后不会留下名字，只有指针与偏移量。这就意味着我们很难由此构造一个支持自省特性的为对象系统。但是，就`LOLI`的设计目标而言，结构体已经足够了。

这里的核心思路是，在C结构体的头部拓展出一个公共的头部用以标记类型，并以这样的新结构体作为`LoliObject`的主体。

首先用一个宏来从结构体名中扩展`LoliType`的名字。

```C
#define $cls(structure) _CLS_ ## structure
```

`$`是C语言中构成符号的合法字符之一，`LOLI`中所有的宏的名字都以`$`开始以示区分。然后编写另外一个用于定义新结构体的宏：

```C
#define $mkcls(structure) \
const LoliType $typeinfo(structure) = {#structure, &LOLIMETA}; \
typedef struct {                    \
    const LoliType * const parent;  \ // 标记类型
    structure              data;    \ // 保存数据
} $cls(structure)                   \
```

这样每次从结构体静态衍生一个类，就是静态地建立了对应的结构和一个对应的`LoliType`。`LoliType`用作`LoliType`所有`LoliObject`首部的公共数据段：

```C
typedef struct _LOLI_TYPE LoliType;
struct _LOLI_TYPE {
    const char * const name;        // 记录名字
    const LoliType * const parent;  // 记录父类
};
```

其中`LOLIMETA`是定义的基类。

```C
const LoliType LOLIMETA = {"LOLIMETA", NULL};
```

“构造方法”这样的东西对一个如此简陋的系统还是太奢侈了，现在我们直接使用字面值构建`LoliObject`。

```C
#define $imm(structure, literal...) \
(($cls(structure)) { &$typeinfo(structure), ((structure) literal) })
```

为了编译这样的语句，你可能需要开启编译器的`C99`支持。此宏之于C就像立即数之于汇编，这也就是这个宏为什么被命名为`$imm`。


让我们继续写几个宏：

```C
// 构造方法名
#define $mname(structure, name) _M_ ## structure ## _ ## name

// 定义方法
#define $defmethod(ret_t, structure, name, ...) \
    ret_t $mname(structure, name) (structure *self, ## __VA_ARGS__)

// 调用方法
#define $(structure, name, instance, ...) \
    $mname(structure, name)(&instance.data, ## __VA_ARGS__)
```

好了，到现在我们已经写了将近……呃……20行代码了。在继续之前，先初步地测试一下吧。

```C
$mkcls(RGBColor);

$defmethod(int, RGBColor, gray_level) {
    return (self->R + self->G + self->B)/3;
}

$defmethod(void, RGBColor, print) {
    printf("%s: %02X%02X%02X\n", self->desc,
            self->R, self->G, self->B);
}
    
int main() {
    $cls(RGBColor) c = $imm(RGBColor, {0x66, 0xCC, 0xFF, "天依蓝"});
    printf("%d\n", $(RGBColor, gray_level, c));
    // 187
    $(RGBColor, print, c);
    // 天依蓝: 66CCFF
    return 0;
}
```

收工。这个很原始的系统已经可以运作了。

## 原始的方法绑定 ##

到目前为止，我们一共写了不足60行代码。但是你有没有觉得诸如

```C
$(RGBColor, print, c);
```

这样的写法很糟糕？明明在Loli类实例的前面添加了类型信息，为什么还要先申明类型才能调用方法呢？下面我们来尝试解决这个问题。

一个可能的解决方案是，利用方法的名字和实例的类型来查找需要的方法。这个想法似乎不错，但是C并没有提供足够的自省能力。看来又得自己动手了。

来看看这个改造：

```C
typedef struct _LOLI_TYPE LoliType;

struct _LOLI_TYPE {
    const char * const name;
    const LoliType * const parent; 
    LoliMethod methods[METHOD_MAX];
};
```

`LoliType.methods`用于在类型中注册方法。除了这种老套的解决方案，我们还可以像`Common Lisp Object System`那样，将方法注册到**广义函数**中去。不过那这样的实现会更加复杂，在起步初期，我们还是选择一些比较简单的解决方案吧！

> 0. 这里的广义函数不是定义在从函数集到数集的映射，就像你们信号老师（即将）说的那样。
<br/>
> 0. 是的，用一个数组来记录方法十分不妥。但是我在这里只是想展示一下思路，至于如何高效实现嘛……那些技术需要的篇幅恐怕远甚于此，留待日后再说吧。

在数组这简单的数据结构上写出一个简介的查找算法应该不难。但是界定方法序列结束就需要一些诸如置零的技巧了。

```C
LoliMethod *findmethod(const char* name, LoliType *type) {
    int i = 1;
    while(i<METHOD_MAX) {
        if (!type->methods[i].name)
            break;
        if (strcmp(type->methods[i].name, name) == 0)
            return type->methods[i];
        ++i;
    }
    return type->methods[0];
}
```

如何定制方法查找失败的行为呢？一个**临时**解决方案就是在`LoliType.methods[0]`中放入一个`method_notfound`之类的方法，比如：

```C
void method_notfound() {
    fprintf(stderr, "METHOD NOT FOUND");
}
```

看上去很简单，不是吗？但是却有一个小问题：如何把函数放进`LoliType.methods`里？在`mian`里手动添加？不，还可以有更好的解决方案，不过可能需要重新认识一下C：事实上，`main`函数并不是一个C语言编写的应用真正开始执行的地方——如其不然，谁来构造并初始化栈，谁又来给`main`函数提供`argc, argv, envp`这三个参数呢？

既然在`main`开始之前，早就有一系列的指令搞定了诸如初始化调用栈之类的事情，那么这些初始化动作能不能定制呢？能。但是下面的内容**不是C语言本身的特性，而是编译器提供的扩展**。

`gcc/clang`都实现了`__attribute__`这个指令，其中`__attribute__((constructor))`就是用于定制自定义初始化动作的，而且这些动作可以定义多个。

现在事情变得容易处理多了，只要我们在定义`LoliType`的时候添加一个全局计数器，并处绑定`method_notfound`：

```C
#define $mtcounter(structure) _M_COUNTER_ ## structure

#define $mkcls(structure) \
LoliType $typeinfo(structure) = {#structure,  \
                                 &LoliObject, \
                                 {(LoliMethod) {"NOTFOUND",         \
                                                method_notfound}}}; \
\                                                
size_t $mtcounter(structure) = 1;   \
typedef struct {                    \
    LoliType * const type;          \
    structure              data;    \
} $cls(structure)                   \
```

然后编写一组这样的宏用于绑定方法：

```C
#define $mname(structure, name) _M_ ## structure ## _ ## name

#define $defmethod(ret_t, structure, name, ...) \
    ret_t $mname(structure, name) (structure *self, ## __VA_ARGS__); \
\
    void __attribute__((constructor)) binding_ ## structure ## _ ## name(){ \
        $typeinfo(structure).methods[$mtcounter(structure)++] = \
        (LoliMethod) {# name, \
                      $mname(structure, name)}; \
    } \
\
    ret_t $mname(structure, name) (structure *self, ## __VA_ARGS__)
```

就好了。

现在来解决下一个问题：如何调用方法。

方法的**绑定**是相对容易实现的，只要把指向调用方法的`LoliType.data`的指针传给方法本身就好了。但是如何解决类型的问题呢？

在开始解决这个问题之前，我需要在重申一下刚刚发生的事情：我们并没有扩展语言，而是仅仅透过一组函数和几个宏创造了一个模拟**类**结构的语法糖。这也就是说，C语言本身的运行时特性并没有因为我们几十行代码的努力而让步。

简言之，**我们在`Loli`下写的代码会被预处理器展开成标准的C代码，然后编译成静态文件。**

**没有动态特性！！！**

这实在是太糟糕了。为了避免手写汇编，或者现在大量编写处理类型的系统，想象一个**十分粗糙**的解决方案：强制方法返回`void*`，然后交由用户强制转换并及时释放它。

```C
#define $defmethod(ret_t, structure, name, ...) \
    void* $mname(structure, name) (structure *self, ## __VA_ARGS__);

#define $ret(type, value) \
    void *__$$__ = malloc(sizeof(type)); \
    strncpy(__$$__, (void*)&value, sizeof(type)); \
    return __$$__;

#define $retvoid \
    return NULL;
```

此时，用于调用方法的代码也需要修改：

```C
#define $lolicall(method, object, ...) \
    (((void* (*)())(method->function))(object, ## __VA_ARGS__))

#define $(mname, instance, ...) \
    $lolicall(findmethod(#mname, instance.type), \
              &(instance.data),                  \
              ## __VA_ARGS__)
```

```C
#define $lolicall(method, object, ...) \
    (((void* (*)())(method->function))(object, ## __VA_ARGS__))

#define $(mname, instance, ...) \
    $lolicall(findmethod(#mname, instance.type), \
              &(instance.data),                  \
              ## __VA_ARGS__)
```

又多了几十行代码，让我们重新测试一下。

```C
$mkcls(RGBColor);

$defmethod(int, RGBColor, gray_level) {
    int rtv = (self->R + self->G + self->B)/3;
    $ret(int, rtv);
}

$defmethod(void, RGBColor, print) {
    printf("%s: %02X%02X%02X\n", self->desc,
            self->R, self->G, self->B);
    $retvoid;
}

$defmethod(int, RGBColor, select, char name) {
    int v = 0;
    switch (name) {
        case 'R':
            v = self->R;
            break;           
        case 'G':
            v = self->G;
            break;
        default:
            v = self->B;
    }
    $ret(int, v);
}

int main() {
    $cls(RGBColor) c = $imm(RGBColor, {0x66, 0xCC, 0xFF, "天依蓝"});
    printf("@ %d\n", *(int*)$(gray_level, c));
    printf("R %d\n", *(int*)$(select, c, 'R'));
    printf("G %d\n", *(int*)$(select, c, 'G'));
    printf("B %d\n", *(int*)$(select, c, '?'));
    $(print, c);
    $(oops, c);
    return 0;
}
```
Bingo!

# 走向继承 #

## 1+1 ?= 2 ##
> 线性叠加与内存联合。

## 方法查找 ##
> 支持集成后的方法查找与绑定。
 
## MRO ##
> 借鉴Python的**MRO**实现多继承。

