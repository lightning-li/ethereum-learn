### 交易签名以及哈希值的计算

**参考 [eip155](https://github.com/ethereum/eips/issues/155)**

在以太坊中，tx = [nonce, gasPrice, gas, to, value, data, v, r, s]，其中 eip155 之前，(r, s, v) = sign(rlp.encode(tx[:-3]))，eip155 之后，(r, s, v) = sign(rlp.encode(tx[:-3] + [chainId, 0, 0]))，hash(tx) = keccak256(rlp.encode(tx))；使用 [pyethereum](https://github.com/ethereum/pyethereum) 库计算交易哈希值：

```
In [1]: import rlp

In [2]: from ethereum import transactions, utils

In [3]: nonce = 1

In [4]: gasPrice = 50 * pow(10, 9)

In [5]: to = "2525252525252525252525252525252525252525"

In [6]: private_key = "5757575757575757575757575757575757575757575757575757575757575757"

In [7]: gas = 200000

In [8]: data = ''

In [9]: value = pow(10, 18)

In [12]: tx = transactions.Transaction(nonce, gasPrice, gas, to, value, data, 1, 0, 0)

In [13]: sig = tx.sign(private_key)

In [14]: tx.v, tx.r, tx.s
Out[14]:
(28,
 101912087796757166702718993441740664977869670498269350823539550609489070477520L,
 7977227560646687443041581723152133713969920118482322767730773813537862499586L)

In [15]: tx.hash
Out[15]: '\xa5_\xd2\xbf\xce1\xc1\xe5\xb6u\xd9\x19`\xfa~\x9c\xd7\x8a\xa8Z\x8d\x82\x91\xd7\x19\x03\xbc2KfJ\xf7'

In [16]: tx.hash.encode("hex")
Out[16]: 'a55fd2bfce31c1e5b675d91960fa7e9cd78aa85a8d8291d71903bc324b664af7'

In [17]: utils.sha3(rlp.encode(tx)).encode("hex")
Out[17]: 'a55fd2bfce31c1e5b675d91960fa7e9cd78aa85a8d8291d71903bc324b664af7'

// eip155 提议增加chainId
In [18]: sig = tx.sign(private_key, network_id=1)

In [19]: sig.v, sig.r, sig.s
Out[19]:
(38,
 107391112126426437763509853950770984364199519236549915339101439338348479768362L,
 18088074028265079293467323141260314974426590311632536364616548465792139408866L)

In [20]: tx.hash.encode("hex")
Out[20]: 'dbe9621bb3c5e3ae8553b0cdb2885f6fd3604093ecd01f6c596fcdc157c8253e'

In [21]: utils.sha3(rlp.encode(tx)).encode("hex")
Out[21]: 'dbe9621bb3c5e3ae8553b0cdb2885f6fd3604093ecd01f6c596fcdc157c8253e'

// 得到公钥地址
In [23]: tx.sender.encode("hex")
Out[23]: '8b75db8a458f25a728dcbc237c10e89cea11d176'

// privtopub 是从 bitcoin 中库导入的函数，得出的公钥是未压缩的，前缀是 04，后面跟着两个 256 bit，分别代表公钥的 x 坐标与 y 坐标，如附图所示
In [31]: utils.sha3(utils.privtopub(private_key.decode("hex"))[1:])[12:].encode("hex")
Out[31]: '8b75db8a458f25a728dcbc237c10e89cea11d176'

// privtoaddr 中使用到了 privtopub 函数
In [33]: utils.privtoaddr(private_key).encode("hex")
Out[33]: '8b75db8a458f25a728dcbc237c10e89cea11d176'
```

**Note:** 在 web3.js 提供的 personal.sign 以及 eth.sign API 中，会对将要进行签名的数据 msg 进行变化：keccak256("\x19Ethereum Signed Message:\n" + len(msg) + msg)，见 https://github.com/ethereum/EIPs/issues/191

----

![比特币未压缩公钥示意图](/images/2017/06/bitcoin_pubkey.png)
