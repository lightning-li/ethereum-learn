#### 在 cpp-ethereum 中添加预编译合约

作者：李康

现有的以太坊中存在一系列的预编译合约，包括 `ecrecover`、`sha256`、`ripemd160`、`identity`、`modexp`、`alt_bn128_G1_add`、`alt_bn128_G1_mul`、`alt_bn128_pairing_product`。预编译合约存在的作用在于将一些常用、使用 solidity 语言实现极其复杂并且效率低下的功能固化在协议中，另外由于 EVM 操作码只有 256 个，是比较稀少的，应该尽量留出来给必要的功能。

那么如何在以太坊中添加自己的预编译合约并且在 solidity 中调用该合约呢？下面以 cpp-ethereum 为例，来讲解如何操作。首先来看一下我们操作过程中会使用到的文件:

- `ethvm/main.cpp` : 该文件是 cpp-ethereum 编译成功后生成的可执行命令 ethvm 的入口文件

- `libethcore/Precompiled.cpp` : 该文件包含了所有预编译合约的实现，我们增添的预编译合约放在该文件实现

- `libethashseal/genesis/test/byzantiumTest.cpp` : 该文件是测试以太坊大都会拜占庭分叉的配置文件，我们需要在给配置文件中的 `accounts` 字段下，增加我们的预编译合约声明。**注 :** `ethvm` 可执行命令可自由选择加载某一配置文件，我们需要在加载的配置文件中添加预编译合约的声明，在此，我选择了 `byzantiumTest` 配置文件

下面来看一下具体的改变，所有改变已上传至 https://github.com/lightning-li/cpp-ethereum

```
// libethashseal/genesis/test/byzantiumTest.cpp
"accounts": {
		"0000000000000000000000000000000000000001": { "wei": "1", "precompiled": { "name": "ecrecover", "linear": { "base": 3000, "word": 0 } } },
		"0000000000000000000000000000000000000002": { "wei": "1", "precompiled": { "name": "sha256", "linear": { "base": 60, "word": 12 } } },
		"0000000000000000000000000000000000000003": { "wei": "1", "precompiled": { "name": "ripemd160", "linear": { "base": 600, "word": 120 } } },
		"0000000000000000000000000000000000000004": { "wei": "1", "precompiled": { "name": "identity", "linear": { "base": 15, "word": 3 } } },
		"0000000000000000000000000000000000000005": { "wei": "1", "precompiled": { "name": "modexp" } },
		"0000000000000000000000000000000000000006": { "wei": "1", "precompiled": { "name": "alt_bn128_G1_add", "linear": { "base": 500, "word": 0 } } },
		"0000000000000000000000000000000000000007": { "wei": "1", "precompiled": { "name": "alt_bn128_G1_mul", "linear": { "base": 2000, "word": 0 } } },
		"0000000000000000000000000000000000000008": { "wei": "1", "precompiled": { "name": "alt_bn128_pairing_product" } },
		"000000000000000000000000000000000000000f": { "wei": "1", "precompiled": { "name": "selfadd", "linear": { "base": 3000, "word": 0 }}}
	}
```

在地址 "000000000000000000000000000000000000000f" 出添加了名为 selfadd 的预编译合约，其中 "linear" 的含义是执行该预编译合约的 gas 花费，其中 base 代表的是基本的花费，word 指的是该预编译合约的入参大小每 32 个字节所需的花费，因此总的花费为 `base + (_in.size() + 31) / 32 * word`。因此如若不指定 "linear" 字段，我们在 Precompiled.cpp 中还需增加额外的 ETH_REGISTER_PRECOMPILED_PRICER，用来供 evm 判断执行预编译合约所需的 gas 花费

```
// libethcore/Precompiled.cpp

// selfadd precompiled contract
// in : u256 a
// out : (a + a) % 256

ETH_REGISTER_PRECOMPILED(selfadd)(bytesConstRef _in) {
	int a = *(_in.data() + 31);

	std::vector<byte> res(1, 0);
	res[0] = (a + a) % 256;
	cout << "selfadd res " << int(res[0]) << endl;
	return {true, res};
}
```

该预编译合约实现的功能及其简单，要求传进的参数是 256 位，然后取其最后一个字节的整数值，然后 double 并且对 256 取余，然后返回结果，结果是一个 pair 实例，第一个元素代表该合约执行成功与否，第二个参数是一个 vector<byte> 变量，代表输出的结果。

```
// ethvm/main.cpp

if (mode == Mode::Statistics)
	{
		cout << "Gas used: " << res.gasUsed << " (+" << t.baseGasRequired(se->evmSchedule(envInfo.number())) << " for transaction, -" << res.gasRefunded << " refunded)\n";
		cout << "Output: " << toHex(output) << "\n";
		LogEntries logs = executive.logs();
		cout << logs.size() << " logs" << (logs.empty() ? "." : ":") << "\n";
		for (LogEntry const& l: logs)
		{
			cout << "  " << l.address.hex() << ": " << toHex(t.data()) << "\n";
			cout << "log data field ";
			string ss = "";
			for (auto i = l.data.begin(); i != l.data.end(); ++i) {
				if (int(*i) / 16 < 10) {
					ss.push_back(int(*i) / 16 + 48);
				} else {
					ss.push_back(int(*i) / 16 - 10 + 97);
				}
				if (int(*i) % 16 < 10) {
					ss.push_back(int(*i) % 16 + 48);
				} else {
					ss.push_back(int(*i) % 16 - 10 + 97);
				}
			}
			cout << ss << endl;
			for (h256 const& t: l.topics)
				cout << "    " << t.hex() << "\n";
		}

		cout << total << " operations in " << execTime << " seconds.\n";
		cout << "Maximum memory usage: " << memTotal * 32 << " bytes\n";
		cout << "Expensive operations:\n";
		for (auto const& c: {Instruction::SSTORE, Instruction::SLOAD, Instruction::CALL, Instruction::CREATE, Instruction::CALLCODE, Instruction::DELEGATECALL, Instruction::MSTORE8, Instruction::MSTORE, Instruction::MLOAD, Instruction::SHA3})
			if (!!counts[(byte)c].first)
				cout << "  " << instructionInfo(c).name << " x " << counts[(byte)c].first << " (" << counts[(byte)c].second << " gas)\n";
	}
```

在 ethvm/main.cpp 中添加输出日志的代码，便于我们查证合约执行结果

接下来我们需要在 solidity 中演示如何调用我们刚刚已经写好的预编译合约。

```
pragma solidity ^0.4.11;

contract R {
    bool public flag;
    event Res(bytes32 haha);

    function selfadd(uint8 v) {
        // We do our own memory management here. Solidity uses memory offset
        // 0x40 to store the current end of memory. We write past it (as
        // writes are memory extensions), but don't update the offset so
        // Solidity will reuse it. The memory used here is only needed for
        // this context.

        // FIXME: inline assembly can't access return values
        bool ret;

        bytes32 haha;
        assembly {
            let size := mload(0x40)
            mstore(size, v)
            // NOTE: we can reuse the request memory because we deal with
            //       the return code
            ret := call(3000, 15, 0, size, 32, add(size, 32), 32)
            haha := mload(add(size, 32))
        }
        Res(haha);
        flag = ret;
    }

}
```

关于 assembly 的语法请查看 http://solidity.readthedocs.io/en/develop/assembly.html 。

在 EVM 中存在三种类型的存储，memory、stack、storage，memory 用于存储一些临时变量，stack 用于 EVM 的执行环境，storage 用于存储永久的临时变量，solidity 在 EVM 底层的 memory 的 0x40 位置处存放了当前消耗的 memory 大小，每当所需的内存扩大时，solidity 会更新 memory 0x40 处的值。`mload(0x40)` 作用是将 memory 0x40 处的值压到 stack 上，比如 solidity 编译器会将它翻译成 `PUSH1 40 MLOAD`。`mstore(size, v)` 作用是将 v 存储在 memory[size...size+31] 处。`call(3000, 15, 0, size, 32, add(size, 32), 32)` 意思是提供 3000 gas 调用地址位于 15 处的智能合约，0 代表传给该智能合约的 ether value。第一个 size 代表 input 起始位置，紧随其后的 32 代表将 input (包括本身) 往后的 32 个字节作为输入传递给智能合约。add(size, 32) 代表智能合约返回结果的 output 起始地址，紧随其后的 32 代表返回结果的大小，即将返回结果赋予 memory[add(size, 32)...add(size, 63)]。

将该代码编译成字节码，即

``` 60606040526000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff16806317b020ab14610048578063890eba681461006e57600080fd5b341561005357600080fd5b61006c600480803560ff1690602001909190505061009b565b005b341561007957600080fd5b61008161011b565b604051808215151515815260200191505060405180910390f35b60008060405183815260208082016020836000600f610bb8f1925060208101519150507f9d720c3d3607e5e125654a222aa9aa0dcadc78fe38dd87183bcd002043086bd08160405180826000191660001916815260200191505060405180910390a1816000806101000a81548160ff021916908315150217905550505050565b6000809054906101000a900460ff16815600a165627a7a7230582013d98234d627d3cc0ccf04c3301571e481f0423ec3cc1befc12855d1edda77190029
```
将该字节码保存在 build/ethvm/contract1.hex 文件中

通过编译 cpp-ethereum 生成的 ethvm 命令来调用该合约代码，如下：

```
➜  /Users/likang/cpp-ethereum/build git:(develop) ✗ >> ethvm/ethvm ethvm/contract1.hex --input "0x17b020ab0000000000000000000000000000000000000000000000000000000000000082" --network Byzantium

// output result

NoProof
0000000000000000000000000000000000000045 has 0 wei
32 0 0
Does tx have zero signature : 0
selfadd res 4
-------SLOAD-------
0
0
-------FINISH------
Gas used: 25277 (+21464 for transaction, -0 refunded)
Output:
1 logs:
  1122334455667788991011121314151617181920: 17b020ab0000000000000000000000000000000000000000000000000000000000000082
log data field 0400000000000000000000000000000000000000000000000000000000000000
    9d720c3d3607e5e125654a222aa9aa0dcadc78fe38dd87183bcd002043086bd0
118 operations in 0.000189 seconds.
Maximum memory usage: 0 bytes
Expensive operations:
  SSTORE x 1 (20000 gas)
  SLOAD x 1 (200 gas)
  CALL x 1 (0 gas)
  MSTORE x 3 (9 gas)
  MLOAD x 4 (12 gas)
```

可以发现打出的 log 日志中，data field 为 0400000000000000000000000000000000000000000000000000000000000000，可见系统将其转换成了小端模式，印证了预编译合约的功能 : (0x82 + 0x82) % 256 = 4

未来工作展望：

将预编译合约以关键字的形式实现，例如在 solidity 中直接使用 `selfadd` 关键字，应该会需要修改 solidity 编译器。
