### 如何实现一个去中心化的 Dropbox 存储

原标题：Secret Sharing and Erasure Coding: A Guide for the Aspiring Dropbox Decentralizer

作者：Vitalik Buterin   &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;译者：李康

本文翻译自 https://blog.ethereum.org/2014/08/16/secret-sharing-erasure-coding-guide-aspiring-dropbox-decentralizer/

在去中心化计算的应用中，有一个激动人心的应用，在过去的一年里引起了相当大的兴趣，那就是受激励的去中心化在线文件存储系统的概念。目前，如果你想你的文件或者数据安全地在云端备份，你有3种选择：(1). 上传它们到自己的服务器，(2). 使用一个中心化的应用，如 Google drive 或者 Dropbox，或者是 (3). 使用已经存在的去中心化的应用，如 Freenet。这些方法都有它们自己的缺点：第一种方法有着昂贵的建立和维护费用；第二种方法依赖于一个单可信任实体，并且常常涉及重大价格上涨；第三种方法速度慢，对每一位用户在空间容量方面有着很高的限制，因为它依赖于用户自愿奉献存储空间。受激励的文件存储协议有潜力成为第四种方法，通过无中心化地激励执行者 (存储用户数据的客户) 参与其中，提供高容量存储与高质量服务。

大量的平台，包括 [Storj](http://storj.io/)、[Maidsafe](http://maidsafe.net/)，某种程度上，[Permacoin](http://cs.umd.edu/~amiller/permacoin.pdf) 与 [Filecoin](http://filecoin.io/)，正在尝试处理这个问题，该问题在某种意义上看起来是简单的，所有的工具要么已经存在了，要么正在构建的过程中，我们所需要的就是实现而已。然而，该问题其中的一小部分尤其重要：我们如何合适地引进冗余性？冗余对于安全来说至关重要，尤其是在一个去中心化的网络中，业余爱好者与临时性用户占大部分，我们绝对不能依赖于单节点保持在线。我们可以简单地复制用户数据，让一些节点存储单独的拷贝，问题是：我们能做的更好吗？事实证明，我们当然可以。

#### Merkle Trees 与 Challenge-Response 协议

在我们进入最为重要的冗余性部分之前，我们首先讲解一些更容易的部分：我们如何创建一个至少激励一个实体保持文件的最为基本的系统？没有激励的系统中，问题将会变得更加容易，你上传文件，等待其它的用户来下载它，当你再次需要该文件的时候，通过文件的哈希来发出一个查询请求。如果我们想引进激励机制，问题某种程度上变得更加困难，但是在大事的计划中，依旧不是那么难。

在文件存储的上下文中，有两种实体你可以激励。第一种是你发出一个下载文件的请求时，实际向你发送文件的实体。这很容易可以做；最好的策略是一种类似于简单的 tit-for-tat 游戏，发送者发送 32 kb，你发送 0.0001 个币，然后发送者发送另一 32 kb，以此类推。注意在没有冗余以及大文件的情形下，这个策略是极易受到敲诈勒索攻击的。一个文件的 99.99% 对于你来说是毫无用处的，所以存储者有机会敲诈你，让你为文件的最后一部分付出高昂的费用。对该问题最聪明的解决办法是，让文件是自冗余的，使用一种特殊的编码来扩展该文件，如文件 11.11% 自冗余，那么该文件的任意 90% 都可以来恢复原文件的内容，并且对存储者隐藏该文件的自冗余百分比。然而，随后针对不同的目的，我们将讨论一个非常类似的算法。所以现在，接受该问题已被解决。

第二种我们可以激励的行为实体是长期存储该文件的实体。该问题有些困难，如何在不真正传输整个文件的情况下，证明你存储着该文件？幸运地是，有一个不是很难实现的解决方案，使用在加密经济中，被用来构建信誉的结构：Merkle trees。

![haha](/images/2016/10/haha.jpg)

准确来说，某些情况下，Patricia Merkle 是更好的，尽管古老原始的 Merkle 树也能胜任。

基本的方法就是这个。首先，将文件分散为小块。或许每一块在 32 bytes 与 1024 bytes 之间，添加全 0 的块，直到块数量达到 `n = 2 ^ k` (添补的步骤是可以避免的，但是它使得算法编码和解释更加简单)。接下来，我们构建树。重命名 `n` 个块为 `chunk[n]` 到 `chunk[2n-1]`，并且使用以下规则重新构建块  `1` 到 `n-1`：`chunk[i] = sha3([chunk[2*i], chunk[2*i+1]])`。这使得你计算出块 `n/2` 到 `n-1`，然后 `n/4` 到 `n/2 - 1`，然后重复上述步骤直至得到一个 *root*， `chunk[1]`。

![Merkle](/images/2016/10/merkle.png)

现在，注意如果你仅仅存储根节点，忘记 `chunk[2]...chunk[2n-1]`，存储其它块的实体通过几百字节就能向你证明，他们拥有某个特定块。这个算法相对来说比较简单。首先，我们定义一个函数 `partner(n)`，如果 `n` 是奇数，输出 `n-1`，如果 `n` 是偶数，输出 `n+1` - 简单来说，就是给定一个块，找出和该块链接在一起产生父区块的块。然后，如果你想证明 `chunk[k]` (`n <= k <= 2n - 1`) 的所有权，提交 `chunk[partner(k)]`，`chunk[partner(k/2)]` (除法在这里意味着向下取整，例如 `11 / 2 = 5`)，`chunk[partner(k/4)]` 等等直到 `chunk[1]`，与真正的 `chunk[k]` 一起。本质上，我们提供的是从块 `k` 到根节点的整个分支。验证者使用 `chunk[k]` 与 `chunk[partner(k)]` 构建 `chunk[k/2]`，使用刚生成的 `chunk[k/2]` 与 `chunk[partner(k/2)]` 构建 `chunk[k/4]`，直到验证者得到 `chunk[1]`，该树的根节点，如果根节点与验证者自存的根节点匹配，那么该证据就是有效的，否则无效。

![Merkle_verify](/images/2016/10/merkle_verify.png)

上图是块 10 的证据，包括 (1). chunk 10，(2). chunk 11 (`11=partner(10)`)，4 (`4 = partner(10/2)`)，3 (`3 = partner(10/4)`)。验证过程从块 10 开始，使用它们的 partner 来产生父区块，直到产生根节点 chunk 1，看是否匹配验证者节点自存储的根节点。

注意证据隐式地包含了索引--在哈希之前有时你需要将 partner 加到左边，有的时候是右边，如果用来验证证据的索引是不同的，那么该证据将不再匹配。因此，如果我要求一个 `chunk 422` 的证据，你提供了一个即使有效的 `chunk 587` 的证据，我也会注意到出了错误。同样地，如果没有 Merkle Tree 的整个相关的部分，那么你没有任何办法提供一个有效证据；如果你试图传输伪造的数据，在某一点，哈希将不会匹配，导致最终的根节点会是不同的。

现在，让我们回顾一下这个协议。按照上述描述，我在文件之外构建了一棵 Merkle 树，然后上传给某个实体。然后，每 12 个小时，我找一个位于 `[0, 2^k-1]` 中