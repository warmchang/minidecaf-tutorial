## 代码生成

本阶段代码生成的重点是处理 `*` 和 `&` 这两个运算符，而对于类型系统和强制类型转换属于语义检查，无需生成代码。

首先，对于 `*` 运算，只需一条 Load 指令即可完成解引用这一操作。但对于取地址符 `&` 就不是很直观了，因为 `&` 号后面可能不会立即跟一个变量如 `&a`，而是会有形如 `&*p` 这样的，虽然两个操作可以相互抵消，但如果将其解释为“先解引用，再取地址”，就难以生成对应的汇编代码，因为变量的地址信息会在解引用后丢失。此外，对于像 `*` 号出现在赋值语句左边的情况，也不是很好处理。因此，在讨论如何生成代码之前，我们先来讨论一下“左值”的概念。

### 左值

我们都知道，赋值语句可以写成 `a = 123` 的形式，而不能是 `123 = a`，这是因为赋值号左边的是一个**左值**（lvalue），要求与一个地址相关联；而右边的 `123` 是一个**右值**（rvalue），并不需要关联一个地址。通俗地讲，左值是一个**有地址的值**，例如：

```c
int a;
int* p;
a = 1;      // a 是左值
*&a = 2;    // *&a 是左值
p = &a;     // p, a 是左值
*p = 3 + a; // *p 是左值
```

从上述代码可以看出，赋值号左边的以及 `&` 后面的都要求是左值，需要知道它们的地址才能进行计算；而像几个常量和最后一行的 `a` 这样的是右值，只需知道其值就行了。

有了左值的概念，就比较容易处理本节一开始提到的哪些问题了。例如对于 `&a` 和 `&*p`，可看做求左值 `a` 和 `*p` 的地址（不考虑空指针，`*p` 的地址就是 `p` 的值），对于赋值 `*p = 1`，可以看做先求左值 `*p` 的地址，再向该地址写入 1。

因此，在生成代码的过程中，对于左值，我们不能生成其具体的值，而要先生成它们的地址。对于一个表达式，我们需要先确定那些部分是左值，并求得它们的地址作为中间结果，然后才能生成正确的代码。通过观察，我们可以归纳出左值的判定方法：

1. 出现在赋值号 `=` 的左边，或是取地址符 `&` 的右边；
2. 是一个标识符，或是 `*` 开头的表达式（`(*p)` 等有括号的形式也算）。

在建立了 AST 后，我们可以很容易求出哪些节点是左值，可将其作为节点的属性，以供之后使用。

此外，我们还需要进行左值检查，以拒绝编译形如 `123 = a` 的错误输入。根据上述左值的判定方法，如果表达式的某一部分满足第 1 条，但不满足第 2 条，就是不合法的。

### 基于左值的代码生成框架

现在，我们假设已经求出了 AST 的哪些节点是左值。在之后的遍历 AST 生成汇编码时，只需对左值生成地址，对右值生成值。在我们的文法中，左值一定出现在 `<factor>` 非终结符，其文法为：

```
<factor> ::= Identifier | '*' <factor> | '&' <factor> | ...
```

下面给出了用于生成汇编码的伪代码：

```js
function visitFactor(ctx) {
    if (ctx ::= Ident) {                // <factor> ::= Identifier
        emitAddress(Ident);             // 计算变量 Ident 的地址
        if (ctx.isLValue) {
            // do nothing               // 如果是左值，直接返回其地址
        } else {
            emitLoad();                 // 否则，再生成一条 Load 指令来取得变量的值
        }
    } else if (ctx ::= '*' factor) {    // <factor> ::= '*' <factor>
        visitFactor(factor);            // 访问子 factor 计算其值
        if (ctx.isLValue) {
            // do nothing               // 如果是左值，直接返回该值
        } else {
            emitLoad();                 // 否则，再生成一条 Load 指令来解引用
        }
    } else if (ctx ::= '&' factor) {    // <factor> ::= '&' <factor>
        visitFactor(factor);            // 子 factor 一定是左值，直接访问以得到其地址
    } else {
        // ...
    }
}
```

### 生成左值的地址

如上一节所述，左值只可能出现在两种地方：

1. 标识符：地址即该变量的地址；
2. 解引用符 `*` 之后：地址即之后那部分的值；

第二种情况可直接忽略，我们只需考虑如何计算一个变量的地址。在上一个 lab 我们引入了全局变量，所以变量可分为两种：

* 局部变量：保存在栈上，使用 `lw t0, offset(fp)` 指令获取其值。由于我们使用栈来传参，所以参数也可认为是局部变量。
* 全局变量：保存在数据段，使用 `lui t1, %hi(N); lw t0, %lo(N)(t1)` 指令获取其值。

要从取值变成取地址，只需将 `lw` 改成 `addi`：

```asm
addi t0, fp, offset     # 获取局部变量的地址，保存到 t0

lui t1, %hi(N)
addi t0, t1, %lo(N)     # 获取全局变量 N 的地址，保存到 t0
```

之后，可再用一条 Load 或 Store 指令来实现获取变量的值或给变量赋值：

```asm
lw t1, 0(t0)    # 读取内存地址 t0 中的内容到 t1
sw t1, 0(t0)    # 将 t1 写入内存地址 t0
```

### 赋值语句

最后，我们还需要修改赋值语句的生成过程。在之前的语法分析阶段，赋值语句的文法已经改为了

```diff
-<exp> ::= Identifier "=" <exp> | <conditional-exp>
+<exp> ::= <factor> "=" <exp> | <conditional-exp>
```

其中要求 `<factor>` 是一个左值。

我们在之前生成赋值语句时是这么做的：

```asm
生成 expr 的代码      # 结果存在 t0
sw t0, offset(fp)   # offset 为变量在栈帧中的位置
```

现在，已经没有变量了，而是变成了左值。我们就先生成计算左值地址的代码，然后再生成一条 Store 指令即可：

```asm
生成 factor 的代码  # 结果（左值的地址）存在 t0
生成 expr 的代码    # 结果存在 t1
sw t1, 0(t0)      # 将 t1 写入内存地址 t0
```