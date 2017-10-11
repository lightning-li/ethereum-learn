#### 以太坊状态清理

**作者：李康**

在 2016 年 9 月份，以太坊网络遭遇了 DDOS 攻击，由于以太坊协议的漏洞（可以使用 SUICIDE 操作码廉价地生成新账户），使得攻击者可以在很少的花费下创建大量的空账户 - empty account（zero balance、nonce、code），空账户功能性上与未存在账户 - non-existence account 相似，但是不同点在于空账户会存在于以太坊的 account state trie 中，所以这一次 DDOS 攻击导致以太坊节点存储空间爆炸（大概增加了 1900 万空账户，但是账户与 account state trie 中的节点不是 1：1 的关系，因为有中间节点，所以真正增加的存储空间大于 1900 万账户所需空间）。为了解决该问题，以太坊进行了硬分叉 spurious dragon，旨在清除空账户，减小以太坊节点所需存储空间，修补协议漏洞（SUICIDE 创建新账户时需要额外的 25000 gas），并且引进了新的协议规则：1). empty account 不再允许被创建(原先可以导致空账户产生的操作将会使账户变为 non-existence)。 2). 可以通过交易与空账户交互（touch）来删除它们。

一个在硬分叉 spurious dragon 之前可以产生大量空账户的合约例子：

```
pragma solidity ^0.4.15;

contract C {
  function del(address s) {
      selfdestruct(s);
  }
}

contract A {
    uint public a;
    C cc;
    event Log(address);

    function() payable{

    }
    function set(address s) {
        cc = C(s);
        a = 5;
    }

    function haha() {
        a = 3;
        Log(msg.sender);

    }
    function des() {
        for (uint i = 0; i < 100; ++i) {
            cc.del(address(i + 6));
        }
        a = 100;
    }
}
```

合约 A 调用 C 的 del 函数时，会调用 selfdestruct 函数，根据黄皮书对 SUICIDE 操作码的解释 "Halt execution and register account for later deletion"，意味着 C 合约的执行会立即终止，但是在该交易执行完毕后，才会删除该账户，也就意味着可以多次调用 del 即 selfdestruct 函数，导致产生大量的空账户。

---
参考文献：
1. https://ethereum.stackexchange.com/questions/11312/how-was-the-state-bloat-attack-that-led-to-the-eip-150-hardfork-performed

2. https://ethereum.stackexchange.com/questions/10134/why-were-empty-accounts-allowed-to-be-on-the-blockchain

3. https://www.reddit.com/r/ethereum/comments/5es5g4/a_state_clearing_faq/
