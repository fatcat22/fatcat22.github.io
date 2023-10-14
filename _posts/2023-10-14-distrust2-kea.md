---
layout: post
title:  "04-互不信任之二：KEA"
categories: zkp
tags: 原创 zkp groth16 
author: fatcat22
mathjax: true
---

* content
{:toc}



[上一篇文章](https://yangzhe.me/2023/10/13/distrust1-encrypt-random)里，我们通过使用同态加密的方式，加密了 verifier 产生的随机数，让 prover 无法计算 $t(s)$ ；verifier 的验证也变成了 $g^{f(s)}={(g^{h(s)})}^{t(s)}$ 。但这样就万事大吉了吗？

不是的，prover 仍然是有作弊的可能的。注意 verifier 的验证等式 $g^{f(s)}={(g^{h(s)})}^{t(s)}$ ，其实也等价于 $g^{f(s)}={(g^{t(s)})}^{h(s)}$ 。虽然 prover 计算不出来 $t(s)$ ，但 $g^{t(s)}$ 是可以计算出来的。所以 prover 可以随便选一个数 $r$ ，令 $Z_h=g^r$ ，$Z_f=(g^{t(s)})^r$ ，然后把 $Z_f$ 和 $Z_h$ 传给 verifier 验证。verifier 此时验证的是 $Z_f=(Z_h)^{t(s)}$ ，也即 $(g^{t(s)})^r=(g^r)^{t(s)}$ ，显然这个等式是成立的，verifier 可以验证通过。但 prover 并没有真正计算多项式的值，只是随便选了个 $r$ 而已。

造成这个问题原因，是 verifier 并不知道 prover 是不是真的进行计算了。所以 verifier 要想办法确认 prover 给的值，真的是通过计算多项式得来的，而不是通过其它方式（比如随机值）来的。

实话说，并没有一种方式可以让 verifier 能够「确认 prover 给的值，真的是通过计算多项式得来的，而不是通过其它方式来的」（或者存在这种方式，只是我还不知道）。但换一种思路，verifier 虽然不可能知道 prover 给的值是怎么来的，但在给定 prover 两个值让 prover 分别计算的情况下，verifier 可以确认 prover 对这两个值的计算是不是使用了同一种计算方式（比如同一多项式）。所以在这种限制下，prover 就不能使用猜测这种作弊方式了。下面我们就来了解一下这种限制方式：KEA 。

# 1. KEA
KEA 的全称是 `Knowledge-of-Exponent Assumption` ，这也是一种加密方式吧，这里我们并不会作特别深入的说明，只知道怎么用这种方式就行了。感兴趣的小伙伴可以去搜索详细解释的相关论文和文章。

假设 Alice 有一个值 $s$ 想让 Bob 帮忙做一个指数运算，具体指数值是多少 Alice 不关心，Alice 关心的是 Bob 只是帮忙做了个指数运算，而没有做其它运算（比如对 $s$ 做减法）。所以 Alice 可以：
1. 生成一个随机数 $\alpha$ （alpha）
2. 计算 $s' = s^\alpha$ 
3. 将 $(s, s')$ 发送给 Bob

Bob 收到 $(s, s')$ 后，开始它的运算：
1. 随机选择一个数 $c$ 
2. 计算 $b=s^c$ ，$b'={(s')}^c$ 
3. 将 $(b, b')$ 发给 Alice

Alice 收到 $(b, b')$ 后，验证：$b^\alpha=b'$ ，验证通过可以确认 Bob 只对 $s$ 作了指数运算。

$b^\alpha=b'$ 意味着什么呢？将 $b$ 和 $b'$ 的计算过程代入这个等式，可以得到 ${(s^c)^\alpha={(s')}^c}$  ，再将 $s'$ 的计算过程代入等式，可以得到 ${(s^c)^\alpha={(s^\alpha)}^c}$ 。显然只要 Bob 老老实实做了指数运算，不管 $c$ 的值是多少，这个等式都是成立的。但如果没做指数运算，比如做了个减法运算，那么 $b^\alpha=b'$ 就变成了 $(s-c)^\alpha=(s^\alpha-c)$  ，显然这个等式是不成立的。

这里还有一个关键是，Bob 无法知道 Alice 随机生成的 $\alpha$ 值是多少。如果这个值被 Bob 知道了，那它依然可以构造任意的 $b$ 和 $b'$ ，满足 $b^\alpha=b'$ 。

我们再来看看使用 KEA 这种方式，prover 依然使用文章开头提到的作弊方式会怎么样。首先 verifier 传给值 $g^{s^i}$ 和 ${(g^{s^i})}^\alpha$ 让 prover 验证。prover 依然使用随机数 $r$ ：
- 使用 $g^{s^i}$ 计算时，得到  $Z_h=g^r,\ Z_f=(g^{t(s)})^r$
- 使用 ${(g^{s^i})}^\alpha$ 计算时，得到  $Z'_h=g^r,\ Z'_f= {\left(\left(g^{t(s)}\right)^r\right)}^\alpha$
然后将 $(Z_f,Z_h)$ 和 $(Z'_f, Z'_h)$ 分别传回给 verifier 验证。

这时 verifier 除了验证 $Z_f={(Z_h)}^{t(s)}$ 和 $Z'_f={(Z'_h)}^{t(s')}$ 外，还要验证：
- ${(Z_f)}^\alpha=Z'_f$
- ${(Z_h)}^\alpha=Z'_h$

但显然由于 $Z_h$ 与 $Z'_h$  相等，都等于 $g^r$ ，所以这个验证肯定不能过（如果选用不同的随机数 $r'$ ，那么 ${(Z_f)}^\alpha=Z'_f$ 又将验证不通过）。

总之，引入 $\alpha$ 就是引入了一种校验方式，可以让 prover 按要求的那样进行计算，否则就会被 verifier 检查出来。我们后面还会遇到这样的情形，到时还是通过引入更多类似 $\alpha$ 值的方式来解决。

# 2. 改进后的证明过程
知道了 KEA 是什么了以后，我们可以使用这种方式限制 prover 的计算方式，让 ta 也只能进行指数运算，并且两次运算使用的方法（多项式）是一样的，不能不计算（使用随机数）和乱计算（使用除指数运算外的运算方式）。（巧的是，使用同态加密后，prover 的运算就真的只是指数运算，世界上哪有这么巧的事^__^）

> 为了表示方便，后面对于：
> 1. 使用给定值 $s$ 计算某个多项式 $f(s)$ 的值的表示方式，都使用 $f$ 来代替 $f(s)$ 
> 2. 加密计算 $f(s)$ 时，使用 $E_f$ 代替 $E(f(s))$
> 这样的形式在一坨算式里更简单，也更好理解。

**前提**:
1. prover 知道多项式 $f(x)$, $h(x)$, $t(x)$ 
2. verifier 知道多项式 $t(x)$

**verifier**:
1. verifier 生成一个随机数 $s$ 和 $\alpha$  
2. 分别计算 $E(s^i)=g^{s^i}$ 和 ${E(s^i)}^{\alpha}=g^{ {\alpha}s^i}$, $i\in\{0,1,...,n\}$，$n$ 为多项式的最高次幂。
3. 把所有 $(E(s^i), {E(s^i)}^\alpha)$ 传给 prover

**prover**:
1. 接收 $E(s^i)=g^{s^i}$ ，${E(s^i)}^\alpha=g^{ {\alpha}s^i}$， $i\in\{0,1,...,n\}$
2. 使用 $E(s^i)$ 计算：  
   $$\begin{align}E_f&=(g^{s^n})^{a_n}(g^{s^{n-1}})^{a_{n-1}}...(g^{s^1})^{a_1}g^{a_0}=g^f  \\
   E_h&=g^h \end{align}$$
3. 同样使用 ${E(s^i)}^\alpha$ 计算 ：  
   $$\begin{align}E_{f'}&={(g^{ {\alpha}s^n})}^{a_n}{(g^{ {\alpha}s^{n-1}})}^{a_{n-1}}...{(g^{ {\alpha}s^1})}^{a_1}{(g^{\alpha})}^{a_0}={({(g^{s^n})}^{a_n}{(g^{s^{n-1}})}^{a_{n-1}}...{(g^{s^1})}^{a_1}g^{a_0})}^{\alpha}={g^f}^\alpha=g^{ {\alpha}f} \\
   E_{h'}&=g^{ {\alpha}h} \end{align}$$
4. 将计算得出的值 $(E_f, E_{f'}, E_h, E_{h'})$  传回给 verifier.

**verifier**:
1. 使用未加密的值计算 $t(s)$
2. 验证 ${(E_f)}^{\alpha}=E_{f'}$ 。也即验证 ${(g^f)}^{\alpha}=g^{ {\alpha}f}$
3. 验证 ${(E_h)}^{\alpha}=E_{h'}$ 。也即验证 ${(g^h)}^{\alpha}=g^{ {\alpha}h}$
4. 验证：$E_f={(E_h)}^t$ 。也即验证  $g^f={(g^h)}^t$

> 使用现在的证明过程，如果 prover 还想随便选一个数 $r$ ，令 $Z_h=g^r$ ，$Z_f=(g^{t(s)})^r$  ，由于 prover 不知道 $\alpha$ 的值，所以它无法计算出 $Z_{h'}={(Z_h)}^\alpha$ 和 $Z_{f'}={(Z_f)}^{\alpha}$ 使其能够通过 verifier 的 第 2 和 3 步的验证。


# 3. 再次改进
现在我们已经有了很大的改进，对 prover 作了很多限制，使其无法作弊。但 prover 也可能会对 verifier 存在不信任，即 prover 会担心 verifier 是不是有方法可以知道自己的「知识」，如果 prover 的「知识」暴露的话，那就不是零知识证明了。

其实当 prover 生成的多项式阶数比较小时，verifier 是有可能通过暴力枚举的方式，计算出 prover 的多项式的。比如 prover 有一个 3 阶多项式 $f(x)=a_2x^2+a_1x^1+a_0$ 。这个多项式有 3 个系数，虽然每个系数理论上的取值范围无限大，但现实中多项式的系数很可能在一个很小的范围内。我们假设所有系数取值范围是 100 ，那么这 3 个系数共有 $100^3$ 即 100 万种可能的组合。通过 prover 给的值，verifier 是可以通过暴力枚举的方式找出这些可能的组合的；如果 verifier 和 prover 多交互几次，verifier 就可能一步步筛选这些可能的组合，最终找到正确的参数组合方式。

那怎么改进呢？产生这个问题原因是 verifier 明确知道 prover 返回的值是什么。如果让 verifier 不知道 prover 返回的具体值是什么，还可以进行验证，那就可以解决这个问题了。

根据前面我们总结的证明过程，prover 最终返回 $(E_f, E_{f'}, E_h, E_{h'})$  ，而 verifier 则验证：

$$\begin{align}
{(E_f)}^{\alpha}=E_{f'},\qquad i.e.\ {(g^f)}^{\alpha}=g^{ {\alpha}f}\\
{(E_h)}^{\alpha}=E_{h'},\qquad i.e.\ {(g^h)}^{\alpha}=g^{ {\alpha}h}\\
E_f={(E_h)}^t,\qquad i.e.\ g^f={(g^h)}^t
\end{align}$$


如果让 prover 使用 KEA 的方式加密，那 verifier 就即可以验证，又不能猜出多项式的系数了。比如 prover 计算得出 $(E_f, E_{f'}, E_h, E_{h'})$ 后，随机一个随机数 $\delta$ ，然后用 $\delta$ 分别加密这四个值得到：$$({E_f}^\delta, {E_{f'}}^\delta, {E_h}^\delta, {E_{h'}}^\delta)$$ 然后将加密后的值发给 verifier 。由于现在这四个值加密了，除非 verifier 知道 $\delta$ 的取值，否则再暴力去猜就没意义了。另一方面，verifier 仍然可以通过验证：

$$\begin{align}
{({E_f}^\delta})^{\alpha}={E_{f'}}^\delta,\qquad i.e.\ {({E_f}^\alpha})^{\delta}={E_{f'}}^\delta\\
{({E_h}^\delta)}^{\alpha}={E_{h'}}^\delta,\qquad i.e.\ {({E_h}^\alpha)}^{\delta}={E_{h'}}^\delta\\
{E_f}^\delta={({E_h}^\delta)}^t,\qquad i.e.\ {(E_f)}^\delta={({E_h}^t)}^\delta
\end{align}$$


可以看到，除了每个等式两边都进行了一个 $\delta$ 次方的指数运算，其它的与前面 verifier 的验证是一模一样的，所以 prover 使用 $\delta$  进行加密后，并不影响 verifier 的验证。

# 4. 最终的证明过程
**前提**:
1. prover 知道多项式 $f(x)$, $h(x)$, $t(x)$ 
2. verifier 知道多项式 $t(x)$

**verifier**:
1. verifier 生成一个随机数 $s$ 和 $\alpha$ 
2. 分别计算 $E(s^i)=g^{s^i}$  和 ${E(s^i)}^\alpha=g^{ {\alpha}s^i} ，$ $i\in\{0,1,...,n\}$，$n$ 为多项式的最高次幂。
3. 把所有 $(E(s^i), {E(s^i)}^\alpha)$ 传给 prover

**prover**:
1. 接收 $E(s^i)=g^{s^i}$ ，${E(s^i)}^\alpha=g^{ {\alpha}s^i}$， $i\in\{0,1,...,n\}$
2. 使用 $E(s^i)$ 计算：  
$$\begin{align}
E_f&=(g^{s^n})^{a_n}(g^{s^{n-1}})^{a_{n-1}}...(g^{s^1})^{a_1}g^{a_0}=g^f\\
E_h&=g^h
\end{align}$$  
3. 同样使用 ${E(s^i)}^\alpha$ 计算 ：  
$$\begin{align}
E_{f'}&={(g^{ {\alpha}s^n})}^{a_n}{(g^{ {\alpha}s^{n-1}})}^{a_{n-1}}...{(g^{ {\alpha}s^1})}^{a_1}{(g^{\alpha})}^{a_0}={({(g^{s^n})}^{a_n}{(g^{s^{n-1}})}^{a_{n-1}}...{(g^{s^1})}^{a_1}g^{a_0})}^{\alpha}={g^f}^\alpha=g^{ {\alpha}f}\\
E_{h'}&=g^{ {\alpha}h}
\end{align}$$
4. 取随机数 $\delta$ ，分别加密 $(E_f, E_{f'}, E_h, E_{h'})$ ，得到：  
$$\begin{align}
E'_f&={(E_f)}^\delta\\
E'_{f'}&={(E_{f'})}^\delta\\
E'_h&={(E_h)}^\delta\\
E'_{h'}&={(E_{h'})}^\delta\\
\end{align}$$    
5. 将计算得出的值 $$({E'_f}, E'_{f'}, E'_h, E'_{h'})$$ 传回给 verifier.

**verifier**:
1. 使用未加密的值计算 $t(s)$
2. 验证 $${(E'_f)}^{\alpha}=E'_{f'}$$。也即验证 ${({(E_f)}^\delta)}^\alpha={(E_{f'})}^\delta$，即 ${({(g^f)}^\delta)}^{\alpha}={(g^{ {\alpha}f})}^\delta$
3. 验证 $${(E'_h)}^{\alpha}=E'_{h'}$$ 。也即验证 ${({(E_h)}^\delta)}^\alpha={(E_{h'})}^\delta$，即 ${({(g^h)}^\delta)}^{\alpha}={(g^{ {\alpha}h})}^\delta$
4. 验证：$E'_f={(E'_h)}^t$ 。也即验证${(E_f)}^\delta={({(E_h)}^\delta)}^t$， 即 ${(g^f)}^\delta={({(g^h)}^\delta)}^t$


#  5. 总结
这篇文章我们解决了即使使用同态加密后，依然存在的一个漏洞，即 prover 依然可以通过生成随机值的方式糊弄 verifier 。解决的方法就是使用 KEA 。KEA 的核心思想是，通过随机数 $\alpha$ 产生两个相关的值 $s$ 和 $s'=s^\alpha$ ，然后让 prover 对这两个值分别进行指数运算，产生结果 $b$ 和 $b'$ ；如果 $b^\alpha=b'$ 则说明 prover 使用的是同一方式对 $s$ 和 $s'$ 进行的指数运算，verifier 至少相信 prover 没有糊编乱造，是真的进行了某种计算的。

至此，在目前的验证过程中，我们基本解决了 prover 和 verifier 互不信任的问题。当然随着后面的验证过程引入新的知识，还会产生类似的信任问题，但基本上都可以使用 KEA 的这种方式解决掉。

目前我们的验证过程是**交互式**的，即 prover 和 verifier 一问一答。但现实中这种方式是不太可行的，因为一方面这要求 prover 一直在线（这本身就是极大的限制，尤其在区块链领域），另一方面真正产生 proof 的过程很慢，而 verifier 又可能有很多个，prover 根据处理不过来。所以我们要想办法，将交互式的验证变成非交互式的，这个我们后面再说。