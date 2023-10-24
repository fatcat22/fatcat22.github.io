---
layout: post
title:  "plonk 中的批量多项式承诺"
categories: zkp
tags: 原创 zkp polynomial-commitment batched-kzg 
author: fatcat22
mathjax: true
---

* content
{:toc}




在前面的文章时，我们介绍了 kzg 多项式承诺（如果您对 kzg 承诺还不熟悉，建议先读[介绍 kzg 的文章](https://yangzhe.me/2023/10/20/kzg/)，再读这篇）。kzg 承诺是一种证明多项式的方案，但它一次只能证明一个多项式。而在 plonk 论文里根据算法的需要，对多项式承诺作了扩展，可以让 plonk 算法一次证明多个多项式。这篇文章我们就来详细了解一下 plonk 论文里的这个批量多项式承诺（A batched version of the KZG scheme）。

> 虽然叫做 `batched`，但数量越多越复杂。 plonk 算法里只需要证明两个多项式，所以论文里也只介绍了两个多项式时的承诺方法，这里我们也以两个多项式为例进行介绍。

在开始之前我们先明确一下符号，所谓「批量多项式承诺」（batched version of the KZG scheme），就是要一次性证明 $t$ （$t \ge 2$ ）个多项式 $f_1(x),...,f_t(x)$，分别在 $t$ 个不同的点（$z_1, ..., z_t$）的取值是正确的。「一次性证明」，就是指通过一个双线性配对等式就可以证明，不需要多个。

# 1. 单个点上的承诺
我们首先看一个简单的情况，即所有 $t$ 个点都相同的情况：$z_1 = ... = z_t = z$ 。

> 下面的介绍中有些符号与 plonk 论文中的不太一样，比如加密值我们用 $s$ 表示，而论文里用 $x$ 表示。因为我们在之前一系列文章中一直这么用，已经习惯了，所以这里就不再为了跟论文一样而改动了。

与之前文章介绍的普通 kzg 一样，我们要先通过 trusted setup 生成加密值 $s$ ，然后承诺者就可以计算得到每个多项式的承诺值 $cm_i = [f_i(s)]_1$ 。

同样与普通 kzg 类似，批量承诺也要计算商多项式，只不过不同的是，这里的商多项式是累加到一起的一个多项式。在普通 kzg 里，对于每个多项式 $f_i(x)$，它的商多项式是 $q_i(x) = \frac{f_i(x)-f_i(z)}{x-z}$，其中 $f_i(z)$ 就是之前我们介绍 kzg 的文章里的 $y$ ；但在这里的批量承诺中，这些商多项式是累加起来的，如下：

$$
h(x) = \sum\limits_{i=1}^t{\gamma}^{i-1}\frac{f_i(x)-f_i(z)}{x-z}
$$

其中 $\gamma$ 是一个随机数，目的是为了增强「零知识性」，让验证者更难地猜出原多项式是什么。如果用我们前面介绍普通 kzg 文章里的表示方式，单个多项式的商多项式表示成 $q(x) = \frac{f(x)-y}{x-z}$ ，那么 $h(x)$ 可以写成：$h(x) = \sum\limits_{i=1}^t{\gamma}^{i-1}q_i(x)$ 。

然后承诺者就可以给出 $W = [h(s)]_1$ 。（这里用 $W$ 表示商多项式在加密点 $s$ 上椭圆曲线上的值，是 plonk 论文是的表示方式）

那验证者如何验证呢？还记得普通 kzg 算法中，验证等式为：$e([q(s)]_1, [s-z]_2) = e([f(s)]_1-[y]_1, H)$ 。那在当前的批量版本中，$[q(s)]_1$ 需要换成 $W$ ；并且既然 $W$ 是多个商多项式组合起来的，那么普通版本中的 $[f(s)]_1$ 和 $[y]_1$ 也需要使用相应的组合方式进行组合。其中 $[f(s)]_1$ 是承诺，$[y]_1$ 是多项式在 $z$ 点的值，所以承诺应该替换成：

$$F = \sum\limits_{i=1}^t{\gamma}^{i-1}cm_i$$  

> 前面我们提到过，$cm_i = [f_i(s)]_1$ 

而普通 kzg 版本中的 $[y]_1$ 应该替换为：

$$v = \sum\limits_{i=1}^t{\gamma}^{i-1}[f_i(z)]_1$$

所以最终普通 kzg 版本里的 $[f(s)]_1 - [y]_1$ 应该被替换为：

$$F - v$$

最终，验证者使用如下等式进行验证：

$$e(W, [s-z]_2) = e(F - v, H)$$

以上就是批量承诺的单点版本。可以看到，所有符号的意义、包括验证等式的形式，与普通 kzg 版本完全一样，只不过商多项式、承诺、$y$ 这三个在椭圆曲线上的值，都使用了累加版本的 $W$ 、$F$ 和 $v$ 进行了替换。**所以其核心思想就是每个多项式的 $[q(s)]_1$ 、$C$ 、$y$ 分别进行累加，得到多个多项式的各个值聚合在一起的 $W$、 $F$、 $v$ ，然后使用相同形式的等式进行验证**。

> 在 plonk 论文中，$s_i$ 代表 $[f_i(z)]_1$ ，而 $x$ 则和我们文章里的 $s$ 意义一样。另外论文中的验证等式为 $e(F-v, [1]_2)\cdot e(-W, [s-z]_2)=1$ ，这与我们给出的意义完全一样。

# 2. 两个点上的承诺
接下来我们介绍复杂一些的、也是 plonk 证明中用到的，两个点上的承诺。所谓两个点上的承诺，严格来说，是指有两个点 $z$ 和 $z'$ ，然后要证明拥有 $t_1$ 个多项式 $$f_i(x)_{i\in[t_1]}$$ ，在 $z$ 点的值分别是 $f_i(z)$ ；并且拥有另外 $t_2$ 个多项式 $$f'_i(x)_{i\in[t_2]}$$ ，在 $z'$ 点的值分别是 $f'_i(z')$。

前面介绍的只有一个点 $z$ 时的思路主要是将每个多项式的各个不同意义的值分别累加，最后使用形式上与普通 kzg 完全一样的验证等式进行验证。那么这里扩展成两个点的时候，我们依然也可以使用这个思路，将每个点单独构造 $W$ 、$F$ 和 $v$ 值，然后将它们累加到一起，最后的验证等式依然和普通 kzg 的形式一样。

具体来说，我们先计算经过 $z$ 点的 $t_1$ 个多项式的商多项式的累加，这与前面介绍的单个点时是一样的：

$$
h(x) = \sum\limits_{i=1}^{t_1}{\gamma}^{i-1}\frac{f_i(x)-f_i(z)}{x-z}
$$

类似的，再计算经过 $z'$ 点的 $t_2$ 个多项式的商多项式的累加：

$$
h'(x) = \sum\limits_{i=1}^{t_1}{\gamma'}^{i-1}\frac{f'_i(x)-f'_i(z')}{x-z'}
$$

注意在计算 $h'(x)$ 时，随机数也重新选了一个，记为 $\gamma'$ 。

最终承诺者可以给出 $W = [h(s)]_1$ ，$W' = [h'(s)]_1$。

然后验证者要如何验证呢？与单个点的版本类似，验证者先计算经过 $z$ 点的 $t_1$ 个多项式的承诺 $F$，及在 $z$ 点的取值的累加 $v$ ：

$$\begin{gather}
F = \sum\limits_{i=1}^{t_1}cm_i \\
v = \sum\limits_{i=1}^{t_1}[f_i(z)]_1
\end{gather}$$

同样的，还要计算经过 $z'$ 点的 $t_2$ 个多项式的 $F'$ 和 $v'$：

$$\begin{gather}
F' = \sum\limits_{i=1}^{t_2}cm'_i \\
v' = \sum\limits_{i=1}^{t_2}[f'_i(z')]_1
\end{gather}$$

最终验证者需要将 $F-v$ 和 $F' - v'$ 组合在一起：

$$
(F-v)+r'(F'-v')
$$

这里的 $r'$ 代表的是另个一个随机数。

然后我们开始尝试将 $W$ 和 $W'$ 、$z$ 和 $z'$ 也代入验证公式，形成最后的验证。在单个点的承诺中，验证等式为：$$e(W,[s-z]_2)=e(F-v,H)$$ ，我们的思路是将 $$F'-v'$$ 、$W'$ 和 $z'$ 分别加到对应的位置上，比如引入新的随机数 $r'$ 后， $F'-v'$ 和 $F-v$ 组合在一起，形成 $(F-v) + r'(F'-v')$ 。但这里我们遇到一个问题，就是 $z'$ 怎么和 $z$ 组合。直接加上变成 $s-z+s-z'$ ？或者相乘变成 $(s-z)(s-z')$ 好像都不对。究其原因，在于 $s-z$ 跟 $W$ 是相乘的。让我们暂时用一个不太正确但方便的记法，让 $F_{raw}$ 、$v_{raw}$ 、$W_{raw}$ 暂时只代表一个普通值，而不是椭圆曲线上的点（如 $W_{raw}$ 代表 $h(s)$ 而非 $$[h(s)]_1$$），那么在单个点的版本中，实际上我们要验证的是 $F_{raw}-v_{raw} = W_{raw}(s-z)$ 。这里可以清晰的看到，$W_{raw}$ 和 $s-z$ 是相乘的关系，没法像 $$F'_{raw}-v'_{raw}$$ 一样直接加到 $F_{raw}-v_{raw}$ 上进行组合。那要怎么办呢？

很简单，我们把 $F_{raw}-v_{raw}=W_{raw}(s-z)$ 拆开，去掉乘法就可以了。所以它就等价变成了：

$$(F_{raw}-v_{raw})+zW_{raw} = sW_{raw}$$

所以我们也可以使用双线性配对去验证上面这个等式。如果是这样，那单个点版本中的最终验证的双线性等式就变成了：

$$e((F-v)+zW, H) = e(W, [s]_2)$$

这个等式与之前的验证等式是等价的，但变成这样就方便使用加法组合了。

刚才我们看到，对于经过 $z$ 点的多项式，有：

$$(F_{raw}-v_{raw})+zW_{raw} = sW_{raw}$$

所以类似的，对于经过 $z'$ 点的多项式，引入新的随机变量 $r'$ 后，也有：

$$r'(F'_{raw}-v'_{raw})+r'z'W'_{raw} = sr'W'_{raw}$$

将这两个等式等号两边分加相加得到：

$$
(F_{raw}-v_{raw})+r'(F'_{raw}-v'_{raw}) +zW_{raw} + r'z'W'_{raw} = (W_{raw}+r'W'_{raw})s
$$

所以最终的验证用的双线性配对等等式就变成了：

$$
e((F-v)+r'(F'-v')+zW + r'z'W', H) = e(W + r'W', [s]_2)
$$

为了表示方便以及跟论文里保持一致，我们给 $F$ 新的意义，让它代表整个原来的 $(F-v)+r'(F'-v')$ ，所以：

$$
F = \left(\sum\limits_{i=1}^{t_1}cm_i - \sum\limits_{i=1}^{t_1}[f_i(z)]_1\right) + r'\left(\sum\limits_{i=1}^{t_2}cm'_i - \sum\limits_{i=1}^{t_2}[f'_i(z')]_1\right)
$$

所以最终的验证等式变成了：

$$
e(F + zW + r'z'W', H) = e(W+r'W', [s]_2)
$$

这就是两个点上的多项式承诺。

# 3. 总结
所谓批量多项式承诺，其中心思想就是将简单情况下的承诺、商多项式、$y$ 值，通过加法引入新的随机数的方式组合起来；当有两个点 $z$ 和 $z'$ 时，组合有些麻烦，就对验证等式做了一些转换即可。

在 plonk 算法中，最终验证者进行验证时，也会按照本文介绍的方法和方式，生成 $W$ 、$W'$ 等数据，到时候我们再对照本文的介绍，进行梳理和理解。



