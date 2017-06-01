#### 智能合约实战(三)

**本文主要介绍合约创建合约，合约与合约之间的相互调用，以及转移以太币到账户等操作**

以下是一个简单的合约示例：

```
pragma solidity ^0.4.8;

contract SimpleStorage {
    uint public storedData;
    uint public extraData;
    SimpleStorageCreator creator;
    mapping (address => uint) public haha;
    event valueChanged(address sender, uint a, uint b);
    event fallbackCalled(address caller);
    event constructorCalled(address caller);
    event constructbySelfCalled(address caller, bytes32 name);

    function () payable{
        fallbackCalled(msg.sender);
    }
    function SimpleStorage() {
        constructorCalled(msg.sender);
        creator = SimpleStorageCreator(msg.sender);
    }
    function set(uint a, uint b) returns (uint) {
        storedData = a;
        extraData = b;
        haha[msg.sender] = a + b;
        valueChanged(msg.sender, a, b);
        return a + b;
    }
    function pay() payable {
        haha[msg.sender] = msg.value;
    }
    function constructbySelf(bytes32 name) {
        creator.create(name);
        constructbySelfCalled(msg.sender, name);
    }
    function del() {
        delete storedData;
        delete haha[msg.sender];
    }
}

contract SimpleStorageCaller {
    event RecvEther(address sender, uint value);

    function set(address ssaddress, uint a, uint b) {
        SimpleStorage s = SimpleStorage(ssaddress);
        s.set(a, b);

    }
    function setbyAddress(address ssaddress) {
        //can't use hex string to call function
        //if (!ssaddress.call.gas(1000000).value(0)("0x1ab06ee500000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000020"))
        //if (!ssaddress.call.gas(10000000).value(1000000)("0x1b9265b8"))
        //call.value default gas is 34050
        if (!ssaddress.call.value(0)(bytes4(bytes32(sha3('set(uint256,uint256)'))), 32, 64))
            throw;
        if (!ssaddress.call.value(2 ether)("\x1b\x92e\xb8"))
            throw;
        if (!ssaddress.send(2 ether))
            throw;
    }
    function recvEther() payable {
        RecvEther(msg.sender, msg.value);
    }
}

contract SimpleStorageCreator {
    mapping (bytes32 => address) public deployedContracts;
    event contractCreated(bytes32 indexed name, address indexed c);
    function create(bytes32 name) returns (address ca){
        deployedContracts[name] = new SimpleStorage();
        contractCreated(name, deployedContracts[name]);
        return deployedContracts[name];
    }

}
```

将其放入[在线编译器](https://remix.ethereum.org/)中，得到合约的 ABI 等信息，在本地私有链网络中部署：

1. 首先部署 SimpleStorageCreator 合约，然后调用该合约的 create 函数创建一个 SimpleStorage 合约的实例；

2. 接着部署 SimpleStorageCaller 合约，然后调用该合约的 recvEther 函数接收以太币，接着通过调用 set 与 setbyAddress 函数来改变第 1 步中产生的 SimpleStorage 实例的状态；

3. 最后使用生成的 SimpleStorage 实例调用 SimpleStorageCreator 再次生成一个新的 SimpleStorage 实例

```
// deploy SimpleStorageCreator contract
> var ballot_sol_simplestoragecreatorContract = web3.eth.contract([{"constant":true,"inputs":[{"name":"","type":"bytes32"}],"name":"deployedContracts","outputs":[{"name":"","type":"address"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"name","type":"bytes32"}],"name":"create","outputs":[{"name":"ca","type":"address"}],"payable":false,"type":"function"},{"anonymous":false,"inputs":[{"indexed":true,"name":"name","type":"bytes32"},{"indexed":true,"name":"c","type":"address"}],"name":"contractCreated","type":"event"}]);
> var ballot_sol_simplestoragecreator = ballot_sol_simplestoragecreatorContract.new(
   {
     from: web3.eth.accounts[0],
     data: '0x6060604052341561000c57fe5b5b6108688061001c6000396000f30060606040526000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff16806323b94b54146100465780637368a8ce146100aa575bfe5b341561004e57fe5b61006860048080356000191690602001909190505061010e565b604051808273ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200191505060405180910390f35b34156100b257fe5b6100cc600480803560001916906020019091905050610141565b604051808273ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200191505060405180910390f35b60006020528060005260406000206000915054906101000a900473ffffffffffffffffffffffffffffffffffffffff1681565b600061014b610284565b809050604051809103906000f080151561016157fe5b60006000846000191660001916815260200190815260200160002060006101000a81548173ffffffffffffffffffffffffffffffffffffffff021916908373ffffffffffffffffffffffffffffffffffffffff16021790555060006000836000191660001916815260200190815260200160002060009054906101000a900473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff1682600019167f2ab63b650b80cd61a629723d60e296d9f72bafe303d88536727e84377708b73960405180905060405180910390a360006000836000191660001916815260200190815260200160002060009054906101000a900473ffffffffffffffffffffffffffffffffffffffff1690505b919050565b6040516105a8806102958339019056006060604052341561000c57fe5b5b7f46062ebc56804d834366ac861d4e38edf94333ab8ccca994d5b3b0b311dfd0d733604051808273ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200191505060405180910390a133600260006101000a81548173ffffffffffffffffffffffffffffffffffffffff021916908373ffffffffffffffffffffffffffffffffffffffff1602179055505b5b6104e6806100c26000396000f30060606040523615610081576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff168063107b401f146100ed5780631ab06ee5146101375780631b9265b8146101745780632a1afcd91461017e5780635dab8af6146101a4578063609d3334146101c8578063b6588ffd146101ee575b6100eb5b7fb86da4c4811a5ccb3d035797a21634b07481267aba3135b71b181ac96345cf2833604051808273ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200191505060405180910390a15b565b005b34156100f557fe5b610121600480803573ffffffffffffffffffffffffffffffffffffffff16906020019091905050610200565b6040518082815260200191505060405180910390f35b341561013f57fe5b61015e6004808035906020019091908035906020019091905050610218565b6040518082815260200191505060405180910390f35b61017c6102ed565b005b341561018657fe5b61018e610334565b6040518082815260200191505060405180910390f35b34156101ac57fe5b6101c660048080356000191690602001909190505061033a565b005b34156101d057fe5b6101d8610468565b6040518082815260200191505060405180910390f35b34156101f657fe5b6101fe61046e565b005b60036020528060005260406000206000915090505481565b60008260008190555081600181905550818301600360003373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020819055507f11ee77a57d176eec0c855796ed2e9f1be7314279895c6cc8a5e823e426b16d10338484604051808473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001838152602001828152602001935050505060405180910390a181830190505b92915050565b34600360003373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020819055505b565b60005481565b600260009054906101000a900473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16637368a8ce826000604051602001526040518263ffffffff167c0100000000000000000000000000000000000000000000000000000000028152600401808260001916600019168152602001915050602060405180830381600087803b15156103d857fe5b6102c65a03f115156103e657fe5b50505060405180519050507fc741b49632e665de715035d7664543151610b8b242d71fe7eb5120cdafa7a8d63382604051808373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff16815260200182600019166000191681526020019250505060405180910390a15b50565b60015481565b600060009055600360003373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff168152602001908152602001600020600090555b5600a165627a7a723058207b7ecf199a0f3c5ece72a982609c856c3da393a4d8d78b694c49ebaa28a7bff60029a165627a7a72305820c687e4906e825de4ff53122103711057b2266c1bb8818235d7ca71343ec4fd4d0029',
     gas: '4700000'
   }, function (e, contract){
    console.log(e, contract);
    if (typeof contract.address !== 'undefined') {
         console.log('Contract mined! address: ' + contract.address + ' transactionHash: ' + contract.transactionHash);
    }
 })

> miner.start()
null
> null [object Object]
Contract mined! address: 0x2a7e33dac11c8e6164acb448637092be816b1488 transactionHash: 0x21c9a676cae0a501829a4bc788adc4b9dc851c4c0c622b7997405cbce08807c1

> miner.stop()
true

//call create function to create SimpleStorage Instance
> var simpleStorageCreatorInstance = ballot_sol_simplestoragecreatorContract.at("0x2a7e33dac11c8e6164acb448637092be816b1488")
undefined
> personal.unlockAccount(eth.coinbase)
Unlock account 0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37
Passphrase:
true

//notice : set the transaction gasLimit, because default gasLimit is not enough to create contract
> simpleStorageCreatorInstance.create.sendTransaction("aaa", {from : eth.coinbase, gas : 1000000})
"0x698a40f0884fd5111fbd5c1e3dd91d13c7df4a0f09a8565cb068da4bd1945654"
> miner.start()
null
> miner.stop()
true

// 创建的 SimpleStorage 合约地址
> simpleStorageCreatorInstance.deployedContracts("aaa")
"0xe0b759ca7448210b1e73fb32c8b12f880130aad2"

//logs 中的第一项是 SimpleStorage 构造函数中发生的事件函数: constructorCalled(address)
//第二项是自身发生的事件函数：contractCreated。通过 topics 中的第一项可以看出。

> web3.sha3("constructorCalled(address)")
"0x46062ebc56804d834366ac861d4e38edf94333ab8ccca994d5b3b0b311dfd0d7"
> web3.sha3("contractCreated(bytes32,address)")
"0x2ab63b650b80cd61a629723d60e296d9f72bafe303d88536727e84377708b739"

> eth.getTransactionReceipt("0x698a40f0884fd5111fbd5c1e3dd91d13c7df4a0f09a8565cb068da4bd1945654")
{
  blockHash: "0x89243162564fb7be70e8e1d5b6fc6151fc8ba4069b6e2526420bd5894e4b6184",
  blockNumber: 142,
  contractAddress: null,
  cumulativeGasUsed: 349238,
  from: "0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37",
  gasUsed: 349238,
  logs: [{
      address: "0xe0b759ca7448210b1e73fb32c8b12f880130aad2",
      blockHash: "0x89243162564fb7be70e8e1d5b6fc6151fc8ba4069b6e2526420bd5894e4b6184",
      blockNumber: 142,
      data: "0x0000000000000000000000002a7e33dac11c8e6164acb448637092be816b1488",
      logIndex: 0,
      removed: false,
      topics: ["0x46062ebc56804d834366ac861d4e38edf94333ab8ccca994d5b3b0b311dfd0d7"],
      transactionHash: "0x698a40f0884fd5111fbd5c1e3dd91d13c7df4a0f09a8565cb068da4bd1945654",
      transactionIndex: 0
  }, {
      address: "0x2a7e33dac11c8e6164acb448637092be816b1488",
      blockHash: "0x89243162564fb7be70e8e1d5b6fc6151fc8ba4069b6e2526420bd5894e4b6184",
      blockNumber: 142,
      data: "0x",
      logIndex: 1,
      removed: false,
      topics: ["0x2ab63b650b80cd61a629723d60e296d9f72bafe303d88536727e84377708b739", "0x6161610000000000000000000000000000000000000000000000000000000000", "0x000000000000000000000000e0b759ca7448210b1e73fb32c8b12f880130aad2"],
      transactionHash: "0x698a40f0884fd5111fbd5c1e3dd91d13c7df4a0f09a8565cb068da4bd1945654",
      transactionIndex: 0
  }],
  logsBloom: "0x0000008000000010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004000a800000000000000000000000000000080000000000040000000000400000000002000100000000000000000000000020000000000000000000000000040000000000000000000000000000000000000000000000000020000000000000400000000000000000000000000000000001000000000000000000000000010000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000",
  root: "0x6156ba393a7edca776920e38ec7df6fc9da533dc5d24d975eb465ff6eeab551a",
  to: "0x2a7e33dac11c8e6164acb448637092be816b1488",
  transactionHash: "0x698a40f0884fd5111fbd5c1e3dd91d13c7df4a0f09a8565cb068da4bd1945654",
  transactionIndex: 0
}

// deploy SimpleStorageCaller contract
> var ballot_sol_simplestoragecallerContract = web3.eth.contract([{"constant":false,"inputs":[],"name":"recvEther","outputs":[],"payable":true,"type":"function"},{"constant":false,"inputs":[{"name":"ssaddress","type":"address"},{"name":"a","type":"uint256"},{"name":"b","type":"uint256"}],"name":"set","outputs":[],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"ssaddress","type":"address"}],"name":"setbyAddress","outputs":[],"payable":false,"type":"function"},{"anonymous":false,"inputs":[{"indexed":false,"name":"sender","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"RecvEther","type":"event"}]);
> var ballot_sol_simplestoragecaller = ballot_sol_simplestoragecallerContract.new(
   {
     from: web3.eth.accounts[0],
     data: '0x6060604052341561000c57fe5b5b6103ae8061001c6000396000f30060606040526000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff1680635c890712146100515780638308d7e91461005b578063b8518596146100a3575bfe5b6100596100d9565b005b341561006357fe5b6100a1600480803573ffffffffffffffffffffffffffffffffffffffff16906020019091908035906020019091908035906020019091905050610147565b005b34156100ab57fe5b6100d7600480803573ffffffffffffffffffffffffffffffffffffffff169060200190919050506101e8565b005b7ff26e958dbf975a119527fcf7e79a1c63c6245ee7206c2393ac4ebb7b3d434ee73334604051808373ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff1681526020018281526020019250505060405180910390a15b565b60008390508073ffffffffffffffffffffffffffffffffffffffff16631ab06ee584846000604051602001526040518363ffffffff167c01000000000000000000000000000000000000000000000000000000000281526004018083815260200182815260200192505050602060405180830381600087803b15156101c857fe5b6102c65a03f115156101d657fe5b50505060405180519050505b50505050565b8073ffffffffffffffffffffffffffffffffffffffff16600060405180807f7365742875696e743235362c75696e7432353629000000000000000000000000815250601401905060405180910390207c0100000000000000000000000000000000000000000000000000000000900490602060406040518463ffffffff167c0100000000000000000000000000000000000000000000000000000000028152600401808360ff1681526020018260ff1681526020019250505060006040518083038185886187965a03f1935050505015156102c35760006000fd5b8073ffffffffffffffffffffffffffffffffffffffff16671bc16d674ec8000060405180807f1b9265b800000000000000000000000000000000000000000000000000000000815250602001905060006040518083038185876187965a03f19250505015156103325760006000fd5b8073ffffffffffffffffffffffffffffffffffffffff166108fc671bc16d674ec800009081150290604051809050600060405180830381858888f19350505050151561037e5760006000fd5b5b505600a165627a7a723058201dc97e127cda430da996488f5aacb680af1cf34fa6e046118c7293ed652484e30029',
     gas: '4700000'
   }, function (e, contract){
    console.log(e, contract);
    if (typeof contract.address !== 'undefined') {
         console.log('Contract mined! address: ' + contract.address + ' transactionHash: ' + contract.transactionHash);
    }
 })
> miner.start()
null
> null [object Object]
Contract mined! address: 0xce0963867986bd654631b5f373c571c6599c41d8 transactionHash: 0x394f7a75ec92947bb5bc341e5402334b7240158812bd29c99873f75e540f2e9f

> miner.stop()
true

//call recvEther function
> var simpleStorageCallerInstance = ballot_sol_simplestoragecallerContract.at("0xce0963867986bd654631b5f373c571c6599c41d8")
undefined
> simpleStorageCallerInstance.recvEther.sendTransaction({from : eth.coinbase, value : web3.toWei(7, "ether")})
"0x2a2fde605785cdb98781a75302515ef49f7f6adc00544e571b3a643ada435773"
> miner.start()
null
> miner.stop()
true
> eth.getBalance("0xce0963867986bd654631b5f373c571c6599c41d8")
7000000000000000000

// call set function
> simpleStorageCallerInstance.set.sendTransaction("0xe0b759ca7448210b1e73fb32c8b12f880130aad2", 10, 20, {from : eth.coinbase})
"0x18ad69f414e031238f67e6f99cff0e94eabd349bdc467beb5dd8024b1691d361"
> miner.start()
null
> miner.stop()
true

//验证被调用合约的状态是否正确改变
> var ballot_sol_simplestorageContract = web3.eth.contract([{"constant":true,"inputs":[{"name":"","type":"address"}],"name":"haha","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"a","type":"uint256"},{"name":"b","type":"uint256"}],"name":"set","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[],"name":"pay","outputs":[],"payable":true,"type":"function"},{"constant":true,"inputs":[],"name":"storedData","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"name","type":"bytes32"}],"name":"constructbySelf","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"extraData","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[],"name":"del","outputs":[],"payable":false,"type":"function"},{"inputs":[],"payable":false,"type":"constructor"},{"payable":true,"type":"fallback"},{"anonymous":false,"inputs":[{"indexed":false,"name":"sender","type":"address"},{"indexed":false,"name":"a","type":"uint256"},{"indexed":false,"name":"b","type":"uint256"}],"name":"valueChanged","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"caller","type":"address"}],"name":"fallbackCalled","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"caller","type":"address"}],"name":"constructorCalled","type":"event"},{"anonymous":false,"inputs":[{"indexed":false,"name":"caller","type":"address"},{"indexed":false,"name":"name","type":"bytes32"}],"name":"constructbySelfCalled","type":"event"}]);
undefined
> var simpleStorageInstance = ballot_sol_simplestorageContract.at("0xe0b759ca7448210b1e73fb32c8b12f880130aad2")
undefined
> simpleStorageInstance.storedData
function()
> simpleStorageInstance.storedData()
10
> simpleStorageInstance.extraData()
20
> simpleStorageInstance.haha("0xce0963867986bd654631b5f373c571c6599c41d8")
30

// call setbyAddress function
> simpleStorageCallerInstance.setbyAddress.sendTransaction("0xe0b759ca7448210b1e73fb32c8b12f880130aad2", {from : eth.coinbase})
"0x6bba77c6532b4319451eba12d3a4545cf73020d4f5ffca65a7cd60af7c930d0b"
> miner.start()
null
> miner.stop()
true

// 转账成功
> eth.getBalance("0xce0963867986bd654631b5f373c571c6599c41d8")
3000000000000000000
> eth.getBalance("0xe0b759ca7448210b1e73fb32c8b12f880130aad2")
4000000000000000000
> simpleStorageInstance.storedData()
32
> simpleStorageInstance.extraData()
64
> simpleStorageInstance.haha("0xce0963867986bd654631b5f373c571c6599c41d8")
2000000000000000000

// 交易收据: 其中 logs 有两项，一项是由 ssaddress.call.value(0)(bytes4(bytes32(sha3('set(uint256,uint256)'))), 32, 64) 触发的 valueChanged 事件，另一项是由 ssaddress.send(2 ether) 触发的 SimpleContract 的 fallback 函数引起的事件，即 fallbackCalled 事件
> web3.sha3("fallbackCalled(address)")
"0xb86da4c4811a5ccb3d035797a21634b07481267aba3135b71b181ac96345cf28"
> web3.sha3("valueChanged(address,uint256,uint256)")
"0x11ee77a57d176eec0c855796ed2e9f1be7314279895c6cc8a5e823e426b16d10"
> eth.getTransactionReceipt("0x6bba77c6532b4319451eba12d3a4545cf73020d4f5ffca65a7cd60af7c930d0b")
{
  blockHash: "0x60a6b55888b7439a169b60ca4e7464bb29c818417b95038e40ab37274aa6c2ae",
  blockNumber: 181,
  contractAddress: null,
  cumulativeGasUsed: 62306,
  from: "0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37",
  gasUsed: 62306,
  logs: [{
      address: "0xe0b759ca7448210b1e73fb32c8b12f880130aad2",
      blockHash: "0x60a6b55888b7439a169b60ca4e7464bb29c818417b95038e40ab37274aa6c2ae",
      blockNumber: 181,
      data: "0x000000000000000000000000ce0963867986bd654631b5f373c571c6599c41d800000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000040",
      logIndex: 0,
      removed: false,
      topics: ["0x11ee77a57d176eec0c855796ed2e9f1be7314279895c6cc8a5e823e426b16d10"],
      transactionHash: "0x6bba77c6532b4319451eba12d3a4545cf73020d4f5ffca65a7cd60af7c930d0b",
      transactionIndex: 0
  }, {
      address: "0xe0b759ca7448210b1e73fb32c8b12f880130aad2",
      blockHash: "0x60a6b55888b7439a169b60ca4e7464bb29c818417b95038e40ab37274aa6c2ae",
      blockNumber: 181,
      data: "0x000000000000000000000000ce0963867986bd654631b5f373c571c6599c41d8",
      logIndex: 1,
      removed: false,
      topics: ["0xb86da4c4811a5ccb3d035797a21634b07481267aba3135b71b181ac96345cf28"],
      transactionHash: "0x6bba77c6532b4319451eba12d3a4545cf73020d4f5ffca65a7cd60af7c930d0b",
      transactionIndex: 0
  }],
  logsBloom: "0x00000000000000000000000000000000000000000000000000000080000000100000000000000000000000000000000000000000000002000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000080000000000000000040000000000000000000000000000000000000000000000000000000000000000",
  root: "0xc9eb43e1ac6c51c20e05939da1036f45ba9ac030160b89e0b44c96737363be5b",
  to: "0xce0963867986bd654631b5f373c571c6599c41d8",
  transactionHash: "0x6bba77c6532b4319451eba12d3a4545cf73020d4f5ffca65a7cd60af7c930d0b",
  transactionIndex: 0
}

// SimpleStorage call SimpleStorageCreator
> simpleStorageInstance.constructbySelf.sendTransaction("bbb", {from : eth.coinbase, gas : 1000000})
"0x778b61d675b67f786a03033caa384eb6ee9c27c1ab68d2d8a9119b0e02a8799b"
> miner.start()
null
> miner.stop()
true

// 第二次创建生成合约的地址，与下面交易收据 logs 第二项中 topics 的第二项、第三项相同
> simpleStorageCreatorInstance.deployedContracts("bbb")
"0x0e344a0a77447b01c0991335301b79e1d052734a"
// 交易收据：logs 有三项，分别是 constructorCalled、contractCreated、constructbySelfCalled 事件
> web3.sha3("constructorCalled(address)")
"0x46062ebc56804d834366ac861d4e38edf94333ab8ccca994d5b3b0b311dfd0d7"
> web3.sha3("contractCreated(bytes32,address)")
"0x2ab63b650b80cd61a629723d60e296d9f72bafe303d88536727e84377708b739"
> web3.sha3("constructbySelfCalled(address,bytes32)")
"0xc741b49632e665de715035d7664543151610b8b242d71fe7eb5120cdafa7a8d6"
> eth.getTransactionReceipt("0x778b61d675b67f786a03033caa384eb6ee9c27c1ab68d2d8a9119b0e02a8799b")
{
  blockHash: "0xee6ccffe8225635a69f193d9283e265dbfb98845bf29d705e753216944bc2683",
  blockNumber: 188,
  contractAddress: null,
  cumulativeGasUsed: 352719,
  from: "0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37",
  gasUsed: 352719,
  logs: [{
      address: "0x0e344a0a77447b01c0991335301b79e1d052734a",
      blockHash: "0xee6ccffe8225635a69f193d9283e265dbfb98845bf29d705e753216944bc2683",
      blockNumber: 188,
      data: "0x0000000000000000000000002a7e33dac11c8e6164acb448637092be816b1488",
      logIndex: 0,
      removed: false,
      topics: ["0x46062ebc56804d834366ac861d4e38edf94333ab8ccca994d5b3b0b311dfd0d7"],
      transactionHash: "0x778b61d675b67f786a03033caa384eb6ee9c27c1ab68d2d8a9119b0e02a8799b",
      transactionIndex: 0
  }, {
      address: "0x2a7e33dac11c8e6164acb448637092be816b1488",
      blockHash: "0xee6ccffe8225635a69f193d9283e265dbfb98845bf29d705e753216944bc2683",
      blockNumber: 188,
      data: "0x",
      logIndex: 1,
      removed: false,
      topics: ["0x2ab63b650b80cd61a629723d60e296d9f72bafe303d88536727e84377708b739", "0x6262620000000000000000000000000000000000000000000000000000000000", "0x0000000000000000000000000e344a0a77447b01c0991335301b79e1d052734a"],
      transactionHash: "0x778b61d675b67f786a03033caa384eb6ee9c27c1ab68d2d8a9119b0e02a8799b",
      transactionIndex: 0
  }, {
      address: "0xe0b759ca7448210b1e73fb32c8b12f880130aad2",
      blockHash: "0xee6ccffe8225635a69f193d9283e265dbfb98845bf29d705e753216944bc2683",
      blockNumber: 188,
      data: "0x000000000000000000000000a3ac96fbe4b0dce5f6f89a715ca00934d68f6c376262620000000000000000000000000000000000000000000000000000000000",
      logIndex: 2,
      removed: false,
      topics: ["0xc741b49632e665de715035d7664543151610b8b242d71fe7eb5120cdafa7a8d6"],
      transactionHash: "0x778b61d675b67f786a03033caa384eb6ee9c27c1ab68d2d8a9119b0e02a8799b",
      transactionIndex: 0
  }],
  logsBloom: "0x00000080000000000000000000080000000100000000000000000000000000000000000000000000000000000000000000000000040002800000000000000000000040000000000000000000840000000000400000000000000100000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000400000000000000000000004000000000021000400000000000000000000010000001000000020000000000000000800000000000000004000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000001000000000000",
  root: "0x3f15654a1db329092852c31ab221828d3a7f29275c84558bba96d866d8d287b9",
  to: "0xe0b759ca7448210b1e73fb32c8b12f880130aad2",
  transactionHash: "0x778b61d675b67f786a03033caa384eb6ee9c27c1ab68d2d8a9119b0e02a8799b",
  transactionIndex: 0
}
```

bingo~
