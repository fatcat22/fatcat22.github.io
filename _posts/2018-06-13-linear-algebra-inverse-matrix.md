---
layout: post
title:  "线性代数：2.逆矩阵"
categories: 数学
tags: 原创 数学 线性代数
excerpt: 这是我从一个程序员的角度学习线性代数的笔记和体会
mathjax: true
author: fatcat22
---

* content
{:toc}

### 逆问题
在之前的文章中，我们曾介绍过矩阵和逆矩阵。在这一篇文章中，我们会专门学习一下逆矩阵的相关知识。我们已经知道对于 $Ax = y$ ，使 $Cy = x$ 成立的矩阵 $C$ 就是 $A$ 的逆矩阵，记为 $A^{-1}$ 。那么为什么要求逆矩阵？什么情况下会用到逆矩阵呢？

其实逆矩阵对应的就是逆问题，所谓逆问题，就是知道了结果，想要求得原因的问题。比如某个实景经过一个倒霉的失焦矩阵拍摄，变成了模糊的图片，我们想恢复成清晰的原景；再比如我们想恢复一段不清晰的音频。这些都是从结果推原因的应用，都有逆矩阵的身影。  
当然在这些实际应用中不会只求一个逆矩阵就能完美解决问题，但等以后我们遇到这种问题再专门写文章说吧，这里我们主要关注逆矩阵。

那么一个矩阵一定存在逆矩阵吗？其实这是这篇文章的一个主要问题。对于一个程序员来说，除非自己要写一个库，不然只要知道哪个库的哪个方法可以计算逆矩阵就可以了。但我们通知对矩阵是否可逆的学习，可以加深对矩阵的了解，也算是练习“内功”了。

### 逆矩阵的存在性
显然不是所有的矩阵都存在逆矩阵。从映射的角度看，所谓 $A^{-1}$ 是指将 $y = Ax$ 的中所有的 $y$ 点再映射回 $x$ 点，那么显然如果 $A$ 不是方阵，则肯定不存在逆矩阵。比如 $A$ 是一个 $2{\times}3$ 矩阵，即从3维映射到2维，那么映射过程中肯定丢了一个维度的数据，再想映射回去的时候，这些丢失的数据就找不到了；再比如 $A$ 是一个 $3{\times}2$ 矩阵，即从2维映射到3维，那么在3维空间中，肯定存在没有与2维空间中的点对应的点，想将这些点映射回去的时候，也是不行的。即使是方阵，也可能不存在逆矩阵。比如：

$$
A = \begin{pmatrix} 1 & 2 & 3 \\ 2 & 4 & 6 \\ 10 & 20 & 30\end{pmatrix}
$$

那么什么情况下存在逆矩阵呢？

#### 矩阵的核与像
先说概念。
在 $y = Ax$ 中，使所有 $Ax = 0$ 的向量（点） $x$ 的集合，称为 $A$ 的核，记为 $KerA$ ；对于给定的 $A$，选取不同的 $x$ ，在 $A$ 的作用下， $y=Ax$ 组成的集合，称为 $A$ 的像，记为 $ImgA$ 。

核与像的概念初看起来比较抽象，其实也很好理解。  
从概念看， $KerA$ 是一个向量（点）的集合，这些向量满足 $Ax = 0$ ，其实这是“压缩扁平化”的一种数学表达而已。我们假设存在两个点 $x$ 和 $x^{\prime}$ ，经过 $A$ 的映射都到达了 $y_0$ 点，那么可知 $Ax = Ax^{\prime} = y_0$ ，从而有 $Ax - Ax^{\prime} = 0$ ，即 $A(x - x^{\prime}) = 0$ 。此时我们令 $z = x - x^{\prime}$ ，则 $Az = 0$ 。 $z$ 向量正好符合 $KerA$ 的定义。  
像的概念较核更好理解一些。仔细看上面的定义，我们选取任意的 $x$ 映射到 $y$ ，等于将原空间中所有点映射到了新的空间中，这些映射后的点所组成的图像就称为“像”（有点像拍照片的意思....）。

或者说， $KerA$ 是将要被压缩的图像（ $KerA$ 指代的是被压缩的点的集合，这些点组成了被压缩的图像）； $ImgA$ 是映射以后图像。

核与像还有维度的概念，分别记作 $dim\ KerA$ 和 $dim\ ImgA$ 。核的维度是指核的集合中的点所组成的图形的维度，假如 $KerA$ 中所有点都在同一条同直线上，那么 $dim\ KerA$ 为1；假如 $KerA$ 中所有点在一个平面中，那么 $dim\ KerA$ 为2。  
像的维度也比较好理解，就是被称为“像”的图像的维度。比如对于

$$A = \begin{pmatrix}1 & 0 & 0 \\ 0 & 1 & 0 \\ 0 & 0 & 0 \end{pmatrix}$$

对于任意的向量 $x = (x_1, x_2)^T$ ，映射后都是 $(x_1, x_2, 0)^T$ ，虽然映射后的点处在三维空间中，但这些点组成的图形却只是一个平面，所以 $dim\ ImgA$ 为2。

从核与像的维度出发，我们可以得到维数定理：  对于一个m $\times$ n维矩阵 $A$ 来说，有  
$dim\ KerA + dim\ ImgA = n$  
直观上理解， $A$ 是一个从n维空间到m维空间的映射，那么映射过程中损失掉的维度和映射以后的图像的维度，肯定和原维度n相等。

那么核与像的概念和逆矩阵有什么关系呢？

如果 $dim\ KerA$ 不为0，那么 $A$ 肯定是不可逆的。其实这与前面以映射的角度来说明可逆性是一致的。假设矩阵 $A$ 将两个点 $a$ 和 $a^{\prime}$ 同时映射到了 $b$点，那么

$$
b = Aa\quad 且 \quad b = Aa^{\prime} \\
则 \quad 0 = Aa - Aa^{\prime} = A(a - a^{\prime}) \\
令 \quad u = a - a^{\prime} \\
则 \quad 0 = Au
$$

可见，存在非零向量 $u$ 令 $Au = 0$ 与将两个不同的点映射到同一个点的说法是一样的。

如果 $ImgA$ 中的点不足以覆盖整个目标空间，那么 $A$ 也肯定是不可逆的。这也与映射的说法等价。如果在目标空间中存在某个点不属于 $ImgA$ ，那么你想把这个点映射回原来的点是不可能的。


#### 矩阵的单射与满射
终于可以说说单射与满射了。所谓单射，是指目标空间中的点如果有原来的点与之对应，那么这个点是唯一的；所谓满射，是指目标空间中的任一点都有原来的点与之对应。

当一个矩阵即是单射又是满射时，称之为双射（我记得上学时有一个名词叫一一映射，应该也是指的这个）。

结合上面说的核与像，可以看出它们与单射和满射的概念是等价的：  
$KerA$ 中只有零向量 <=> 单射  
$ImgA$ 包含目标空间中所有点 <=> 满射

可以看到，当一个矩阵是双射时，它一定是可逆的。因为一方面，映射后的目标空间中的任一点都有映射前的点与之对应（满射）；另一方面，这个对应原点又是唯一的（单射）。所以肯定能从映射后的点映射回映射前的点。


#### 列的线性无关和瓶颈型分解

说了这么多，到底怎么才能知道矩阵是否可逆呢？别着急，其实这篇文章的内容特别简单，之所以写这么多，是为了能从各个方面来看待矩阵，更加熟悉矩阵。

对于简单的矩阵，其实用眼看就能知道是否可逆。看什么呢？首先肯定得是方阵；其次看列向量是否线性无关，也就是某些列是否可用其它列表示。比如：

$$
A = \begin{pmatrix}
  1 & 0 & 0 \\
  0 & 1 & 0 \\
  0 & 0 & 0
\end{pmatrix} \quad
B = \begin{pmatrix}
  1 & 2 & 3 \\
  5 & 10 & 15 \\
  8 & 16 & 24
\end{pmatrix}
$$

对于矩阵 $A$ 来说，很明显是不可逆的；对于矩阵 $B$ 来说，第二列等于第一列乘以2，第三列等于第一列乘以3，所以这三个列向量是线性相关的， $A$ 也不可逆。

其实对于所有列向量是线性相关的矩阵，都可以将其分解成两个矩阵相乘的形式，左边的矩阵各列就是所有线性无关的列向量，右边的矩阵是组成左边列向量的系数。这么说很抽象，看个例子就明白了。比如对于上面的矩阵 $B$ ，可以做如下分解：

$$
B = \begin{pmatrix}
  1 & 2 & 3 \\
  5 & 10 & 15 \\
  8 & 16 & 24
\end{pmatrix} \quad = \quad
\begin{pmatrix} 1 \\ 5 \\8 \end{pmatrix}
\begin{pmatrix} 1 & 2 & 3 \end{pmatrix}
$$

再比如下面这个“复杂”一点的矩阵（第三列等于第一列乘以10加上第二列）：

$$
C = \begin{pmatrix}
  1 & 1 & 11 \\
  3 & 7 & 37 \\
  5 & 8 & 58
\end{pmatrix} \quad = \quad
\begin{pmatrix}
  1 & 1 \\
  3 & 7 \\
  5 & 8
\end{pmatrix}
\begin{pmatrix}
  1 & 0 & 10 \\
  0 & 1 &  1
\end{pmatrix}
$$

这就是所谓的瓶颈型分解。


#### 矩阵的秩
在我看来秩就是人为起的一个名字，因为从概念上就可以看出来： $A$ 的秩就是 $dim\ ImgA$ ，即矩阵的像的维数。 $A$ 的秩记为 $rank A$ 。

概念很简单，那么为什么把像的维度单独拿出来起个名字呢？因为秩可以判断矩阵是否可逆。

怎么判断？以及为什么？

先说怎么判断。对于m ${\times}$ n维矩阵 $A$ ，如果 $rank A = n$ ，则 $A$ 肯定是可逆的。  
根据维数定理和秩的定义，可以知道对于m ${\times}$ n维矩阵 $A$ 来说，有 $dim\ KerA + rankA = n$ 。如果 $rank A = n$ ，那么 $KerA$ 就为0。根据秩的定义和上面我们讲单射和满射时提到的，可以看出：  
$rank A = n$ $\quad$ <=> $\quad$ $dim\ ImgA = n$ $\quad$ => 满射  
$rank A = n$ $\quad$ => $\quad$ $dim\ KerA = 0$ $\quad$ => 单射  
可以看到，当 $rank A$ = 0时，代表矩阵 $A$ 是一个双射，所以是可逆的。


### 如何求解矩阵的逆
我们不是为了考试，所以完全可以使用程序库来计算逆矩阵。但为了加深对矩阵的理解，我仍然想把手工计算逆矩阵的过程展现一遍。


#### 手工计算逆矩阵
在解线性方程组 $Ax = y$ 时，我们可以使分块矩阵 $\left(\begin{array}{c|c} A & y\end{array}\right)$ 变换为 $\left(\begin{array}{c|c} I & y^{\prime}\end{array}\right)$ 的形式，得到的 $y^{\prime}$ 就是线性方程组的解。求解逆矩阵时，我们也可以这么做。将分块矩阵 $\left(\begin{array}{c|c}A & I\end{array}\right)$ 变换为 $\left(\begin{array}{c|c}I & X\end{array}\right)$ ，此时 $X$ 就是矩阵 $A$ 的逆。

下面用一个简单的二维矩阵演示一下：
假设

$$A=\begin{pmatrix}2 & 3 \\ 1 & 4\end{pmatrix}$$

求 $A$ 的逆矩阵。

$$
\left(\begin{array}{cc|cc}
  2 & 3 & 1 & 0\\
  1 & 4 & 0 & 1
\end{array}\right) \\
第一行乘以\frac{1}{2} \quad -> \quad
\left(\begin{array}{cc|cc}
  1 & \frac{3}{2} & \frac{1}{2} & 0\\
  1 & 4 & 0 & 1
\end{array}\right) \\
第一行乘以(-1)加上第二行 \quad -> \quad
\left(\begin{array}{cc|cc}
  1 & \frac{3}{2} & \frac{1}{2} & 0\\
  0 & \frac{5}{2} & -\frac{1}{2} & 1
\end{array}\right) \\
第二行乘以\frac{2}{5} \quad -> \quad
\left(\begin{array}{cc|cc}
  1 & \frac{3}{2} & \frac{1}{2} & 0\\
  0 & 1 & -\frac{1}{5} & \frac{2}{5}
\end{array}\right)
$$

现在左边已经是一个上三角矩阵了，只要再把上三角的元素变成0就可以了：

$$
将第二行乘以-\frac{3}{2}，然后加到第一行 \quad -> \quad
\left(\begin{array}{cc|cc}
  1 & 0 & \frac{4}{5} & -\frac{3}{5}\\
  0 & 1 & -\frac{1}{5} & \frac{2}{5}
\end{array}\right)
$$

至此，右半部分的矩阵即为 $A$ 的逆矩阵，即

$$
A^{-1} = \begin{pmatrix}
  \frac{4}{5} & -\frac{3}{5}\\
  -\frac{1}{5} & \frac{2}{5}
\end{pmatrix}
$$


#### python库求解逆矩阵
numpy.linalg.inv方法可以直接得到逆矩阵，非常简单。


### 总结
这篇文章探究的是逆矩阵，但其实主要集中在可逆性的判断上。从中引出了核、像、单射、满射等概念。对于简单的情况，我们通过观察列的线性相关性就能知道是否存在逆矩阵；而对于复杂的情况，通过求矩阵的秩可以知道是否可逆。


### 参考资料
1. 《程序员的数学 - 线性代数》
