# Storage 读写

EVM 中的 Storage(永久存储) 与 Memory 类似的地方是通常使用的时候都是写入或者读取一个`bytes32`。但是使用Storage存储数据并不会产生像Memory一样的扩展成本。

因为Storage中的数据会永久存储在区块链中，最终导致使用成本非常昂贵。

最常用的Storage操作 opcode 为：[SLOAD、SSTORE](https://ethereum.org/en/developers/docs/evm/opcodes/)

SLOAD：从指定的存储地址(slot)读取  32bytes

SSTORE：向指定的存储地址(slot)写入 32bytes



[SLOAD](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#a6-sload)的成本：

1. 当在一个 transaction 中第一次访问某个 slot 时：
   SLOAD: 2100 gas 
2. 当在一个 transaction 中非第一次访问某个 slot 时：
   SLOAD:  100 gas

[SSTORE](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#a7-sstore)的成本：

1. SSTORE的成本计算相对复杂，可以直接查看：https://github.com/wolflo/evm-opcodes/blob/main/gas.md#a7-sstore

2. 简略来说成本的部分计算过程如下:

   ```c
   int gasCost = 0;
   if (新存储与原存储值相等){
      gasCost += 2200;
   } else{
      if(原存储为空){
         gasCost += 22100;
      } else if(新存储为空){
         gasCost += 5000; //注意,此处会产生gas退还(refund),但是不会直接体现的SSTORE的执行成本中，需要交易执行完成之后才会执行gas退还。
      } else{
         gasCost += 5000;
      }
   }
   // 上面仅仅是为了避免陷入复杂的规则中，而提供了近似的简化规则。
   ```
   
   



写入以及读取存储的示例：

<div>
Code
<a style="float: right;" href="https://remix.ethereum.org/?#language=yul&amp;version=0.8.26&amp;code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IEdQTC0zLjAKcHJhZ21hIHNvbGlkaXR5IF4wLjguMjA7CmNvbnRyYWN0IFN0b3JhZ2UgewogICAgZnVuY3Rpb24gc3RvcmFnZVdyaXRlKCkgcHVibGljIHsKICAgICAgICBhc3NlbWJseSAoIm1lbW9yeS1zYWZlIikgewogICAgICAgICAgICBzc3RvcmUoMHgxMDAsIDEpCiAgICAgICAgfQogICAgfQoKICAgIGZ1bmN0aW9uIHN0b3JhZ2VSZWFkKCkgcHVibGljIHZpZXcgcmV0dXJucyAodWludDI1NiByZXQpIHsKICAgICAgICBhc3NlbWJseSAoIm1lbW9yeS1zYWZlIikgewogICAgICAgICAgICByZXQgOj0gc2xvYWQoMHgxMDApCiAgICAgICAgfQogICAgfQp9" >open in Remix</a>
</div>

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.20;
contract Storage {
    function storageWrite() public {
        assembly ("memory-safe") {
            sstore(0x100, 1)
        }
    }

    function storageRead() public view returns (uint256 ret) {
        assembly ("memory-safe") {
            ret := sload(0x100)
        }
    }
}
```

在上面的示例中我们通过`sstore`把`1`存储到了存储`storage[0x100:0x120]`中，然后在通过sload`读取这个存储中的数据并输出。



下面我们通过几个示例来直观地感受在不同情况下执行存储指令时的gas消耗:
<div>
Code
<a style="float: right;" href="https://remix.ethereum.org/?#language=yul&amp;version=0.8.26&amp;code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IEdQTC0zLjAKcHJhZ21hIHNvbGlkaXR5IF4wLjguMjA7CmltcG9ydCAiaGFyZGhhdC9jb25zb2xlLnNvbCI7Cgpjb250cmFjdCBTdG9yYWdlIHsKICAgIGJ5dGVzMzIgaW50ZXJuYWwgY29uc3RhbnQga2V5MSA9IGJ5dGVzMzIodWludDI1NigxMDEpKTsKICAgIGJ5dGVzMzIgaW50ZXJuYWwgY29uc3RhbnQga2V5MiA9IGJ5dGVzMzIodWludDI1NigxMDIpKTsKICAgIGJ5dGVzMzIgaW50ZXJuYWwgY29uc3RhbnQga2V5MyA9IGJ5dGVzMzIodWludDI1NigxMDMpKTsKCiAgICBjb25zdHJ1Y3RvcigpIHsKICAgICAgICBieXRlczMyIF9rZXkxID0ga2V5MTsKICAgICAgICBieXRlczMyIF9rZXkyID0ga2V5MjsKICAgICAgICBhc3NlbWJseSAoIm1lbW9yeS1zYWZlIikgewogICAgICAgICAgICBzc3RvcmUoX2tleTEsIDB4ZjEpCiAgICAgICAgICAgIHNzdG9yZShfa2V5MiwgMHhmMikKICAgICAgICB9CiAgICB9CgogICAgZnVuY3Rpb24gc3RvcmFnZVdyaXRlMSgpIHB1YmxpYyB7CiAgICAgICAgYnl0ZXMzMiBfa2V5MSA9IGtleTE7CiAgICAgICAgdWludDI1NiBnYXMxID0gZ2FzbGVmdCgpOwogICAgICAgIGFzc2VtYmx5ICgibWVtb3J5LXNhZmUiKSB7CiAgICAgICAgICAgIHNzdG9yZShfa2V5MSwgMHhmMSkKICAgICAgICB9CiAgICAgICAgdWludDI1NiBnYXMyID0gZ2FzbGVmdCgpOwogICAgICAgIGNvbnNvbGUubG9nKCJTU1RPUkUgXHQgbmV3X3ZhbCA9PSBjdXJyZW50X3ZhbDpcdCAlZGdhcyIsIGdhczEgLSBnYXMyKTsKICAgIH0KCiAgICBmdW5jdGlvbiBzdG9yYWdlV3JpdGUyKCkgcHVibGljIHsKICAgICAgICBieXRlczMyIF9rZXkyID0ga2V5MjsKICAgICAgICB1aW50MjU2IGdhczEgPSBnYXNsZWZ0KCk7CiAgICAgICAgYXNzZW1ibHkgKCJtZW1vcnktc2FmZSIpIHsKICAgICAgICAgICAgc3N0b3JlKF9rZXkyLCAwKQogICAgICAgIH0KICAgICAgICB1aW50MjU2IGdhczIgPSBnYXNsZWZ0KCk7CiAgICAgICAgY29uc29sZS5sb2coIlNTVE9SRSBcdCBjdXJyZW50X3ZhbCAhPSAwLCBuZXdfdmFsID09IDA6XHQgJWRnYXMiLCBnYXMxIC0gZ2FzMik7CgogICAgICAgIGFzc2VtYmx5ICgibWVtb3J5LXNhZmUiKSB7CiAgICAgICAgICAgIHNzdG9yZShfa2V5MiwgMHhmMikKICAgICAgICB9CiAgICB9CgogICAgZnVuY3Rpb24gc3RvcmFnZVdyaXRlMygpIHB1YmxpYyB7CiAgICAgICAgYnl0ZXMzMiBfa2V5MyA9IGtleTM7CiAgICAgICAgdWludDI1NiBnYXMxID0gZ2FzbGVmdCgpOwogICAgICAgIGFzc2VtYmx5ICgibWVtb3J5LXNhZmUiKSB7CiAgICAgICAgICAgIHNzdG9yZShfa2V5MywgMHgwMSkKICAgICAgICB9CiAgICAgICAgdWludDI1NiBnYXMyID0gZ2FzbGVmdCgpOwogICAgICAgIGNvbnNvbGUubG9nKCJTU1RPUkUgXHQgY3VycmVudF92YWwgPT0wLCBuZXdfdmFsICE9IDA6XHQgJWRnYXMiLCBnYXMxIC0gZ2FzMik7CiAgICB9Cn0=" >open in Remix</a>
</div>

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.20;
import "hardhat/console.sol";

contract Storage {
    bytes32 internal constant key1 = bytes32(uint256(101));
    bytes32 internal constant key2 = bytes32(uint256(102));
    bytes32 internal constant key3 = bytes32(uint256(103));

    constructor() {
        bytes32 _key1 = key1;
        bytes32 _key2 = key2;
        assembly ("memory-safe") {
            sstore(_key1, 0xf1)
            sstore(_key2, 0xf2)
        }
    }

    function storageWrite1() public {
        bytes32 _key1 = key1;
        uint256 gas1 = gasleft();
        assembly ("memory-safe") {
            sstore(_key1, 0xf1)
        }
        uint256 gas2 = gasleft();
        console.log("SSTORE \t new_val == current_val:\t %dgas", gas1 - gas2);
    }

    function storageWrite2() public {
        bytes32 _key2 = key2;
        uint256 gas1 = gasleft();
        assembly ("memory-safe") {
            sstore(_key2, 0)
        }
        uint256 gas2 = gasleft();
        console.log("SSTORE \t current_val != 0, new_val == 0:\t %dgas", gas1 - gas2);

        assembly ("memory-safe") {
            sstore(_key2, 0xf2)
        }
    }

    function storageWrite3() public {
        bytes32 _key3 = key3;
        uint256 gas1 = gasleft();
        assembly ("memory-safe") {
            sstore(_key3, 0x01)
        }
        uint256 gas2 = gasleft();
        console.log("SSTORE \t current_val ==0, new_val != 0:\t %dgas", gas1 - gas2);
    }
}
```

执行完成上面的三个函数后的输出：

```shell
SSTORE new_val == current_val:          2208 gas
SSTORE current_val != 0, new_val == 0:  5007 gas
SSTORE current_val ==0, new_val != 0:  22108 gas
```



在一个空的 slot 写入数据需要消耗 22k gas，很多时候为了避免这种昂贵的消耗我们会通过下面两种方法来降低gas消耗:

1. 多个变量共用一个 storage slot。
   - 即使不使用 inline assembly, 也可以通过 `struct`来实现，这个我们后面会讲到。
2. 预期未来某个 slot 还会使用的情况下则不要彻底清空数据
   - 例如原本清空数据是执行`SSTORE(x, 0)`, 而我们可以使用`SSTORE(x, 1)`来代替，这样以后再向 slot x 写入数据是就仅仅需要支付 5k gas。
2. 尽量避免使用 storage。
   - 在一些受控的模块化组合合约中，我们往往需要定义每个外部合约的类型，通常来讲可以通过 storage 来存储。但是为了极致地追求gas效率,我们也可以通过生成指定规则的外部合约地址来预设外部合约的类型，例如通过对外部合约地址的模运算来确定这个外部合约地址的类型。( 例如通过执行 `(0x.....01) % 255` 来把地址进行分类，而不需要额外的存储读写)。
