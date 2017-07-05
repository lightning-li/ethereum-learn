## 智能合约最佳实践

翻译自 https://github.com/ConsenSys/smart-contract-best-practices

### 使用 solidity 编写智能合约的安全建议
---
#### 外部调用

##### 尽可能避免外部调用

(TODO)

##### 注意 `send()`，`transfer()`，`call.value()()` 三者之间的权衡

当发送以太币的时候，请注意使用 `someAddress.send()`，`someAddress.transfer()` 与 `someAddress.call.value()()` 之间的相对权衡。

- `x.transfer(y)` 等价于 `if (!x.send(y)) throw;` Send 是 transfer 的底层实现，建议尽可能地使用 `transfer`

- `someAddress.send()` 与 `someAddress.transfer()` 对于[重入]()攻击来说是安全的。尽管这些方法仍然会触发代码的执行，但是被调用的合约只被允许花费 2300 gas，这些 gas 目前仅仅够记录一个事件

- `someAddress.call.value()()` 将会发送提供的以太币并且触发代码执行。执行的代码被给予了当前所有剩余可用的 gas，使得这种类型的价值转移易受到重入攻击

使用 `send()` 或 `transfer()` 将会防止重入攻击，但是这么做的代价是与默认函数(fallback funtion) 花费超过 2300 gas 的合约是不兼容的，

一种尝试平衡该折衷的模式是实现 [push and pull]() 机制，使用 `send()` 或 `transfer()` 来实现 push 部分，使用 `call.value()()` 来实现 pull 部分。

值得指出的是，使用 `send()` 或 `transfer()` 进行价值转移本身并不能使合约对重入攻击保持安全性，而只能使特定的价值转移对重入攻击保持安全性。
