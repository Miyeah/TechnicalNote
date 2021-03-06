# SICP

## what is data 
P123
> In general, we can think of data as defined by some collection of selectors and constructors, together with specified conditions that these procedures must fulfill in order to be a valid representation.

## Church numerals
P126
> (define zero (lambda (f) (lambda (x) x))) (define (add-1 n)
(lambda (f) (lambda (x) (f ((n f) x)))))
this representation is known as Church numerals, after its inventor, Alonzo Church, the logician who invented the λ- calculus.

翻译成 js 是
```
const zero = 
    (f) => 
      (x) => x

const addOne = 
  (n) =>
    (f) =>
      (x) =>
        f(n(f)(x))
```

## 2.4 数据表示
### Tagged data 代标志的数据
给每一种数据带上标志，调用函数的时候通过标志来确定调用什么样的方法。缺点是函数需要知道所有的标志，添加新的标志需要修改所有的函数。编写新的数据类型的时候要注意不能重名。

### Data-Directed 数据导向
通过维护一个表格（数据类型 -> 方法），来决定函数分配(dispatch)，类似于 C++ 的虚表。称这个表为函数分配表。

### 消息传递 Message passing
所需要的操作通过字符作为`消息`传入, 通过构建一个`智能操作结构`，智能体现在可以根据消息分配函数。`智能操作结构` 决定了函数分配表中的列.

### 引用透明
p 315
> A language that supports the concept that “equals can be substituted for equals” in an expression without changing the value of the expres- sion is said to be referentially transparent. 

# y-combinator Ex 4.21
y-combinator 用于解决匿名函数递归的问题
参考[康托尔、哥德尔、图灵——永恒的金色对角线(rev#2)](http://mindhacks.cn/2006/10/15/cantor-godel-turing-an-eternal-golden-diagonal/)

以阶乘函数举例
```
(define (fac x)
  (if (= x 1) 
    1
    (* x (fac (- x 1)))
  )
)
```

递归的时候我们需要知道当前的名字（调用当前函数的方法），如果我们不知道当前函数的名字（匿名函数）还有办法递归么？

一个较为自然的想法是，把调用方法加在参数中，如下
```
(define fac 
(lambda (self x) 
  (if (= x 1) 
    1
    (* x (fac self (- x 1)))
  )
))

(fac fac 10) ;Value: 3628800
```
虽然这样子使得 lambda 内部并没有使用 fac 这个名字，但是我们仍然需要在某个地方保存着 lambda 的调用方式，另外 (fac 10 fac) 这样的调用很不优雅。

现在，假设我们已经实现了匿名函数的递归，可以通过 (real_fac 10) 这样的方式来调用，我们可以改造一下之前完成的 fac
```
(define (get_real real_fac)
  (lambda (x) 
    (if (= x 1) 
      1
      (* x (real_fac (- x 1)))
    )
  )
)
```
表面上看起来差别不大，而且用到了一个假想中的函数，但是注意，现在的 fac 已经变成了 real-fac。这里发生了什么？ 使用一个 真\*递归函数可以将一个匿名函数变成它自己，抽象一下就是
```
(get_real f) = f
```

我们如果能够找到一个 f 就能到得到这个 f(神经病啊！！)，也就是 get_real 这个函数的不动点(f(x)=x)
我们继续假设，有这么一个函数 Y，它能找到任何函数的不动点，即
```
Y(F) = f

where F(f) = f
```

也就是说：
```
Y(F) = F(Y(F))
```
所以现在的问题就是 如何构造这个 Y 函数的了，它就是 y-combinator

最开始的时候，我们通过(fac fac 10) 的方式得到了一个较丑陋的方式，但是确实最接近 real-fac 的尝试了，从这个思路入手，我们把 作为real-fac， 注意(fac fac)省略了参数n (柯理化)
```
(define generator
  (lambda self (get_real (self self))
)
```

这里基本上已得到了答案
```
(define Y
   (lambda f
        (let ((g (lambda (x) 
               (f (lambda arg (apply (x x) arg))))))
          (g g)
     )
   )
)
```

测试一下
```
(define f
  (lambda (real_func)
    (lambda (n)
      (if (zero? n)
          1
          (* n (real_func (- n 1)))))))
```

写成 js 是：
```
const y = f => { const g = self => x =>  f(self(self))(x); return g(g); }
```

测试一下
```
y(real =>x=> (x == 0 ? 1 : x * real(x-1)))(10) // 3628800
```

这里的用 js 实现的确切叫 z-combinator，即在 applicative order 中的实现，见下面一节。
注意，
```
x =>  f(self(self))(x)
```
不能写成
```
f(self(self))
```
区别如下：
```
const show = () => {console.log('!!!')}
const f = (x) => {x(); return (y) => (y())}

const g = (x) => f(show)(x) // nothing
const h = f(show) // !!!
```
表面上看起来是仍是需要定义变量名的, 但你可以写成这样了
```
(self => x => f(self(self))(x))((self => x => f(self(self))(x)))
```
把 f 替换成你想要的函数即可。（js 由于没有惰性求值，很容易得到一个死递归，需要注意）

# normal order and applicative order
4.2.1
lazy 和 normal-order 之间的差别有一点点微妙。一般而言，lazy 指的是一种求值的机制，而 normal-order 指的是一种语言的语义，和求值的方式无关。但是这不是一个固定的区别，这两个术语经常互用。

接下来又讲了 strict 和 non-strict 分别对应 non-lazy 和 lazy。
在纯的 applicative order 语言中所有的参数都是 strict。在纯的 normal-order 语言中，可能是 strict 或者 non-strict 的，或者由使用者自定义。

假设我们定义 unless
```
(define (unless condition process else-process)
  (if condition else-process process)
)
```
看起来了很好，但是在 applicative-order 有下面的问题
```
(unless (= b 0)
  (/ a b)
  0
)
```
其中 (/ a b) 是求值之后传给 if 的，会产生异常。

```
(define (factorial n)
  (unless (= n 1)
    (* n (factorial (- n 1)))
    1
  )
)
```
会发生一直递归下去，因为即使(= n 1) 也会 执行 (* n (factorial (- n 1)))，从而一直执行下去。

在 lazy 的定义下，可以这样定义
```
(define ones (cons 1 ones))
```
奇怪的是 scheme 里不是无限递归，可能和 scheme 的实现有关吧。 js 这样的表达式确实的无限递归的。

lazy 的一个问题是何时求值，sicp 中的做法是primitive function 或者  if(condition)，这个规范定义好就行。




