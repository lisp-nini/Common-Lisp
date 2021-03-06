转载请说明来自Lisp中文社区官方群1群（39128390），小妮窝窝（307730136）。

这一篇作为对(1)的补充。

#CL的宏可以看成是阉割版的C宏#

今天提到了C的宏和CL的macro（下边都是指defmacro所定义的宏）的比较。

实际上，C的宏只做无脑的文本替换，这是它相比CL的macro的优势。如果你见过gcc 4.0的源代码，你会感叹C的宏原来可以这么“强大”（或者，“还可以这样子用”），而正因为C的宏使用了无脑的替换，才使得你拥有如此强大的能力————你完全可以让C的宏生成一堆乱七八糟的文本而不遵循语法，然后，这些乱七八糟的文本里面还能进一步的嵌套宏，这些宏进一步处理这些乱七八糟的文本，直到最终生成的文本，都可以是不符合语法的东西。相比之下，CL的宏，每一次调用的返回，都必须返回一个完整的符合语法的代码（即使是仅仅作替换用的symbol macro，也是如此），从这个意义上来说，这相当于一个阉割版的C宏。 

除了上面所说的，还有一点事实是大家需要知道的。虽然CL世界鼓励你用宏做DSL，但是除非这个DSL仅仅使用C宏就能简单的做出来，否则一般不会有人去做，就算做了，也往往做的十分垃圾。CL提供了标准的“`”、“,”、“,@”等操作符，让你做简单的替换（类似文本替换，但是替换的内容实际上是一个对象，比方说，一个符号，或者一个数字什么的；不过既然是对象，那么内容就不可以是残缺不全的乱七八糟的文本了），这些操作符经常被用于defmacro中，用于实现类似C宏参数的功能。CL在对宏的简单替换方面，确实提供了这些便利（虽然其实好少好少），所以，实现（受限的）C风格的这样子的宏，跟在C里面实现一样简单（因此到哪里都能看到这种风格的宏）。

但是，如果做大一点呢？比方说，做个需要做parse的（比如说，实现标准库里面的loop宏那样子的有自己一套专用语法东西）。我们说过，CL几乎不对parse提供任何便利。注意我用了“几乎”，因为其实还是有的，但是很多人其实并不知晓。实际上 ，defmacro可以在参数定义中指定括号可以怎样套（无脑的哦，不要想得太高级。参数定义的语法在CL里面称为lambda list，CL一共有10套不同的lambda list分别对应不同的场合，defmacro就有自己专门一套），括号里面可以有几个内容。因此，defmacro也能很容易实现这种风格语法的宏（我用了“语句”是为了让不懂CL的人看懂，你们就别吐槽了）：

```
(with-output-to-shit (Var) 语句1 语句2 ... 语句n) =>  按顺序执行所有的语句，语句中可以使用变量名为Var的变量向Var输出内容，最后整个with-output-to-shit返回结束时Var里面d所有内容。
```

这里的(Var)，是固定的无脑的东西，它说明了这里要套一个括号，因而可以使用defmacro所提供的便利（lambda list）来定义；后面的一大串语句，则可以使用&rest（其实就是varargs）在定义参数的时候做声明，一下子把所有后面的内容一次性拿到。所以CL里面，这种风格的宏也很常见，因为实现起来很容易。而这种宏，使用#define也一样很容易实现。

如果语法再稍微复杂一点呢？这时候就麻烦了，因为CL几乎不为此提供便利，你必须自己去parse。如果你的这门DSL只parse当前reader（我说过，这是个lexer）返回的内容，那你只需要macro就行了。如果你还想修改CL的文法，你就要使用reader macro，实现自己的lexer。这一切都是手动的，没有任何工具帮助你（那种你只要描写文法就给你生成解析程序的功能是没有的，不要想了），你只能从零做起。所以，讲CL高级编程的书（这些书是不会告诉你CL背后真正复杂的那些特性的）都在讲如何使用宏去实现一个DSL，你完全可以认为就是一本简要的编译器入门教材。CL这种“提供了个hook就跑”的特点，解释了几个问题（下面的每一项，其实从0做起都是蛮困难的，做过compiler的童鞋应该都明白）：

* CL世界里面，真正用宏实现一个需要自己解析语法的DSL，很少。大多数宏都是C风格的宏能实现的。
* 就算是有，往往语法也不会太过复杂。
* 这些DSL的实现的解析能力通常都很弱，大部分几乎都不对语法错误做相应的处理。偶尔会遇到一两个良心的，会对极少数（真的是“极少数”）语法错误做一小点（真的只是“一小点”）做出错提示。这些实现对所输入的代码的解析往往都仅仅建立在假设代码是完全正确的基础上，所以你经常会遇到这样的问题：
 - 语法错误的代码被当成正确的解析，然后生成错误的代码，这些错误的代码只有在运行时才表现出来；
 - 解析时解析程序遇到错误的输入（也就是你的代码），由于没有对错误输入做处理，解析程序直接挂掉。这时候往往是一个未处理的异常（我喜欢说“异常”，如果我说condition没有人听得懂）被抛出，然后程序停下来，然后你会看到挂掉时候的backtrace以及异常信息。这时候，除非这个解析程序是你自己写的或者你读过源代码，否则，面对这一堆陌生的backtrace，你根本不知道你输入的代码到底错在哪里。当然，如果很幸运没有挂掉，那么就是上面说的“语法错误的代码被当成正确的解析”那种情况了。
* 这个DSL的实现根本不会对生成的代码加上任何的调试信息，如果出错了，你往往只能看到一大堆莫名其妙的代码和符号。
 - 他们擅长使用临时的无意义的符号，比方说#:G1234，所以这时候你看到的出错信息里面只有一堆莫名其妙的符号，根本不知道对应的源代码是什么，也不知道如何修改。
 - DSL的实现所直接生成的代码，在执行前就已经消失了，所以你看到的backtrace也是一堆莫名其妙的函数调用。这时候，对于低级点的，尽管可以用macroexpand等函数让宏再展开一次来查看生成什么代码（生成的过程，也就是生成的理由，你只能直接去看源代码，因为文档也不会告诉你这个事情），但由于使用的临时符号基本上都是调用gensym这个函数来产生的全局唯一的临时符号，所以每次的临时符号都不同，你不可能将对应的临时符号和出错时的临时符号直接对上号，并且生成的代码往往排版相当糟糕而且无任何注释。而对于高级点的宏，它还要根据当前的环境来生成宏，这时候你根本营造不出生成时的环境，所以如果还是使用macroexpand等函数，那么将产生错误的代码。对于这种高级点的宏，你只能让代码在生成的时候加入足够的调试信息，并使用调试器来查看和调试这些生成的代码。但是，各家CL实现在这方面（调试信息、调试器）都很弱，所以基本上是不可行的。当然，lispworks也许会是个例外（他是唯一一个提供较为稳定且界面接近现代调试器的主流CL实现，尽管功能和界面也比较弱），但你也不要期望说调试一段意义不明的毫无注释的生成后的代码会有多么容易（注：CL标准库里面有一大堆的宏，而且都相当的常用，所以就算你不用第三方的任何东西，如果你用这样子的方法去调试，你也要经常面对这些意义不明毫无注释的代码）。
* 主流CL实现里面，标准库的宏，几乎都存在和上面所说的“解析能力通常都很弱”那样子的问题（明年CL就30岁了，这些CL实现有的从70年代做到现在了）。所以当你们看到做g2这等“神器”开发的媛媛童鞋在吐槽g2里面有连异常处理的语法都写错并顺利通过编译的老代码的时候，你们要明白问题出在哪里（他们用的是lispworks）。

总之一句话，尽管CL世界鼓励用CL的宏做DSL，但CL的宏最经常用在C宏类似的场合，起到类似的作用，而且还不能返回乱七八糟的内容。而且，用CL的宏做DSL确实蛮辛苦的。

#FAQ#

Q: _我想问问Common Lisp能不能不用宏？_

A: 可以，但是由于标准库里面大量的使用了宏，而且是在最常用的地方（defun, defvar, defparamter, defmacro等等等等全是宏哦），所以我相信你不会愿意完全不使用宏。当然，你不自己定义宏，完全没问题。

Q: _相对于使用宏，为什么不直接用函数返回个代码段？_

A: 宏本身就是个返回代码段的函数。简单的说，defmacro帮你做了这样子的事情：定义一个“返回代码段的函数”，这个函数和普通函数的区别仅仅在于其调用接口是定死的；然后，将这个函数安装到对应的hook。
 
2013-08-25 by 小妮
 

