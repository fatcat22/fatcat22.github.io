---
layout: post
title:  "09-最终的证明过程"
categories: zkp
tags: 原创 zkp groth16 
author: fatcat22
mathjax: true
---

* content
{:toc}




经过不断的改进，我们终于迎来了最终的证明过程。所以这篇文章里没有别的，只是完整的、最终的证明过挰。但最终的证明过程我们分两部分来说，第一部分是我们根据之前的文章得到的证明过程，这部分是我们一惯的、熟悉的；第二部分我们将展示 groth16 这种具体的零知识证明算法的论文里的证明过程，这部分是正式的，与我们熟悉的有的差别，但我们会对其进行解释。这样一来，我们不但熟悉了零知识证明的内在逻辑，也掌握了 groth16 的正式的表达方式（这方便我们去看其它的资料和代码，因为多数资料还是以论文里的表述方式和算法进行表达的）。

> groth16 是一种目前比较常用的，也比较简单的零知识证明算法。前面的文章我们一直没提，但事实是从这一系列介绍零知识证明的[第一篇](https://yangzhe.me/2023/10/11/first-story-of-zk/)到当前这篇，都是以 groth16 作为最终目标进行介绍的。

# 1. 最终证明过程
这一小节我们展示根据我们前面文章的内容，得到的最终的一个证明过程。

> 下面过程中， $n$ 始终为变量的数量（即 witness 数量）；$d$ 始终为约束的数量（即 r1cs 中等式的数量）

>  prover 使用 $\delta$ 值加密的这个过程（即保持零知识）是可选的，可以有也可以没有，所以下面与这个过程相关的操作都使用$\color{red}{红色标记}$ 。


## 1.1 common setup
1. 多方参与 trusted setup ，得到 $(g, g^{s^i})$ ，其中 $i$ 的最大值足够大，一般远超过 r1cs 中的等式数量 $d$ 。

## 1.2 生成 r1cs 和 QAP
1. prover 根据具体问题，生成 r1cs .
2. prover 根据 r1cs 生成 QAP: $l_i(x)$、$r_i(x)$ 、$o_i(x)$ 、$t(x)$  , $i \in \{0,1..,n\}$ ，其中 $n$ 为变量的个数  ；另外令 $d$ 为 r1cs 中约束数量，令 witness 中序号从  0 到 m 的值为 public input 。

## 1.3 QAP setup
1. 生成随机数 ${\rho}_l$ 、${\rho}_r$ 、${\alpha}_l$ 、${\alpha}_r$ 、${\alpha}_o$ 、$\beta$ 、$\gamma$ 
2. 令 ${\rho}_o = {\rho}_l \cdot {\rho}_r$ ，并生成新的 generator $g_l = g^{\rho_l}$ ，$g_r = g^{\rho_r}$ ，$g_o = g^{\rho_o}$
3. prover 使用 common setup 中生成的值 ，分别计算：  
$$\begin{align}
&\{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)}\}_{i \in \{0,1,...,n\}}, \\
&\{g_l^{\alpha_ll_i(s)}, g_r^{\alpha_rr_i(s)}, g_o^{\alpha_oo_i(s)}, g_l^{ {\beta}l_i(s)} \cdot g_r^{ {\beta}r_i(s)} \cdot g_o^{ {\beta}o_i(s)}\}_{i \in \{m+1,...,n\}}, \\
&g_o^{t(s)}, g^{\alpha_l}, g^{\alpha_r}, g^{\alpha_o}, g^{\gamma}, g^{\beta\gamma}, \\
& \color{red}{g_l^{t(s)}, g_r^{t(s)}, g_o^{t(s)}, g_l^{\alpha_lt(s)}, g_r^{\alpha_rt(s)}, g_o^{\alpha_ot(s)}, g_l^{ {\beta}t(s)}, g_r^{ {\beta}t(s)}, g_o^{ {\beta}t(s)}}
\end{align}$$ 
4. 在第 3 步的基础上，多方参与 trusted setup 。方法是每一方都重新生成第 1 步中的随机数，然后在上一方的第 3 步的基础上，继续对这些值进行加密。（多方加密后，随机数的值已不是原来的值，而是各方的随机数相乘的结果，但为了表示方便，我们保持随机数的命名不变）
5. 最终得到 ：  
   $$\begin{align}
proving\ key&: \begin{cases}
\{g^{s^i}\}_{i \in \{1,...,d\}}, \\
\{g_l^{l_i(s)},\ g_r^{r_i(s)},\ g_o^{o_i(s)}\}_{i \in \{0,...,n\}}, \\
\{g_l^{\alpha_ll_i(s)},\ g_r^{\alpha_rr_i(s)},\ g_o^{\alpha_oo_i(s)},\ g_l^{ {\beta}l_i(s)} \cdot g_r^{ {\beta}r_i(s)} \cdot g_o^{ {\beta}o_i(s)}\}_{i \in \{m+1,...,n\}}, \\
\color{red}{g_l^{t(s)}, g_r^{t(s)}, g_o^{t(s)}, g_l^{\alpha_lt(s)}, g_r^{\alpha_rt(s)}, g_o^{\alpha_ot(s)}, g_l^{ {\beta}t(s)}, g_r^{ {\beta}t(s)}, g_o^{ {\beta}t(s)}}
\end{cases} \\ \\
verification\ key&: \begin{cases}
g, \\
g_o^{t(s)}, \\
\{g_l^{l_i(s)},\ g_r^{r_i(s)},\ g_o^{o_i(s)}\}_{i \in \{0,...,m\}}, \\
g^{\alpha_l},\ g^{\alpha_r},\ g^{\alpha_o},\ g^{\gamma}, g^{\beta\gamma}
\end{cases}
\end{align}$$

## 1.4 prover
1. 得到 witness 的具体取值（即每个 $v_i$ 的值）
2. $\color{red}{生成随机数 \{\delta_l\ , {\delta}_r,\ {\delta}_o\}}$，并使用 proving key 中的值，计算：  
   $$\begin{align}
 g_l^{L_p(s)} &= \color{red}{\left(g_l^{t(s)}\right)^{\delta_l} \cdot} \prod\limits_{i=m+1}^n{\left( g_l^{l_i(s)} \right)^{v_i}} = g_l^{\color{red}{\delta_lt(s) +} \left(v_{m+1}l_{m+1}(s)+...+v_nl_n(s)\right)} = E_{L_p} \\
 g_r^{R_p(s)} &= \color{red}{\left(g_r^{t(s)}\right)^{\delta_r} \cdot} \prod\limits_{i=m+1}^n{\left( g_r^{r_i(s)} \right)^{v_i}} = g_r^{\color{red}{\delta_rt(s) +} \left(v_{m+1}r_{m+1}(s)+...+v_nr_n(s)\right)} = E_{R_p} \\
 g_o^{O_p(s)} &= \color{red}{\left(g_o^{t(s)}\right)^{\delta_o} \cdot} \prod\limits_{i=m+1}^n{\left( g_o^{o_i(s)} \right)^{v_i}} = g_o^{\color{red}{\delta_ot(s) +} \left(v_{m+1}o_{m+1}(s)+...+v_no_n(s)\right)} = E_{O_p} \\
\end{align}$$
3. 同样使用 $\color{red}{随机数 \{\delta_l,\ {\delta}_r,\ {\delta}_o\}}$ 及 proving key 中的值计算：  
   $$\begin{align}
g_l^{L'_p(s)} &= \color{red}{\left(g_l^{\alpha_lt(s)}\right)^{\delta_l} \cdot} \prod\limits_{i=m+1}^n{\left( g_l^{\alpha_ll_i(s)} \right)^{v_i}} = g_l^{\alpha_l\left(\color{red}{\delta_lt(s)+} \left( v_{m+1}l_{m+1}(s) + ... + v_nl_n(i) \right)\right)} = \left(g_l^{L_p(s)}\right)^{\alpha_l} = E_{L'_p} \\
g_r^{R'_p(s)} &= \color{red}{\left(g_r^{\alpha_rt(s)}\right)^{\delta_r} \cdot} \prod\limits_{i=m+1}^n{\left( g_r^{\alpha_rr_i(s)} \right)^{v_i}} = g_r^{\alpha_r\left(\color{red}{\delta_rt(s)+} \left( v_{m+1}r_{m+1}(s) + ... + v_nr_n(i) \right)\right)} = \left(g_r^{R_p(s)}\right)^{\alpha_r} = E_{R'_p} \\
g_o^{O'_p(s)} &= \color{red}{\left(g_o^{\alpha_ot(s)}\right)^{\delta_o} \cdot} \prod\limits_{i=m+1}^n{\left( g_o^{\alpha_oo_i(s)} \right)^{v_i}} = g_o^{\alpha_o\left(\color{red}{\delta_ot(s)+} \left( v_{m+1}o_{m+1}(s) + ... + v_no_n(i) \right)\right)} = \left(g_o^{O_p(s)}\right)^{\alpha_o} = E_{O'_p} 
\end{align}$$
5. 计算 $h(x) = \frac{L(x) \times R(x)-O(x)}{t(x)} \color{red}{+\delta_rL(x) + \delta_lR(x) + \delta_l\delta_rt(x) - \delta_o}$ （生成 $L(x)$ 、$R(x)$ 和 $O(x)$ 时都要用到 witness 即 $v_i$ 的值，所以 prover 只有在拿到具体的 witness 值以后才能计算出 $h(x)$ ），并使用 proving key 中的 ${\{g^{s^i}\}_{i \in \{0,...,d\}}}$ 计算得出 $g^{h(s)}$ 。
6.  计算用来约束变量一致性的 $g^Z$ （注意，下面等式中虽然用 $\left(g_l^{L_p(s)}g_r^{R_p(s)}g_o^{O_p(s)}\right)^{\beta}$ 表示 $g^Z$ ，但这只是为了理解方便，可以在后面 verifier 对 $g^Z$ 进行验证时，方便的看出结果是一致的；并不代表 prover 可以这样计算 $g^Z$ ，因为 prover 并不知道 $\beta$ 的值，因为 $\beta$ 的值被 $\gamma$ 加密了） ：  
   $$
g^{Z(s)} = \color{red}{\left(g_l^{ {\beta}t(s)}\right)^{\delta_l} \cdot \left(g_r^{ {\beta}t(s)}\right)^{\delta_r} \cdot \left(g_o^{ {\beta}t(s)}\right)^{\delta_o} \cdot} \prod\limits_{i=m+1}^{n}{\left(g_l^{ {\beta}l_i(s)}g_r^{ {\beta}r_i(s)}g_o^{ {\beta}o_i(s)}\right)^{v_i}} = \left(g_l^{L_p(s)}g_r^{R_p(s)}g_o^{O_p(s)}\right)^{\beta} = E_Z
$$
7. 将计算得出的值 $$(E_{L_p},E_{L'_p},E_{R_p},E_{R'_p},E_{O_p},E_{O'_p},E_h,E_Z)$$ 传回给 verifier.

## 1.5 verifier
1. 使用 public input/output 和 verification key 中的值计算 $g_l^{L_v}$ 、$g_r^{R_v}$ 、$g_o^{O_v}$ （注意 $g_l^{l_0(s)}$ 、$g_r^{r_0(s)}$ 和 $g_o^{o_0(s)}$ 是单列出来的，并没放入 $\prod$  中，因为 $v_0$ 必须是 1 ）：  
   $$\begin{align}
g_l^{L_v(s)} &= g_l^{l_0(s)} \cdot \prod\limits_{i=1}^m{\left(g_l^{l_i(s)}\right)^{v_i}} = E_{L_v} \\
g_r^{R_v(s)} &= g_r^{r_0(s)} \cdot \prod\limits_{i=1}^m{\left(g_r^{r_i(s)}\right)^{v_i}} = E_{R_v}\\
g_o^{O_v(s)} &= g_o^{o_0(s)} \cdot \prod\limits_{i=1}^m{\left(g_l^{o_i(s)}\right)^{v_i}} = E_{O_v} \\
\end{align}$$
2. 验证 $\alpha$ 约束的正确性（注意 $g_l = g^{\rho_l}, g_r = g^{\rho_r}, g_o = g^{\rho_o}$；另外可以看到，无论 prover 是否使用 $\delta$ 加密，对 verifier 的验证都是没有影响的）：  
   $$\begin{align}
e\left(E_{L_p}, g^{\alpha_l}\right) &= e\left(E_{L'_p}, g\right), &&即  \\ 
&&&e\left(g_l^{L_p(s)}, g^{\alpha_l}\right) = e\left(\left(g_l^{L_p(s)}\right)^{\alpha_l}, g\right) \ \Rightarrow \  e\left(g^{\rho_lL_p(s)}, g^{\alpha_l}\right) = e\left(\left(g^{\rho_lL_p(s)}\right)^{\alpha_l}, g\right) \\
e\left(E_{R_p}, g^{\alpha_r}\right) &= e\left(E_{R'_p}, g\right), &&即 \\
&&&e\left(g_r^{R_p(s)}, g^{\alpha_r}\right) = e\left(\left(g_r^{R_p(s)}\right)^{\alpha_r}, g\right) \ \Rightarrow \  e\left(g^{\rho_rR_p(s)}, g^{\alpha_r}\right) = e\left(\left(g^{\rho_rR_p(s)}\right)^{\alpha_r}, g\right) \\
e\left(E_{O_p}, g^{\alpha_o}\right) &= e\left(E_{O'_p}, g\right), &&即 \\
&&& e\left(g_o^{O_p(s)}, g^{\alpha_o}\right) = e\left(\left(g_o^{O_p(s)}\right)^{\alpha_o}, g\right) \ \Rightarrow \  e\left(g^{\rho_oO_p(s)}, g^{\alpha_o}\right) = e\left(\left(g^{\rho_oO_p(s)}\right)^{\alpha_o}, g\right)
\end{align}$$
3. 验证变量一致性：  
   $$\begin{gather}
e\left(g_l^{L_p(s)}g_r^{L_r(s)}g_o^{L_o(s)}, g^{\beta\gamma}\right)=e\left(g^{Z(s)}, g^{\gamma}\right) \\ \Downarrow \\
e\left(g_l^{L_p(s)}g_r^{L_r(s)}g_o^{L_o(s)}, g^{\beta\gamma}\right)=e\left(\left(g_l^{L_p(s)}g_r^{L_r(s)}g_o^{L_o(s)}\right)^{\beta}, g^{\gamma}\right) \\
\end{gather}$$
4. 验证多项式承诺（注意 $\rho_o = \rho_l\rho_r$ ）：  
   $$\begin{gather}
e\left(E_{L_p}\cdot E_{L_v}, E_{R_p} \cdot E_{R_v}\right) = e\left(g_o^{t(s)}, E_h\right) \cdot e\left(E_{O_p} \cdot E_{O_v},g\right) \\ \Downarrow \\
e\left(g_l^{L_p}g_l^{L_v}, g_r^{R_p}g_r^{R_v}\right) = e\left(g_o^{t(s)}, g^{h(s)}\right) \cdot e\left(g_o^{O_p}g_o^{O_v}, g\right) \\ \Downarrow \\
e\left(g^{\rho_l(L_p+L_v)}, g^{\rho_r(R_p+R_v)}\right) = e\left(g^{\rho_ot(s)},g^{h(s)}\right) \cdot e\left(g^{\rho_o(O_p+O_v)},g\right) \\ \Downarrow \\
e\left(g,g\right)^{\rho_l{\rho}_rL(s)R(s)} = e(g,g)^{\rho_l{\rho}_r(t(s)h(s)+O(s))}
\end{gather}$$


# 2. 最终的证明过程-正式篇
这一小节，我们展示一下 groth16 论文里的证明过程。因为上面我们得出的证明只是为了我们好理解所推导出来的，但业界还是以论文里的证明过程为准。所以为了能在学习完我们的文章后，可以继续方便的去看其它资料、论文、代码，我们需要对正式的证明过程作一个说明。

首先我们要说明一下正式情况下是如何定义双线性配对的。在 groth16 的论文里，有这样的预先定义：
- 定义三个素数 $p$ 阶乘法循环群（group of prime order $p$）$\mathbb{G}_1$ 、$\mathbb{G}_2$ 、$\mathbb{G}_T$
- 令 $e: \mathbb{G}_1 \times \mathbb{G}_2 \rightarrow \mathbb{G_T}$ 为这三个乘法循环群的双线性映射
- 令 $g$ 为 $\mathbb{G}_1$ 的生成元；$h$ 为 $\mathbb{G}_2$ 的生成元；$e(g,h)$ 为 $\mathbb{G}_3$ 的生成元
有了这样的定义后，后面使用 $[a]_1$ 的形式代表 $g^a$ ；$[b]_2$ 的形式代表 $h^b$ ；$[c]_T$ 的形式代表 $e(g,h)^c$ 。（其实有的资料里也使用 $a \cdot G_1$ 这种方式代表 $g^a$ ，写法不一样意义都是一样的）

>需要说明一下的是，在刚才的定义之下，双线性映射 $e$ 有这样的性质： $e([a]_1, [b]_2)=e(g,h)^{ab}=[ab]_T$ ，还请特别注意一下（因为我们之前一直用同一个生成元 $g$ ，所以这里可能会有点迷惑）。

其次还要说明的是，我们之前使用的 $l_i(x)$ 、$r_i(x)$ 、$o_i(x)$ ，在 groth16 里换成了 $u_i(x)$ 、$v_i(x)$ 、$w_i(x)$ ，虽然表示方法不一样，但意义是完全一样的。

## 2.1 common setup
多方参与计算：每个人生成随机数 $\alpha$ 、$\beta$ 、$\gamma$ 、$\delta$ 、$x$ ， 并在前面人基础上，生成加密值 CRS $\tau = (\alpha, \beta, \gamma, \delta, x)$ 。（这里的 $x$ 相当于我们之前的 $s$ ）

## 2.2 生成 r1cs 和 QAP
1. prover 根据具体问题，生成 r1cs .
2. prover 根据 r1cs 生成 QAP: $u_i(x)$、$v_i(x)$ 、$w_i(x)$ 、$t(x)$  , $i \in \{0..,m\}$ ，其中 $m$ 为变量的个数 。

## 2.3 QAP setup
生成 $\sigma=([\sigma]_1,[\sigma]_2)$ ，其中：

$$\begin{align}
\sigma_1 &= 
\begin{cases}
\alpha, \beta, \gamma \\
\{x^i\}_{i \in [0,..,n-1]} \\
\left\{\frac{ {\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)}{\gamma}\right\}_{i \in [0,..,\mathscr{l}]} \\
\left\{\frac{ {\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)}{\delta}\right\}_{i \in [{\mathscr{l}+1,..m}]} \\
\left\{\frac{x^it(x)}{\delta}\right\}_{i \in [0,..,n-2]}
\end{cases} \\ \\
\sigma_2 &= 
\begin{cases}
\beta, \gamma, \delta, \left\{x^i\right\}_{i \in [0,..,n-1]}
\end{cases}
\end{align}$$

注意上面 $\sigma_1$ 和 $\sigma_2$ 的各项值虽然没有用 $[a]_1$ 这样的表达形式，但 $\sigma = ([\sigma]_1, [\sigma]_2)$ ，所以这些都实际上都是在加密域计算的。

注意如果选取的生成元 $g$ 和 $h$ 是一样的（groth16 算法是允许的），那么这里 $\sigma_1$ 里的 $\beta$ 和 $\gamma$ 和 $\sigma_2$ 里的就是一样的；否则这俩值是不一样的。


## 2.4 prover
假设 prover 得到 witness = $(a_0, a_1, ..., a_m)$ 。其中 $a_0$ 总是等于 1 ；$(a_0, ..., a_{\mathscr{l}})$ 为 public input ，$a_{\mathscr{l}+1},...,a_m$ 为private input 。

得到 witness 后，prover 需要计算 $h(x)$ 多项式。

prover 选择随机数 $r$ 和 $s$ ，计算 $\pi = \left([A]_1, [B]_2, [C]_1\right)$ ，其中：

$$\begin{align}
A &= \alpha + \sum\limits_{i=0}^m{a_iu_i(x)} + r\delta \\ \\
B &= \beta + \sum\limits_{i=0}^m{a_iv_i(x)} + s\delta \\ \\
C &= \frac{\sum\limits_{i=\mathscr{l}+1}^m{a_i\Big({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\Big)}+h(x)t(x)}{\delta} + As + Br - rs\delta
\end{align}$$

同样的，虽然上面 $A$ 、$B$ 、$C$ 里的值没有用 $[a]_1$ 这种表示方式，但 $\pi = ([A]_1, [B]_2, [C]_1)$ ，所以实际上计算都是在加密域的。并且其实 prover 只知道 witness $a_i$ 的值，对于其它值如 $\alpha$ ，他是不知道原始值的。

最后 prover 将 $\pi$ 传给 verify 进行验证。

## 2.5 verifier
verifier 拿到 prover 的 $\pi$ 以后，使用以下等式进行验证：

$$
e([A]_1, [B]_2) = e([\alpha]_1, [\beta]_2) \cdot e\left(\left[\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)}{\gamma}}\right]_1,[\gamma]_2\right) \cdot e([C]_1, [\delta]_2)
$$

如果等式成立，则验证成功，否则失败。（这个等式代入 $[A]_1$ 、$[B]_2$ 、$[C]_1$ 的值后进行拆解，可以发现是真的可以相等的，过程繁锁但不难，所以这里就不展示了）

我们接下来将 $A$ 、$B$ 、$C$ 的值代入这个等式，看看这个等式意味着什么，为什么可以成立。

首先左边 $e([A]_1, [B]_2)$ 也就是 $e(g, h)^{A \cdot B}$ ，所以我们只计算 $A \cdot B$ 即可：

$$\begin{align}
& A \cdot B \\ 
&= \left(\alpha + \sum\limits_{i=0}^m{a_iu_i(x)} + r\delta \right)\left( \beta + \sum\limits_{i=0}^m{a_iv_i(x)} + s\delta \right) \\
&= {\alpha}{\beta} + {\alpha}\sum\limits_{i=0}^m{a_iv_i(x)} + {\alpha}s{\delta} + \beta\sum\limits_{i=0}^m{a_iu_i(x)} + \sum\limits_{i=0}^m{a_iu_i(x)}\sum\limits_{i=0}^m{a_iv_i(x)} + s{\delta}\sum\limits_{i=0}^m{a_iu_i(x)} + {\beta}r{\delta} + r{\delta}\sum\limits_{i=0}^m{a_iv_i(x)} + rs{\delta}^2 \\
&= {\alpha}{\beta} + {\alpha}s{\delta} + {\beta}r{\delta} + rs{\delta}^2 + (\alpha+r\delta)\sum\limits_{i=0}^m{a_iv_i(x)} + (\beta+s\delta)\sum\limits_{i=0}^m{a_iu_i(x)}+  \\ &\sum\limits_{i=0}^m{a_iu_i(x)}\sum\limits_{i=0}^m{a_iv_i(x)}
\end{align}$$

最后一行就是 $A\cdot B$ 的结果（为了方便，最后一行将所有不带 $\sum$ 的计算放到了最前面，同时将最复杂的两个 $\sum$ 相乘的计算放到了最后面）。

我们再看看等式右侧：  
$$e([\alpha]_1,[\beta]_2){\cdot}e\left(\left[\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)}{\gamma}}\right]_1,[\gamma]_2\right){\cdot}e([C]_1,[\delta]_2)$$  
也就是 $e(g, h)^{\alpha\beta+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)}{\gamma}}{\gamma}+C\delta}$ ，所以我们只需要计算指数部分：

$$\begin{align}
&{\alpha}{\beta}+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)}{\gamma}}{\gamma}+C\delta \\
&= {\alpha}{\beta} + \sum\limits_{i=0}^{\mathscr{l}}{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)} + \left( \frac{\sum\limits_{i=\mathscr{l}+1}^m{a_i\Big({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\Big)}+h(x)t(x)}{\delta} + As + Br - rs\delta \right)\delta \\
&= {\alpha}{\beta} + \sum\limits_{i=0}^{\mathscr{l}}{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)} + \sum\limits_{i={\mathscr{l}}+1}^m{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)}+h(x)t(x)+\left(\alpha + \sum\limits_{i=0}^m{a_iu_i(x)} + r\delta\right)s\delta + \left(\beta + \sum\limits_{i=0}^m{a_iv_i(x)} + s\delta\right)r\delta - rs{\delta}^2 \\
&= {\alpha}{\beta} + \sum\limits_{i=0}^m{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)}+h(x)t(x)+{\alpha}s{\delta}+s{\delta}\sum\limits_{i=0}^m{a_iu_i(x)}+rs{\delta}^2+{\beta}r{\delta}+r{\delta}\sum\limits_{i=0}^m{a_iv_i(x)} \\
&= {\alpha}{\beta}+{\alpha}s{\delta}+{\beta}r{\delta}+rs{\delta}^2 + (\alpha++r\delta)\sum\limits_{i=0}^m{a_iv_i(x)} + (\beta+s\delta)\sum\limits_{i=0}^m{a_iu_i(x)} + \\
&\sum\limits_{i=0}^m{a_iw_i(x)}+h(x)t(x)
\end{align}$$

最后一行即是最终的结果（最后一行一方面将 $\sum\limits_{i=0}^m{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}$ 拆成了 $\beta\sum\limits_{i=0}^m{a_iu_i(x)}+\alpha\sum\limits_{i=0}^m{a_iv_i(x)}+\sum\limits_{i=0}^m{a_iw_i(x)}$ ，另一方面将各子运算重新排序，使相同的运算与前面 $A\cdot B$ 的结果保持一致）。可以看到，前面大部分运算是可以抵消掉的，最后只剩下了：

$$
\sum\limits_{i=0}^m{a_iu_i(x)}\sum\limits_{i=0}^m{a_iv_i(x)} = \sum\limits_{i=0}^m{a_iw_i(x)}+h(x)t(x)
$$

这跟我们之前的文章里 $L(x)R(x) = O(x) + h(x)t(x)$ 是完全一样的。


## 2.6 正式证明过程的解析
可以看到论文里的整个过程比我们的简洁多了。但你心里可能也会有疑问：那我们之前为了阻止 prover 作弊，引入了很多的随机数和检查 ，这个简洁的证明看上去随机数比我们的少了很多，verifier 检查也只有一个等式，能解决 prover 作弊的问题吗？我们这里就一一分析一下，顺便了解一下这个正式的过程里各变量（$\alpha$ 、$\beta$ 等）起的作用。

我们先列举一下之前为了防止 prover 作弊，我们都引入了哪些随机数：
1. 诚实计算问题：为了保证 prover 真的使用多项式计算，而不是随便猜一个数糊弄 verifer ，我们引入了三个 $\alpha$ 变量；verifier 增加了三个检查（如 $e(g^L, \alpha)=e(g^{ {\alpha}L},g)$）。
2. witness 一致性问题：为了保证 prover 在计算 $g^L$ 、$g^R$ 、$g^O$ 时使用的是完全一样的 witness $(v_0, v_1, ...)$ ，我们引入了三个 $\beta$ 变量（后来通过优化，缩减为一个 $\beta$ ），为了隐藏 $\beta$ 而引入了 $\gamma$ ；verifier 增加了一个对 $g^Z$ 的检查 $e\left(g_l^{L_p(s)}g_r^{L_r(s)}g_o^{L_o(s)}, g^{\beta\gamma}\right)=e\left(g^{Z(s)}, g^{\gamma}\right)$。
3. 为了优化，减少 verifier 计算量，引入了 $\rho_l$ 和 $\rho_r$ 两个随机数

虽然然没使用 $\rho_l$ 的 $\rho_r$ ，但正式的证明过程中 verifier 依然只做了四次双线性配对的运算，比我们证明过程少很多，所以最后一条我觉得显而易见，只要前面两条的安全性能保证，那第 3 条肯定是没问题的，我们就不再讨论了。下面我们重点讨论前两条。


## 2.7 诚实计算问题
我们先来看看第一个问题，为了防止 prover 随便弄个值糊弄 verifier ，我们引入了三个 $\alpha$ 变量，verifier 也增加了三个检查。那么这个正式的证明过程能防止这个情况吗？

我们现在假设 prover 作弊，ta 不真正的通过多项式计算得到 $A$ 、$B$ 和 $C$ ，而是通过「精心准备」三个值企图让 verifier 通过验证。那么 prover 该如何「精心准备」呢？

从前面 $A$ 、$B$ 、$C$ 三个等式我们可以看出来，prover 真正不想计算的部分分别是 $\sum\limits_{i=0}^m{a_iu_i(x)}$ 、$\sum\limits_{i=0}^m{a_iv_i(x)}$ 、 $\sum\limits_{i=\mathscr{l}+1}^m{a_i\Big({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\Big)}$ 以及 $h(x)$ 和 $t(x)$ ，这里我们先不管 $h(x)$ 和 $t(x)$ ，先看看前面三个值。假如 prover 「精心准备」的这三个值分别是 $R_A$ 、$R_B$ 和 $R_C$ ，那这三个值是满足什么条件，才能骗过 verifier ，让 verifier 验证成功呢？

为了更直观，我们先把这三个值代入到 $A$ 、$B$ 、$C$ 中：

$$\begin{align}
A &= \alpha + R_A + r\delta \\ \\
B &= \beta + R_B + s\delta \\ \\
C &= \frac{R_C+h(x)t(x)}{\delta} + As + Br - rs\delta
\end{align}$$

我们使用 verifier 的验证等式，将上面的值代入，最后就能看出 $R_A$ 、$R_B$ 和 $R_C$ 需要满足什么条件，才能让 verifier 验证成功了。verifier 验证等式的左边是 $e([A]_1,[B]_2)$ ，即 $e(g,h)^{A{\cdot}B}$，所以需要计算 $A{\cdot}B$ 的值：

$$\begin{gather}
A \cdot B = \\
(\alpha + R_A + r\delta)(\beta + R_B + s\delta)= \\
\alpha\beta + {\alpha}s\delta + {\beta}r\delta + rs{\delta}^2 + {\alpha}R_B + {\beta}R_A + R_AR_B + s{\delta}R_A + r{\delta}R_B
\end{gather}$$

verifier 验证等式的右边是 $$e([\alpha]_1,[\beta]_2){\cdot}e\left(\left[\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}\gamma}\right]_1,[\gamma]_2\right){\cdot}e([C]_1,[\delta]_2)$$ ，即 $e(g,h)^{\alpha\beta+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma}+{\delta}C}$ ，所以：

$$\begin{gather}
{\alpha}{\beta}+\sum\limits_{i=0}^{\mathscr{l}}{a_i(\frac{ {\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma}+{\delta}C = \\
{\alpha}{\beta}+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma}+{\delta}\left(\frac{R_C+h(x)t(x)}{\delta} + As + Br - rs\delta\right) = \\
{\alpha}{\beta}+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma}+R_C+h(x)t(x)+{\delta}sA + {\delta}rB-rs{\delta}^2 = \\
{\alpha}{\beta}+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma}+R_C+h(x)t(x)+{\delta}s({\alpha}+R_A+r{\delta}) + {\delta}r(\beta+R_B+s\delta)-rs{\delta}^2 = \\
{\alpha}{\beta}+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma}+R_C+h(x)t(x)+{\alpha}s\delta+s{\delta}R_A+rs{\delta}^2+{\beta}r{\delta}+r{\delta}R_B
\end{gather}$$

注意我们现在正在模拟 prover 寻找作弊方法，所以一直保留着 $\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma}$ 没动，一方面是因为这一块是 verifier 计算的，所以目前必须如实计算；另一方面也是因为 $\frac{ {\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)}{\gamma}$ 这个值是在 setup key 中，prover 只能直接拿来用，自己是计算不出来的。

现在 verifier 的左边和右边我们都计算出来了，为了方便我们再重新写一下，并将可以约掉的值标记出来：

$$\begin{align}
left&: \\
& \underline{\alpha\beta} + \underline{ {\alpha}s\delta} + \underline{ {\beta}r\delta} + \underline{rs{\delta}^2} + {\alpha}R_B + {\beta}R_A + R_AR_B + \underline{s{\delta}R_A} + \underline{r{\delta}R_B} \\
right&: \\
& \underline{\alpha\beta}+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma}+R_C+h(x)t(x)+\underline{ {\alpha}s\delta}+\underline{s{\delta}R_A}+\underline{rs{\delta}^2}+\underline{ {\beta}r{\delta}}+\underline{r{\delta}R_B}
\end{align}$$

可以看到最后只剩下：

$$
{\alpha}R_B + {\beta}R_A + R_AR_B = \sum\limits_{i=0}^{\mathscr{l}}{a_i(\frac{ {\beta}u_i(x)+{\alpha}v_i(x)+w_i(x))}{\gamma}}{\gamma} + R_C + h(x)t(x)
$$

可见只要 $R_A$ 、$R_B$ 、$R_C$ 的取值能满足这个等式，就可以让 verifier 验证成功。但是 prover 能算出这样的三个值吗？你可能会想这还不简单，等式都摆在这了，随机取 $R_A$ 和 $R_B$ 的值，然后用等式计算出 $R_C$ 不就行了嘛。但是有一点我们要注意，这个等式是在非加密域的等式，但实际上，prover 是不知道 $\alpha$ 、$\beta$ 和 $\gamma$ 的值的，prover 知道的只是 $[\alpha]_1$ 、$[\beta]_1$ 、$[\gamma]_1$ 以及 $[\beta]_2$ 、$[\gamma]_2$ 。所以在不知道 $\alpha$ 、$\beta$ 和 $\gamma$ 的值的情况下，prover 实际上是无法算出 $R_A$ 、$R_B$ 和 $R_C$ 的值，满足上面等式的，除非老老实实通过多项式计算。

你可能还会想，上面的等式不在加密域里，只是为了写文章表示方便，实际上 prover 可以使用像 verifier 一样的计算方法，随机选定 $R_A$ 和 $R_B$ ，然后通过双线性配对计算，求解出 $R_C$ 呢？这个肯定是不行的。为了说明方便，我们简化一下，假设有人随便选了两个保密的数 $a$ 和 $b$ ，然后只告诉你 $e(g^a,g^b)$ 的值（假设为 $Y$），让你通过 $Y = e(g^x,g)$ 的方式求解出 $x$ 。这种问题有一种专业的叫法，叫「NP 问题」，就像告诉你 hash ，让你求出原数据一样，是几乎不可能的。双线性配对的运算也保证了这一点，让 $Y=e(g^x,g)$ 中求解 $x$ 值的问题是 NP 问题，几乎是求解不出来的（但如果知道了 $Y$ 和 $x$ ，验证却是非常简单的）。

所以总得来说，我们模拟 prover 作弊失败了，这得亏于 $\alpha$ 、$\beta$ 和 $\gamma$ 这三个值，当然在当前这个问题里，有一个就够了，就足以让 prover 无法作弊了。

## 2.8 witness 一致性问题
接下来我们看看第二个问题，即 witness 一致性的问题。当初为了保证 prover 使用的 witness $v_i$ 在每个多项式 $L(x)$ 、$R(x)$ 和 $O(x)$ 都是一样的，我们最终引入了 $\beta$ 和 $\gamma$ ，并让 verifier  增加了一次双线性配对运算，对 $g^Z$ 的验证。现在在正式的证明过程中，虽然也有 $\beta$ 和 $\gamma$ ，但我们不确定它们的意义是不是一样的；另外 verifier 只有一个验证等式，没有额外的验证 $g^Z$ 的过程，这能保证 witness 一致性问题吗？

我们可以先想像一下，假设 prover 在计算 $A$ 时用一套 witness ，在计算 $B$ 时用另一套 witness ，那在计算 $C$ 时，应该用 $A$ 那套还是 $B$ 那套呢？还是再重新用一套完全不一样的呢？事实上无论 $C$ 用哪套，都不可能让 verifier 验证成功。为了理解方便，我们把 $A$ 、$B$ 、$C$ 重新写一下：

$$
\begin{align}
A &= \alpha + \sum\limits_{i=0}^m{a_iu_i(x)} + r\delta \\
B &= \beta + \sum\limits_{i=0}^m{a_iv_i(x)} + s\delta \\
C &= \frac{\sum\limits_{i=\mathscr{l}+1}^m{a_i\Big({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\Big)}+h(x)t(x)}{\delta} + As + Br - rs\delta
\end{align}
$$

验证时左侧是 $A \cdot B$ ，将 $A$ 和 $B$ 相乘将会导致 $a_iu_i(x)$ 会有 $\beta$  参数，即 ${\beta}a_iu_i(x)$ ；同样 $a_iv_i(x)$ 会有 $\alpha$ 参数。右侧有 $C$ ，与 ${\beta}a_iu_i(x)$ 和 ${\alpha}a_iv_i(x)$ 对应的就是 $C$ 中的 $\frac{ {\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)}{\delta}$ 了，注意这个值是在 setup 时算好的，prover 并不能把它拆开（比如给 $\frac{ {\beta}u_i(x)}{\delta}$ 乘以一个值，而给 $\frac{ {\alpha}v_i(x)}{\delta}$ 乘以另一个不同的值），只能给 ${\beta}u_i(x)$ 和 ${\alpha}v_i(x)$ 乘以相同的值，所以如果 $A$ 和 $B$ 中的 $a_i$ 值不同，$C$ 是无法计算出一个值，让等式成立的。 

你可能还会想，我只要让整个等式成立行了，而整个等式成立不一定要让左边的 ${\beta}a_iu_i(x)$ 等于右边的 ${\beta}a_iu_i(x)$ ，$h(x)$ 不是 prover 自己算的吗？我只要调节 $h(x)$ 的值，让整个等式成立就行。这样一来就要看你 prover 能不能算出这要的 $h(x)$ 的值来了。与上一小节类似，要计算这个值就肯定涉及到了 $\alpha$ 、$\beta$ 这些随机变量的值，而 prover 是不可能知道这些值的。所以无论如何，prover 如果使用不同的 witness 来计算 $A$ 和 $B$ ，都会被发现。

下面我们用一个简单的具体例子来验证一下刚才所说的（如果你理解了上面所说的，例子部分可以跳过）。我们简单一点，假设 witness 只有两个。正确的 witness 为 $(a_0, a_1)$ ，其中 $a_0$ 是 public input ； prover 作弊时，在计算 $A$ 时使用 witness $(a'_0, a'_1)$ ，在计算 $B$ 时使用 witness $(b'_0, b'_1)$ ，在计算 $C$ 时使用 witness $(c'_1)$（$C$ 的计算只用到了 private input 部分） 。

我们在前面详细拆解过 verifier 等式的左右两部分，左侧比较简单我们直接摘抄一遍就行：（这里我们用不到与 witness 无关的部分（比如 $\alpha\beta$），所以为了更简洁把这些运算都去掉了）：

$$
(\alpha+r\delta)\sum\limits_{i=0}^m{a_iv_i(x)} + (\beta+s\delta)\sum\limits_{i=0}^m{a_iu_i(x)}+ \sum\limits_{i=0}^m{a_iu_i(x)}\sum\limits_{i=0}^m{a_iv_i(x)}
$$

我们把 $(a'_0, a'_1)$ 和 $(b'_0, b'_1)$ 代入，得到：

$$
(\alpha+r\delta)\sum\limits_{i=0}^1{b'_iv_i(x)}+(\beta+s\delta)\sum\limits_{i=0}^1{a'_iu_i(x)}+ \sum\limits_{i=0}^1{a'_iu_i(x)}\sum\limits_{i=0}^1{b'_iv_i(x)}
$$

verifier 右侧比较麻烦，需要我们重新计算一遍：

$$\begin{align}
&{\alpha}{\beta}+\sum\limits_{i=0}^{\mathscr{l}}{\frac{a_i\left({\beta}u_i(x)+{\alpha}v_i(x)+w_i(x)\right)}{\gamma}}{\gamma}+C\delta \\
&= {\alpha}{\beta}+a_0({\beta}u_0(x)+{\alpha}v_0(x)+w_0(x))+ {\delta}\left(\frac{c'_1\left({\beta}u_1(x)+{\alpha}v_1(x)+w_1(x)\right)+h(x)t(x)}{\delta}+s\left(\alpha+\sum\limits_{i=0}^1{a'_iu_i(x)}+r\delta\right)+r\left(\beta+\sum\limits_{i=0}^1{b'_iv_i(x)}+s\delta\right)-rs\delta\right) \\
&= {\alpha}{\beta}+{\beta}a_0u_0(x)+{\alpha}a_0v_0(x)+a_0w_0(x)+{\beta}c'_1u_1(x)+{\alpha}c'_1v_1(x)+c'_1w_1(x)+h(x)t(x)+s\alpha\delta+s{\delta}\sum\limits_{i=0}^1{a'_iu_i(x)}+rs{\delta}^2+r\beta\delta+r{\delta}\sum\limits_{i=0}^1{b'_iv_i(x)}
\end{align}$$

对最后一个等式进行整理（去掉与 witness 无关的项，比如 $\alpha\beta$ ；移动各项的位置），我们可以得到：

$$
\bigg({\alpha}a_0v_0(x)+{\alpha}c'_1v_1(x)+r{\delta}\sum\limits_{i=0}^1{b'_iv_i(x)\bigg)+
\bigg({\beta}a_0u_0(x)+{\beta}c'_1u_1(x)+s{\delta}\sum\limits_{i=0}^1{a'_iu_i(x)}\bigg)+
\bigg(a_0w_0(x)+c'_1w_1(x)\bigg)+
h(x)t(x)}
$$

可以看到如果要想 verifier 左右两侧的值相等，需要：

$$\begin{align}
(\alpha+r\delta)\sum\limits_{i=0}^1{b'_iv_i(x)} &= \bigg({\alpha}a_0v_0(x)+{\alpha}c'_1v_1(x)+r\delta\sum\limits_{i=0}^1{b'_iv_i(x)}\bigg) \\
(\beta+s\delta)\sum\limits_{i=0}^1{a'_iu_i(x)} &= \bigg({\beta}a_0u_0(x)+{\beta}c'_1u_1(x)+s{\delta}\sum\limits_{i=0}^1{a'_iu_i(x)}\bigg) \\
\sum\limits_{i=0}^1{a'_iu_i(x)}\sum\limits_{i=0}^1{b'_iv_i(x)} &= \bigg(a_0w_0(x)+c'_1w_1(x)\bigg)+
h(x)t(x)
\end{align}$$

第一个等式要成立，只能是 $a_0 = b'_0, c'_1 = b'_1$ ；第二个等式要成立，只能是 $a_0 = a'_0; c'_1 = a'_1$；第二个等式要成立，只能是 $a_1 = c'_1$ 。要想 verifier 验证通过，三个等式必须同时成立，所以：

$$\begin{gather}
a_0 = b'_0 = a'_0 \\
a_1 = c'_1 = a'_1 = b'_1
\end{gather}$$

也就是要想验证通过，prover 所选的 witness 都要分别等于 $(a_0, a_1)$ ，所以可见 prover 是无法在同一次证明中使用不同 witness 的。

你可能还会想，我只需要 verifier 等式左右两侧的值最终相等就行，不需要像上面那样，让每一部分相等。这也不是不可以，如果这样的话，只能调整 $h(x)$ 的值了，只为整个等式中只有 $h(x)$ 是 prover 完全可以控制的。但 prover 要想计算 $h(x)$ 的值，就会像上一小节一样，涉及到 $\alpha$ 、$\beta$ 这些随机变量的值，而 prover 是不可能知道这些随机变量的值的，所以 prover 就不可能通过计算找到一个合适的 $h(x)$ ，使得 witness 不是同一套，又通使上面左右两侧的值相等的。

## 2.9 零知识
之前我们讲过，prover 为了在某些情况下不让 verifier 猜出一些「知识」，会在生成 proof 时引入随机数 $\delta_l$ 、$\delta_r$ 和 $\delta_o$ 。在正式的证明过程中，也有为了这个目的而引入的随机数，那就是 $r$ 和 $s$ ，它们被分别用来计算 $A$ 和 $B$ ，然后通过巧妙的构造 $C$ ，让 verifier 在验证时，左右两侧跟 $r$ 和 $s$ 有关的值又能抵消掉，前面介绍 verifier 过程时已经演示过，这里就不再赘述了。 


# 3. 总结
经过差不多 10 篇文章的介绍，我们终于把整个 groth16 的证明过程、原理讲完了。groth16 是目前业界可用的零知识证明算法里相对简单的算法，但万变不离其宗，相信搞明白了 groth16 ，我们再去理解其它算法会容易很多。

现在我们回想整个过程，其实所谓零知识证明，最直接的，是证明一个多项式整除的问题。而所谓零知识，一方面是将「知识」通过等价转化为多项式来隐藏原知识（计算），另一方面通过在验证时隐藏 private 输入、只公开可以公开的 public 输入，达到隐藏知识的目的。其它所有手段、随机数，以及最终比较难理解的 $A$ 、$B$ 、$C$ ，都是因为互不信任，防止对方作弊而采取的手段。