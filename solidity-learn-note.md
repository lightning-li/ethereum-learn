## Solidity 学习笔记

#### 1. 可见性

函数可以被定义为 `external`，`public`，`internal` 或者 `private`，默认是 `public`；对于状态变量来说，没有 `external` 类型，默认是 `internal`。

- `external`：External 函数是合约接口的一部分，意味着它们可以被其它合约调用，也可以通过交易来调用。一个 external 函数 f 不能在内部调用（例如，f() 是不可行的，但是 this.f() 是可行的）。当接收大型数据数组的时候，external 函数有时是更高效的。

- `public`：对于函数来说，可以在内部调用，或者通过消息调用，也可以被其它合约调用；对于状态变量来说，会自动生成一个 `getter` 函数

- `internal`：这些类型的函数和状态变量只能够从内部访问（例如，从当前合约内部，或者是子合约的内部），无需使用 this

- `private`：这些类型的函数和状态变量只能够从定义它们合约的内部访问

对于 public 和 external 类型的函数来说，区别在于一个可以在内部调用，一个不可以，而且 external 在传递大型数组时效率更高，具体原因请查看
https://ethereum.stackexchange.com/questions/19380/external-vs-public-best-practices

#### 2. call/callcode/delegatecall

**delegatecall 的意思是：我是一个合约，我允许（委托）你对我的存储做任何事情，所以必须信任被调用合约不会随意修改调用合约的存储**

delegatecall 是 callcode 的一个 bug fix，delegatecall 保留了 callcode 没有保留的 msg.sender 与 msg.value，下面来看一个具体的例子：

```
contract D {
  uint public n;
  address public sender;
  function callSetN(address _e, uint _n) {
    _e.call(bytes4(sha3("setN(uint256)")), _n); // E's storage is set, D is not modified
  }

  function callcodeSetN(address _e, uint _n) {
    _e.callcode(bytes4(sha3("setN(uint256)")), _n); // D's storage is set, E is not modified
  }

  function delegatecallSetN(address _e, uint _n) {
    _e.delegatecall(bytes4(sha3("setN(uint256)")), _n); // D's storage is set, E is not modified
  }
}

contract E {
  uint public n;
  address public sender;
  function setN(uint _n) {
    n = _n;
    sender = msg.sender;
    // msg.sender is D if invoked by D's callcodeSetN. None of E's storage is updated
    // msg.sender is C if invoked by C.foo(). None of E's storage is updated
  }
}

contract C {
    function foo(D _d, E _e, uint _n) {
        _d.delegatecallSetN(_e, _n);
    }
}
```

当合约 D CALL 合约 E 的时候，代码运行在 E 的上下文环境中：合约 E 的存储被使用。

当合约 D CALLCODE 合约 E 的时候，代码运行在 D 的上下文环境中，即合约 E 的代码存在于合约 D 中。当该代码修改存储的时候，发生变化的是合约 D 的存储，而不是 E 的。

当合约 D CALLCODE 合约 E 的时候，在 E 中调用 `msg.sender`，得到的结果是 D。

当一个合约 C 调用 D，然后 D 再 DELEGATECALL 合约 E，此时在 E 中调用 `msg.sender`，得到的结果是 C，也就是说 E 中有着与 D 中相同的 `msg.sender` 与 `msg.value`。 
