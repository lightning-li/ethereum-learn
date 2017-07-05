## Solidity 学习笔记

#### 1. 可见性

函数可以被定义为 `external`，`public`，`internal` 或者 `private`，默认是 `public`；对于状态变量来说，没有 `external` 类型，默认是 `internal`。

- `external`：External 函数是合约接口的一部分，意味着它们可以被其它合约调用，也可以通过交易来调用。一个 external 函数 f 不能在内部调用（例如，f() 是不可行的，但是 this.f() 是可行的）。当接收大型数据数组的时候，external 函数有时是更高效的。

- `public`：对于函数来说，可以在内部调用，或者通过消息调用，也可以被其它合约调用；对于状态变量来说，会自动生成一个 `getter` 函数

- `internal`：这些类型的函数和状态变量只能够从内部访问（例如，从当前合约内部，或者是子合约的内部），无需使用 this

- `private`：这些类型的函数和状态变量只能够从定义它们合约的内部访问

对于 public 和 external 类型的函数来说，区别在于一个可以在内部调用，一个不可以，而且 external 在传递大型数组时效率更高，具体原因请查看
https://ethereum.stackexchange.com/questions/19380/external-vs-public-best-practices
