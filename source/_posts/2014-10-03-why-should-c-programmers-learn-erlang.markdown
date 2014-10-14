---
layout: post
title: "为什么每一个C程序员都应该学学Erlang"
date: 2014-10-03 09:51:04 +0800
comments: true
categories: Erlang Language 
---

这是一篇翻译文章，原文[在此][1]。水平有限，见笑了。

2013年底初接触 [Erlang][2] ，真的被她震撼到了！虽然一开始很不习惯其语法和变量只能绑定不能改变值等各种新奇的东西，写起代码来也很别扭，但是通过一天的接触学习，毫不夸张地说，我爱上她了。

你要问为什么？
爱难道需要理由吗？当然要，翻译这篇文章，就是为了回答这个问题！

<!-- more -->

很可惜的是，后来由于各种原因包括工作变动，并没有继续深入学习使用过 Erlang ，仅以此文，开启重新学习 Erlang 的计划！

以下是正文：

----------

# 前言

如果有人跳出来说一种语言比另一种语言好，通常这将引发这两个语言阵营之间一场激烈的争论。一旦你使用某种语言很长时间，你会成为她的狂热粉丝，并不自觉地试图去捍卫她。不管你承不承认，（此时的）你已经跳入了一个陷阱，你所能看到的东西被极大地限制了（，变得狭隘）。

对此，《肖生克的救赎》给出了很好地注解：
![The Shawshank Redemption][3]
> * [Red] 这些墙很有趣。刚入狱的时候，你痛恨周围的高墙；慢慢地，你习惯了生活在其中；最终你会发现自己不得不依靠它而生存。这就叫体制化。

所以趁着在体制化的泥潭中还没有陷得太深，让我们尝点新鲜的玩意儿吧———学习一门 C 家族以外的语言，一门带你进入全新领域的语言。

对，`Erlang` 看起来是个很不错的选择！

----------

# 为什么是 Erlang ?

Erlang 最初由 Ericsson 公司为了其“下一代”交换机而开发出来的，已经有了 20 多年的历史（诞生于 1987 年，1998 年开源）。

Erlang 以 [Prolog][4] 语言为基础，加入了 [函数式编程][5] 的特性，加之其难以理解的语法，使她看起来真是一门古怪的语言。

然而，Erlang 这门语言融合了几乎最优秀的设计哲学，（她的设计思想）领先当今时代至少 10 年。

（真这么牛逼？）容我慢慢道来。

## 几乎无副作用

作为一门函数语言， Erlang 摒弃了 `共享状态` 的概念。变量只能被 `绑定`，（一旦完成绑定，变量的值就）不能被改变。

```sh
1> X = 1.
1
2> X = X + 1.
** exception error: no match of right hand side value 2
3> X = 2.
** exception error: no match of right hand side value 2
```

这保证了你能写出没有副作用的函数，也就是说，用相同的参数多次调用一个函数，返回值是一样的。这是 Erlang 一个很重要的优点，它带来的好处是：

> * **你（几乎）不需要去保护所谓的 `临界区`。** 想一想并发环境下的 C 代码，你需要使用若干 `同步原语` 来保护数据不被破坏。糟糕的同步实现还会给系统带来底下的性能和不稳定性。这是让很多 C 程序员头疼的地方。
> * **底层优化变得容易。** 因为编译器知道一个变量一旦完成绑定将不再改变，它可以对寄存器的使用采用相当激进的优化策略。
> * **垃圾回收变得容易。** 可怜的 Java 虚拟机，为了确定一个变量是否可回收可没那么轻松，因为它可能被别的变量所引用，可能还会被使用或者改变。Erlang 不同，一个变量的作用域仅在当前函数，没有其他的代码会访问到它，更别说改变它的值了。(感兴趣的话，可以进一步了解一下 [Erlang 的垃圾收集和内存管理][6]。)


## 内建的 异步/并发 机制

Erlang 内建了对 `并发/异步` 机制的支持，其背后的思想是 [Actor 模式][7]。在 1986 年的时候，在那个 `多核`，`多线程`，甚至 `SMP`（对称多处理）这些术语还不为人所知的年代，这是多么富有远见的创造啊！

### 轻量级进程

为了支持并发，Erlang 在她的虚拟机中创建自己的轻量级进程。
> * 你可以同时创建 `上百万` 个进程，也不会有操作系统资源瓶颈的问题。
> * 并且，进程的创建是如此快速———在奔腾4 CPU上这个速度是 350,000 个/每秒。
> * Erlang 进程占用的内存资源也非常小——仅在 KB 的粒度（最小可达到约300字节）。操作系统层次的进程动辄 MB 的粒度
> * 进程调度方面，Erlang 支持 [软实时][8] 调度。并且 上下文切换 的代价非常小——在现代处理器上，Erlang 进程间切换仅需要 `16` 个指令周期和 `20` 纳秒（[参考][9]）。如果你对进程调度感兴趣，可参阅 [Erlang 如何调度][10]。


### 消息传递

Erlang 使用 `消息传递` 来进行 `进程间通信`，这是继承自 Actor 模式 的方法。
每个 Erlang 进程都有用来存储传入 `消息` 的 `信箱`。
得益于内建进程和进程间的消息传递等机制， Erlang 是完全异步的。

#### 实例

`代码`：
```erlang
-module(echo_server).
-export([rpc/2, loop/0]).

rpc(Pid, Request) ->
    Pid ! {self(), Request},
    receive
        Response ->
            Response
    end.

loop() ->
    receive
        {From, {message, Message}} ->
            From ! {ok, Message},
            loop();
        {From, Request} ->
            From ! {error, Request},
            loop()
    end.
```

`命令行执行`：
```sh
1> c(echo_server).
{ok,echo_server}
2> Pid = spawn(fun echo_server:loop/0).
<0.42.0>
3> echo_server:rpc(Pid, {message, "Hello world!"}).
{ok,"Hello world!"}
4> echo_server:rpc(Pid, {message1, "Hello world!"}).
{error,{message1,"Hello world!"}}
```

希望你没被上面这些语法和函数式编程的语句吓到。我不打算对代码细节进行展开，简单点说，这段代码创建了一个回显服务器，并向该服务器发送消息。

使用 `spawn`，`!`(消息传递关键字) 和 `receive`，我们仅用几行代码就完成了服务端的创建。对比一下，在 C，Java，Python，Ruby 或者 node.js 中如何完成相同的任务，你就会发现 Erlang 是多么的优雅！

`Actor` 模式对于 `并发` 是如此的重要，以至于一些现代的语言比如 [Golong][11] 也在语言层面上对它进行支持。我对 Golang 并不熟悉，但是（我知道）它允许共享内存，因此在 Golang 中进行并发编程可能仍然需要同步的工作。并且，我认为 Golang 并不支持软实时，因为这是 Erlang 的设计目标而非 Golang 的。

诸如 Java，Python 和 Rudy 等其他语言，则使用 `库` 的方式对 Actor 模式进行支持。对其 `效率` 我持质疑的态度。只有把 `协程` 及其 `调度` 放到 `虚拟机` 层面，你才能获得最好的性能。（这段话不甚理解，mark）


## 可扩展性

从前面的例子可以看出在 Erlang 里我们的代码看起来结构相对松散些（是这个意思不？）。加上内建的并发支持，Erlang 应用程序相当容易扩展。你可以从一个节点将代码分发到多个节点，分发到局域网内多台机器上，甚至分发到其他网络的服务器上，而只需要额外一点点代码。这是因为：

> * Erlang 允许你在远程节点上创建进程
> * Erlang 允许你用如同跟本地进程交互一样的方式，与远程进程进行交互

对前面的程序稍加改动 (添加一个函数), 我们就能够将其分发到 2 个 Erlang 节点上去：

```erlang
start() -> register(?MODULE, spawn(fun loop/0)).
```

```sh
➜  Erlang-programming-examples  erl -sname weasley
Erlang R16B (erts-5.10.1) [source] [64-bit] [smp:8:8] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Eshell V5.10.1  (abort with ^G)
(weasley@cnrd-tchen-mbp)1> c(echo_server).
{ok,echo_server}
(weasley@cnrd-tchen-mbp)2> echo_server:start().
true

➜  Erlang-programming-examples  erl -sname potter
Erlang R16B (erts-5.10.1) [source] [64-bit] [smp:8:8] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Eshell V5.10.1  (abort with ^G)
(potter@cnrd-tchen-mbp)1> rpc:call('weasley@cnrd-tchen-mbp', echo_server, rpc, [{message, "Hurry up, Harry!"}]).
{ok,"Hurry up, Harry!"}
```


## 热更新

这是每个系统程序员梦寐以求的特性啊！看看你为客户提供的那些实现起来单调乏味的所谓 `热补丁` 的解决方案，不断的删改，看起来又很不得体，有着很大的局限性。但是 Erlang 正好相反，她yi yi zhongsupports hot code reload in an elegant manner.

沿用前面的例子，我们通过修改一点代码以便待会能够动态更新运行中的代码：


```erlang
-module(echo_server_general).
-export([start/2, rpc/2, swap_code/2]).

start(Name, Mod) ->
    register(Name, spawn(fun() -> loop(Name, Mod) end)).

swap_code(Name, Mod) ->
    rpc(Name, {swap_code, Mod}).

rpc(Name, Request) ->
    Name ! {self(), Request},
    receive
        {Name, Response} -> Response
    end.

loop(Name, Mod) ->
    receive
        {From, {swap_code, NewMod}} ->
            From ! {Name, ack},
            loop(Name, NewMod);
        {From, Request} ->
            Response  = Mod:handle(Request),
            From ! {Name, Response},
            loop(Name, Mod)
    end.

-module(echo_server).
-export([echo/2, handle/1]).

echo(Name, Message) -> echo_server_general:rpc(Name, {echo, Message}).

handle({echo, Message}) -> {ok, Message}.
```

运行：

```sh
(weasley@cnrd-tchen-mbp)1> echo_server_general:start(s, echo_server).
true
(weasley@cnrd-tchen-mbp)2> echo_server:echo(s, "hello world").
{ok,"hello world"}
```

现在新需求来了——回显服务器需要把消息的首字母转换成大写形式再回显发送。一般情况下，我们需要停掉服务器程序，替换代码，然后重启服务器。但在 Erlang 中，不需要这么麻烦。

代码修改：

```erlang
-module(echo_server).
-export([echo/2, handle/1]).

echo(Name, Message) -> echo_server_general:rpc(Name, {echo, Message}).

handle({echo, Message}) -> {ok, capfirst(Message)}.

capfirst([H|T]) when H >= $a, H =< $z ->
    [H + ($A - $a)|T];
capfirst(Others) -> Others.
```

运行：

```sh
(weasley@cnrd-tchen-mbp)4> c(echo_server).
{ok,echo_server}
(weasley@cnrd-tchen-mbp)5> echo_server_general:swap_code(s, echo_server).
ack
(weasley@cnrd-tchen-mbp)6> echo_server:echo(s, "hello world").
{ok,"Hello world"}
```

热更新的特性不仅对于有着高可用性要求的服务非常有用，对于软件的开发生命周期同样很有帮助。你不需要浪费许多时间在停服，更新程序，然后重启这种事情上，而且有时候仅仅只是更新了很少量的几行代码而已。

我看过几篇声称热更新在现实中并不是很有用的文章，他们担心热更新的方式会带来模块版本管理上的混乱。这是可以理解的，人们对于未知的事物，对于超出其知识范围和视野的事物，总会有恐惧心理的。（我相信）10年以后，当有相应的软件管理工具和理论问世以后，热更新的强大之处将会彰显出来。（我只能说）Erlang 是如此的前卫，以致于在她问世 20 多年之后，她的设计哲学仍然领先于当今时代。


## 容错机制

高达 `nine nines`（99.9999999%）的系统可用性是软件工业一直在追求的目标。然后，人总是会犯错的，其创造出来的软件当然也免不了出错。`墨菲法则` 说，任何可能出错的事终将出错。认识到我们终将无法避免错误，那么最重要的问题就是：软件如何健壮地应对错误（并继续正常运作）？

其他的一些语言鼓励程序员使用防御性的编程策略来保证程序不致因为某些错误而奔溃，但 Erlang 的思想则是 “`让它奔溃吧`” ！这是为何？

如果为你建造的房子是这样的，只要某个窗户坏掉了整栋房子就会倒塌，你会住进去吗？答案是显而易见的！你需要的房子应该在这种情况下仍然可以安全居住，稍后叫人来修窗户就行了。

这就是用 C 和 Erlang 编写的程序之间的区别。只要知道如何从错误中恢复，软件出错就不是一个多么严重的问题了。在 Erlang 里：

> * 一个进程的奔溃不会影响到其他不相关的进程
> * 一个进程出错以后在退出前，会向与它关联的进程发送通知（hey，我要退出了！）
> * 可以使用监控者来恢复奔溃的程序，比如重启进程。

## 速度

与 C 语言相比，Erlang 代码在性能上有几点劣势：

> * 代码运行于虚拟机之上
> * 运行时类型推导
> * 模式匹配
> * 其他...

那么 Erlang 到底有多慢？参考[这篇文章][12]，我在自己的mbp上做了如下测试：

```sh
➜  comparison  time ./euler12.bin
842161320
./euler12.bin  5.90s user 0.01s system 99% cpu 5.910 total
➜  comparison  time erl -noshell -s euler12 solve
842161320
erl -noshell -s euler12 solve  11.09s user 0.19s system 100% cpu 11.269 total
➜  comparison  time pypy euler12.py
842161320
pypy euler12.py  9.92s user 0.05s system 96% cpu 10.305 total
➜  comparison  time ./euler12.py
842161320
./euler12.py  66.32s user 0.04s system 99% cpu 1:06.43 total
```

可以看到，C 代码几乎比 Erlang 快了 1 倍。（但是）考虑到 Erlang 带来的好处，这已经是很不错的数据了。想想 （Erlang 所擅长的）I/O 密集型、并发/分布式场景，Erlang 毫无疑问地胜出了。

虽然现在 Erlang 在性能上无法超越 C，但将来单个芯片裸片（DIE）上有成百上千个核的时候，这一天会到来的。

## 应用

有不少著名的软件是基于 Erlang 开发的：

> * [CouchDB][13]. 一个 NoSql 数据库。
> * [RabbitMQ][14]. 一个分布式消息队列系统。
> * [ejabberd][15]. 一个即时通信服务器。
> * [AXD301 ATM switch][16]. 这很可能是世界上唯一一个能够达到 `nine nines` 可用性的系统了，它已经连续正常运行了超过 20 年！
> * 还有很多其他的公司，包括 `Amazon`, `Facebook`, `Yahoo!`, `T-Mobile` 等，都在他们的系统中使用了 Erlang，参考 [Where is Erlang used and why?][17]

----------

# 如何学习 Erlang

Erlang 学习起来确实相当困难，但是一旦你掌握了她，你就是 `the king`。我很欣赏下面这段话，摘自著名 Erlang Web MVC 框架 [Chicago Boss][18] 的作者 Evan Miller 撰写的一篇名为 [Joy of erlang][19] 的文章：

> * In the movie Avatar, there's this big badass bird-brained pterodactyl thing called a Toruk that the main character must learn to ride in order to regain the trust of the blue people. As a general rule, Toruks do not like to be ridden, but if you fight one, subdue it, and then link your Blue Man ponytail to the Toruk's ptero-tail, you get to own the thing for life. Owning a Toruk is awesome; it's like owning a flying car you can control with your mind, which comes in handy when battling large chemical companies, impressing future colleagues, or delivering a pizza. But learning to ride a Toruk is dangerous, and very few people succeed.

![阿凡达][20]

这段话生动地反映了我学习 Erlang 的困难程度。顺便说一句，我距离攻克她还远着呢～～～

在学习 Erlang 之前，我熟练掌握了 C、Python 和 Javascript，对 C++、Java 和 Ruby 也有所涉略。我的思维也是早已习惯了命令式编程，因此最大的挑战就是要去适应函数式编程，这需要做出思维上巨大的转变。

## 变量不变

变量只能绑定某个值，而无法更改，这让我突然间不会编程了！
举个例子，要求在不使用任何辅助函数（例如map）的情况下实现 upper(str) 函数。

在 Python 中这简直太简单且直接了：
```py
def upper(str):
    str1 = ''
    for c in str:
        str1 += c.upper()
    return str1
```

但是在 Erlang 中，无法改变变量的值，该如何实现呢？
诀窍在于一个累积器的概念，下面这个范例，请在以后编写 Erlang 程序的时候时刻记着。代码看起来是这样的：

```erlang
upper(S) ->
    upper(S, []).

upper([], N) ->
    lists:reverse(N);
upper([H|T], N) when H > $a, H =< $z ->
    upper(T, [H + ($A - $a)|N]);
upper([H|T], N) ->
    upper(T, [H|N]).
```

这个函数本身不包含任何内部变量的状态改变，但我们使用一个通过参数传入的累积器，实现了同样的功能。这是函数式编程里面的一个很基础的范例，理解并记住它，要么请放弃 Erlang。

## 模式匹配

模式匹配使得 Erlang 程序看起来既优雅又易于理解。

在 C 语言中，一个函数在它的作用域中必须是唯一的，你不能定义了一个函数 `fun(x)`，再定义一个函数 `fun(x,y)`，但在 Erlang 中就没有这个限制了。你可以随意定义多个（同名）函数，只要它们的参数是不一样的，你也可以为一个函数定义多个子句，只要它们匹配的模式不同。

仍旧以上面的 `upper()` 函数为例，调用 `upper("#hello")` 时:

> * 由于只有一个参数，该调用匹配 `upper(S)`，因此接着调用 `upper("#hello", [])`
> * `"#hello"` 不匹配 `[]`，并且第一个字符  `"#"` 不匹配 guard 条件，故调用 `upper("#hello", [])` 匹配 `upper([H|T], N)`，因此接着调用 `upper("hello", [$#])`
> * 接下来的各个调用匹配 `upper([H|T], N) when H > $a, H =< $z`，每个调用将一个字符转换为大写，调用序列为：
```erlang
upper("ello", [$H, $#])
upper("llo", [$E, $H, $#])
upper("lo", [$L, $E, $H, $#])
upper("o", [$L, $L, $E, $H, $#])
upper([], [$O, $L, $L, $E, $H, $#])
```
> * 最后，`upper([], [$O, $L, $L, $E, $H, $#])` 匹配 `upper([], N), lists:reverse(N)`

模式匹配让你能够将代码逻辑进行分拆，这使得函数的每个子句变得更小、更具可读性。这就是为什么我们经常可以看到 C 函数动辄超过 100 行代码，而 Erlang 函数通常不会超过 20 行的原因。

## 递归魔法

学习 C 编程的时候，我被告知 `递归是有害的`，应当尽可能避免使用它。但在函数式编程的世界里，完全就是另外一回事了。

回头看看上文的所有代码，你有看到循环语句吗？没有。递归函数呢？到处都是。在需要迭代或递归的场景里，递归取代了 `for/while` 循环，代码看起来非常简洁。为什么会是这样呢？回想一下 C 语言的 `for` 循环，其实包含了状态改变，每次循环里修改迭代器的值，直到它满足了退出的条件，跳出循环。但 Erlang 不允许内部状态的改变，所以她使用递归取代了通常的 `for/while` 循环。

但是，对此你心有疑虑...

### 递归的性能很差

这其实是一个 `伪命题`。

让我举个例子。我们通常认为，通过寄存器传递参数要远比通过堆栈传递快得多，函数调用中，Intel CPU 由于缺少足够的通用寄存器（相比较RISC CPU），因此它非常依赖堆栈的 `push/pop` 操作。为了提高性能它引入了寄存器栈，这让 `push/pop` 操作变得和操作寄存器一样快。

让我们看回递归，我们在 C 中很少使用递归，并且根据 `80/20` 法则（什么意思？），没有必要马上对递归进行优化，对吧？而对 Erlang 而言，递归就是代码的生命，编译器岂敢不对其进行优化！所以说吧，你的直觉是不对的 —— 递归并非天生就是很慢的，在 C 里面它确实慢，但在 Erlang 里，它非常快。

### 递归很容易导致堆栈溢出

这个论断也并不十分准确，如果你以正确的方式编写程序的话。在递归中 Erlang 对堆栈操作做了优化，尤其是尾递归。举例说明：

```erlang
fac(1) -> 1;
fac(N) when N > 0 -> N * fac(N-1).
```

这不是一个尾递归，因为堆栈数据必须保存下来做进一步计算。让我们把它改成尾递归的版本：

```erlang
fac1(N) -> fac1(N, 1).

fac1(1, M) -> M;
fac1(N, M) when N > 0 -> fac1(N-1, N * M).
```

每进行一次递归调用，我们都能够安全地把上一次递归调用产生的堆栈数据丢弃，这就是尾递归。尾递归的好处是，你只需要保存少量并且固定大小的内存数据，所以不需要担心长时间运行的递归程序会有堆栈溢出的问题。

在 Erlang 里你可以尽可能地使用尾递归编程。

## 函数式编程

特别关注一下那些可以被递归访问的数据，例如 list（包括string），list 可以做如下处理：

```erlang
retrieve([H|T]) -> {H, T}.

%% or even
retrieve1([H1, H2|T]) -> {H1, H2, T}
```

请牢记 map/reduce 的概念。

对于算法问题，想一想如何通过数学公式来实现它，如果落地到了具体的数学公式了，那么这样的问题应该很容易用 Erlang 来编程解决。分而治之，不要被细节问题所迷惑。

我们用 atoi(S) 函数为例，公式如下：

```sh
         |- S[0] is '-':  -1 * atoi(S[1:], 0)
atoi(S) -+
         |- else:         atoi(S, 0)

              |- S is []:       Acc
atoi(S, Acc) -+- S[X] is digit: atoi(S[X+1:], 10 * Acc + digit(S))
              |- else:          Acc
```

实现代码例子如下:

```erlang
-module(math).
-export([atoi/1]).

atoi([$-|S]) ->
    -1 * atoi(S, 0);
atoi(S) ->
    atoi(S, 0).

atoi([], Acc) -> Acc;
atoi([H|T], Acc) when H >= $0, H =< $9 ->
    atoi(T, 10 * Acc + (H - $0));
atoi(_S, Acc) -> Acc.
```

比较一下 atoi() 函数的 C 实现，Erlang 的实现版本更直观反映了其算法逻辑，你几乎能一次就编写出正确的代码。

## 拥抱进程

进程使用起来并不难，它以一种结构相对松散的、异步的方式组织起你的系统。你只需要去习惯它，将它当作一个 worker，一个 object，或者其他类似的东西。

## 让它出错吧

C 程序员通常需要在编程时考虑覆盖尽可能多的参数值条件，而 Erlang 程序员则不需要如此，看这个例子：

```erlang
%% good code
fac(N) when is_integer(N), N > 0 -> fac(N, 1).

fac(1, Acc) -> Acc;
fac(N, Acc) -> fac(N-1, Acc * N).

%% bad code
fac(N) when is_integer(N), N > 0 -> fac(N, 1).
fac(N)                           -> {error, "argument must be positive integer"}

fac(1, Acc) -> Acc;
fac(N, Acc) -> fac(N-1, Acc * N).
```

用非法参数调用第一段代码将会导致异常：

```sh
1> c(math).
{ok,math}
2> math:fac(10).
3628800
3> math:fac(-10).
** exception error: no function clause matching math:fac(-10) (math.erl, line 14)
```

我们需要为负数的情况编写代码吗？C 程序需要，Erlang 不需要！用负数调用时，第二段代码使得程序得以继续运行，但是给调用者增加了不必要的处理，调用者需要为此添加额外的代码，给程序徒增复杂性。还是直接让进程退出吧，我们只在真正需要异常处理的场景里为此操心就行了。

----------

# Erlang 学习资料

恭喜你！读完了这篇文章，你将可以开始学习驯服属于你的那只 Toruk了，下面是对你很有帮助的一些 `武器`：

> * **Joe Armstrong** 的经典之作 《 **Programming Erlang - software for a concurrent world** 》。学习一门语言最好的途径是阅读该门语言作者的作品，我力荐你好好读一读这本书，尤其是里面有关 `顺序编程`、`并发编程` 和 `OTP` 的章节。
> * [Erlang doc][21]。有任何疑问时作为参考。
> * 用 Erlang 编程解决你日常工作中遇到的问题，以下例子可供参考：
  * 给定一个文件，计算经典的 wordcount
  * 实现 cloc (给定一个目录，计算不同语言的代码行数)
  * 编写一个消息服务器，将 topN 未读消息保存在内存，其余的保存在 mnesia 中
  * 编写一个 markdown 格式解析器
  * 基于 mochiweb 编写一个 web 框架
  * 编写一个 L3/L4 防火墙软件用于处理 handles symmetric NAT translation (to focus on the problem itself you should use the data from tcpdump).
> * 阅读开源项目代码，例如 ranch, cowboy, boss_db, 等等。

Hope you have fun!

----------


  [1]: http://tchen.me/posts/2013-07-22-why-should-c-programmers-learn-erlang.html
  [2]: http://www.erlang.org/
  [3]: http://tchen.me/assets/files/snapshots/institutionalized.jpg
  [4]: http://en.wikipedia.org/wiki/Prolog
  [5]: http://en.wikipedia.org/wiki/Functional_programming
  [6]: http://stackoverflow.com/questions/10221907/garbage-collection-and-memory-management-in-erlang
  [7]: http://en.wikipedia.org/wiki/Actor_model
  [8]: http://en.wikipedia.org/wiki/Real-time_computing#Soft
  [9]: http://stackoverflow.com/questions/2708033/technically-why-are-processes-in-erlang-more-efficient-than-os-threads
  [10]: http://jlouisramblings.blogspot.com/2013/01/how-Erlang-does-scheduling.html
  [11]: http://golang.org/
  [12]: http://stackoverflow.com/questions/6964392/speed-comparison-with-project-euler-c-vs-python-vs-Erlang-vs-haskell
  [13]: http://couchdb.apache.org/
  [14]: http://www.rabbitmq.com/
  [15]: http://www.ejabberd.im/
  [16]: http://www.adelcogroup.com/EricssonAXD301.htm
  [17]: http://stackoverflow.com/questions/1636455/where-is-Erlang-used-and-why
  [18]: http://www.chicagoboss.org/ng-used-and-why
  [19]: http://www.evanmiller.org/joy-of-erlang.html
  [20]: http://tchen.me/assets/files/snapshots/toruk.jpg
  [21]: http://www.erlang.org/doc/
