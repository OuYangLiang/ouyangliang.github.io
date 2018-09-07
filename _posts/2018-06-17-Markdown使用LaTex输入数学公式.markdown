---
layout: post
title:  "Markdown使用LaTex输入数学公式"
date:   2018-06-17 08:19:16 +0800
categories: 数学
keywords: latex
description: Markdown使用LaTex输入数学公式
commentId: 2018-06-17
---

## 插入数学公式的方法

<kbd>行内公式</kbd>

`$数学公式$`，例如`$\sum_{i=1}^n a_i=0$`，输出为：$\sum_{i=1}^n a_i=0$

<kbd>块级公式</kbd>

`$$数学公式$$`，例如`$$\sum_{i=1}^n a_i=0$$`，输出为：

$$\sum_{i=1}^n a_i=0$$

## 上标和下标

使用^来表示上标，_来表示下标，同时如果上下标的内容多于一个字符，可以使用{}来将这些内容括起来当做一个整体。与此同时，上下标是可以嵌套的。

例如：`$$x = a_{1}^n + a_{2}^n + a_{3}^n$$`，输出为：

$$x = a_{1}^n + a_{2}^n + a_{3}^n$$

## 括号

`()`，`[]`和`|`都表示它们自己，但是`{}`因为有特殊作用因此当需要显示大括号时一般使用\lbrace和\rbrace来表示。

例如：`$$f(x, y) = 100 * \lbrace[(x + y) * 3] - 5\rbrace$$`，输出为：

$$f(x, y) = 100 * \lbrace[(x + y) * 3] - 5\rbrace$$

## 省略号

数学公式中常见的省略号有两种，`\ldots`表示与文本底线对齐的省略号，`\cdots`表示与文本中线对齐的省略号。

例如：`$$f(x_1,x_2,\ldots,x_n) = x_1^2 + x_2^2 + \cdots + x_n^2$$`，输出为：

$$f(x_1,x_2,\ldots,x_n) = x_1^2 + x_2^2 + \cdots + x_n^2$$

## 分数

分数使用`\frac{分子}{分母}`这样的语法，不过推荐使用`\cfrac`来代替`\frac`，显示公式不会太挤。

例如：`$$\cfrac{1}{4} + \cfrac{1}{2} = \cfrac{3}{4}$$`，输出为：

$$\cfrac{1}{4} + \cfrac{1}{2} = \cfrac{3}{4}$$

## 开方

开方使用`\sqrt[次数]{被开方数}`这样的语法。

例如：`$$\sqrt[3]{27} = 3$$`，输出为：

$$\sqrt[3]{27} = 3$$

## 间隔空间

MathJax通过内部策略自己管理公式内部的空间，因此a︹︹b与a︹︹︹︹︹b（︹表示空格）都会显示为ab。可以通过在ab间加入\空格或\;增加些许间隙，\quad 与 \qquad 会增加更大的间隙。

例如：`$a\;b$`或`$a\quad b$`或`$a\qquad b$`，输出为：

$$a\;b，a\quad b，a\qquad b$$

$$ 矢量表示

矢量用`\vect`标记实现，语法格式如下：`\vec{矢量值}`

例如：`$$\vec{a} \cdot \vec{b}=0$$`，输出为：

$$\vec{a} \cdot \vec{b}=0$$

## 连线符号

`$$\overline{a+b+c+d}$$`，输出为：

$$\overline{a+b+c+d}$$

`$$\underline{a+b+c+d}$$`，输出为：

$$\underline{a+b+c+d}$$

`$$\overbrace{a+\underbrace{b+c}_{1.0}+d}^{2.0}$$`，输出为：

$$\overbrace{a+\underbrace{b+c}_{1.0}+d}^{2.0}$$

## 希腊字母

<div class="row">
<div class="col-sm-10">
<table class="table table-bordered table-condensed table-striped text-center">
<tr class="info"><td>代码</td><td>大写</td><td>代码</td><td>小写</td></tr>
<tr><td>A</td><td>$A$</td><td>\alpha</td><td>$\alpha$</td></tr>
<tr><td>B</td><td>$B$</td><td>\beta</td><td>$\beta$</td></tr>
<tr><td>\Gamma</td><td>$\Gamma$</td><td>\gamma</td><td>$\gamma$</td></tr>
<tr><td>\Delta</td><td>$\Delta$</td><td>\delta</td><td>$\delta$</td></tr>
<tr><td>E</td><td>$E$</td><td>\epsilon</td><td>$\epsilon$</td></tr>
<tr><td>Z</td><td>$Z$</td><td>\zeta</td><td>$\zeta$</td></tr>
<tr><td>H</td><td>$H$</td><td>\eta</td><td>$\eta$</td></tr>
<tr><td>\Theta</td><td>$\Theta$</td><td>\theta</td><td>$\theta$</td></tr>
<tr><td>I</td><td>$I$</td><td>\iota</td><td>$\iota$</td></tr>
<tr><td>K</td><td>$K$</td><td>\kappa</td><td>$\kappa$</td></tr>
<tr><td>\Lambda</td><td>$\Lambda$</td><td>\lambda</td><td>$\lambda$</td></tr>
<tr><td>M</td><td>$M$</td><td>\mu</td><td>$\mu$</td></tr>
<tr><td>N</td><td>$N$</td><td>\nu</td><td>$\nu$</td></tr>
<tr><td>\Xi</td><td>$\Xi$</td><td>\xi</td><td>$\xi$</td></tr>
<tr><td>O</td><td>$O$</td><td>\omicron</td><td>$\omicron$</td></tr>
<tr><td>\Pi</td><td>$\Pi$</td><td>\pi</td><td>$\pi$</td></tr>
<tr><td>\Sigma</td><td>$\Sigma$</td><td>\sigma</td><td>$\sigma$</td></tr>
<tr><td>T</td><td>$T$</td><td>\tau</td><td>$\tau$</td></tr>
<tr><td>\Upsilon</td><td>$\Upsilon$</td><td>\upsilon</td><td>$\upsilon$</td></tr>
<tr><td>\Phi</td><td>$\Phi$</td><td>\phi</td><td>$\phi$</td></tr>
<tr><td>X</td><td>$X$</td><td>\chi</td><td>$\chi$</td></tr>
<tr><td>\Psi</td><td>$\Psi$</td><td>\psi</td><td>$\psi$</td></tr>
<tr><td>\Omega</td><td>$\Omega$</td><td>\omega</td><td>$\omega$</td></tr>
</table>
</div>
</div>

## 关系运算符

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-center">
<tr class="info"><td>代码</td><td>符号</td></tr>
<tr><td>\pm</td><td>$\pm$</td></tr>
<tr><td>\times</td><td>$\times$</td></tr>
<tr><td>\div</td><td>$\div$</td></tr>
<tr><td>\mid</td><td>$\mid$</td></tr>
<tr><td>\nmid</td><td>$\nmid$</td></tr>
<tr><td>\cdot</td><td>$\cdot$</td></tr>
<tr><td>\circ</td><td>$\circ$</td></tr>
<tr><td>\ast</td><td>$\ast$</td></tr>
<tr><td>\bigodot</td><td>$\bigodot$</td></tr>
<tr><td>\bigotimes</td><td>$\bigotimes$</td></tr>
<tr><td>\bigoplus</td><td>$\bigoplus$</td></tr>
<tr><td>\leq</td><td>$\leq$</td></tr>
<tr><td>\geq</td><td>$\geq$</td></tr>
<tr><td>\neq</td><td>$\neq$</td></tr>
<tr><td>\approx</td><td>$\approx$</td></tr>
<tr><td>\equiv</td><td>$\equiv$</td></tr>
<tr><td>\sum</td><td>$\sum$</td></tr>
<tr><td>\prod</td><td>$\prod$</td></tr>
<tr><td>\coprod</td><td>$\coprod$</td></tr>
</table>
</div>
</div>

## 集合运算符

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-center">
<tr class="info"><td>代码</td><td>符号</td></tr>
<tr><td>\emptyset</td><td>$\emptyset$</td></tr>
<tr><td>\in</td><td>$\in$</td></tr>
<tr><td>\notin</td><td>$\notin$</td></tr>
<tr><td>\subset</td><td>$\subset$</td></tr>
<tr><td>\supset</td><td>$\supset$</td></tr>
<tr><td>\subseteq</td><td>$\subseteq$</td></tr>
<tr><td>\supseteq</td><td>$\supseteq$</td></tr>
<tr><td>\bigcap</td><td>$\bigcap$</td></tr>
<tr><td>\bigcup</td><td>$\bigcup$</td></tr>
<tr><td>\bigvee</td><td>$\bigvee$</td></tr>
<tr><td>\bigwedge</td><td>$\bigwedge$</td></tr>
<tr><td>\biguplus</td><td>$\biguplus$</td></tr>
<tr><td>\bigsqcup</td><td>$\bigsqcup$</td></tr>
</table>
</div>
</div>

## 对数运算符

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-center">
<tr class="info"><td>代码</td><td>符号</td></tr>
<tr><td>\log</td><td>$\log$</td></tr>
<tr><td>\lg</td><td>$\lg$</td></tr>
<tr><td>\ln</td><td>$\ln$</td></tr>
</table>
</div>
</div>

## 三角运算符

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-center">
<tr class="info"><td>代码</td><td>符号</td></tr>
<tr><td>\bot</td><td>$\bot$</td></tr>
<tr><td>\angle</td><td>$\angle$</td></tr>
<tr><td>\sin</td><td>$\sin$</td></tr>
<tr><td>\cos</td><td>$\cos$</td></tr>
<tr><td>\tan</td><td>$\tan$</td></tr>
<tr><td>\cot</td><td>$\cot$</td></tr>
<tr><td>\sec</td><td>$\sec$</td></tr>
<tr><td>\csc</td><td>$\csc$</td></tr>
</table>
</div>
</div>

## 微积分运算符

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-center">
<tr class="info"><td>代码</td><td>符号</td></tr>
<tr><td>\prime</td><td>$\prime$</td></tr>
<tr><td>\int</td><td>$\int$</td></tr>
<tr><td>\iint</td><td>$\iint$</td></tr>
<tr><td>\iiint</td><td>$\iiint$</td></tr>
<tr><td>\iiiint</td><td>$\iiiint$</td></tr>
<tr><td>\oint</td><td>$\oint$</td></tr>
<tr><td>\lim</td><td>$\lim$</td></tr>
<tr><td>\infty</td><td>$\infty$</td></tr>
<tr><td>\nabla</td><td>$\nabla$</td></tr>
<tr><td>\mathrm{d}</td><td>$\mathrm{d}$</td></tr>
</table>
</div>
</div>

## 箭头符号

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-center">
<tr class="info"><td>代码</td><td>符号</td></tr>
<tr><td>\uparrow</td><td>$\uparrow$</td></tr>
<tr><td>\downarrow</td><td>$\downarrow$</td></tr>
<tr><td>\Uparrow</td><td>$\Uparrow$</td></tr>
<tr><td>\Downarrow</td><td>$\Downarrow$</td></tr>
<tr><td>\rightarrow</td><td>$\rightarrow$</td></tr>
<tr><td>\leftarrow</td><td>$\leftarrow$</td></tr>
<tr><td>\Rightarrow</td><td>$\Rightarrow$</td></tr>
<tr><td>\Leftarrow</td><td>$\Leftarrow$</td></tr>
<tr><td>\longrightarrow</td><td>$\longrightarrow$</td></tr>
<tr><td>\longleftarrow</td><td>$\longleftarrow$</td></tr>
<tr><td>\Longrightarrow</td><td>$\Longrightarrow$</td></tr>
<tr><td>\Longleftarrow</td><td>$\Longleftarrow$</td></tr>
</table>
</div>
</div>

## 几个示例

`$$\sum_{a=0}^n \frac{1}{a^2}$$`，输出为：

$$\sum_{a=0}^n \frac{1}{a^2}$$

`$$\int_0^1 x^2 {\rm d}x$$`，输出为：

$$\int_0^1 x^2 {\rm d}x$$

`$$\lim_{n\rightarrow\infty} \frac{n+(-1)^{n-1}}{n} = 1$$`，输出为：

$$\lim_{n\rightarrow\infty} \frac{n+(-1)^{n-1}}{n} = 1$$
