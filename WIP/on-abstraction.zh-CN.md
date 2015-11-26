# On Abstraction

[Original post](https://blog.ethereum.org/2015/07/05/on-abstraction/) by Vitalik Buterin on July 5th, 2015

_Special thanks to Gavin Wood, Vlad Zamfir, our security auditors and others for some of the thoughts that led to the conclusions described in this post_

_特别感谢Gavin Wood, Vlad Zamfir, 我们的安全审计专家，和所有为本文中描述到的结论贡献了想法的人们。_

One of Ethereum’s goals from the start, and arguably its entire raison d’être, is the high degree of abstraction that the platform offers. Rather than limiting users to a specific set of transaction types and applications, the platform allows anyone to create any kind of blockchain application by writing a script and uploading it to the Ethereum blockchain. This gives an Ethereum a degree of future-proof-ness and neutrality much greater than that of other blockchain protocols: even if society decides that blockchains aren’t really all that useful for finance at all, and are only really interesting for supply chain tracking, self-owning cars and self-refilling dishwashers and playing chess for money in a trust-free form, Ethereum will still be useful. However, there still are a substantial number of ways in which Ethereum is not nearly as abstract as it could be.

以太坊项目最初的目标之一，甚至可说是她存在的全部意义，在于这个平台提供的高度抽象。相对于限制用户使用特定的交易类型和应用，这个平台允许任何人通过编写脚本然后上传到以太坊区块链的方式，来构建任何种类的区块链应用。这给以太坊带来了远超其它区块链协议的未来适应度和中立性：即使区块链最终被认为对于金融不是那么有用，即使区块链只被用于供应链追踪，"自有"汽车，自补充洗碗机或者是免信任的带赌注象棋游戏，以太坊依然有用武之地。然而，以太坊在许多方面还远没有抽象到她应有的程度。

## Cryptography
## 密码学

Currently, Ethereum transactions are all signed using the ECDSA algorithm, and specifically Bitcoin’s secp256k1 curve. Elliptic curve signatures are a popular kind of signature today, particularly because of the smaller signature and key sizes compared to RSA: an elliptic curve signature takes only 65 bytes, compared to several hundred bytes for an RSA signature. However, it is becoming increasingly understood that the specific kind of signature used by Bitcoin is far from optimal; ed25519 is increasingly recognized as a superior alternative particularly because of its simpler implementation, greater hardness against side-channel attacks and faster verification. And if quantum computers come around, we will likely have to move to Lamport signatures.

目前，以太坊中的交易都是通过[ECDSA算法](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)(椭圆曲线数字签名算法)签名的，更具体地说采用了在比特币中使用的secp256k1曲线。椭圆曲线签名是现在最流行的一种签名算法，很大程度上是因为其相对于RSA算法更小的签名和密钥长度：一个椭圆曲线签名只需要65字节，而RSA签名则是数百字节。但是人们越来越认识到比特币使用的这种签名远不是最优方案，越来越多的人认为[ed25519](http://ed25519.cr.yp.to/ed25519-20110926.pdf)算法会是一个更好的选择，因为它有[更简单的实现](https://github.com/vbuterin/ed25519/blob/master/ed25519.py)，更好的旁路攻击抗性，以及更快的验证速度。如果量子计算机问世，我们则更可能转向[Lamport签名算法](https://bitcoinmagazine.com/6021/bitcoin-is-not-quantum-safe-and-how-we-can-fix/)。

One suggestion that some of our security auditors, and others, have given us is to allow ed25519 signatures as an option in 1.1. But what if we can stay true to our spirit of abstraction and go a bit further: let people use whatever cryptographic verification algorithm that they want? Is that even possible to do securely? Well, we have the ethereum virtual machine, so we have a way of letting people implement arbitrary cryptographic verification algorithms, but we still need to figure out how it can fit in.

我们的安全审计专家和一些朋友给出过的一个建议是，在1.1版本中加入ed25519签名算法作为可选项。但是我们是否能忠于我们的抽象精神再走远一步：让人们可以选择他们想要的任何密码学验证算法呢？有可能安全的做到吗？没错，我们有以太坊虚拟机，这让人们可以实现任意的密码学验证算法，但我们还需要弄明白**如何**让这些算法融入到系统中。

Here is a possible approach:

一个可能的方法是：

1. Every account that is not a contract has a piece of “verification code” attached to it.
2. When a transaction is sent, it must now explicitly specify both sender and recipient.
3. The first step in processing a transaction is to call the verification code, using the transaction’s signature (now a plain byte array) as input. If the verification code outputs anything nonempty within 50000 gas, the transaction is valid. If it outputs an empty array (ie. exactly zero bytes; a single \x00 byte does not count) or exits with an exception condition, then it is not valid.
4. To allow people without ETH to create accounts, we implement a protocol such that one can generate verification code offline and use the hash of the verification code as an address. People can send funds to that address. The first time you send a transaction from that account, you need to provide the verification code in a separate field (we can perhaps overload the nonce for this, since in all cases where this happens the nonce would be zero in any case) and the protocol (i) checks that the verification code is correct, and (ii) swaps it in (this is roughly equivalent to “pay-to-script-hash” in Bitcoin).

1. 每一个非合约账户都可以附带一段“验证程序代码”。
2. 交易发送出去的时候，必须显式的指明发送者和接受者。
3. 处理交易的第一步是调用验证程序，用交易的签名（现在是一个普通的字节数组了）作为输入。如果验证程序在50000 gas之内产生了任何非空的输出，这个交易就是有效的。如果输出为空（也就是说0个字节，一个`\x00`字节不算）或者程序抛异常退出，则交易无效。
4. 为了让没有以太币的用户也可以创建账户，我们让协议支持用户离线生成一段验证程序并且使用这段程序的哈希值作为地址。人们可以向这个地址发送资金。当用户第一次从这个账户发起交易的时候，他需要另外输入对应的验证程序代码（或许可以重用nonce字段来实现, 因为可以知道这种情况下nonce都应该是0），而协议会 (i)检查验证程序正确（对应哈希值), (ii)将其作为此账户的验证程序代码（大致上等价于比特币中的"[pay-to-script-hash](https://en.bitcoin.it/wiki/Pay_to_script_hash)"机制)。

This approach has a few benefits. First, it does not specify anything about the cryptographic algorithm used or the signature format, except that it must take up at most 50000 gas (this value can be adjusted up or down over time). Second, it still keeps the property of the existing system that no pre-registration is required. Third, and quite importantly, it allows people to add higher-level validity conditions that depend on state: for example, making transactions that spend more GavCoin than you currently have actually fail instead of just going into the blockchain and having no effect.

这个方法有几个好处。首先，它没有指定使用**任何**密码学算法或是签名格式，只要求这个算法最多消耗50000 gas（这个数值可以随时间上下调整）。其次，它保留了现有系统无需预先注册的性质。第三也是很重要的一点，它允许人们加入更高层面的依赖状态(译注: state, 指区块链上保存的数据)的有效性条件：例如一个花费超过你当前持有的GavCoin的交易直接是无效的，而不是进入区块链但不产生效果。

However, there are substantial changes to the virtual machine that would have to be made for this to work well. The current virtual machine is designed well for dealing with 256-bit numbers, capturing the hashes and elliptic curve signatures that are used right now, but is suboptimal for algorithms that have different sizes. Additionally, no matter how well-designed the VM is right now, it fundamentally adds a layer of abstraction between the code and the machine. Hence, if this will be one of the uses of the VM going forward, an architecture that maps VM code directly to machine code, applying transformations in the middle to translate specialized opcodes and ensure security, will likely be optimal – particularly for expensive and exotic cryptographic algorithms like zk-SNARKs. And even then, one must take care to minimize any “startup costs” of the virtual machine in order to further increase efficiency as well as denial-of-service vulnerability; together with this, a gas cost rule that encourages re-using existing code and heavily penalizes using different code for every account, allowing just-in-time-compiling virtual machines to maintain a cache, may also be a further improvement.

然而为了良好的实现这个方案虚拟机需要大量改动。现在的虚拟机被设计成能很好的处理256位的数字，很适合目前使用的哈希算法和椭圆曲线签名算法，但对于非256位的算法不是最优的。此外，无论虚拟机设计的多么好，本质上它终究是在字节码和机器码中间加了一层。因此，如果这是未来的虚拟机需要处理的场景，一个可以将虚拟机字节码直接映射为机器码，并且在此过程中通过特殊转换保证安全性的架构可能是最好的 - 尤其适合zk-SNARKs这类费时又奇特的算法。在此之上，我们还需要注意减少虚拟机的“启动成本”，以进一步提高效率，同时避免拒绝服务攻击(denial-of-service vulnerability)；最后，增加一条鼓励重用已有代码和重罚给每一个账户使用不同代码的gas消耗规则，会有利于虚拟机做即时编译优化，也是进一步优化的手段。

## The Trie
## The Trie

(译注：Trie是一种数据结构，又可称作前缀树或是字典树，本文中保留不译。)

Perhaps the most important data structure in Ethereum is the Patricia tree. The Patricia tree is a data structure that, like the standard binary Merkle tree, allows any piece of data inside the trie to be securely authenticated against a root hash using a logarithmically sized (ie. relatively short) hash chain, but also has the important property that data can be added, removed or modified in the tree extremely quickly, only making a small number of changes to the entire structure. The trie is used in Ethereum to store transactions, receipts, accounts and particularly importantly the storage of each account.

以太坊中最终要的数据结构也许就是[Patricia Tree](https://github.com/ethereum/wiki/wiki/Patricia-Tree)了。(译注：Patricia tree是基于trie的一种数据结构。这里Vitalik谈论的实际上是Merkle Patricia Tree, Patricia Tree的一种变体。下同。) Patricia Tree是一种类似标准Merkle二叉树的数据结构，可以通过对数大小(也就是说，相对很小)的哈希链可靠的验证验证树中任意数据，更重要的是还能非常迅速的进行插入，删除和更新操作，这些操作仅对树做很小的变动。以太坊用这种Trie来保存交易，交易收据，账户信息和账户持久数据。

One of the often cited weaknesses of this approach is that the trie is one particular data structure, optimized for a particular set of use cases, but in many cases accounts will do better with a different model. The most common request is a heap: a data structure to which elements can quickly be added with a priority value, and from which the lowest-priority element can always be quickly removed – particularly useful in implementations of markets with bid/ask offers.

这种实现常被提及的一个弱点是，Trie是一种很特殊的数据结构，为一组特殊的场景优化，而很多时候另外的数据结构更适合。最常见的需求是堆（heap）：一种可以快速插入带有权重节点的数据结构，支持快速的移除权重最低的元素 - 这个性质在实现交易撮合的时候特别有用。

Right now, the only way to do this is a rather inefficient workaround: write an implementation of a heap in Solidity or Serpent on top of the trie. This essentially means that every update to the heap requires a logarithmic number of updates (eg. at 1000 elements, ten updates, at 1000000 elements, twenty updates) to the trie, and each update to the trie requires changes to a logarithmic number (once again ten at 1000 elements and twenty at 1000000 elements) of items, and each one of those requires a change to the leveldb database which uses a logarithmic-time-updateable trie internally. If contracts had the option to have a heap instead, as a direct protocol feature, then this overhead could be cut down substantially.

当前，得到一个堆的唯一方法是一种很没效率的变通(workaround)：用Solidity或是Serpent语言[在trie的基础上实现一个堆](https://github.com/ethereum/serpent/blob/develop/examples/cyberdyne/heap.se). 这代表着对heap的每一次更新操作都会带来对下层trie的对数级次更新（eg. 对于1000个元素的堆，带来10次更新，对于1000000个元素的堆，20次更新），而对于下层trie的每次更新操作又需要更新对数级个元素（也就是同样的，对于1000个元素的trie，需要更新10个元素，对于1000000个元素的trie, 需要更新20个），最后每一个trie元素的更新又导致下层的leveldb数据库去更新其内部的，更新操作同样需要对数时间复杂度的，trie. 如果以太坊提供原生实现的堆做为选项供合约使用，这种开销可以极大的降低。

One option to solve this problem is the direct one: just have an option for contracts to have either a regular trie or a heap, and be done with it. A seemingly nicer solution, however, is to generalize even further. The solution here is as follows. Rather than having a trie or a treap, we simply have an abstract hash tree: there is a root node, which may be empty or which may be the hash of one or more children, and each child in turn may either be a terminal value or the hash of some set of children of its own. An extension may be to allow nodes to have both a value and children. This would all be encoded in RLP; for example, we may stipulate that all nodes must be of the form:

解决此问题的一个直接方法是：给合约提供是使用标准trie或是堆的选项，结束。然而进一步通用化看上去会是更好的解决方案。该方案如下。与其使用trie或者堆，我们仅提供一个抽象哈希树：它有一个根节点，可以是空的或是其一个或多个子节点的哈希值，而每一个子节点又可以是终点或是其子节点的哈希值。允许节点既有自己的值又有子节点可以是一个扩展。这些都会编码为RLP格式（译注: Recursive Length Prefix, 以太坊使用的序列化格式），比如，我们可以规定所有节点都必须表示为：

```
[val, child1, child2, child3....]
```

Where val must be a string of bytes (we can restrict it to 32 if desired), and each child (of which there can be zero or more) must be the 32 byte SHA3 hash of some other node. Now, we have the virtual machine’s execution environment keep track of a “current node” pointer, and add a few opcodes:

这里`val`必须是一个字节串（如有需要可以限制长度为32），每一个childX（可以有0个或是多个）必须是某个子节点的SHA3散列值。然后我们让虚拟机执行环境记住一个“当前节点”的指针，并增加一些指令：


* GETVAL: pushes the value of the node at the current pointer onto the stack
* SETVAL: sets the value at the of the node at the current pointer to the value at the top of the stack
* GETCHILDCOUNT: gets the number of children of the node
* ADDCHILD: adds a new child node (starting with zero children of its own)
* REMOVECHILD: pops off a child node
* DESCEND: descend to the kth child of the current node (taking k as an argument from the stack)
* ASCEND: ascend to the parent
* ASCENDROOT: ascend to the root node

* `GETVAL`: 将当前节点的值压入栈
* `SETVAL`: 将当前节点的值设为栈顶值
* `GETCHILDCOUNT`: 获取子节点数量
* `ADDCHILD`: 增加一个子节点（从0个子节点开始）
* `REMOVECHILD`: 弹出一个子节点
* `DESCEND`: 下指到当前节点的第k个子节点（参数k从栈中取得）
* `ASCEND`: 上指到父节点
* `ASCENDROOT`: 上指到根节点

Accessing a Merkle tree with 128 elements would thus look like this:

一个包含128个元素的Merkle Tree的读取看起来就像这样：

```
def access(i):
    ~ascendroot()
    return _access(i, 7)

def _access(i, depth):
    while depth > 0:
        ~descend(i % 2)   
        i /= 2
        depth -= 1
    return ~getval()
```

Creating the tree would look like this:

树的构造看起来像这样：

```
def create(vals):
    ~ascendroot()
    while ~getchildcount() > 0:
        ~removechild()
    _create(vals, 7)

def _create(vals:arr, depth):
    if depth > 0:
        # Recursively create left child
        ~addchild()
        ~descend(0)
        _create(slice(vals, 0, 2**(depth - 1)), depth - 1)
        ~ascend()
        # Recursively create right child
        ~addchild()
        ~descend(1)
        _create(slice(vals, 2**(depth - 1), 2**depth), depth - 1)
        ~ascend()
    else:
        ~setval(vals[0])
```

Clearly, the trie, the treap and in fact any other tree-like data structure could thus be implemented as a library on top of these methods. What is particularly interesting is that each individual opcode is constant-time: theoretically, each node can keep track of the pointers to its children and parent on the database level, requiring only one level of overhead.

显然，无论是trie, 堆还是其它任何类似树的数据结构，都可以实现为基于这些指令的库。这里最有意思的是每一条指令都是常数时间复杂度的：理论上，每个节点都可以记下指向其子节点和父节点的数据库层的指针，只需增加一层开销。

However, this approach also comes with flaws. Particularly, note that if we lose control of the structure of the tree, then we lose the ability to make optimizations. Right now, most Ethereum clients, including C++, Go and Python, have a higher-level cache that allows updates to and reads from storage to happen in constant time if there are multiple reads and writes within one transaction execution. If tries become de-standardized, then optimizations like these become impossible. Additionally, each individual trie structure would need to come up with its own gas costs and its own mechanisms for ensuring that the tree cannot be exploited: quite a hard problem, given that even our own trie had a medium level of vulnerability until recently when we replaced the trie keys with the SHA3 hash of the key rather than the actual key. Hence, it’s unclear whether going this far is worth it.

然而这个方案也有其缺陷。特别需要注意的是如果我们失去对树的结构的控制，我们也就失去了做优化的能力。目前大多数以太坊客户端，包括C++, Go和Python，都有一个高层缓存，可以将涉及存储层多次读写的交易中的读写操作优化到常数时间复杂度。如果trie不再是标准数据结构，类似的优化将成为不可能。此外，每一个不同的trie都需要对应的gas消耗规则，以及确保其不被攻击利用的机制：这是个非常难的问题，即使是我们自己的trie实现，在最近用key的SHA3散列值代替实际值做key的改动之前，也是一个中等级别的漏洞。因此，目前还不清楚这个抽象是否值得去做。

# Currency
# 货币
