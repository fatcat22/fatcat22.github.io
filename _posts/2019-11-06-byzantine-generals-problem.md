---
layout: post
title:  "拜占庭将军问题"
categories: blockchain
tags: 原创 consensus byzantine 
excerpt: 接触区块链，经常会听到有人提到「拜占庭将军问题」（The Byzantine Generals Problem）。那么这到底是一个什么问题呢？这篇文章我们就来详细的探讨一下。
author: fatcat22
mathjax: true
---

* content
{:toc}





# 引言

接触区块链，经常会听到有人提到「拜占庭将军问题」（The Byzantine Generals Problem），所以这篇文章里，我们就详细探讨一下这个「问题」。

本篇文章的讨论多数来自于「拜占庭将军问题」的[原始论文](https://people.eecs.berkeley.edu/~luca/cs174/byzantine.pdf)，有一些细节参考了网上[这篇文章](https://github.com/alan2lin/byzantine_demo)。


# 什么是拜占庭将军问题

「拜占庭将军问题」来源于这样一个场景：  
拜占庭帝国的军队正在围攻一座城市。这支军队被分成了多支小分队，驻扎在城市周围的不同方位，每支小分队由一个将军领导。这些将军们彼此之间只能依靠信使传递消息（无法聚在一起开个会）。每个将军在观察自己方位的敌情以后，会给出一个各自的行动建议（比如进攻、撤退或按兵不动），但最终的需要将军们达成一致的作战计划并共同执行，否则就会被敌人各个击破。但是，这些将军中可能有叛徒，他们会尝试阻止其他忠诚的将军达成一致的作战计划。

这就是拜占庭将军的「问题」：只能依靠通信相互交流，又不知道谁是叛徒，怎么能不受叛徒们的影响，让这些忠诚的将军快速的达到一致的作战计划呢？

很显然，将这一场景套用到计算机系统中也是非常适用的：在一个分布式系统中，针对每个运算，每台独立的机器也需要最终达成一致的结果。但每台计算机之间也只能依靠网络通信（显然它们无法聚在一起开会），每台计算机都有出错的可能（被攻击，或故障），从而变成「叛徒」干扰正常的计算机达成一致。

这一问题是由 Leslie·Lamport 等人在[这篇论文](https://people.eecs.berkeley.edu/~luca/cs174/byzantine.pdf)中提出的，并在论文中给出了理论可行的解决方案和证明。不过在学习这些解决方案之前，我们还要对「拜占庭将军问题」作进一步的推导，看一下解决方案只要符合哪些要求，就能解决拜占庭将军问题。

## 推导

根据刚才的描述可以总结出，忠诚的将军们使用的算法，需要保证以下两点：  
**A. 所有忠诚的将军可以达成一致的作战计划**  
**B. 少数的叛徒不会导致忠诚的将军们无法达成一致**

（论文中第二条本来是「少数的叛徒不会导致忠诚的将军们采用错误的计划」（A small number of traitors cannot cause the loyal generals to adopt a bad plan），并在接下来对什么是「bad plan」作了一个间接的解释，我的理解是，在这里只要是不一致的行动计划，都是「bad plan」，无论是「进攻」还是「撤退」）

如果我们假设 $v(i)$ 为第 $i$ 个将军的作战计划，且每个将军使用某种方法将序列 $v(1),v(2),...,v(n)$ 转换成一个单一的作战计划。那么只要所有的将军使用相同的方法转换 $v(1),v(2),...,v(n)$ ，条件A就可以被满足。条件B可以通过某种更有弹性的方法来满足，比如获取到消息序列  $v(1),v(2),...,v(n)$  以后，通过投票的方式，少数服从多数，得出一个最终的、一致的作战计划。

问题似乎很简单就被解决了，但我们忽略了一点，想要满足上面两个条件，所有忠诚的将军必须拿到同样的「 $v(1),v(2),...,v(n)$ 」。而叛徒可能给不同的将军发送不同的消息，使这些将军拿到的是不同的消息序列，这种情况下我们刚才的解决方法就无法成立了。所以我们必须想办法保证：  
**1. 所有忠诚的将军必须拿到相同的消息序列 $v(1),v(2),...,v(n)$**

只要某个算法能保证条件1，那么条件A和条件B也是可以保证的，因而这个算法就可以解决将军们的问题。

那么如何保证条件1呢？有些将军（叛徒）可能给不同的将军发送不同的值，而要使所有忠诚的将军拿到相同的消息序列，这些忠诚的将军们就不能直接使用自己接收到的值。也就是说，将军们最终使用的 $v(i)$ 不一定就是将军$i$当初发送的原始值，即使将军$i$是一个忠诚的将军。为什么这么说呢？因为如果让将军们必须直接使用其它将军发送的原始值，那么由于叛徒会给不同将军发送不同值，所以条件1永远也无法达成。所以对于叛徒发出的值，将军们不能直接使用。但将军们无法区分谁是叛徒谁是忠诚的，所以如果不多加注意，就会导致抛弃或修改某个忠诚的将军发出原始值的情况。但这显然是不对的，对于忠诚的将军，我们应该必须使用他发出的原始值，所以我们必须保证：  
**2. 如果将军$i$ 是忠诚的，那么它发出的原始值 $v(i)$ 必须被其他忠诚的将军所采用**

条件2只提到了「如果将军$i$是忠诚的」的情况，如果将军$i$是叛徒呢？根据条件1，我们依然要保证所有忠诚的将军拿到相同的值。所以我们可以得出：  
**1'. 对于任意一个将军$i$，无论他是忠诚的还是叛徒，当他发出值时，所有的忠诚的将军都将采用相同的值，即 $v(i)$（虽然可能不是将军$i$发送的原始值）**

至此，只要某个算法能保证条件1'（对任意一个将军发出的值，其他忠诚的将军都将使用相同的值），就能保证条件1（所有忠诚的将军肯定会拿到相同的消息序列）。再加上条件2（忠诚的将军发出的原始值肯定会被其它忠诚的将军采用），就可以保证条件A和条件B的成立，因而这个算法就可以解决将军们的问题。

注意条件2和条件1'考虑的都是单个的将军发送的值的情况，而只要这两个条件成立，我们就可以保证解决将军们的问题。因此我们将问题的关键转移到了「单个的将军如何发送他的值给其它人」上来。我们把这一问题表达为「司令官发送命令给他的副官们」，得到了如下拜占庭将军问题：
> 拜占庭将军问题：  
> 一位司令官必须发送命令给他的 n - 1 个副官们，且满足条件：  
> IC1. 所有忠诚的副官获得相同的命令  
> IC2. 如果司令官是忠诚的，那么所有忠诚的副官都会获得司令官发出的原始命令

这里 IC1 对应着条件1'； IC2 对应着条件2，不同的是 IC2 特别强调了「司令官」，而条件2只说「对任意一个将军」。后面我们会看到，司令官是会轮换的，任意一个将军都可能成为司令官，所以这两个说明其实是一样的。

至此，我们得出了确定性的拜占庭将军问题的定义。只要能保证 IC1 和 IC2，就可以解决将军们的问题。

（这里可能会有人疑惑：IC1 和 IC2 到底是「问题」，还是「解决条件」。我觉得「问题」和「解决条件」是一致的。它们构成了问题，也是解决问题的关键，解决了它们，就解决了问题）


# 解决方案的不可能性

介绍完拜占庭将军问题以后，你可能会觉得其实也挺简单。不过论文中也提出了「解决方案的不可能性」（IMPOSSIBILITY RESULTS），指出在某些条件下，将军们无论如何是无法达成一致的。我们需要先对这个作一个介绍，然后再去介绍解决方案。

我们这里先说结论：在使用口头消息的情况下，不存在将军总数小于 $3m+1$ 的解决方案，能够在存在 $m$ 个叛徒的情况下解决拜占庭将军问题。也就是说，如果有 $m$ 个叛徒，则至少需要 $3m+1$ 个将军，才能让忠诚的将军们达成一致。

什么是口头消息呢？所谓口头消息，是指内容完全由发送者或转发者控制的消息。将军们使用这种消息传递信息时，叛徒可以修改自己收到的任意消息，再转发给其人，而其它人无法识别出这个消息是否被修改过。（与口头消息对应的是签名消息，即消息是经过原始发送者签名的，收到消息的人可以根据签名验证此消息是否被修改过）

下面我们简单证明一下这个结论。首先来证明最简单的情况，即当叛徒数量 m=1 时，不存在将军总数 <= 3 的解决方案，可以解决拜占庭将军问题。下面这张图我直接截取自原始论文：

![](/pic/byzantine-generals-problem/3-generals-impossibility.png)

我们先来看 Fig.1 代表的第一种情况。此时司令官是忠诚的，2号副官是叛徒。司令官给所有副官发送了「attack」命令，但副官2是叛徒，所以他欺骗副官1自己收到了「retreat」命令。因为副官1也不知道谁是叛徒，所以此时副官1会很迷茫，不知道谁说的是真的：如果他相信副官2，那么他将不会与忠诚的司令官达成一致；如果他相信司令官，倒是暂时正确了，但事情还未结束。

我们再来看 Fig.2 代表的第二种情况。此时司令官是叛徒，两个副官是忠诚的。司令官给不同的副官发送了不同的命令，副官2也如实将司令官的命令发给了副官1。但此时副官1依然会迷茫，因为他仍然收到了两个不同的命令。可以看到，对于副官1来说，当前收到的消息与刚才第一种情况下收到的消息是一样的，他依然不知道该相信谁。在第一种情况里，选择相信司令官是暂时正确的，但在当前的情况里却是错误的。

所以结合这两种情况来看，无论副官1选择相信谁，都有可能是错误的。所以可以得出结论，在只有一个叛徒但将军总量 <= 3 的情况下，无法保证忠诚的将军们达成一致。

现在我们再来看叛徒数量 m>1 的情况。m>1 时论文的证明很有意思，它将 m=1 时的每一位将军想象成同类将军的代表，比如在 Fig.1 中，司令官和副官1分别代表 $m$ 个忠诚的将军，副官2代表 $m$ 个叛徒；在 Fig.2 中，司令官代表 $m$ 个叛徒，副官1和副官2分别代表 $m$ 个忠诚的将军。因此也可以得出， $m>1$ 但将军总量 <= $3m$ 时，无法保证忠诚的将军们达成一致。

以上就是这一「解决方案的不可能性」的证明过程。但我总是感觉这个证明有些「玄乎」，感觉能理解，但好像又不是真的可以理解。论文中也提到了类似的问题，然后将严谨的证明过程指向了另一篇论文。不过我们不是作学术研究，就不继续追究下去了。


# 口头消息的解决方案

刚才我们介绍了不可能存在解决方案的情况，那么现在我们就来介绍一下，存在解决方案的时，如何解决拜占庭将军问题。

在这一小节里我们假设将军们使用「口头消息」（Oral Message）传递消息。前面我们已经介绍过，所谓口头消息，是指内容完全由发送者或转发者控制的消息。将军们使用这种消息传递信息时，叛徒可以修改自己收到的任意消息，再转发给其人，而其它人无法识别出这个消息是否被修改过。另外，我们还要对将军们的消息传递系统作如下的限制：
> A1. 每一个消息都可以在两个将军间正确的传递，传递过程中不会被修改  
> A2. 消息的接收者知道这个消息的真正发送者是谁  
> A3. 消息的缺失是能够被将军们发现的  

A1 和 A2 可以防止叛徒干扰两个将军之间的通信：A1 让叛徒无法在通信过程中修改信息；A2 让叛徒无法冒充其它将军发送消息。A3 可以防止叛徒通过不发消息的方式影响一致共识的达成。并且如果将军们发现缺少某个消息，他们可以使用统一的默认值（比如 RETREAT）来替代缺失的消息。

另外，我们还假设每个将军都可以直接发消息给其它任意一个将军，不需要中间有人代理转发。

下面我们先给出算法的定义，然后再详细解释一下算法过程。

我们首先要定义一个函数 $majority$，这个函数有这样的功能：如果大多数的 $v_i$ 等于 $v$，那么 $majority(v_1, ..., v_{n-1}) = v$。比如可以这样实现这个函数：
1. $majority$ 函数可以取出现次数最多的那个值，否则为一个默认值（如 RETREAT）
2. $majority$ 函数可以取所有值的中位数（如果这些值可以排序的话）

我们定义使用口头消息解决拜占庭将军问题的算法为 $OM(m)$，其中 $m$ 代表叛徒的数量且 $m>=0$，$n$ 代表将军的总数量且 $n>=3m+1$。 $OM(m)$ 算法如下：  

$OM(0)$:  
(1). 司令官把他的值发送给每一位副官。  
(2). 每一位副官使用从司令官那里接收到的值作为**共识值**。如果没接收到任何值则使用默认值作为共识值。

$OM(m), m > 0$:  
(1). 司令官把它的值发送给每一位副官。对于每一个副官 $i$，假设 $v_i$ 为自己从司令官那里接收到的值（如果没接收到则为默认值）。  
(2). 每一个副官 $i$ 作为新的司令官、其它所有副官（当然不包含自己和当前司令官啦）作为新的副官，执行算法 $OM(m-1)$。（在算法 $OM(m-1)$ 中新司令官 $i$ 会将他接收到的值 $v_i$ 发送他的副官们）。  
(3). 假设 $v_{i}^{\prime}$ 为第二步中的副官们对第二步中的新司令官 $i$ 发送的值达成的共识值，则当前副官们的**共识值**为 $majority(v_{1}^{\prime}, v_{2}^{\prime}, ..., v_{n-1}^{\prime})$。

这里我们要马上解释一下 **共识值** 这个概念（这个词是我自己创造的，原论文中没有）。所谓「共识值」，是指副官们一致认同的司令官发送的值。这里的关键是并不是司令官发什么值，副官们就接受并相信什么值；而是副官之间要对司令官发送的值进行「探讨」，最终形成一个一致意见，接受并相信一个相同的值，这个值就是「共识值」。如果司令官是叛徒，那么这个值不一定就是司令官发出的原始值。比如有 4 个将军，司令官是叛徒，他给其他 3 个副官发送的值分别是 a、b、c。那么 3 个副官最终的共识值应该是 $majority(a, b, c)$。

$OM(m)$ 算法使用了递归，这使它非常难以理解。但我们这里先不详细考虑算法本身的流程，而是从更高一层的角度想想，为什么会产生拜占庭将军问题，以及应该怎样应对。

产生拜占庭将军问题的原因我认为是：每个忠诚的将军不知道谁是叛徒，从而产生了对其他将军的不信任。

那怎么应对呢？既然是不知道该相信谁且多数将军是忠诚的，那么只能使用 $majority$ 函数产生一个「多数人的意见」，并服从这个意见。所以当副官们接收到司令官的命令时，副官们不知道司令官是不是叛徒，从而不能轻易相信司令官，只能依然副官们「相互沟通」后，达成一个可以相信的「共识值」。

副官们沟通的方法就是将司令官排除出去，每个副官将自己接收到的值发给其他所有人，这样每个副官就能知道司令官发给所有人值了。然后副官们就使用 $majority$ 函数从这些值中计算出一个「共识值」。

但问题是，在这些沟通的副官中，仍可能会有叛徒，他发出来的值可能根本不是从司令官接收到的值，且可能发给其他每个人的值都不一样，这样每个忠诚的副官拿到的就不是一样的序列 $v_1, ..., v_{n-1}$，还是没法达成一致的。

其实在副官们的「沟通」过程中，每当一位副官向其他人发送自己的值时，其他人也不知道这位副官是否是叛徒，因此也不应该直接相信他，而是应该所有接收他消息的人再次「相互沟通」，以便对这位副官发送的值达成共识。这里就产生了递归：每一位副官向其他人发送自己接收到的值时，也可以看作自己是「司令官」，其他人是自己的副官。其他的副官在彩纳自己发送的值之前，首先要充分的「相互沟通」达成一个共识值。

以上基本上就是 $OM(m)$ 算法的要素和递归过程。这么来看，这个算法的逻辑就比较容易理解了。

那什么时候递归结束呢？或者说，什么时候可以相信司令官发送的值，而不用再「相互沟通」了呢。在 $OM(m)$ 算法中，每递归一轮，$m$ 的值就减 1，当 $m=0$ 时，副官们就不用再「相互沟通」了，而是直接将司令官发送的值作为当前的共识值。至于为什么这样是正确的，我们一会再来说明。

下面我们通过实例，来加深对这一算法的理解。首先看一看论文中比较简单的 $m=1$ 时的例子。

首先看司令官是忠诚的情况，此时有一个副官是叛徒：
![](/pic/byzantine-generals-problem/byz-om1-sample-commander-loyalty.png)

从图中可以看到，在 OM(1) 的第 (1) 步中，司令官开始给每个副官发送一个值，本例中司令官是忠诚的，所以他给每个副官的值都是一样的，为 a。

在 OM(1) 的第 (2) 步中，每个副官各自作为司令官，进入到 $OM(m-1)$ 算法中，将自己收到的值发给其他副官。此时 $m=0$，所以其它副官把此时自己收到的值作为这一层次的共识值。这一步执行完以后，每个副官都会收到其它两个副官发送的值，再加上自己在第 (1) 步中收到的值，此时每个副官共收到了 3 个值。然后进入到下一步中。

在 OM(1) 的第 (3) 步中，每个副官利用 $majority$ 函数，计算自己收到的 3 个值。由于 L1 自己从 L0 那里收到了一个 a 值；且在 OM(0) 中从 L2 那里收到的也是 a 值，从 L3 那里收到的是 x，所以 L1 的共识值应该是 a。L2 的情况与 L1 类似，他的共识值也是 a。所以 L0、L1、L2 三个忠诚的将军达成了一致，他们的共识值都是 a。

我们再来看看司令官是叛徒的情况；
![](/pic/byzantine-generals-problem/byz-om1-sample-commander-traitor.png)

此时在 OM(1) 的第 (1) 步中，由于司令官 L0 是叛徒，所以他可能发给每个人的值都不一样（以尽最大可能达到干优忠诚将军们的目的），所以我们假设 L1、L2、L3 收到的值各不相同，分别为 a、b、c。

下面的步骤与刚才类似，我们不再赘述。总之 L1、L2、L3 分别收到了除自己这外的其他两个副官发送的值，所以他们最后每个人收到的三个值其实都是 a、b、c。当他们使用 $majority$ 函数计算共识值时，由于输入值是相同的，所以输出的肯定是同一个值。因此最终三个忠诚的将军仍然达成了一致的共识值 t，虽然这个值可能与司令官 L0 发送的值完全不一样。

 $m=1$ 时 $OM(m)$ 算法很简单，也很容易理解。以此为预热，我们来看一下复杂一些的情况，比如 $m=3$ 时，$OM(3)$ 是如何执行的。我们下面的示意图使用的图例与刚才完全一致，只是更复杂而已。由于 $OM(3)$ 的情况实在太复杂，如果全部用图示意出来，图片会非常的大，也会非常复杂，这样反而失去了图片示意的意义。因此在下面这张图中，每一个步骤我只示意了第一种情况和最后一种情况，其它情况都用省略号代替了。另外在乍用这张图理解 $OM(3)$ 算法时，建议先纵向看，即先沿着某一分支一直向下看，看明白了再看其它分支。最后再横向综合起来看。
 
 另外，也由于 $OM(3)$ 太复杂，所以这里也只示意了叛徒作为司令官的情况。忠诚将军作为司令官的情况是类似的（甚至会更简单些）。
 
 下面就是 $OM(3)$ 的示意图（如果图片太小，可以尝试点击图片看大图，或在新标签中打开图片，或图片另存为本地图片后查看）：
![](/pic/byzantine-generals-problem/byz-om3-sample.png)

虽然整个过程变复杂了，但细节上与刚才介绍的 $OM(1)$ 是相同的，因此就不每一步都说明介绍了。需要特别提出来的是，在 $OM(1)$ step (1) 中、L9 作为司令官时，最终忠诚的将军们是无法对 L9 发送的信息达成一致的共识值的（$OM(1)$ step (3) 中），但这并不妨碍忠诚将军们达到最终的一致共识。

与前面介绍过的 $OM(1)$ 中判徒作为司令官的情况类似，这里忠诚将军们虽然最终达成了一致（上图中用值 C 表示），但这个值可能与最开始司令官发出来的值不一样。

在原论文中还给出了这个算法的正确性证明。这里我们就不进行证明了，只是聊一下证明相关的思路，以及为什么当 m=0 时各将军就不再相互确认，而是直接将接收到的值作为当前的共识值。由于在 $OM$ 算法中，每进行一轮都会忽略当前司令官、剩下的副官们相互发送消息确认。那么我们设想两个极端的情况：  
第一种情况是每次忽略的都是叛徒；  
第二种情况是每次忽略的都是忠诚将军。

在第一种情况下，每次忽略的都是叛徒，因此当最后 m=0 时，一共忽略了 m 个叛徒，但总共也就 m 个叛徒。也就是说，在这种情况下当最后 m=0 时，剩下的所有将军都是忠诚的，因此他们肯定可以达成一致的共识值。

在第二种情况下，每次忽略的都是忠诚将军，因此最后当 m=0 时，一共忽略了 m 个忠诚将军，此时剩下的将军中，有 m 个是叛徒，有 m+1 个是忠诚将军。忠诚将军的数量大于叛徒的数量，因此还是可以达成一致的共识值的。

因此无论哪种情况下，在 $OM(1)$ 的第 (3) 步中，忠诚将军们肯定是可以达成一致共识的。论文中证明的关键也是无论什么时候，忠诚将军的数量总是多于叛徒的。


# 签名消息的解决方案

回看一下口头消息的解决方案，还是很复杂的。其实口头消息的解决方案之所以复杂，就是因为叛徒可以随意改更忠诚将军的消息，而别人无法发现消息被改。如果我们让忠诚将军的消息无法篡改，那么问题就变得简单多了。这就是签名消息的解决方案。

前面介绍口头消息时，我们对将军们之间的消息传递系统作了一些限制（前面的 A1 - A3）。在使用签名消息时，我们要在之前限制的基础上，再加一条：
> A4.   
> (a) 忠诚将军的签名是无法被修改的；任何改动包含了忠诚将军签名的消息的行为，都可以被发现  
> (b) 任意一个人都可以验证属于某将军的签名是不是真的是他自己签的

注意这里我们并没有对叛徒的签名作任何的限制。我们可以允许叛徒之间相互修改和伪造彼此的签名，而不被别人发现。

既然我们现在使用了消息签名，那么之前将军数量必须大于等于 $3m+1$ 才能达成共识的限制就可以去除了。事实上，我们可以让任何数量的将军团体在存在 $m$ 个叛徒的情况下达成共识（这里虽然说任意数量，但如果总数小于 $m+2$ 将没有意义。因为「达成共识」意味着两个或两个以上的人，只有一个忠诚将军或跟本没有忠诚将军谈不上「达成共识」）。

在给出解决方案之前，我们首先要定义一个函数 $choice$。这个函数输入一个值的集合，输出一个单一值。我们对这个函数有两个要求：
1. 如果集合 $V$ 由单个元素 $v$ 组成，那么 $choice(v) = v$
2. $choice(\varnothing)$ 始终为相同的默认值，比如 RETREAT。共中 $\varnothing$ 代表集合为空

另外在算法中，我们使用 $x:i$ 代表由将军 $i$ 签名的值 $x$。类似的 $v:j:i$ 代表值 $v$ 先由将军 $j$ 签名得到 $v:j$，然后 $v:j$ 又被将军 $i$ 签名，得到 $v:j:i$。

我们定义使用签名消息解决拜占庭将军问题的算法为 $SM(m)$，其中 $m$ 代表叛徒的数量且 $m>=0$。 $SM(m)$ 算法如下：  
(0). 每个将军 i 初始化自己的值集合 $V_i = \varnothing$。  
(1). 司令官将要发送的值签名，然后发送签名后的值。  
(2). 对于每个副官 $i$：  
　　　(A) 如果副官 $i$ 之前没接收过任何将军发过来的任何值，且值的形式是 $v:0$，那么：  
　　　　　(i)  将 $v$ 加入到 $V_i$ 中（此时 $V = \{v\}$）  
　　　　　(ii) 将 $v:0:i$ 发送给其他副官  
　　　(B) 如果副官 $i$ 接收到了一个形式如 $v:0:j_1:...:j_k$ 这样的值，并且 $v$ 不在 $V_i$ 中，那么：  
　　　　　(i)  将 $v$ 加入到 $V_i$ 中  
　　　　　(ii) 如果 $k<m$，那么此副官将值 $v:0:j_1:...:j_k:i$ 发送给除 $j_1, ..., j_k$ 之外的所有其它副官  
　　　(C) 如果副官 $i$ 接收到一个已经存在于 $V_i$ 中的值，则忽略它。  
(3). 对于每个副官 $i$，当自己不会再接收到更多值的时候，它将 $choice(V_i)$ 作为最终自己的共识值。


（论文中解释了第 (3) 步中如判断自己不会再接收到更多的值，但我觉得不太靠谱。这里只是算法，不是实现，所以就姑且认为可以判断吧）

可以看到，$SM$ 算法要比 $OM$ 算法详细、简单，尤其是 $SM$ 算法没有用到递归来解决问题。注意在第 (2) 步中，(A)、(B)、(C) 三种情况是互斥的，即只要执行了某一情况，就不会再去执行另一情况。可以把第 (2) 步理解成编程语言中的 switch/case 语句。

这个算法的关键是维护一个集合 $V$，如果新收到的值不在这个集合中，就将它加入进去，并对这个值签名后再将其发送给其他人；如果已存在于集合中，就忽略不管了。 仔细理解这一过程你就会发现，**$SM$ 的关键思想是通过消息不可篡改这一特性，让所有忠诚将军成为「一体」，就像同一个人一样。** 当某个值第一次被忠诚将军接收到时，无论是第 (2) 步的 情况 (A) 还是 情况 (B)，这个忠诚将军都会将这个值发送给其他所有没给这个值签过名的人；由于签名值的不可篡改，最终其他所有忠诚将军的集合 $V_i$ 中肯定也会存在这个值。因此除非消息到达不了忠诚将军那里，只要到达，就会「全体忠诚将军都有」。这样看来，无论叛徒怎么干扰，都无济于事了，因为最终忠诚将军们的集合 $V_i$ 肯定都是一样的，因此 $choice$ 最终的结果也肯定是一样的。所不同的是，如果司令官是忠诚的，那么最终所有忠诚将军的集合 $V_i$ 中肯定只有一个值，就是司令官原始发送的值；而如果司令官是叛徒，他给不同的副官发送了不同的值，那么最终所有忠诚将军的集合 $V_i$ 中会有多个值（但整个集合还是相等的）。

这里还要解释一下为什么在算法的 (2)(B)(ii) 步骤中，只有 $k<m$ 的情况下，才会将值签名后，发给其他副官。因为接收到的值 $v:0:j_1:...:j_k$ 有 $k+1$ 个签名（注意最前面还有编号为 0 的司令官的签名）。如果 $k>=m$，那么 $k+1>=m+1$，即当 $k>=m$ 时，至少有 $m+1$ 个将军对这个值签过名了，而这 $m+1$ 个将军中至少有 1 个忠诚的将军。也就是说当 $k>=m$ 时至少有 1 个忠诚的将军的集合 $V$ 中已经有这个值了。刚才我们说过，忠诚的将军们是「一体的」，只要有一个忠诚的将军有这个值了，其它忠诚将军也肯定会有这个值。所以当 $k>=m$ 时就没必要再给别人发送这个值了。

$SM$ 算法比较容易理解，所以我们这里只看一下论文中给出的简单例子即可。论文中的例子中总共有 3 个将军，其中有 1 个叛徒，并且司令官就是这个叛徒（注意这个例子并不需要如 $OM$ 算法那样需要 4 个将军才能达成共识）。如下图（图片来自原论文）：
![](/pic/byzantine-generals-problem/sm1_sample.png)

在上图中，叛徒司令官给两个副官分别发送了不同的值。副官1收到值后发现这是自己第一次收到值且是司令官发来的，于是执行步骤 (2)(A)，将 "attack" 加入到了自己的集合 $V_1$ 中，然后将 "attack":0:1 发送给副官2；类似的副官 2 在收到司令官发来的值后，也将 “retreat” 加入到自己的集合 $V_2$ 中，然后将 "retreat":0:2 发送给副官1。

副官1接收到副官2发来的消息后，发现自己的集合中没有 "retreat" 这个值，因此他将这个值加入到了自己的集合 $V_1$ 中。此时副官1的集合为 ${"attack", "retreat"}$。

副官2接收到副官1发来的消息后，也发现自己的集合中没有 "attack" 这个值，因此他也将这个值加入到了自己的集合 $V_2$ 中。此时副官2的集合为 ${"retreat", "attack"}$。

最后我们可以看到，两个忠诚的副官的集合都是一样的。因此 $choice(V_1)$ 和 $choice(V_2)$ 的值也肯定是一样的（不管这个值是什么）。所以最终忠诚的将军达成了共识。


# 总结

通过这篇文章，我们详细了解了什么是拜占庭将军问题，以及原论文给出的基于口头消息和签名消息的解决方法。「拜占庭将军问题」是区块链共识的一个基础（我觉得也是各种分布式系统的共识的基础），了解了这个问题，有利于我们以后理解其它很多的共识算法。

拜占庭将军问题的原论文虽然给出了基于口头消息和签名消息的解决方案，但这些方案并不能很好的应用于实际生产环境中（比如基于口头消息的方法，通信量太大；基于签名消息的方法，如果司令官一直是叛徒，那这个系统虽然可以达成一致，但也干不了什么正事）。因此很多前辈和大牛给出了不少其他可实际应用的解决方案，我们以后的文章会继续学习这些方案。

限于作者水平，文章中难免有错误的地方，如果发现感谢您能不吝指正。