#### Golem 多方签名钱包解析

**注意：修改了 submitTransactionWithSignatures 函数，由于 web3.eth.sign 行为发生了改变，见 https://github.com/ethereum/EIPs/issues/191**
```
pragma solidity ^0.4.8;


/// @title Multisignature wallet - Allows multiple parties to agree on transactions before execution.
/// @author Stefan George - <stefan.george@consensys.net>
contract MultiSigWallet {

    event Confirmation(address sender, bytes32 transactionHash);
    event Revocation(address sender, bytes32 transactionHash);
    event Submission(bytes32 transactionHash);
    event Execution(bytes32 transactionHash);
    event Deposit(address sender, uint value);
    event OwnerAddition(address owner);
    event OwnerRemoval(address owner);
    event RequiredUpdate(uint required);

    mapping (bytes32 => Transaction) public transactions;
    mapping (bytes32 => mapping (address => bool)) public confirmations;
    mapping (address => bool) public isOwner;
    address[] owners;
    bytes32[] transactionList;
    uint public required;

    struct Transaction {
        address destination;
        uint value;
        bytes data;
        uint nonce;
        bool executed;
    }

    modifier onlyWallet() {
        if (msg.sender != address(this))
            throw;
        _;
    }

    modifier signaturesFromOwners(bytes32 transactionHash, uint8[] v, bytes32[] rs) {
        for (uint i=0; i<v.length; i++)
            if (!isOwner[ecrecover(transactionHash, v[i], rs[i], rs[v.length + i])])
                throw;
        _;
    }

    modifier ownerDoesNotExist(address owner) {
        if (isOwner[owner])
            throw;
        _;
    }

    modifier ownerExists(address owner) {
        if (!isOwner[owner])
            throw;
        _;
    }

    modifier confirmed(bytes32 transactionHash, address owner) {
        if (!confirmations[transactionHash][owner])
            throw;
        _;
    }

    modifier notConfirmed(bytes32 transactionHash, address owner) {
        if (confirmations[transactionHash][owner])
            throw;
        _;
    }

    modifier notExecuted(bytes32 transactionHash) {
        if (transactions[transactionHash].executed)
            throw;
        _;
    }

    modifier notNull(address destination) {
        if (destination == 0)
            throw;
        _;
    }

    modifier validRequired(uint _ownerCount, uint _required) {
        if (   _required > _ownerCount
            || _required == 0
            || _ownerCount == 0)
            throw;
        _;
    }

    function addOwner(address owner)
        external
        onlyWallet
        ownerDoesNotExist(owner)
    {
        isOwner[owner] = true;
        owners.push(owner);
        OwnerAddition(owner);
    }

    function removeOwner(address owner)
        external
        onlyWallet
        ownerExists(owner)
    {
        isOwner[owner] = false;
        for (uint i=0; i<owners.length - 1; i++)
            if (owners[i] == owner) {
                owners[i] = owners[owners.length - 1];
                break;
            }
        owners.length -= 1;
        if (required > owners.length)
            updateRequired(owners.length);
        OwnerRemoval(owner);
    }

    function updateRequired(uint _required)
        public
        onlyWallet
        validRequired(owners.length, _required)
    {
        required = _required;
        RequiredUpdate(_required);
    }

    function addTransaction(address destination, uint value, bytes data, uint nonce)
        private
        notNull(destination)
        returns (bytes32 transactionHash)
    {
        transactionHash = sha3(destination, value, data, nonce);
        if (transactions[transactionHash].destination == 0) {
            transactions[transactionHash] = Transaction({
                destination: destination,
                value: value,
                data: data,
                nonce: nonce,
                executed: false
            });
            transactionList.push(transactionHash);
            Submission(transactionHash);
        }
    }

    function submitTransaction(address destination, uint value, bytes data, uint nonce)
        external
        returns (bytes32 transactionHash)
    {
        transactionHash = addTransaction(destination, value, data, nonce);
        confirmTransaction(transactionHash);
    }

    function submitTransactionWithSignatures(address destination, uint value, bytes data, uint nonce, uint8[] v, bytes32[] rs)
        external
        returns (bytes32 transactionHash)
    {
        transactionHash = addTransaction(destination, value, data, nonce);
        bytes memory prefix = "\x19Ethereum Signed Message:\n32";
        bytes32 prefixedHash = sha3(prefix, transactionHash);
        confirmTransactionWithSignatures(transactionHash, prefixedHash, v, rs);
    }


    function addConfirmation(bytes32 transactionHash, address owner)
        private
        notConfirmed(transactionHash, owner)
    {
        confirmations[transactionHash][owner] = true;
        Confirmation(owner, transactionHash);
    }

    function confirmTransaction(bytes32 transactionHash)
        public
        ownerExists(msg.sender)
    {
        addConfirmation(transactionHash, msg.sender);
        executeTransaction(transactionHash);
    }

    function confirmTransactionWithSignatures(bytes32 transactionHash, bytes32 prefixedHash, uint8[] v, bytes32[] rs)
        public
        signaturesFromOwners(prefixedHash, v, rs)
    {
        for (uint i=0; i<v.length; i++)
            addConfirmation(transactionHash, ecrecover(prefixedHash, v[i], rs[i], rs[i + v.length]));
        executeTransaction(transactionHash);
    }

    function executeTransaction(bytes32 transactionHash)
        public
        notExecuted(transactionHash)
    {
        if (isConfirmed(transactionHash)) {
            Transaction tx = transactions[transactionHash];
            tx.executed = true;
            if (!tx.destination.call.value(tx.value)(tx.data))
                throw;
            Execution(transactionHash);
        }
    }

    function revokeConfirmation(bytes32 transactionHash)
        external
        ownerExists(msg.sender)
        confirmed(transactionHash, msg.sender)
        notExecuted(transactionHash)
    {
        confirmations[transactionHash][msg.sender] = false;
        Revocation(msg.sender, transactionHash);
    }

    function MultiSigWallet(address[] _owners, uint _required)
        validRequired(_owners.length, _required)
    {
        for (uint i=0; i<_owners.length; i++)
            isOwner[_owners[i]] = true;
        owners = _owners;
        required = _required;
    }

    function()
        payable
    {
        if (msg.value > 0)
            Deposit(msg.sender, msg.value);
    }

    function isConfirmed(bytes32 transactionHash)
        public
        constant
        returns (bool)
    {
        uint count = 0;
        for (uint i=0; i<owners.length; i++)
            if (confirmations[transactionHash][owners[i]])
                count += 1;
            if (count == required)
                return true;
    }

    function confirmationCount(bytes32 transactionHash)
        external
        constant
        returns (uint count)
    {
        for (uint i=0; i<owners.length; i++)
            if (confirmations[transactionHash][owners[i]])
                count += 1;
    }

    function filterTransactions(bool isPending)
        private
        returns (bytes32[] _transactionList)
    {
        bytes32[] memory _transactionListTemp = new bytes32[](transactionList.length);
        uint count = 0;
        for (uint i=0; i<transactionList.length; i++)
            if (   isPending && !transactions[transactionList[i]].executed
                || !isPending && transactions[transactionList[i]].executed)
            {
                _transactionListTemp[count] = transactionList[i];
                count += 1;
            }
        _transactionList = new bytes32[](count);
        for (i=0; i<count; i++)
            if (_transactionListTemp[i] > 0)
                _transactionList[i] = _transactionListTemp[i];
    }

    function getPendingTransactions()
        external
        constant
        returns (bytes32[] _transactionList)
    {
        return filterTransactions(true);
    }

    function getExecutedTransactions()
        external
        constant
        returns (bytes32[] _transactionList)
    {
        return filterTransactions(false);
    }
}
```
在 https://remix.ethereum.org/ 中编译该合约，在右侧 Execution environment 中选择 Web3 provider，输入本地节点监听的端口，即可连接到本地搭建的私有网络中。

1. 首先部署该合约，在右侧合约下方有 Create 按钮，输入合约构造函数所需的参数，然后点击该按钮部署至私有链中，注意在命令行终端中，将创建合约的账户解锁 `personal.unlockAccount`，接着启动挖矿功能 `miner.start()`。本例中使用 2-of-3 的多方签名模式：

  ```
  Create : ["0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37", "0x0f5578914288da3b9a3f43ba41a2e6b4d3dd587a", "0x802c12459243cdafd34112561c478cefe3e714db"], 2
  ```
  其中 "0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37" 设为账户 A，"0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37" 设为账户 B，"0x802c12459243cdafd34112561c478cefe3e714db" 设为账户 C。
  部署成功之后，可得到合约地址 `0x9f94a748d21961f9dafc96274834793e16a4fac0`，在终端可以得到该合约实例：

  ```
> var multisigwallet_sol_multisigwalletContract = web3.eth.contract([{"constant":true,"inputs":[{"name":"","type":"bytes32"},{"name":"","type":"address"}],"name":"confirmations","outputs":[{"name":"","type":"bool"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"destination","type":"address"},{"name":"value","type":"uint256"},{"name":"data","type":"bytes"},{"name":"nonce","type":"uint256"}],"name":"submitTransaction","outputs":[{"name":"transactionHash","type":"bytes32"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"owner","type":"address"}],"name":"removeOwner","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"isOwner","outputs":[{"name":"","type":"bool"}],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"transactionHash","type":"bytes32"}],"name":"confirmationCount","outputs":[{"name":"count","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"_required","type":"uint256"}],"name":"updateRequired","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"","type":"bytes32"}],"name":"transactions","outputs":[{"name":"destination","type":"address"},{"name":"value","type":"uint256"},{"name":"data","type":"bytes"},{"name":"nonce","type":"uint256"},{"name":"executed","type":"bool"}],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"transactionHash","type":"bytes32"}],"name":"isConfirmed","outputs":[{"name":"","type":"bool"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"owner","type":"address"}],"name":"addOwner","outputs":[],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"transactionHash","type":"bytes32"}],"name":"confirmTransaction","outputs":[],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"destination","type":"address"},{"name":"value","type":"uint256"},{"name":"data","type":"bytes"},{"name":"nonce","type":"uint256"},{"name":"v","type":"uint8[]"},{"name":"rs","type":"bytes32[]"}],"name":"submitTransactionWithSignatures","outputs":[{"name":"transactionHash","type":"bytes32"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"transactionHash","type":"bytes32"},{"name":"prefixedHash","type":"bytes32"},{"name":"v","type":"uint8[]"},{"name":"rs","type":"bytes32[]"}],"name":"confirmTransactionWithSignatures","outputs":[],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"transactionHash","type":"bytes32"}],"name":"executeTransaction","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"getPendingTransactions","outputs":[{"name":"_transactionList","type":"bytes32[]"}],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"required","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"getExecutedTransactions","outputs":[{"name":"_transactionList","type":"bytes32[]"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"transactionHash","type":"bytes32"}],"name":"revokeConfirmation","outputs":[],"payable":false,"type":"function"},{"inputs":[{"name":"_owners","type":"address[]"},{"name":"_required","type":"uint256"}],"payable":false,"type":"constructor"},{"payable":true,"type":"fallback"},{"anonymous":false,"inputs":[{"indexed":false,"name":"sender","type":"address"},{"indexed":false,"name":"transactionHash","type":"bytes32"}],"name":"Confirmation","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"sender","type":"address"},{"indexed":false,"name":"transactionHash","type":"bytes32"}],"name":"Revocation","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"transactionHash","type":"bytes32"}],"name":"Submission","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"transactionHash","type":"bytes32"}],"name":"Execution","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"sender","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Deposit","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"owner","type":"address"}],"name":"OwnerAddition","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"owner","type":"address"}],"name":"OwnerRemoval","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"required","type":"uint256"}],"name":"RequiredUpdate","type":"event"}]);
> var multiSigWalletInstance = multisigwallet_sol_multisigwalletContract.at("0x9f94a748d21961f9dafc96274834793e16a4fac0")
  ```

2. 接着我们向该多方签名钱包存入一笔钱，调用该合约的 fallback 函数（在调用合约不存在的函数时，也会调用默认的 fallback 函数）：

  ```
> personal.unlockAccount(eth.coinbase)
Unlock account 0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37
Passphrase:
true
> eth.sendTransaction({from : eth.coinbase, to : "0x9f94a748d21961f9dafc96274834793e16a4fac0", value : web3.toWei(9, "ether")})
"0xc6dd7d6cf4cbfcde3f4b1c4c0cd09eeaf115eb5f5d4d594fc486230bfb185919"
> miner.start()
null
> eth.getBalance("0x9f94a748d21961f9dafc96274834793e16a4fac0")
9000000000000000000
> miner.stop()
true
  ```

3. 有两种模式提交交易来转移资金：第一种， 使用 `submitTransaction` 函数，即 A、B、C 三方中任意两方使用相同的参数（接收方、金额、data、nonce，4 元组来标识唯一一笔交易）来调用该函数，一旦达到多方签名阈值，钱包自动将以太币转入接收方账户中；第二种，使用 `submitTransactionWithSignatures` ，A、B、C 任意一方调用该函数，均可在参数中添加他人对钱包转账交易的签名，达到阈值即可。

  第一种方法：账户 A 与账户 B 均提交一笔确认交易
  ```
> miner.start()
> eth.coinbase
"0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37"
> personal.unlockAccount(eth.coinbase)
Unlock account 0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37
Passphrase:
true
> multiSigWalletInstance.submitTransaction.sendTransaction("0x703bfe10ec3da7b6655e9787f21ccd05b53265e5", 100, "ts", 0, {from : eth.coinbase, gas : 1000000})
"0x9483344d99cc903169136ad84f458297cec19f777385a30432b001c6382fb26a"
> eth.getBalance("0x703bfe10ec3da7b6655e9787f21ccd05b53265e5")
0
> personal.unlockAccount("0x0f5578914288da3b9a3f43ba41a2e6b4d3dd587a")
Unlock account 0x0f5578914288da3b9a3f43ba41a2e6b4d3dd587a
Passphrase:
true
> > multiSigWalletInstance.submitTransaction.sendTransaction("0x703bfe10ec3da7b6655e9787f21ccd05b53265e5", 100, "ts", 0, {from : "0x0f5578914288da3b9a3f43ba41a2e6b4d3dd587a", gas : 1000000})
"0xf6938769a24b3aed8030c20534e0a1964e4d6e616d2b0ac94901211904c83d88"
> > eth.getBalance("0x703bfe10ec3da7b6655e9787f21ccd05b53265e5")
100
> miner.stop()
  ```

  第二种方法：账户 A 提交一笔交易，其中交易的数据中包含账户 A 与 B 对同一笔转账交易（四元组）哈希的签名
  ```
  //获取要签名的转账交易 <"0x703bfe10ec3da7b6655e9787f21ccd05b53265e5", 200, "ts2", 1> 哈希值，其中地址为 160 位，200 为 uint256 类型，"ts2" 为 bytes 类型, 1 为 uint256 类型，solidity-sha3 与 web3.sha3 略有差别，详细见 https://github.com/raineorshine/solidity-sha3/blob/master/src/index.js
  > web3.toHex(200)
  "0xc8"
  > web3.toHex("ts2")
  "0x747332"
  // 将 value 与 nonce 值分别扩充到 256位，其余不变
  > txHash = web3.sha3("703bfe10ec3da7b6655e9787f21ccd05b53265e500000000000000000000000000000000000000000000000000000000000000c87473320000000000000000000000000000000000000000000000000000000000000001",{encoding:"hex"})
"0xd0dcee45718975f75da779890289089a71657e8923b81329473a6bb17c521c5c"
> Asig = web3.eth.sign(eth.coinbase, txHash)
"0xefa03c079a7631cb31250994412deac02f9aa41ac8faf0519a0f51dea17c715a60f31bc4808f16a1a16c63a7b5afbaca1d86c91bbec0c9b4f46cd38869691cf91c"
> Bsig = web3.eth.sign(personal.listAccounts[1], txHash)
"0xbcc7f2cc99b587f0cb2207470e811c1705997f1b37838c1a98c7e905fd5229b8083a0638f01fd53f11fb9f4f4d023cfaa4e4a5a5d931fe125d58a08b6f8e144e1b"
> Ar = Asig.slice(0, 66)
"0xefa03c079a7631cb31250994412deac02f9aa41ac8faf0519a0f51dea17c715a"
> As = "0x" + Asig.slice(66, 130)
"0x60f31bc4808f16a1a16c63a7b5afbaca1d86c91bbec0c9b4f46cd38869691cf9"
> Av = 28
28
> Br = Bsig.slice(0, 66)
"0xbcc7f2cc99b587f0cb2207470e811c1705997f1b37838c1a98c7e905fd5229b8"
> Bs = "0x" + Bsig.slice(66, 130)
"0x083a0638f01fd53f11fb9f4f4d023cfaa4e4a5a5d931fe125d58a08b6f8e144e"
> Bv = 27
27
> multiSigWalletInstance.submitTransactionWithSignatures.sendTransaction("0x703bfe10ec3da7b6655e9787f21ccd05b53265e5", 200, "ts2", 1, [Av, Bv], [Ar, Br, As, Bs], {from : eth.coinbase, gas : 1000000})
"0x442c0533f482ffdcc14c334f36b5d9f8c39a41a4407cfb742bdebf9ad7fa4180"
> miner.start()
null
> eth.getTransactionReceipt("0x442c0533f482ffdcc14c334f36b5d9f8c39a41a4407cfb742bdebf9ad7fa4180")
{
  blockHash: "0xf48630e309251ef21cc05b943982fea52358ec7788b4ecc76c4359032f87fc63",
  blockNumber: 4754,
  contractAddress: null,
  cumulativeGasUsed: 246394,
  from: "0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37",
  gasUsed: 246394,
  logs: [{
      address: "0x9f94a748d21961f9dafc96274834793e16a4fac0",
      blockHash: "0xf48630e309251ef21cc05b943982fea52358ec7788b4ecc76c4359032f87fc63",
      blockNumber: 4754,
      data: "0xd0dcee45718975f75da779890289089a71657e8923b81329473a6bb17c521c5c",
      logIndex: 0,
      removed: false,
      topics: ["0x1b15da2a2b1f440c8fb970f04466e7ccd3a8215634645d232bbc23c75785b5bb"],
      transactionHash: "0x442c0533f482ffdcc14c334f36b5d9f8c39a41a4407cfb742bdebf9ad7fa4180",
      transactionIndex: 0
  }, {
      address: "0x9f94a748d21961f9dafc96274834793e16a4fac0",
      blockHash: "0xf48630e309251ef21cc05b943982fea52358ec7788b4ecc76c4359032f87fc63",
      blockNumber: 4754,
      data: "0x000000000000000000000000a3ac96fbe4b0dce5f6f89a715ca00934d68f6c37d0dcee45718975f75da779890289089a71657e8923b81329473a6bb17c521c5c",
      logIndex: 1,
      removed: false,
      topics: ["0xe1c52dc63b719ade82e8bea94cc41a0d5d28e4aaf536adb5e9cccc9ff8c1aeda"],
      transactionHash: "0x442c0533f482ffdcc14c334f36b5d9f8c39a41a4407cfb742bdebf9ad7fa4180",
      transactionIndex: 0
  }, {
      address: "0x9f94a748d21961f9dafc96274834793e16a4fac0",
      blockHash: "0xf48630e309251ef21cc05b943982fea52358ec7788b4ecc76c4359032f87fc63",
      blockNumber: 4754,
      data: "0x0000000000000000000000000f5578914288da3b9a3f43ba41a2e6b4d3dd587ad0dcee45718975f75da779890289089a71657e8923b81329473a6bb17c521c5c",
      logIndex: 2,
      removed: false,
      topics: ["0xe1c52dc63b719ade82e8bea94cc41a0d5d28e4aaf536adb5e9cccc9ff8c1aeda"],
      transactionHash: "0x442c0533f482ffdcc14c334f36b5d9f8c39a41a4407cfb742bdebf9ad7fa4180",
      transactionIndex: 0
  }, {
      address: "0x9f94a748d21961f9dafc96274834793e16a4fac0",
      blockHash: "0xf48630e309251ef21cc05b943982fea52358ec7788b4ecc76c4359032f87fc63",
      blockNumber: 4754,
      data: "0xd0dcee45718975f75da779890289089a71657e8923b81329473a6bb17c521c5c",
      logIndex: 3,
      removed: false,
      topics: ["0x7e9e1cb65db4927b1815f498cbaa226a15c277816f7df407573682110522c9b1"],
      transactionHash: "0x442c0533f482ffdcc14c334f36b5d9f8c39a41a4407cfb742bdebf9ad7fa4180",
      transactionIndex: 0
  }],
  logsBloom: "0x00000000000000000000100000000000000000000000000000020000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000400000000000000000000000400000000000080400000000001000000000000000000000000000000000000000200000020000000000040000000000000000",
  root: "0xe364f018cc9cd462d46a7180c0eb49ca010dc283821844c3e5023f260db33517",
  to: "0x9f94a748d21961f9dafc96274834793e16a4fac0",
  transactionHash: "0x442c0533f482ffdcc14c334f36b5d9f8c39a41a4407cfb742bdebf9ad7fa4180",
  transactionIndex: 0
}
> eth.getBalance("0x703bfe10ec3da7b6655e9787f21ccd05b53265e5")
300
> miner.stop()
true
  ```

**注意**

上述 `web3.eth.sign(msg)` 函数实际进行的操作是 sign(keccak256("\x19Ethereum Signed Message:\n" + len(message) + message)))，而 solidity 中 `ecrecover` 函数的第一个参数是真正被签名的数据，即需要在合约的 `submitTransactionWithSignatures` 函数中进行转换：
```
transactionHash = addTransaction(destination, value, data, nonce);
bytes memory prefix = "\x19Ethereum Signed Message:\n32";
bytes32 prefixedHash = sha3(prefix, transactionHash);
```

prefixedHash 也可在节点客户端计算出：
```
> prefix = "\x19Ethereum Signed Message:\n32";
"\x19Ethereum Signed Message:\n32"
> prefix = web3.toHex(prefix)
"0x19457468657265756d205369676e6564204d6573736167653a0a3332"
> web3.sha3(prefix + txHash.slice(2), {encoding : "hex"})
"0xdffc185d39cf96dadf4a067c541a03d57155c81bd166fe22a4aee33d13bbad0b"
```

关于 solidity 中 ecrecover 与 web3.eth.sign 使用问题可参考 [Use solidity ecrecover with signature calculated with eth_sign](https://gist.github.com/bas-vk/d46d83da2b2b4721efb0907aecdb7ebd)

---
下面看这样一个问题：

1. A、B、C 拥有一个 2-of-3 的钱包 W1；

2. A、B、D 拥有一个 2-of-3 的钱包 W2；

3. A、B 决定通过 W1 使用 `submitTransactionWithSignatures` 函数向 E 转一笔以太币，参数中包含了 A、B 对四元组转账交易的签名；

4. 则 A、B 或者 D 均可通过上述交易触发 W2 转账给 E，而无需任意其它两者的同意，形成重放攻击。

可修改 `submitTransactionWithSignatures` 如下来避免这个问题：

```
function submitTransactionWithSignatures(address destination, uint value, bytes data, uint nonce, uint8[] v, bytes32[] rs)
        external
        returns (bytes32 transactionHash)
    {
        transactionHash = addTransaction(destination, value, data, nonce);
        bytes memory prefix = "\x19Ethereum Signed Message:\n32";
        bytes32 prefixedHash = sha3(prefix, this, transactionHash);
        confirmTransactionWithSignatures(transactionHash, prefixedHash, v, rs);
    }
```

在 prefixedHash 中添加了 this， 即该合约的地址，在 console 中签名时，将合约地址追加进去，即：

```
> multiSigContractAddress = "0x9f94a748d21961f9dafc96274834793e16a4fac0"
"0x9f94a748d21961f9dafc96274834793e16a4fac0"
> personal.unlockAccount(eth.coinbase)
Unlock account 0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37
Passphrase:
true
> web3.eth.sign(eth.coinbase, multiSigContractAddress + txHash.slice(2, 66))
"0xe7df062a2877a7805dd81256b6e1aa5ba7c4737b262c06197a58ced9b9099a786493ee088622365b5570142863ac2b8140720faf7bb81b21ca7906113169936d1c"
```

由于合约地址不同，可避免重放攻击问题。
