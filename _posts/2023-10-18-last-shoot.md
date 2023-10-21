---
layout: post
title:  "08-最后的冲刺：打补丁"
categories: zkp
tags: 原创 zkp groth16 
author: fatcat22
mathjax: true
---

* content
{:toc}




我们在[上篇文章](https://yangzhe.me/2023/10/17/polynomial-transform-improve)里，对多项式转换作了很多改进，让我们的整个证明过程已经很接近最终的形式了，但仍然有些问题需要改进，这些问题我们将在这篇文章里解决。

> 这篇文章里我们会有很多假设，然后推导验证，这个过程看上去会很晦涩。所以在读这篇文章时如果实在没精力细读其实也没关系，对最终理解零知识证明的影响不大。

# 1. 引入多个 $\alpha$

在上一篇文章里，我们在 trusted setup 步骤的时候使用了同一个 $\alpha$ ；然后我们引入变量多项式 $l_i(x)$ 、$r_i(x)$ 和 $o_i(x)$ 后，也使用了这个相同的 $\alpha$ 对其进行了验证。但这样有一个问题，就是我们无法知道或阻止 prover 去替换 $L(x)$ 、$R(x)$ 和 $O(x)$ 中的变量多项式。

比如 prover 可以令 $L(x)$ 中不止有 $l_i(x)$ ，还可以使用 $r_i(x)$ 或 $o_i(x)$ 。假设 prover 令 $L(s) = v_1r_1(s) + v_2o_2(s)$ ，我们看 verifier 能否检查出来。

首先经过 QAP setup 步骤后我们得到了 $g^{s'r_1(s)}, g^{s'o_2(s)}, g^{ {\alpha}{\alpha}'s'r_1(s)}, g^{ {\alpha}{\alpha}'s'o_2(s)}$ 。那么 prover 就可以计算：

$$\begin{align}
{(g^{s'r_1(s)})}^{v_1}\cdot {(g^{s'o_2(s)})}^{v_2} = g^{v_1s'r_1(s) + v_2s'o_2(s)} = g^{s'L} = E_L \\
\tag{1}{(g^{ {\alpha}{\alpha}'s'r_1(s)})}^{v_1}\cdot {(g^{ {\alpha}{\alpha}'s'o_2(s)})}^{v_2} = g^{v_1{\alpha}{\alpha}'s'r_1(s) + v_2{\alpha}{\alpha}'s'o_2(s)} = {(g^{ {\alpha}'s'L})}^{\alpha} = E_{L'}
\end{align}$$

然后我们省略 $\delta$ 变量后，直接看 verifier 能否对 $E_L$ 和 $L_{L'}$ 验证成功：

$$\begin{align}
e(E_L,g^{\alpha{\alpha}'}) &= e(E_{L'},g) \\ &\Downarrow \\
e(g^{s'v_1r_1(s)+s'v_2o_2(s)},g^{\alpha{\alpha}'}) &= e(g^{s'{\alpha}'v_1{\alpha}r_1(s)+s'{\alpha}'v_2{\alpha}o_2(s)},g) \\ &\Downarrow \\
{(e(g,g))}^{(s'{\alpha}')(v_1{\alpha}r_1(s)+v_2{\alpha}o_2(s))} &= {(e(g,g))}^{(s'{\alpha}')(v_1{\alpha}r_1(s)+v_2{\alpha}o_2(s))}
\end{align}$$

可见 verifier 最终是验证通过的，并没有发现 prover 在 $L(x)$ 中使用了 $r_i(x)$ 和 $o_i(x)$ 。这显然不是我们想要的，我们需要想办法避免这种情况。

这种情况之所以能验证通过，是因为在 QAP setup 阶段使用了同一个 $\alpha'$ ，从而使上面等式 $(1)$ 中计算 $E_{L'}$ 时，可以抽取出共同的 ${\alpha}'$ 变量 。如果我们针对不同的变量多项式使用不同的 $\alpha'$ 值，那么在等式 $(1)$ 中 prover 就无法抽取共同的 $\alpha'$ 变量，从而无法进行这种替换计算了。

使用多个 $\alpha'$ 很简单。在 QAP setup 阶段，大家不是生成一个 $\alpha'$ 随机数，而是生成 ${\alpha}'_l$ 、${\alpha}'_r$ 、${\alpha}'_o$ 三个随机数，最终得到 $\{g^{ {\alpha}l_i(s)}, g^{ {\alpha}r_i(s)}, g^{ {\alpha}o_i(s)}, , g^{ {\alpha}'_l{\alpha}l_i(s)}, g^{ {\alpha}'_r{\alpha}r_i(s)}, g^{ {\alpha}'_o{\alpha}o_i(s)}, \}$ 。

这个时候，如果 prover 还令 $L(x)=v_1r_1(x) + v_2o_2(x)$ ，那么 ta 将得到：

$$
\begin{align}
{(g^{s'r_1(s)})}^{v_1}\cdot {(g^{s'o_2(s)})}^{v_2} = g^{v_1s'r_1(s) + v_2s'o_2(s)} = g^{s'L} = E_L \\
{(g^{ {\alpha}'_r{\alpha}s'r_1(s)})}^{v_1}\cdot {(g^{ {\alpha}'_o{\alpha}s'o_2(s)})}^{v_2} = g^{v_1{\alpha}{\alpha}'_rs'r_1(s) + v_2{\alpha}{\alpha}'_os'o_2(s)} = {(g^{v_1{\alpha}'_rs'r_1(s)+v_2{\alpha}'_os'o_2(s)})}^{\alpha} = E_{L'}
\end{align}
$$

显然使用这个新的 $(E_L, E_{L'})$ 是无法通过 verifier 的验证的。因为 ${\alpha}'_r$ 和 ${\alpha}'_o$ 并不相等。

# 2. 引入多个 $\beta$

> 确切的说，我们还没有 $\beta$ 

我们之前为了保证同一个变量 $v_i$ 在不同的约束中，它们的值必须是一样的而引入了变量多项式。比如在下面的一组约束中，每一个约束的 $\times$ 左侧都是 $a$ ：

$$\begin{align}
a \times b &= c \\
a \times d &= e \\
b \times d &= m 
\end{align}$$

通过使用变量多项式且经过 QAP setup 后，prover 只能将 $l_a(x)$ 乘上 witness $a$ 的值，得到 $al_a(x)$ ，最终得到 $L(x)=al_a(x)+bl_b(x)$ ，从而保证无论 $x=1$ 还是 $x=2$ ，前两个约束的左侧的值都是 $a$ 。

但这个方式只保证了所有约束某一侧（$\times$ 左侧或右侧，或 $=$ 右侧）变量的一致性，没法保证同一约束内变量的一致性。比如考虑下面的约束：

$$\begin{cases}
v_1 \times v_1 = v_2 \\
v_2 \times v_3 = v_4
\end{cases}$$

现有的机制并不能保证第一个约束中 $\times$ 两侧的 $v_1$ 的值必须相同，或者第一个约束等号右侧的 $v_2$ 与 第二个约束 $\times$ 左侧的 $v_2$ 的取值相同。比如 prover 完全可以这样取值：

$$\begin{cases}
2 \times 3 = 6 \\
4 \times 5 = 20
\end{cases}$$

在当前的验证步聚中 ，上面的取值 verifier 也是可以验证通过的，但这显然不是原始的约束表达的意思，所以我们必须想办法约束 prover 。

之前我们通过用变量多项式乘以变量的方式（$vil_i(s)$）限制不同约束等式中的同一变量必须是同一个值，我们也可以用类似的方法，限制相同约束等式中的同一变量必须是同一个值，那就是 $v_i(l_i(s) + r_i(s) + o_i(s))$ 。由于 prover 是在加密的值上进行计算，所以应该是：

$$
g^{v_i(l_i(s)+r_i(s)+o_i(s))}
$$

然后把它们相乘，可以得到：

$$
\prod_\limits{i=1}^{n}g^{v_i(l_i(s)+r_i(s)+o_i(s))} = \prod_\limits{i=1}^{n}{(g^{l_i(s)})}^{v_i} \cdot \prod_\limits{n=1}^{n}{(g^{r_i(s)})}^{v_i} \cdot \prod_\limits{i=1}^{n}{(g^{o_i(s)})}^{v_i}=g^L \cdot g^R \cdot g^O = g^{L+R+O}
$$

为了表示方便，我们定义一个新的多项式 $Z$ ，令 $g^Z = g^{L+R+O}$。根据之前我们的证明过程，prover 本来就会计算 $g^L$、$g^R$ 和 $g^O$ ，并且会传给 verifier 。所以如果 prover 在计算 $g^L$、$g^R$ 和 $g^O$ 时是对的，即同一个变量 $v_i$ 在 $L$ 、$R$ 和 $O$ 中使用的是同一个值，那么 $g^L \cdot g^R \cdot g^O$ 肯定等于 $\prod_\limits{i=1}^{n}g^{v_i(l_i(s)+r_i(s)+o_i(s))}$ 。但 verifier 要如何对这个进行验证呢？

由于 verifier 没有 $v_i$ 的值，所以 $g^Z = \prod_\limits{i=1}^{n}g^{v_i(l_i(s)+r_i(s)+o_i(s))}$ 肯定必须由 prover 提供。这就带来一个问题，即 $g^Z$ 本身就是由 $g^{L+R+O}$ 推导出来的，而 prover 有足够的自由随便计算 $g^Z$ ，那 prover 无论如何怎么得到的 $g^L$、$g^R$ 和 $g^O$ ，prover 只要令 $g^Z = g^L \cdot g^R \cdot g^O$ ，那肯定相等，即使同一变量 $v_i$ 在 $L$ 、$R$ 和 $O$ 中取不同的值。所以为了让 verifier 对 $g^Z$ 的验证有意义，我们必须想办法限制 prover 对 $g^Z$ 的计算方法，prover 必须得使用 $g^L$ 、$g^R$ 和 $g^O$ 参与 $g^Z$ 的计算，但又不能是简单的拼凑出来就行了。要如何才能做到呢？

回想一下我们之前的文章里提到了 KEA ，当时有这样一段描述：
>总之，引入 $\alpha$ 就是引入了一种校验方式，可以让 prover 按要求的那样进行计算，否则就会被 verifier 检查出来

现在我们面临的正是这样一个问题。当时 $\alpha$ 指的是一个在 setup 阶段生成的随机变量。因此，这里我们也可以考虑引入一个随机变量，让 prover 正确的计算 $g^Z$ ，而不仅仅是使用 $g^L$ 、$g^R$ 和 $g^O$ 拼凑出 $g^Z$ 。我们把这个随机变量命名为 $\beta$ 。

那我们要如何使用这个 $\beta$ 呢？还记得在证明过程中，有一个 `QAP setup` 的过程吗？这个过程中，多方参与对 $g^{l_i(s)}$ 、$g^{r_i(s)}$ 和 $g^{o_i(s)}$ 进行了加密。同样的，我们也可以让多方生成随机数 $\beta$ ，对 $g^{l_i(s)+r_i(s)+o_i(s)}$ 进行加密，变成 $g^{ {\beta}(l_i(s)+r_i(s)+o_i(s))}$ （注意这里并不是 $g^{ {\beta}l_i(s)}$ 、$g^{ {\beta}r_i(s)}$ 和 $g^{ {\beta}o_i(s)}$ ），为了表示方便，我们定义一个新的变量 $z_i$ ，令 $g^{z_i} = g^{ {\beta}(l_i(s)+r_i(s)+o_i(s))}$ 。

$g^{z_i}$ 作为 prover key 传给 prover 后，prover 就要用这些值来计算 $g^Z$ ，即：

$$
g^Z = \prod_\limits{i=1}^{n}{(g^{z_i})}^{v_i}
$$

由于 $g^{z_i}$ 是在 `QAP setup` 过程中产生的，prover 只能把它作为一个整体的数进行运算（而不能跟刚才能样，拆成 $\prod_\limits{i=1}^{n}{(g^{l_i(s)})}^{v_i} \cdot \prod_\limits{n=1}^{n}{(g^{r_i(s)})}^{v_i} \cdot \prod_\limits{i=1}^{n}{(g^{o_i(s)})}^{v_i}$ ，即 $g^L \cdot g^R \cdot g^O$）。但如果 prover 是诚实的，$g^Z$ 就会等于 ${(g^L \cdot g^R \cdot g^O)}^{\beta}$ ，因为：

$$\begin{gather}
g^Z = \prod_\limits{i=1}^{n}{(g^{z_i})}^{v_i} = \\
\prod_\limits{i=1}^{n}{(g^{ {\beta}(l_i(s)+r_i(s)+o_i(s))})}^{v_i} = \\
\prod_\limits{i=1}^{n}{(g^{ {\beta}l_i(s)})}^{v_i} \cdot \prod_\limits{n=1}^{n}{(g^{ {\beta}r_i(s)})}^{v_i} \cdot \prod_\limits{i=1}^{n}{(g^{ {\beta}o_i(s)})}^{v_i} = \\
g^{ {\beta}(L+R+O)} = \\
{(g^L \cdot g^R \cdot g^O)}^{\beta}
\end{gather}$$

注意这里与刚才的区别，虽然$g^Z = {(g^L \cdot g^R \cdot g^O)}^{\beta}$ ，但 prover 没有办法使用 $g^L$ 、$g^R$ 和 $g^O$ 拼凑出 $g^Z$ ，因为 ta 并不知道 $\beta$ 的值，所以 ta 只能使用 ${(g^{z_i})}^{v_i}$ 老老实实计算（很显然计算 ${(g^L \cdot g^R \cdot g^O)}^{\beta}$ 是需要未加密的 $\beta$ 值的）。 ${(g^{z_i})}^{v_i}$${(g^{z_i})}^{v_i}$ 可以保证不管是 $l_i$ 、$r_i$ 还是 $o_i$ ，它们使用的都是 $v_i$ ；如果 prover 在 $L$ 、$R$ 或 $O$ 中的 $v_i$ 值不同，那上面这个推导是无法成立的，当然也肯定会被 verifier 发现。

到这里，问题几乎被解决了，但仍然会有一个小问题，即如果只使用一个 $\beta$ ，prover 仍然有可能可以作弊。例如对于某个变量 $v$ （为了清晰，我们去掉了下标 $i$ ，只选择了某一个变量（如 $v_1$）且用 $v$ 来表示 ），本来我们期望：

$$(vl(s)+vr(s)+vo(s)){\beta} = v{\beta}(l(s)+r(s)+o(s))$$

如果 prover 使用的不是同一个变量，这个等式就变成了：

$$(v_ll(s)+v_rr(s)+v_oo(s)){\beta} = v{\beta}(l(s)+r(s)+o(s))$$

如果 $l(s) \neq r(s) \neq o(s)$ 时，这个等式要成立只能 $v_l = v_r = v_o = v$。 但如果 $l(s) \neq r(s) \neq o(s)$ 不成立时，例如 $l(s) = r(s)$ 时，这个等式变成了：

$$\tag{2}(v_ll(s)+v_rl(s)+v_oo(s)){\beta} = v{\beta}(l(s)+l(s)+o(s))$$

即：

$$\tag{3}(v_l + v_r)l(s)+v_oo(s) = 2vl(s) + vo(s)$$

这个等式要成立，只要下面这两个等式成立即可：

$$\begin{cases}
v_l + v_r = 2v \\
v_o = v
\end{cases}$$

即只要 $v_l + v_r = 2v_o$ 成立即可。这个等式成立并一定要必须要 $v_l = v_r = v_o$ （例如 $v_l = 5$，$v_r = 3$ ，$v_o = 4$）

因为存在一定的概率 $l(s) = r(s)$ （也就是说存在一定的概率 $l(s) \neq r(s) \neq o(s)$ 不成立），所以 prover 可以作弊的概率还是有的。之所以可以作弊，是因为在上面等式 $(2)$ 向 $(3)$ 转换时，$\beta$ 可以作为公约数，直接从等号两边去掉。如果不能直接去掉，也是可以 prover 这样作弊的。

所以这跟上一小节里引入多个 $\alpha$ 类似的，这里我们也要引入多个 $\beta$ 来防止 prover 作弊。如果有多个 $\beta$，比如 $${\beta}_l$$ 、$${\beta}_r$$ 和 $${\beta}_o$$ ，那么等式 $$(2)$$ 就变成了：

$$v_l{\beta}_ll(s)+v_r{\beta}_rl(s)+v_o{\beta}_oo(s) = v_{\beta}({\beta}_ll(s)+{\beta}_rl(s)+{\beta}_oo(s))$$

虽然 $l(s) = r(s)$ ，但这个等式要成立，只能 $v_l = v_r = v_o = v_{\beta}$ 。

所以，我们要用 $\beta_l$ 、$\beta_r$ 和 $\beta_o$ 代替 $\beta$。在 `QAP setup` 过程中，每方都生成 3 个随机数 $\beta_l$ 、$\beta_r$ 和 $\beta_o$ ，然后生成 prove key $\{g^{z_i} = g^{\beta_ll_i(s) + \beta_rr_i(s) + \beta_oo_i(s)}\}_{i \in \{1,2...,n\}}$ 和 verification key $\{g^{ {\beta}_l}, g^{ {\beta}_r}, g^{ {\beta}_o}\}$ 。

然后 prover 计算  $g^Z = \prod_\limits{i=1}^{n}{(g^{z_i})}^{v_i}$ ，并在提交给 verifier 验证时，把 $g^Z$ 和 $g^L, g^R, g^O$ 一起提交。

verifier 拿到数据后，验证 $g^Z$ 与 $g^L$ 、$g^R$ 和 $g^O$ 的关系：

$$
e(g^L, g^{ {\beta}_l}) \cdot e(g^R, g^{ {\beta}_r}) \cdot e(g^O, g^{ {\beta}_o}) = e(g^Z, g)
$$

即：

$$e(g,g)^{ {\beta}_lL + {\beta}_rR + {\beta}_oO} = e(g,g)^Z$$

所以虽然 verifier 不知道 $L$ 、$R$ 和 $O$ 是用哪些值生成的，但如果生成的时候相同变量的值不一样，上面 verifier 的验证肯定无法通过。

接下来我们再看看这一小节开始的那个例子，原始的约束是：

$$\begin{cases}
v_1 \times v_1 = v_2 \\
v_2 \times v_3 = v_4
\end{cases}$$

但 prover 并不诚实，在 witness 中 ta 的取值是：

$$\begin{cases}
2 \times 3 = 6 \\
4 \times 5 = 20
\end{cases}$$

可以得到 $L(s)$ 、$R(s)$ 和 $O(s)$：

$$\begin{align}
L(s) &= 2l_1(s) + 4l_2(s) \\
R(s) &= 3r_1(s) + 5r_3(s) \\
O(s) &= 6o_2(s) + 20o_4(s)
\end{align}$$

其中以下变量表达式恒等于 0 ，在 $g^{z_i}$ 中是可以忽略的：

$$
\begin{cases} l_3(x) = 0 \\ l_4(x) = 0 \end{cases} \quad 
\begin{cases} r_2(x) = 0 \\ r_4(x) = 0 \end{cases} \quad
\begin{cases} o_1(x) = 0 \\ o_3(x) = 0 \end{cases}
$$

所以 $g^Z$ 值和计算过程为（由于 prover 给 $v_1$ 和 $v_2$ 取了两个不同的值，但在计算 $g^Z$ 时只能用其中一个，所以我们下面的计算暂时不给 $v_1$ 和 $v_2$ 赋具体的值） ：

$$\begin{align}
& g^Z \\
&= \prod_\limits{i=1}^{4}{(g^{z_i})}^{v_i} \\
&= \prod_\limits{i=1}^{4}{(g^{ {\beta}_ll_i(s) + {\beta}_rr_i(s) + {\beta}_oo_i(s)})}^{v_i} \\
&= {(g^{ {\beta}_ll_1(s) + {\beta}_rr_1(s) + {\beta}_oo_1(s)})}^{v_1} \cdot {(g^{ {\beta}_ll_2(s) + {\beta}_rr_2(s) + {\beta}_oo_2(s)})}^{v_2}\cdot {(g^{ {\beta}_ll_3(s) + {\beta}_rr_3(s) + {\beta}_oo_3(s)})}^5 \cdot {(g^{ {\beta}_ll_4(s) + {\beta}_rr_4(s) + {\beta}_oo_4(s)})}^{20} \\
&= {(g^{ {\beta}_ll_1(s) + {\beta}_rr_1(s)})}^{v_1} \cdot {(g^{ {\beta}_ll_2(s) + {\beta}_oo_2(s)})}^{v_2} \cdot {(g^{ {\beta}_rr_3(s)})}^5 \cdot {(g^{ {\beta}_oo_4(s)})}^{20} \\
&= g^{ {\beta}_l(v_1l_1(s)+v_2l_2(s))} \cdot g^{ {\beta}_r(v_1r_1(s)+5r_3(s))} \cdot g^{ {\beta}_o(v_2o_2(s)+20o_4(s))}
\end{align}$$

现在有了 $L(s)$ 、$R(s)$ 、$O(s)$ 和 $g^Z$ ，verifier 可以开始验证了：

$$\begin{align}
& e(g^L, g^{ {\beta}_l}) \cdot e(g^R, g^{ {\beta}_r}) \cdot e(g^O, g^{ {\beta}_o}) \\
&= e(g,g)^{ {\beta}_lL} \cdot e(g,g)^{ {\beta}_rR} \cdot e(g,g)^{ {\beta}_o} \\
&= e(g,g)^{ {\beta}_l(2l_1(s)+4l_2(s)) + {\beta}_r(3r_1(s)+5r_3(s)) + {\beta}_o(6o_2(s)+20o_4(s))}
\end{align}$$

另一方面：

$$
e(g^Z,g) = e(g,g)^{ {\beta}_l(v_1l_1(s)+v_2l_2(s)) + {\beta}_r(v_1r_1(s)+5+r_3(s)) + {\beta}_o(v_2o_2(s)+20o_4(s))}
$$

如果要通过验证，则需要：

$$\begin{align}
& {\beta}_l(2l_1(s)+4l_2(s)) + {\beta}_r(3r_1(s)+5r_3(s)) + {\beta}_o(6o_2(s)+20o_4(s)) = \\
& {\beta}_l(v_1l_1(s)+v_2l_2(s)) + {\beta}_r(v_1r_1(s)+5+r_3(s)) + {\beta}_o(v_2o_2(s)+20o_4(s))
\end{align}$$

由于 prover 给了 $v_1$ 和 $v_2$ 不同的取值，所以 $v_1$ 的可能取值是 2 和 3 ，$v_2$ 的可能取值是 6 和 4 。但无论怎么取值，上面的等式都无法成立，读者可以自己代入验证一下。总之，prover 的作弊无法通过 verifier 的验证。


# 3. 引入 $\gamma$

前面我们费了好大的劲，引入了三个 $\alpha$  和三个 $\beta$ ，就是为了阻止 prover 作弊。但是不是这样就可以了呢？很遗憾的是，不行，prover 仍然可以随意的修改 $L(x)$ 、$R(x)$ 或 $O(x)$ 的构成。我们举两个例子。

第一个例子，假设我们有以下约束：

$$\begin{cases}
\begin{align}
v_1 \times 1 &= v_2 \\
3v_1 \times 1 &= v_3
\end{align}
\end{cases}$$

prover 按正常逻辑构造变量多项式，所以 $l_{v_1}(1) = 1$ ，$l_{v_1}(2) = 3$ 。但在构造 $L(x)$ 时，ta 作假了，令：$L(x) = v_1l_1(x) + 1$ ，如此一来，$L(1) = v_1 + 1$ ；$L(2) = 3v_1+1$ （其实这里加几都行，我们拿加 1 举例子） ，所以 prover 事实上证明的是：

$$\begin{cases}
\begin{align}
(v_1+1) \times 1 &= v_2 \\
(3v_1+1) \times 1 &= v_3
\end{align}
\end{cases}$$

显然这跟原来要证明的不一样了（一个不太明显的地方是，原来约束里包含了 $v_3 = 3v_2$ 这个逻辑，prover 作假后，这个逻辑没有了），但在目前的机制下，prover 仍然能通过证明。我们来看一下这是如何做到的。

首先 prover 作假得到 $L(x)=v_1l_1(x)+1$ ；然后正常计算 $R(x)$ 和 $O(x)$ 。由于 $R(x)$ 和 $O(x)$ 是正常的，我们主要看看涉及到 $L(x)$ 的计算和验证是如何操作的：
prover : 
- 计算 $h(x) = \frac{L(x) \cdot R(x) - O(x)}{t(x)}$ 
- 计算 $g^{L(s)} = {(g^{l_1(s)})}^{v_1} \cdot g^1=g^{v_1l_1(s)+1}$ 
- 计算 $g^{L'(s)}  = {(g^{ {\alpha}_ll_1(s)})}^{v_1} \cdot g^{ {\alpha}_l}=g^{ {\alpha}_l(v_1l_1(s)+1)}$ 
- 计算 $g^{Z(s)} = \prod_\limits{i=1}^{3}{(g^{ {\beta}_ll_i(s)+{\beta}_rr_i(s)+{\beta}_oo_i(s)})}^{v_i} \cdot g^{ {\beta}_l}=g^{ {\beta}_l(v_1l_1(s)+1) + {\beta}_rR(s)+{\beta}_oO(s)}$
verifier:
- 验证 $e(g^{L'},g) = e(g^L,g^{ {\alpha}_l})$ ，即 $e(g^{ {\alpha}(v_1l_1(s)+1)},g)=e(g^{v_1l_1(s)+1},g^{ {\alpha}_l})$ ，验证通过
- 验证 $e(g^L,g^{ {\beta}_l}) \cdot e(g^R,g^{ {\beta}_r}) \cdot e(g^O,g^{ {\beta}_o})=e(g^Z,g)$ ，即 $e(g,g)^{ {\beta}_l(v_1l_1(s)+1) + {\beta}_rR(s)+{\beta}_oO(s)}=e(g^{ {\beta}_l(v_1l_1(s)+1)+{\beta}_rR(s)+{\beta}_oO(s)},g)$ ，验证通过。
可见 prover 作假后，verifier 并没有发现。

我们再来看另外一个类似的例子。假如我们有以下约束：

$$v_1 \times v_1 = v_2$$

这里很显然两个 $v_1$ 必须是同一个值，我们之前引入三个 $\beta$ 也是为了保证这一点。但不幸的是，目前并没能真的保证这一点。比如 prover 可以令第一个 $v_1 = 2$ ，$v_2 = 10$ ，然后令 $R(x)=2r_{v_1}(x)+3$ ，注意当 $x=1$ 时，$R(1) = 5$ ，$L(s) \times R(1) = O(1)$ 也是成立的，只不过这时不再证明的是 $v_{1}^2 = v_2$ ，而是完全不同的两个数相乘等于 $v_2$ 。

与第一个例子类似，只不过这个例子是改了 $R(x)$ ，给 $R(x)$ 加了一个值（假设为 $c$）。所以 prover 也可以在计算 $g^{R'}$ 时额外乘以 ${\big(g^{\alpha}\big)}^c$ ，在计算 $g^Z$ 时额外乘以 ${\big(g^{\beta}\big)}^c$  ，依然也可以通过 verifier 的验证。

从上面这两个例子中不难发现，prover 之所以可以任意的给 $L(x)$ 、$R(x)$ 和 $O(x)$ 加上额外的值，是因为 ta 知道 $g^{\alpha}$ 和 $g^{\beta}$ 的值，所以我们只要想办法让 prover 不知道这俩值中的任意一个，但 verifier 依然可以用它们进行验证，就能解决这个问题了。

方法很简单，我们引入一个新的随机值 $\gamma$ ，使用它加密 $\beta$ 就可以了。所以有了 $\gamma$ 以后，setup 阶段我们得到的值是

$$..., g^{ {\beta}_ll_i(s)}, g^{ {\beta}_rr_i(s)}, g^{ {\beta}_oo_i(s)}, g^{ {\beta}_l\gamma}, g^{ {\beta}_r\gamma}, g^{ {\beta}_o\gamma}, g^{\gamma}$$

这时我们再来看第一个例子，即 prover 作假得到 $L(x)=v_1l_1(x)+1$ 。由于 $g^{ {\beta}_ll_i(s)+{\beta}_rr_i(s)+{\beta}_oo_i(s)}$ 是 setup 阶段就计算好的，所以上面的例子里 prover 只能这样计算 $g^Z$ ：

$$
g^{Z(s)} = \prod_\limits{i=1}^{3}{(g^{ {\beta}_ll_i(s)+{\beta}_rr_i(s)+{\beta}_oo_i(s)})}^{v_i} \cdot g^{ {\beta}_l\gamma}=g^{ {\beta}_l(v_1l_1(s)+\gamma) + {\beta}_rR(s)+{\beta}_oO(s)}
$$

但 prover 有两个选择计算 $g^L$。第一种方式是跟前第一个例子一样：

$$g^{L(s)} = {(g^{l_1(s)})}^{v_1} \cdot g^1=g^{v_1l_1(s)+1}$$

但这样的话将通不过 verifier 对 $g^Z$ 的验证：

$$\begin{gather}
e(g^L,g^{ {\beta}_l\gamma}) \cdot e(g^R,g^{ {\beta}_r\gamma}) \cdot e(g^O,g^{ {\beta}_o\gamma}) \ne e(g^Z,g^{\gamma}) \\ \Downarrow \\
e(g,g)^{ {\gamma}({\beta}_l(v_1l_1(s)+1) + {\beta}_rR(s)+{\beta}_oO(s))} \ne e(g^{ {\beta}_l(v_1l_1(s)+\gamma)+{\beta}_rR(s)+{\beta}_oO(s)},g^{\gamma}) \\ \Downarrow \\
e(g,g)^{ {\beta}_l(v_1l_1(s)+1) + {\beta}_rR(s)+{\beta}_oO(s)} \ne e(g,g)^{ {\gamma}({\beta}_l(v_1l_1(s)+\gamma)+{\beta}_rR(s)+{\beta}_oO(s))}
\end{gather}$$

如果 prover 用另一种方式计算 $g^L$：

$$g^{L(s)} = {(g^{l_1(s)})}^{v_1} \cdot g^{\gamma}=g^{v_1l_1(s)+\gamma}$$

但 ta 无法使用 $\gamma$ 计算 $g^{L'}$ ，只能这么计算 ：

$$g^{L'(s)}  = {(g^{ {\alpha}_ll_1(s)})}^{v_1} \cdot g^{ {\alpha}_l}=g^{ {\alpha}_l(v_1l_1(s)+1)}$$

如此一来 $g^L$ 虽然能通过 verifier 对 $g^Z$ 的验证，但无法通过对 $g^{L'}$ 的验证：

$$\begin{gather}
e(g^L, g^{ {\alpha}_l}) \ne e(g^{L'},g) \\ \Downarrow \\
e(g^{v_1l_1(s)+\gamma},g^{ {\alpha}_l}) \ne e(g^{ {\alpha}_l(v_1l_1(s)+1)})
\end{gather}$$

所以无论如何，使用了 $\gamma$ 后，prover 再也无法修改 $L(x)$ 、$R(x)$ 和 $O(x)$ 作假了。

最后一说明一点的是，这里不能同时使用 $\gamma$ 对 $\alpha$ 和 $\beta$ 加密，否则 prover 就可以额外乘以 $g^{\gamma}$ 来计算 $g^L$ ，额外乘以 $g^{ {\alpha}_l\gamma}$ 计算 $g^{L'}$ ，依然可以通过验证，那引入 $\gamma$ 就没什么意义了。

同时也要注意，变量多项式 $l(x)$ 、$r(x)$ 和 $o(x)$ 不能是 0 阶的，例如 $l_1(x)= 1$ ，$r_1(x)=0$ ，$o_1(x)=0$ 的情况下，prover 就能从 $g^{ {\beta}_ll_1(s)+{\beta}_rr_1(s)+{\beta}_oo_1(s))}$ 推导出 $g^{ {\beta}_l}$ 的值。


# 4. 优化

之前的一通操作，我们成功阻止了 prover 可能的一些作假行为，但这也令 verifier 引入了一些额外的性能开销，主要是为了验证 $g^Z$ ，跟之前相比：
- verifier 多了 4 个双性线配对的计算 ：$e(g^L,g^{ {\beta}_l\gamma}) \cdot e(g^R,g^{ {\beta}_r\gamma}) \cdot e(g^O,g^{ {\beta}_o\gamma})=e(g^Z,g^{\gamma})$
- verification key 中多了 4 个参数：$g^{ {\beta}_l\gamma}, g^{ {\beta}_r\gamma}, g^{ {\beta}_o\gamma}, g^{\gamma}$ 

这里最重要的就是 verifier 多了 4 个运行开销比较大的双性线配对。我们知道 verifier 验证越快越好、性能开销越小越好，所以能提高一点是一点。

很明显多的这 4 个双性线配对运算是为了验证 $g^Z$ ，更具体的说，主要是为了在加密域将 $L$ 、$R$ 和 $O$ 分别与加密随机值相乘。那么我们就可以想办法，能不能让这个相乘的运算不通过双线性配对运算来操作，而是直接运算完成后，再用双线性配对运算与 $e(g^Z, g^{\gamma})$ 比较呢？

为了清楚，我们这里把 $g^Z$ 与 $g^L$ 、$g^R$ 和 $g^O$ 的关系及验证等式再写一遍：

$$\begin{gather}
g^Z = g^{ {\beta}_lL + {\beta}_rR + {\beta}_oO} \\
e(g^L,g^{ {\beta}_l\gamma}) \cdot e(g^R,g^{ {\beta}_r\gamma}) \cdot e(g^O,g^{ {\beta}_o\gamma})=e(g^Z,g^{\gamma})
\end{gather}$$

一个思路是，我们可以尝试先将 $g^L$ 和 $g^{ {\beta}_l{\gamma}}$ 结合（因为 verification key 中只有使用 $\gamma$ 加密的 $g^{\beta}$），得到 $g^{ {\beta}_l{\gamma}L}$ 、$g^{ {\beta}_r{\gamma}R}$ 和 $g^{ {\beta}_o{\gamma}O}$ ，再将它们直接相乘 ，即 $g^{ {\beta}_l{\gamma}L} \cdot g^{ {\beta}_r{\gamma}R} \cdot g^{ {\beta}_o{\gamma}O}$，就可以得到 $g^{ {\gamma}({\beta}_lL + {\beta}_rR + {\beta}_oO)}$ 了，从而也可以与 $e(g^Z, g^{\gamma})$ 比较了。 但因为 $g^L$ 和 $g^{ {\beta}_l{\gamma}}$ 都是加密值，我们无法直接将它们相乘或相加得到 ${\beta}_l{\gamma}L$ （ 这也是当初引入双性线配对的原因） 。所以这里的关键就是，我们怎么能不使用双性线配对，得到 $g^{ {\beta}_l{\gamma}L}$ 、$g^{ {\beta}_r{\gamma}R}$ 和 $g^{ {\beta}_o{\gamma}O}$ 呢？

我们知道指数运算有一个性质：${(g^a)}^b = g^{ab}$，所以对于 $g^{ {\beta}_l{\gamma}L}$ 来说，我们是否可以把它变成 ${(g^{ {\beta}_l{\gamma}})}^L$ ，也就是说改变了基底，不再是 $g$ ，而是令 $g_l$ 为新的基底，且 $g_l = g^{ {\beta}_l{\gamma}}$  。照这个思路，我们有：

$$\begin{align}
g_l &= g^{ {\beta}_l{\gamma}} \\
g_r &= g^{ {\beta}_r{\gamma}} \\
g_o &= g^{ {\beta}_o{\gamma}}
\end{align}$$

我们看看这么做是不是行得通。

首先对 $g^L$ 和 $g^{L'}$ 的计算和验证是没有影响的，拿 $g_l^L$ 来举例，不管基底怎么变，$g_l^L = \sum\limits_{i=1}^{n}{(g_l^{l_i(s)})}^{v_i}$，${g_l}^{L'}=\sum\limits_{i=1}^{n}{(g^{ {\alpha}_ll_i(s)})}^{v_i}$ ，计算方式不变；验证也不变，$e(g_l^L, g_l^{ {\alpha}_l})=e(g_l^{L'}, g_l)$ 也能验证通过（注意这里新基底中的 $\gamma$ 没用到，没啥意义）。

然后. 看对 $L \times R = O + th$ 这个等式（计算也是不受影响的，我们就不啰嗦了）：

$$\begin{gather}
e(g_l^L, g_r^R) = e(g_o^t, g^h)e(g_o^O,g) \\ \Downarrow \\
e(g,g)^{ {\beta}_l{\gamma}{\beta}_r{\gamma}LR} = e(g,g)^{ {\beta}_o{\gamma}th + {\beta}_o{\gamma}O}
\end{gather}$$

可是上面等式要想验证通过，必须 ${\beta}_o = {\beta}_l{\beta}_r$ 才行。那我们就让它们相等（反正都是加密后的随机数），所以现在基底就变成了：

$$\begin{align}
g_l &= g^{ {\beta}_l{\gamma}} \\
g_r &= g^{ {\beta}_r{\gamma}} \\
g_o &= g^{ {\beta}_l{\beta}_r{\gamma}}
\end{align}$$

这样一改，对 $g^L$ 和 $g^{L'}$ 的计算和验证还是没有影响，且 $L \times R = O + th$ 也可以验证通过了（同样也可以看到，$\gamma$ 对于 $L \times R = O + th$  的计算和验证也没啥意义）。

最后看看对 $g^Z$ 的影响。由于现在不是同一个基底了，所以 prover key 里无法包含 $g^{ {\beta}_ll_i(s)+{\beta}_rr_i(s)+{\beta}_oo_i(s)}$ 用于计算 $g^Z$ 了，而 prover 可以直接用 $g_l^{l_i(s)}$ 、 $g_r^{r_i(s)}$ 和 $g_o^{o_i(s)}$ 计算：
$$g^Z = \prod\limits_{i=1}^{n}{(g_l^{l_i(s)} \cdot g_r^{r_i(s)} \cdot g_o^{o_i(s)})}^{v_i} = g_l^L \cdot g_r^R \cdot g_o^O$$
但这样一来，就失去了 $g^Z$ 的意义，我们本来是要检查约束间同一变量的取值是相同的，但如果使用上面的方式计算，没有任何约束，因为 $g_l^L$ 、$g_r^R$ 和 $g_o^O$ 是 prover 自己计算出来的，可以随便计算，只要把它们乘起来就是 $g^Z$ ，verifier 的验证肯定能通过。所以我们仍然要对参与计算 $g^Z$ 的值加密。

回想之前我们为了保证约束间同一变量的取值是相同的，我们引入了三个 $\beta$ 变量，现在我们又遇到了同样的问题，那我们可以使用同样的解决方案，再引入一个随机变量 A，用来加密 $g^Z$ 的计算要用到的值；也是与之前类似，我们仍要引入一个类似 $\gamma$ 的随机变量 B ，对 A 进行加密。由于我们已经非常习惯 $\beta$ 的 $\gamma$ 的用途了，所以我们这里仍然把 A 命名成 $\beta$ ，把 B 命名成 $\gamma$ 。那我们就要把新基底 $g_l$ 、$g_r$ 和 $g_o$ 中的变量换个名字，就叫 $\rho$ 吧；另外从前面的分析中也可以看到，新基底中的 $\gamma$ 没啥意义，所以我们就直接去掉吧。修改之后的新基底为：

$$\begin{align}
g_l &= g^{ {\rho}_l} \\
g_r &= g^{ {\rho}_r} \\
g_o &= g^{ {\rho}_l{\rho}_r}
\end{align}$$

这样一来，我们令 prover key 中提供 $g_l^{ {\beta}l_i(s)} \cdot g_r^{ {\beta}r_i(s)} \cdot g_o^{ {\beta}o_i(s)}$ ，同时 verification key 中提供对 $\beta$ 加密的值 $g^{ {\gamma}{\beta}}$ 和 $g^{\gamma}$ 。所以 prover 计算 $g^Z$ ：

$$g^Z = \prod\limits_{i=1}^{n}{(g_l^{ {\beta}l_i(s)} \cdot g_r^{ {\beta}r_i(s)} \cdot g_o^{ {\beta}o_i(s)})}^{v_i} = g_l^{ {\beta}L} \cdot g_r^{ {\beta}R} \cdot g_o^{ {\beta}O}$$

verifier 验证 $g^Z$ ：

$$\begin{gather}
e(g_l^L \cdot g_r^R \cdot g_o^O, g^{ {\gamma}{\beta}}) = e(g^Z, g^{\gamma}) \\ \Downarrow \\
e(g_l^L \cdot g_r^R \cdot g_o^O, g^{ {\gamma}{\beta}}) = e(g_l^{ {\beta}L} \cdot g_r^{ {\beta}R} \cdot g_o^{ {\beta}O}, g^{\gamma}) \\ \Downarrow \\
e(g^{ {\rho}_lL} \cdot g^{ {\rho}_rR} \cdot g^{ {\rho}_l{\rho}_rO}, g^{ {\gamma}{\beta}}) = e(g^{ {\rho}_l{\beta}L} \cdot g^{ {\rho}_r{\beta}R} \cdot g^{ {\rho}_l{\rho}_r{\beta}O}, g^{\gamma})
\end{gather}$$

显然也是可以验证通过的。

如此一来，我们为了验证 $g^Z$ ，只多引入了两次双线性配对计算 $e(g_l^L \cdot g_r^R \cdot g_o^O, g^{ {\gamma}{\beta}})$ 和 $e(g^Z, g^{\gamma})$，两个 verification key $\{g^{ {\beta}{\gamma}}, g^{\gamma}\}$  ，而 ${\rho}_l$ 和 ${\rho}_r$ 是不需要放到任何 prover key 或 verification key 中的。并且由于基底的随机化，我们不再需要引入三个 $\beta$ 来保证 $g^Z$ 的正确性了，只要一个就够了。


# 5. public inputs

还记得之前的文章中，我们曾提到过 $v_{one}$ 这个变量吗？当时是在做多项式的转换，我们引入了一个新的变量 $v_{one}$ ，但其实没有约束去保证这个变量的值一定是 1 。为了保证它的值肯定是 1 ，我们想到的办法就是让 verifier 去输入这个值。

另外，verifier 可能还想在验证时，直接输入其它的值，这可能有很多的原因，比如：
- 这些值是必须公开的、还有其它用处。比如一个证明 block 执行的 prover ，生成证明后 prover 告诉所有人这个 block 的 previous hash 和 current hash 是什么；然后 verifier 用这两个 hash 去验证 proof ，成功后则将 current hash 保存，用于验证这个 hash 是否是下一个 block 的 previous hash
- 仅仅是让人信服。比如一个证明信用分是否达标的系统，verifier 将达标的分数线输入进行验证，可以让 verifier 确认达标线是自己输入的这个，而不是其它的值

不管什么原因，都很有必要让 verifier 自己输入一些可以公开的值来进行验证，这些值被称为 `public inputs` （相反的，不公开的、由 prover 输入的值就是 private inputs）。那么这个功能是否能实现呢？以及要怎么做呢？

我们用 $g^L$ 来举例。由于 prover 使用同态加密计算，所以 verifier 可以很简单的给多项式加上一些值，如 $g^L \cdot g^{L_{ver}} = g^{L + L_{ver}}$ 。这样的话，我们可以将原来的 $L$ 多项式拆成两部分：$L(x) = L_v(x) + L_p(x)$ ，其中 $L_v(x)$ 是由 verifier 计算的，$L_p(x)$ 是由 prover 计算并提供给 verifier 进行验证的。这样一来，verifier 就可以利用 public inputs 去计算 $L$ 多项式的一部分，然后与 prover 给的合在一起，验证方式却和以前一样就可以了。

具体来说，我们令前 $m+1$ 个 witness 为 public inputs ，特别的地方是，第 0 个总是 $v_{one}$ （也就是说 verifier 总是给 $v_0$ 赋值为 1 ）；令后面 $n-m$ 个 witness 为 private inputs 。

在 setup 阶段， proving key 中的一部分变成了这样：

$$
(\{g^{s^k}\}_{k \in [0..d]}, \{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)}, g_l^{ {\alpha}_ll_i(s)}, g_r^{ {\alpha}_rr_i(s)}, g_o^{ {\alpha}_oo_i(s)}, g_l^{ {\beta}l_i(s)} \cdot g_r^{ {\beta}r_i(s)} \cdot g_o^{ {\beta}o_i(s)}\}_{i \in \{m+1,...,n\}})
$$

由于 prover 仍然要使用正常的 $L(x)$ 计算 $h(x)$ ，所以所有的 $g^{s^k}$ 还是需要的；但 prover 只需要提供给 verifier $g_l^{L_p(s)}$ 就可以了（$g_l^{ {\alpha}_lL_p(s)}$ 以及多项式 $R$ 、$O$ 类似的），所以上面 proving key 中后面那一串，只要提供 $\{m+1,...,n\}$ 这个范围内的值就可以了。

类似的，在 setup 阶段 verification key 需要增加这些值：

$$(\{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)}\}_{i \in \{0,...,m\}})$$

注意 $i$ 的取值范围是 $\{0,...,m\}$ ，也就是说我们刚才说的，witness 被拆成了两部分，前 $m+1$ 个作为 public inputs 由 verifier 计算。并且这里没有 $g_l^{ {\alpha}_ll_i(s)}$ 等值，因为 verifier 验证不需要。

prover 在生成 proof 时，先计算 $h(x) = \frac{L(x) \cdot R(x) - O(x)}{t(x)}$ ，注意这里的 $L(x)$ 等多项式是完整的，即 $L(x) = L_v(x) + L_p(x)$ 。最终 prover 给 verifier 验证的值有：

$$(g_l^{L_p(s)}, g_r^{R_p(s)}, g_o^{O_p(s)}, g_l^{ {\alpha}_lL_p(s)}, g_r^{ {\alpha}_rR_p(s)}, g_o^{ {\alpha}_oO_p(s)}, g^{Z(s)}, g^{h(s)})
$$

可以看到除了 $g^Z$ 和 $g^h$ ，其它值都是一部分。

verifier 在开始验证前，先计算自己需要计算的部分，如 $g_l^{L_v(s)} = g_l^{l_0(s)} \cdot \prod\limits_{i=1}^{m}{(g_l^{l_i(s)})}^{v_i}$ ；其它 $g_r^{R_v(s)}$ 和 $g_o^{O_v(s)}$ 类似。由于 $v_0$ 总是等于 1 ，所以这里把 $g_l^{l_0(s)}$ 单列出来。
然后 verifier 开始验证，其它的验证不变，比如验证 $g_l^{L_p(s)}$ 和 $g_l^{ {\alpha}L_p(s)}$ ，跟以前完全一样；只是对于 $L \times R = O + th$ 这个的验证，需要加上 verifier 自己计算的值：

$$e(g_l^{L_v(s)} \cdot g_l^{L_p(s)}, \ g_r^{R_v(s)} \cdot g_r^{R_p(s)}) = e(g_o^t, g^h) \cdot e(g_o^{O_v(s)} \cdot g_o^{O_p(s)}, g)$$

如此一来，我们就引入了 public inputs ，可以让 verifier 指定一些输入值，从而让零知识证明用途更加广泛、更加可信。


# 6. 再聊零知识

在之前的文章里，我们聊到过，prover 为了防止 verifier 发现一些自己想要隐藏的知识，会用随机数 $\delta$ 加密生成的值传给 verifier ，让 verifier 验证。最开始的时候，我们验证的是 $f(s) = t(s)h(s)$，使用 $\delta$ 加密则变成了 ${\delta}f(s) = t(s){\delta}h(s)$ ，即 ${(g^{f(s)})}^{\delta}= g^{t(s)}\cdot {(g^{h(s)})}^{\delta}$ 。后来验证改成了 $L(s)R(s) = O(s) + t(s)h(s)$ ， 我们需要把 $L$ 、$R$ 、$O$ 和 $h$ 都加密，所以 $O$ 和 $h$ 需要使用 ${\delta}^2$ 加密，即 ${\delta}L(s){\delta}R(s) = {\delta}^2O(s) + t(s){\delta}^2h(s)$ ，即 $e({(g^L)}^{\delta}, {(g^R)}^{\delta}) = e({(g^O)}^{ {\delta}^2}, g) \cdot e(g^t, {({g^h})}^{ {\delta}^2})$   。这是我们之前的证明过程所做的。

但这样也有个问题，只使用一个 $\delta$ 时，你可能已经猜到了（之前遇到过好几次只使用一个随机变量引发的问题），即 verifier 在某些情况下还是可以猜出一些「知识」，例如：
1. 如果 $g^L = g^R$ ，那即使使用 $\delta$ 加密，verifier 也可以轻易发现，因为是同一个 $\delta$ ，那么 $g^{ {\delta}L}=g^{ {\delta}R}$ 也很容易被发现。
2. 类似的，如果 $g^L$ 和 $g^R$ 存在着类似 $g^L = {(g^R)}^i,\ i \in {1,2,3,...}$ 这样的关系，也可以很快被 verifier 发现。
3. 其它多项式之间的关系，也很容易被发现，例如如果 $e(g^{ {\delta}L},g^{ {\delta}R}) = e(g^{ {\delta}^2O},g)$ ，那么就可以知道 $L \cdot R = O$ 。

可以看到只使用一个 $\delta$ 无法彻底解决隐藏 prover 的「知识」的问题。改进方法也很简单，使用多个 $\delta$ 就可以了：

$${\delta}_lL(s) \cdot {\delta}_rR(s) = {\delta}_oO(s) + t(s)({\Delta} \odot h(s))$$

其中 $\Delta$ 代表我们将要应用到 $h(s)$ 的随机数的值，$\odot$ 代表某种运算，即我们如何将 $\Delta$ 与 $h(s)$ 相结合；这两个符号目前还不确定，需要我们找出来，既能解决刚才单一 $\delta$ 的问题，又可以让 verifier 在不改变验证方式的情况下，验证 $L \cdot R = O + th$ 。

但有一个问题我们忽略了，即 verifier 的 public inputs 。$L(s)$ 是由 $L_v(s)+L_p(s)$ 组成的，如果 verifier 想要验证 ${\delta}_lL(s) \cdot {\delta}_rR(s) = {\delta}_oO(s) + t(s)({\delta}_l{\delta}_rh(s))$ ，它必须将 $L_v(s)$ 也乘上 ${\delta}_l$ 才行；$R_v(s)$ 和 $O_v(s)$ 类似。但 ${\delta}_l$ 是保密的，prover 不可能把这些值给 verifier ，所以 verifier 就无法使用这个方式验证了。

所以我们不能通过将 ${\delta}_l$ 与 $L(s)$ 相乘加密 $L(s)$ ，$R(s)$ 和 $O(s)$ 也是类似的。那么我们就试着改成加法，因为加法在同态加密的计算里也是被支持的：

$$(L(s)+{\delta}_l) \cdot (R(s) + {\delta}_r) = (O(s) + {\delta}_o) + t(s)({\Delta} \odot h(s))$$

现在的重点就是找出 $\odot$ 是什么运算，以及 $\Delta$ 的具体值是啥。我们首先假设 $\odot$ 是乘法，看看是否行得通。此时 $\Delta$ 的值为：

$$\begin{gather}
\Delta = \\ \\
\frac{(L(s)+{\delta}_l)(R(s)+{\delta}_r) - (O(s) + {\delta}_o)}{t(s)h(s)} = \\ \\
\frac{L(s)R(s)-O(s) + {\delta}_rL(s) + {\delta}_lR(s) + {\delta}_l{\delta}_r - {\delta}_o}{t(s)h(s)} = \\ \\
1 + \frac{ {\delta}_rL(s) + {\delta}_lR(s) + {\delta}_l{\delta}_r - {\delta}_o}{t(s)h(s)}
\end{gather}$$

我们可以将每个 $\delta$ 乘上 $t(s)h(s)$ ，如令 ${\delta}_l = {\delta}'_lt(s)h(s)$ ，即 ${\delta}'_l$ 才是真正的单独的一个随机数，这样上面最终的式子就可以整除了：

$$\Delta = 1 + {\delta}'_rL(s) + {\delta}'_lR(s) + {\delta}'_l{\delta}'_rt(s)h(s) - {\delta}'_o$$

但是别忘了我们需要在加密域里计算 $\Delta \times h(s)$ ，也就是 $g^{\Delta h(s)}$ ，由于 $\Delta$ 里包含 $L(s)$ 等，所以要计算 $g^{\Delta h(s)}$ 只有使用双线性配对才能做到。但这又引出另外一个问题，在之前的文章里第一次接触到双线性配对时我们就提过：「经过双线性配对计算生成的值，不能再参与双线性配对的计算」，即如果 $a = e(x, y)$ ，那么 $e(a, b)$ 是无法进行运算的，不管 $b$ 是什么值。但这里计算出来的 $g^{\Delta h(s)}$ 是需要给 verifier 在双线性配对中进行验证中，所以我们又不能使用双线性配对计算 $g^{\Delta h(s)}$ 。所以总得来说，我们无法计算 $g^{\Delta h(s)}$ 。

既然 $\odot$ 是乘法这条路走不通，我们再看看 $\odot$ 是加法的情况。即：

$$\tag{4} (L(s)+{\delta}_l) \cdot (R(s) + {\delta}_r) = (O(s) + {\delta}_o) + t(s)({\Delta} + h(s))$$

所以：

$$\begin{gather}
\Delta = \\ \\
\frac{L(s)R(s) + {\delta}_rL(s) + {\delta}_lR(s) + {\delta}_l{\delta}_r - O(s) - {\delta}_o - t(s)h(s)}{t(s)} \\ \\
= \frac{ {\delta}_rL(s) + {\delta}_lR(s) + {\delta}_l{\delta}_r - {\delta}_o}{t(s)}
\end{gather}$$

为了能令上面的式子整除，我们使用跟刚才一样的技巧，将每个 $\delta$ 乘上 $t(s)$ ，如令 ${\delta}_l = {\delta}'_lt(s)$ ，即 ${\delta}'_l$ 才是真正的单独的一个随机数 。这样 $\Delta$ 就变成了：

$$\Delta = {\delta}'_rL(s) + {\delta}'_lR(s) + {\delta}'_l{\delta}'_rt(s) - {\delta}'_o$$

这时在加密域里，$g^{\Delta}$ 是很好计算的，等于 ${(g^{L(s)})}^{ {\delta}'_r} \cdot {(g^{R(s)})}^{ {\delta}'_l} \cdot {(g^{t(s)})}^{ {\delta}'_l{\delta}'_r} \cdot g^{-{\delta}'_o}$ 。而 $g^{ {\Delta} + h(s)}$ 也很好计算，等于 $g^{\Delta} \cdot g^{h(s)}$ 。所以 $\odot$ 是加法的情况是行得通的；我们将得到的 $\Delta$ 的值代回上面的等式 $(4)$ ，稍加整理可以得到：

$$\begin{align}
L(s) \cdot R(s) &+ \underline{t(s)({\delta}'_rL(s)+{\delta}'_lR(s) + {\delta}'_l{\delta}'_rt(s) - {\delta}'_o)} = \\
O(s) + t(s)h(s) &+ \underline{t(s)({\delta}'_rL(s)+{\delta}'_lR(s) + {\delta}'_l{\delta}'_rt(s) - {\delta}'_o)}
\end{align}$$

可以看到，其实就是在 $L \cdot R = O + th$ 的基础上，在等号两侧都加上了 $t(s)({\delta}'_rL(s)+{\delta}'_lR(s) + {\delta}'_l{\delta}'_rt(s) - {\delta}'_o)$ 。

最后需要强调的是，$\Delta + h(s)$ 是作为一个整体传给 verifier 的，verifier 根本不知道 $\Delta$ 的值，所以 prover 在计算 $h(s)$ 时，应该是 $h(s) = \frac{L(s) \cdot R(s) - O(s)}{t(s)h(s)} + \Delta$ ；类似的， $L_p(s) + {\delta}_l$ 也是作为一个整体传给 verifier 的，只不过是在加密域里将 ${\delta}_l$ 加到 $L_p(s)$ 里；最后别忘了，${\delta}_l = {\delta}'_lt(s)$ ，${\delta}'_l$ 才是一个单纯的随机数，${\delta}_r$ 和 ${\delta}_o$ 也是类似这样的。



# 7. 总结
这篇文章我们聊了很多防止 prover 各种作恶的手段。如文章开头所说，在这个过程中，我作了很多的推导，所以看起来会很繁杂、晦涩。不过好消息是我们很快就要到达终点为啦，加油！