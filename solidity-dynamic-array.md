## 智能合约中的动态数组

以 status 众筹合约中的动态上限合约为例，讲解一下智能合约中动态数组的使用，合约源代码如下：

```
pragma solidity ^0.4.12;


/// @dev `Owned` is a base level contract that assigns an `owner` that can be
///  later changed
contract Owned {

    /// @dev `owner` is the only address that can call a function with this
    /// modifier
    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    address public owner;

    /// @notice The Constructor assigns the message sender to be `owner`
    function Owned() {
        owner = msg.sender;
    }

    address public newOwner;

    /// @notice `owner` can step down and assign some other address to this role
    /// @param _newOwner The address of the new owner. 0x0 can be used to create
    ///  an unowned neutral vault, however that cannot be undone
    function changeOwner(address _newOwner) onlyOwner {
        newOwner = _newOwner;
    }


    function acceptOwnership() {
        if (msg.sender == newOwner) {
            owner = newOwner;
        }
    }
}




/**
 * Math operations with safety checks
 */
library SafeMath {
  function mul(uint a, uint b) internal returns (uint) {
    uint c = a * b;
    assert(a == 0 || c / a == b);
    return c;
  }

  function div(uint a, uint b) internal returns (uint) {
    // assert(b > 0); // Solidity automatically throws when dividing by 0
    uint c = a / b;
    // assert(a == b * c + a % b); // There is no case in which this doesn't hold
    return c;
  }

  function sub(uint a, uint b) internal returns (uint) {
    assert(b <= a);
    return a - b;
  }

  function add(uint a, uint b) internal returns (uint) {
    uint c = a + b;
    assert(c >= a);
    return c;
  }

  function max64(uint64 a, uint64 b) internal constant returns (uint64) {
    return a >= b ? a : b;
  }

  function min64(uint64 a, uint64 b) internal constant returns (uint64) {
    return a < b ? a : b;
  }

  function max256(uint256 a, uint256 b) internal constant returns (uint256) {
    return a >= b ? a : b;
  }

  function min256(uint256 a, uint256 b) internal constant returns (uint256) {
    return a < b ? a : b;
  }
}


/*
    Copyright 2017, Jordi Baylina

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.
 */

/// @title DynamicCeiling Contract
/// @author Jordi Baylina
/// @dev This contract calculates the ceiling from a series of curves.
///  These curves are committed first and revealed later.
///  All the curves must be in increasing order and the last curve is marked
///  as the last one.
///  This contract allows to hide and reveal the ceiling at will of the owner.



contract DynamicCeiling is Owned {
    using SafeMath for uint256;

    struct Curve {
        bytes32 hash;
        // Absolute limit for this curve
        uint256 limit;
        // The funds remaining to be collected are divided by `slopeFactor` smooth ceiling
        // with a long tail where big and small buyers can take part.
        uint256 slopeFactor;
        // This keeps the curve flat at this number, until funds to be collected is less than this
        uint256 collectMinimum;
    }

    address public contribution;

    Curve[] public curves;
    uint256 public currentIndex;
    uint256 public revealedCurves;
    bool public allRevealed;

    /// @dev `contribution` is the only address that can call a function with this
    /// modifier
    modifier onlyContribution {
        require(msg.sender == contribution);
        _;
    }

    function DynamicCeiling(address _owner, address _contribution) {
        owner = _owner;
        contribution = _contribution;
    }

    /// @notice This should be called by the creator of the contract to commit
    ///  all the curves.
    /// @param _curveHashes Array of hashes of each curve. Each hash is calculated
    ///  by the `calculateHash` method. More hashes than actual curves can be
    ///  committed in order to hide also the number of curves.
    ///  The remaining hashes can be just random numbers.
    function setHiddenCurves(bytes32[] _curveHashes) public onlyOwner {
        require(curves.length == 0);

        curves.length = _curveHashes.length;
        for (uint256 i = 0; i < _curveHashes.length; i = i.add(1)) {
            curves[i].hash = _curveHashes[i];
        }
    }


    /// @notice Anybody can reveal the next curve if he knows it.
    /// @param _limit Ceiling cap.
    ///  (must be greater or equal to the previous one).
    /// @param _last `true` if it's the last curve.
    /// @param _salt Random number used to commit the curve
    function revealCurve(uint256 _limit, uint256 _slopeFactor, uint256 _collectMinimum,
                         bool _last, bytes32 _salt) public {
        require(!allRevealed);

        require(curves[revealedCurves].hash == calculateHash(_limit, _slopeFactor, _collectMinimum,
                                                             _last, _salt));

        require(_limit != 0 && _slopeFactor != 0 && _collectMinimum != 0);
        if (revealedCurves > 0) {
            require(_limit >= curves[revealedCurves.sub(1)].limit);
        }

        curves[revealedCurves].limit = _limit;
        curves[revealedCurves].slopeFactor = _slopeFactor;
        curves[revealedCurves].collectMinimum = _collectMinimum;
        revealedCurves = revealedCurves.add(1);

        if (_last) allRevealed = true;
    }

    /// @notice Reveal multiple curves at once
    function revealMulti(uint256[] _limits, uint256[] _slopeFactors, uint256[] _collectMinimums,
                         bool[] _lasts, bytes32[] _salts) public {
        // Do not allow none and needs to be same length for all parameters
        require(_limits.length != 0 &&
                _limits.length == _slopeFactors.length &&
                _limits.length == _collectMinimums.length &&
                _limits.length == _lasts.length &&
                _limits.length == _salts.length);

        for (uint256 i = 0; i < _limits.length; i = i.add(1)) {
            revealCurve(_limits[i], _slopeFactors[i], _collectMinimums[i],
                        _lasts[i], _salts[i]);
        }
    }

    /// @notice Move to curve, used as a failsafe
    function moveTo(uint256 _index) public onlyOwner {
        require(_index < revealedCurves &&       // No more curves
                _index == currentIndex.add(1));  // Only move one index at a time
        currentIndex = _index;
    }

    /// @return Return the funds to collect for the current point on the curve
    ///  (or 0 if no curves revealed yet)
    function toCollect(uint256 collected) public onlyContribution returns (uint256) {
        if (revealedCurves == 0) return 0;

        // Move to the next curve
        if (collected >= curves[currentIndex].limit) {  // Catches `limit == 0`
            uint256 nextIndex = currentIndex.add(1);
            if (nextIndex >= revealedCurves) return 0;  // No more curves
            currentIndex = nextIndex;
            if (collected >= curves[currentIndex].limit) return 0;  // Catches `limit == 0`
        }

        // Everything left to collect from this limit
        uint256 difference = curves[currentIndex].limit.sub(collected);

        // Current point on the curve
        uint256 collect = difference.div(curves[currentIndex].slopeFactor);

        // Prevents paying too much fees vs to be collected; breaks long tail
        if (collect <= curves[currentIndex].collectMinimum) {
            if (difference > curves[currentIndex].collectMinimum) {
                return curves[currentIndex].collectMinimum;
            } else {
                return difference;
            }
        } else {
            return collect;
        }
    }

    /// @notice Calculates the hash of a curve.
    /// @param _limit Ceiling cap.
    /// @param _last `true` if it's the last curve.
    /// @param _salt Random number that will be needed to reveal this curve.
    /// @return The calculated hash of this curve to be used in the `setHiddenCurves` method
    function calculateHash(uint256 _limit, uint256 _slopeFactor, uint256 _collectMinimum,
                           bool _last, bytes32 _salt) public constant returns (bytes32) {
        return keccak256(_limit, _slopeFactor, _collectMinimum, _last, _salt);
    }

    /// @return Return the total number of curves committed
    ///  (can be larger than the number of actual curves on the curve to hide
    ///  the real number of curves)
    function nCurves() public constant returns (uint256) {
        return curves.length;
    }

}
```

首先在 remix 可视化界面创建合约，得到合约地址 0xcdbab7ad4c77046f61d7846a48e5edbe423c2775

假设设置 2 个真实隐藏上限以及 1 个冗余隐藏上限，其中真实隐藏上限为[100, 5, 2, false, 10000]、[200, 10, 4, true, 20000]，首先计算出真实隐藏上限哈希值：

```
// 进入 geth console
// 哈希的形式是 keccak256(100, 5, 2, false, 10000)
// 根据合约代码，除去 _lasts 变量，其余均为 256 位，bool 类型占一个字节，因此是 0x00 或者 0x01

> web3.toHex(100)
"0x64"
> web3.toHex(200)
"0xc8"
> web3.toHex(10000)
"0x2710"
> web3.toHex(20000)
"0x4e20"

> web3.sha3("000000000000000000000000000000000000000000000000000000000000006400000000000000000000000000000000000000000000000000000000000000050000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000002710",{encoding:"hex"})
"0xd995b218064414479be1e91d49a91c4543c87a1835fc9ceb6168cc3232efdf4b"

> web3.sha3("00000000000000000000000000000000000000000000000000000000000000c8000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000004010000000000000000000000000000000000000000000000000000000000004e20",{encoding:"hex"})
"0x1002d8c03b240668e2261b55118f91030c88018d687e3341c9b18af0cc313a83"
```

接着在 geth console 中调用 `setHiddenCurves` 函数

```
> var browser_dynamicceiling_sol_dynamicceilingContract = web3.eth.contract([{"constant":true,"inputs":[{"name":"","type":"uint256"}],"name":"curves","outputs":[{"name":"hash","type":"bytes32"},{"name":"limit","type":"uint256"},{"name":"slopeFactor","type":"uint256"},{"name":"collectMinimum","type":"uint256"}],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"currentIndex","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"nCurves","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"allRevealed","outputs":[{"name":"","type":"bool"}],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"contribution","outputs":[{"name":"","type":"address"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"_curveHashes","type":"bytes32[]"}],"name":"setHiddenCurves","outputs":[],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"_limits","type":"uint256[]"},{"name":"_slopeFactors","type":"uint256[]"},{"name":"_collectMinimums","type":"uint256[]"},{"name":"_lasts","type":"bool[]"},{"name":"_salts","type":"bytes32[]"}],"name":"revealMulti","outputs":[],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"_limit","type":"uint256"},{"name":"_slopeFactor","type":"uint256"},{"name":"_collectMinimum","type":"uint256"},{"name":"_last","type":"bool"},{"name":"_salt","type":"bytes32"}],"name":"revealCurve","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"revealedCurves","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":false,"inputs":[],"name":"acceptOwnership","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[{"name":"_limit","type":"uint256"},{"name":"_slopeFactor","type":"uint256"},{"name":"_collectMinimum","type":"uint256"},{"name":"_last","type":"bool"},{"name":"_salt","type":"bytes32"}],"name":"calculateHash","outputs":[{"name":"","type":"bytes32"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"collected","type":"uint256"}],"name":"toCollect","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"owner","outputs":[{"name":"","type":"address"}],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"_newOwner","type":"address"}],"name":"changeOwner","outputs":[],"payable":false,"type":"function"},{"constant":false,"inputs":[{"name":"_index","type":"uint256"}],"name":"moveTo","outputs":[],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"newOwner","outputs":[{"name":"","type":"address"}],"payable":false,"type":"function"},{"inputs":[{"name":"_owner","type":"address"},{"name":"_contribution","type":"address"}],"payable":false,"type":"constructor"}]);
undefined

> personal.unlockAccount(eth.coinbase)
Unlock account 0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37
Passphrase:
true
> miner.start()
null
> var myInstance = browser_dynamicceiling_sol_dynamicceilingContract.at("0xcdbab7ad4c77046f61d7846a48e5edbe423c2775")
undefined
> myInstance.setHiddenCurves.sendTransaction(["0xd995b218064414479be1e91d49a91c4543c87a1835fc9ceb6168cc3232efdf4b", "0x1002d8c03b240668e2261b55118f91030c88018d687e3341c9b18af0cc313a83", "0x00000000000000000000000000000000000000000000000000123456789abcde"], {from: eth.coinbase, gas : 1000000})
"0x296c1ed8cc13e51cc72180858c49cfe07e47f05a4ff23795113670e539d30cab"
> miner.stop()
true

> myInstance.nCurves()
3

// 我们来看一下真正的交易，可以发现 data 字段多了 0x00...20 与 0x00...03，这是由于函数的参数是动态数组，0x00...20 表示数组从第 32 个字节处开始，接下来是数组元素个数 3
> eth.getTransaction("0x296c1ed8cc13e51cc72180858c49cfe07e47f05a4ff23795113670e539d30cab")
{
  blockHash: "0x54904ffbd1105a68a24d38edf376ab6264bc9c0b5992c32fba8e3d12463b0a01",
  blockNumber: 3787,
  from: "0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37",
  gas: 1000000,
  gasPrice: 50000000000,
  hash: "0x296c1ed8cc13e51cc72180858c49cfe07e47f05a4ff23795113670e539d30cab",
  input: "0x54657f0a00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003d995b218064414479be1e91d49a91c4543c87a1835fc9ceb6168cc3232efdf4b1002d8c03b240668e2261b55118f91030c88018d687e3341c9b18af0cc313a8300000000000000000000000000000000000000000000000000123456789abcde",
  nonce: 36,
  r: "0xcff1d48e39fddf4110a8dcae35022da0306aea65b34e992c5c377e444d15cda4",
  s: "0x2aa7ea973fc63010cce7a7ab106ead4278b251c08ef4accf27d9f189703e984e",
  to: "0xcdbab7ad4c77046f61d7846a48e5edbe423c2775",
  transactionIndex: 0,
  v: "0x26",
  value: 0
}
// 披露上限, 其中 data 字段为：
//In [34]: for i in range(len(a) / 64) :
    ...:     print a[i*64:(i+1)*64]
// 00000000000000000000000000000000000000000000000000000000000000a0
// 0000000000000000000000000000000000000000000000000000000000000100
// 0000000000000000000000000000000000000000000000000000000000000160
// 00000000000000000000000000000000000000000000000000000000000001c0
// 0000000000000000000000000000000000000000000000000000000000000220
// 0000000000000000000000000000000000000000000000000000000000000002
// 0000000000000000000000000000000000000000000000000000000000000064
// 00000000000000000000000000000000000000000000000000000000000000c8
// 0000000000000000000000000000000000000000000000000000000000000002
// 0000000000000000000000000000000000000000000000000000000000000005
// 000000000000000000000000000000000000000000000000000000000000000a
// 0000000000000000000000000000000000000000000000000000000000000002
// 0000000000000000000000000000000000000000000000000000000000000002
// 0000000000000000000000000000000000000000000000000000000000000004
// 0000000000000000000000000000000000000000000000000000000000000002
// 0000000000000000000000000000000000000000000000000000000000000000
// 0000000000000000000000000000000000000000000000000000000000000001
// 0000000000000000000000000000000000000000000000000000000000000002
// 0000000000000000000000000000000000000000000000000000000000002710
// 0000000000000000000000000000000000000000000000000000000000004e20

// 前四项代表四个数组的起始位置，分别为 160、256、352、448
// 后面每三项构成一个数组，第一项是数组元素的个数

> myInstance.revealMulti.sendTransaction([100, 200], [5, 10], [2, 4], [0, 1], ["0x0000000000000000000000000000000000000000000000000000000000002710", "0x0000000000000000000000000000000000000000000000000000000000004e20"], {from : eth.coinbase, gas : 1000000})
"0x5d07eb8ff661bc77b078cf7bd416f92036af2a3bf5d4bac41756c073ba929a9f"

> eth.getTransaction("0x5d07eb8ff661bc77b078cf7bd416f92036af2a3bf5d4bac41756c073ba929a9f")
{
  blockHash: "0x2160f4bc8aeee99eb7841eaa4ad0f587c54227b2d97987b753dce46a2b89ca65",
  blockNumber: 3802,
  from: "0xa3ac96fbe4b0dce5f6f89a715ca00934d68f6c37",
  gas: 1000000,
  gasPrice: 50000000000,
  hash: "0x5d07eb8ff661bc77b078cf7bd416f92036af2a3bf5d4bac41756c073ba929a9f",
  input: "0x627adaa600000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000016000000000000000000000000000000000000000000000000000000000000001c000000000000000000000000000000000000000000000000000000000000002200000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000006400000000000000000000000000000000000000000000000000000000000000c800000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000005000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000027100000000000000000000000000000000000000000000000000000000000004e20",
  nonce: 38,
  r: "0x85eacb0db5e79f65a684efd23917db108e5a776f872a3dd87f1a505d8b5a3126",
  s: "0x323f52a60dd317dc957c241afae6b399cc67d24a1599d98a57739fe70846c62",
  to: "0xcdbab7ad4c77046f61d7846a48e5edbe423c2775",
  transactionIndex: 0,
  v: "0x26",
  value: 0
}
```
