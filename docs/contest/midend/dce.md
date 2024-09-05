# 死代码消除

死代码消除即无用代码消除，死代码和不可达代码是两个概念。前者指的是执行之后没有任何作用的代码（例如：多余的计算），后者指的是永远无法被执行到的代码。

活跃变量分析为每个程序点计算出所有变量的集合的子集，用于表示该点处活跃的变量，所以数据流分析的值集为所有变量的集合的幂集。"活跃"的含义是在程序执行过这一点**之后**，这个变量**当前的值**会被使用到，所以数据流分析是后向的。对于单个语句$S$，传递函数要根据$S$之后活跃的变量计算$S$之前活跃的变量，计算方法为：所有$S$用到的变量在$S$之前都是活跃的，所有$S$之后活跃的变量，如果没有在$S$中被定值，证明未来的那次使用用的还是$S$之前的值，所以也是活跃的。
<!-- 
综合得，传递函数定义为：$f_S(x) = (x - def_S) \cup use_S$。其中 $def_S$ 是$S$中定值的所有变量的集合，$use_S$是$S$中使用的所有变量的集合。

基本块$B$的传递函数定义为：

$$
f_B(x) = f_{S_1}(...f_{S_{n - 1}}(f_{S_n}(x))) \\
= (((((x - def_{S_n}) \cup use_{S_n}) - def_{S_{n - 1}}) \cup use_{S_{n - 1}} ...) - def_{S_1}) \cup use_{S_1}\\
\overset{\mathrm{数学归纳法}}{=} (x - \bigcup_{i = 1}^n def_{S_i}) \cup \bigcup_{i = 1}^n (use_{S_i} - \bigcup_{j = 1}^{i - 1} def_{S_j})
$$

最后一个等号使用数学归纳法来证明，读者自证不难。

定义$def_B = \bigcup_{i = 1}^n def_{S_i}$，$use_B = \bigcup_{i = 1}^n (use_{S_i} - \bigcup_{j = 1}^{i - 1} def_{S_j})$，这样上面的式子就是大家熟悉的形式了。那么这个形式和课堂上定义的$LiveUse$和$Def$是一致的吗？

![](pic/aliveness.png)

先看$use_B$和$LiveUse$，$LiveUse$为$B$中定值之前被引用的变量的集合，而$use_B$定义中每一个求并项都是一条语句的$use$集合减去在这条语句前面的所有$def$集，也就是说如果一个变量在某条语句中被使用了，而且没有在这条语句之前的任何一条语句被定值，那么它属于$use_B$，此外都不属于。显然，这与$LiveUse$的定义是符合的。

再看$def_B$和$Def$，其实很容易可以看出这两个集合并不相同，$def_B$包含了被定值的所有变量，而$Def$要求定值之前没有引用过，所以$Def \subseteq def_B$。然而这个区别不会影响任何计算结果：如果变量$x$满足$x \in def_B, x \notin Def$，则意味着它定值之前被引用过，则$x \in use_B, x \in LiveUse$，则它一定在这一步的结果集合中。

> 所以，$Def$这样的定义是冗余的，只会加大计算$Def$时的计算量，使用$def_B$的定义就会简单一些，而且不会影响最终结果。 -->

这里讲一下几个常见的需要注意的点：

进行死代码删除的时候，如果一条语句**没有副作用**，而且它的赋值目标(如果有的话)不在$out_S$中，那么这条语句就可以删去
   - 所谓的副作用，其实就是除了"改变赋值目标变量"之外，其他所有的作用。显然，tac中没有既没有副作用，同时也没有赋值目标的语句
   - 你在实现的时候可以认为除了`a = call b`之外的所有有赋值目标的语句都是没有副作用的，对于`a = call b`，如果$a \notin out_S$，要求将它优化为`call b`

其实还有一些语句的"副作用"不是很明确，比如除0，有符号整数溢出等(依平台而定)，可能会导致程序崩溃，但是**优化的时候可以不把这当成是副作用**：按照c/c++常用的说法这叫未定义行为，这可以减轻编译器作者的负担，编译器可以假定程序永远没有未定义行为，并以此为依据来优化。

我们的测试样例中不会出现未定义行为，所以可以放心地忽略掉这些副作用。