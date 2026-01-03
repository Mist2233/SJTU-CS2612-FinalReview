## 分离逻辑
现在，我们进入分离逻辑的部分。

分离逻辑主要是针对 While+Dereference 中的解引用部分的语法补充，即如何处理指针和内存。

为了处理内存，分离逻辑将程序状态 $s$ 细化为两个部分：变量 (vars) 和堆内存 (mem)。

它引入了分离合取 (Seperation Conjunction)，符号为 $*$。断言 $P * Q$ 表示，可以将程序状态中的内存拆分成为互不相交的两部分，其中一个满足条件 $P$，而另一个满足条件 $Q$。

即，程序状态 $s$ 满足性质 $P * Q$，当且仅当存在 $s_{1}$ 与 $s_{2}$，使得：
- $s_{1}$ 满足 $P$，$s_{2}$ 满足 $Q$
- $s.\text{vars} = s_{1}.\text{vars} = s_{2}.\text{vars}$，并且 $s.\text{mem} = s_{1}.\text{mem} \uplus s_{2}.\text{mem}$

*注：符号 $\uplus$ 表示“不相交并集”。*

这里，我们用 $\text{store}(a, b)$ 表示地址 $a$ 上储存了 $b$ 这个值，并且仅仅拥有此内存权限。

例如，
- `*x == m && *y == n && x != y` 可以写作 `store(x, m) * store(y, n)`
- `**x == 0 && *x != x` 可以写作 `exists u. store(x, u) * store(u, 0)`

*注：第二个例子里有双重解引用，只需要将 `*x` 记作 `u`，原本的条件就可以改为 `*x == u && *u == 0 && u != x`，那么就可以写出分离逻辑 `store(x, u) * store(u, 0)`*

## 霍尔逻辑规则：引入分离逻辑之后
### 内存赋值规则（正向）
如果前条件 $P$ 能推出：
- 表达式 $e_{1}$ 能够安全求值，并且求值结果为 $a$
- 表达式 $e_{2}$ 能够安全求值，并且求值结果为 $b$
- 存在 $u$，使得 $\text{store}(a, u) * Q$

其中 $a$ 与 $b$ 都是与内存无关的数学式子。

那么，霍尔三元组 $\{ P \} ~ *e_{1} = e_{2} ~ \{ \text{store}(a, b) * Q \}$ 成立。

> 这个规则就是说，内存赋值成功之后，对应地址的值变化，然后其他都不变。

### 变量赋值规则（正向）
如果 $P$ 能推出 $e$ 能够安全求值且求值结果为 $a$，那么：
$$
\{ P \} ~ x = e ~ \{ \exists x'. a[x \mapsto x'] = x ~ \&\& ~ P[x \mapsto x'] \}
$$

其中，$a$ 是与内存无关的数学式子。

> 这个规则就是说，变量赋值成功之后，对应变量的值变化，其他的值不变，但是后条件中的判定需要使用变量的原值，而不是新值。

### 框架规则
如果 $F$ 中不出现被 $c$ 赋值的变量，并且霍尔三元组 $\{ P \} c \{ Q \}$ 成立，那么霍尔三元组 $\{ P * F \} c \{ Q*F \}$ 也成立。

> 这个规则就是说，如果一部分内存中的变量和当前程序运行的变量无关，那么并上这部分内存并不影响原有霍尔逻辑的判定。

### 存在量词规则
如果对于任意 $a$ 都有 $P(a)$ 成立并且霍尔三元组 $\{ P(a) \} c \{ Q \}$ 成立，那么霍尔三元组 $\{ \exists a, P(a) \} c \{ Q \}$ 成立。

> 这个规则就是说，如果对于任意值，霍尔三元组都能成立，那么把前条件换为一个存在条件，这个霍尔三元组还是能成立。

## 描述单链表的谓词
利用分离逻辑，我们可以定义一些新的谓词，从而简洁描述数据结构。以单链表为例。

可以用谓词 `sll(p)` 表示以 `p` 地址为头指针存储了一个单链表，其定义为：
```
p == 0 && emp ||
exists u q, store(p, u) * store(p + 8, q) * sll(q)
```

可以用谓词 `sllseg(p, q)` 表示以 `p` 地址为头指针开始到 `q` 为止是单链表中的一段，其定义为：
```
p == q && emp ||
exists u r, store(p, u) * store(p + 8, r) * sllseg(r, q)
```

## 分离逻辑使用的例子
### 交换地址上的值
下面的霍尔三元组描述了程序“交换地址 `x` 和 `y` 上的值”的内存安全性：
```c
{ store(x, m) * store(y, n) }
t = *x;
*x = *y;
*y = t
{ store(x, n) * store(y, m)}
```

### 单链表取反
下面的程序描述了单链表取反的过程：
```c
while (x != 0) do {
	t = x;				// 用临时变量t存下当前处理的节点指针
	x = *(x + 8);		// 将指针x指向原链表的下一个地址
	*(t + 8) = y;		// 将当前的节点指向y，y是新链表的链表头
	y = t				// 更新链表头y
}
```

上面这个程序，有点难看懂，我们来拆解一下。

在分离逻辑的定义中，一个链表节点的地址为 `p`，它实际上占据了两个连续的内存位置：
- 地址 `p`：存储数据值 `val`
- 地址 `p + 8`：存储指向下一个节点的指针 `next`

结合注释，并理解上面的链表节点定义，程序不难理解。

利用前面定义的 `sll(p)` 谓词，可以如下描述单链表取反程序的内存安全性性质：
```c
{ y == 0 && sll(x) } 	// 这里y == 0表示y目前指向空
while (x != 0) do {
	t = x;
	x = *(x + 8);
	*(t + 8) = y;
	y = t
}
{ x == 0 && sll(y)}
```

如何验证该程序的正确性？联系我们前面的符号执行 [[PLC-8-HoareLogic#符号执行（正向）与验证条件生成]] 的内容，需要证明三件事：
- 循环开始前，`inv` 是否成立？
- 每一轮循环结束后，`inv` 是否成立？
- 循环退出后，`ensure`（后条件） 是否成立？

可以选用 `sll(x) * sll(y)` 作为循环不变量，它表示 x 链表和 y 链表在内存中是互不相交的两部分。

首先，前条件可以推出循环不变量，
```c
y == 0 && sll(x)
|-- y == 0 && emp * sll(x) 
|-- (y == 0 && emp) * sll(x) // 根据sll的定义即可推出下面一条
|-- sll(y) * sll(x)
|-- sll(x) * sll(y)
```

其次，循环体能保持循环不变量：
```rust
{ x != 0 && sll(x) * sll(y) }  
{ exists u z, store(x, u) * store(x + 8, z) * sll(z) * sll(y) }  
// Given u z,  
{ store(x, u) * store(x + 8, z) * sll(z) * sll(y) }  
t = x;  
{ t == x && store(x, u) * store(x + 8, z) * sll(z) * sll(y) }  
x = * (x + 8);  
{ exists x', x == z && t == x' && store(x', u) * store(x' + 8, z) * sll(z) * sll(y) }  
{ x == z && store(t, u) * store(t + 8, z) * sll(z) * sll(y) }  
* (t + 8) = y;  
{ x == z && store(t, u) * store(t + 8, y) * sll(z) * sll(y) }  
y = t  
{ exists y', y == t && x == z && store(t, u) * store(t + 8, y') * sll(z) * sll(y') }  
{ exists y', store(y, u) * store(y + 8, y') * sll(x) * sll(y') }  
{ sll(x) * sll(y) }
```

最后，退出循环时后条件成立：
```c
!(x != 0) && sll(x) * sll(y)
|-- (x == 0) && sll(x) * sll(y)
|-- (x == 0) && sll(y)
```

### 单链表的连接 1
下面的程序描述了单链表的连接：
```c
if (x == 0)
then {
	// res表示链表的头节点
	res = y
}
else {
	res = x;
	nx = * (x + 8);
	// 下面这个while循环，直接走到当前链表的结尾
	while (nx != 0) do {
		x = nx;
		nx = * (x + 8)
	};
	// 把节点y接在当前链表的结尾之后
	* (x + 8) = y
}
```

下面是添加了正向符号执行语句之后的程序：
```c
//@ require sll(x) * sll(y)
//@ ensure sll(res)
if (x == 0)
then {
	res = y
}
else {
	res = x;
	nx = *(x + 8)
	//@ [generated] v == nx && x == res && store(x, u) * store(x + 8, v) * sll(v) * sll(y)
	//@ inv exists u, sllseg(res, x) * store(x, u) * store(x + 8, nx) * sll(nx) * sll(y)
	while (nx != 0) do {
		// @ [generated] nx != 0 && exists u, sllseg(res, x) * store(x, u) * store(x + 8, nx) * sll(nx) * sll(y)
		x = nx;
		// @ [generated] nx != 0 && x == nx && exists u, sllseg(res, x') * store(x', u) * store(x' + 8, nx) * sll(nx) * sll(y)
		nx = *(x + 8)
		// @ [generated] exists u_new, sllseg(res, x) * store(x, u_new) * store(x + 8, nx) * sll(nx) * sll(y)
	};
	//@ [generated] exists u, nx == 0 && sllseg(res, x) * store(x, u) * store(x + 8, nx) * sll(nx) * sll(y)
	*(x + 8) = y
	//@ [generated] exists u, nx == 0 && sllseg(res, x) * store(x, u) * store(x + 8, y) * sll(y)
	// 根据定义，直接得到ensure sll(res) 成立
}
```

演示一下中间一个语句的化简（学会利用定义）：
```rust
nx != 0 && x == nx && exists u, sllseg(res, x') * store(x', u) * store(x' + 8, nx) * sll(nx) * sll(y)
// 用 x == nx 来替换
|-- nx != 0 && x == nx && exists u, sllseg(res, x') * store(x', u) * store(x' + 8, x) * sll(x) * sll(y)
// 注意，这里的sllseg(res, x') * store(x', u) * store(x' + 8, x)可以化简为sllseg(res, x)，然后中间变量u就可以去掉了。
|-- nx != 0 && x == nx && sllseg(res, x) * sll(x) * sll(y)
// 把sll(x)展开，因为下一步要用到x+8
|-- nx != 0 && x == nx &7 sllseg(res, x) * store(x, u_new) * store(x + 8, v) * sll(v) * sll(y)
// 然后，再代入 nx = *(x + 8) 这一句，消去v，就能回到
```

### 单链表的连接 2
下面是用二阶指针实现的单链表连接：
```rust
* phead = x;  
pt = phead;  
while ( * pt != 0) {  
	pt = * pt + 8;  
};  
* pt = y;  
res = * phead
```

请证明，其满足：
```c
{ exists u, store(phead, u) * sll(x) * sll(y) }  
* phead = x; 
//@ [generated] store(phead, x) * sll(x) * sll(y) 
pt = phead;  
//@ [generated] pt == phead && store(phead, x) * sll(x) * sll(y)
//@ inv exists v, SllSeg(phead, pt) * store(pt, v) * sll(v) * sll(y)
while ( * pt != 0 ) {  
	//@ [gene]
	pt = * pt + 8;  
};  
* pt = y;  
res = * phead  
{ exists u, store(phead, u) * sll(res) }
```

证明中应当指明循环语句的循环不变量，每条赋值语句的最强后条件，以及每次使用 Consequence rule 改写前后的断言。另外，证明中可以考虑使用下面谓词：

`SllSeg(p, q)`，定义为：
```
p == q && emp ||
exists u r, store(p, r) * store(r, u) * SllSeg(r + 8, q)
```

这个谓词是理解二阶指针算法的关键。

- **$p$ 的含义**：起始的 **“指针容器地址”**（例如 `phead` 的地址）。
- **$q$ 的含义**：当前的 **“指针容器地址”**（即 `pt` 当前指向的位置，通常是某个节点的 `next` 域的地址）。
- **整体含义**：$SllSeg(p, q)$ 表示从地址 $p$ 到地址 $q$ 之间，已经形成了一段链表路径。
    - 它描述的不是“节点到节点”，而是 **“容器到容器”**。
    - 定义中的 `store(p, r) * store(r, u)` 表示：地址 $p$ 里面存了一个节点的地址 $r$，而这个节点 $r$ 里面存了数据 $u$。
    - 接下来的 `SllSeg(r + 8, q)` 表示：从这个节点的 `next` 域地址（即 $r+8$）开始，继续往后连，直到摸到地址 $q$ 为止。

```c
//@ require exists u_old, store(phead, u_old) * sll(x) * sll(y)
//@ ensure exists u_new, store(phead, u_new) * sll(res)

* phead = x;
//@ [generated] store(phead, x) * sll(x) * sll(y)

pt = phead;
//@ [generated] pt == phead && store(phead, x) * sll(x) * sll(y)
//@ [consequence] SllSeg(phead, pt) * store(pt, x) * sll(x) * sll(y)
// (根据 SllSeg 定义，当 pt == phead 时，SllSeg(phead, pt) 为空内存 emp)

//@ inv exists v, SllSeg(phead, pt) * store(pt, v) * sll(v) * sll(y)
while ( * pt != 0) {
    //@ [generated] * pt != 0 && exists v, SllSeg(phead, pt) * store(pt, v) * sll(v) * sll(y)
    // 根据 sll(v) 且 v != 0 的定义展开:
    //@ [unfold] exists u n, SllSeg(phead, pt) * store(pt, v) * store(v, u) * store(v + 8, n) * sll(n) * sll(y)

    pt = * pt + 8;
    //@ [generated] pt == v + 8 && SllSeg(phead, pt_old) * store(pt_old, v) * store(v, u) * store(v + 8, n) * sll(n) * sll(y)
    //@ [consequence] SllSeg(phead, pt) * store(pt, n) * sll(n) * sll(y)
    // 根据 SllSeg 定义：SllSeg(phead, pt_old) * store(pt_old, v) * store(v, u) 折叠为 SllSeg(phead, v+8)
};

//@ [generated] * pt == 0 && exists v, SllSeg(phead, pt) * store(pt, v) * sll(v) * sll(y)
// 此时 v 必为 0，且 sll(0) 为 emp
//@ [consequence] SllSeg(phead, pt) * store(pt, 0) * sll(y)

* pt = y;
//@ [generated] SllSeg(phead, pt) * store(pt, y) * sll(y)
//@ [consequence] sll_from_phead(phead, res_val)
// (此步将 SllSeg 与结尾的 y 链表合并为以 phead 指向内容为首的完整链表)

res = * phead;
//@ [generated] exists u_new, store(phead, res) * sll(res)
```