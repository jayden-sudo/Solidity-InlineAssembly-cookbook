# Memory 读写

EVM 中的内存为线性内存(紧密且连续排列的)。

Solidity 中的内存使用规范：

```
┌────────────────┬───────────────────────┐
│ Memory offset  │  Notes                │
├────────────────┼───────────────────────┤
│ 0x00           │  scratch space        │
├────────────────┼───────────────────────┤
│ 0x20           │  scratch space        │
├────────────────┼───────────────────────┤
│ 0x40           │  free memory pointer  │
├────────────────┼───────────────────────┤
│ 0x60           │                       │
├────────────────┼───────────────────────┤
│ ...            │                       │
└────────────────┴───────────────────────┘

Memory[0x00:0x40] 为临时存储，应用可以随时读写此 64bytes 而不需要支付内存扩展的成本。
Memory[0x40:0x60] 记录了当前未扩展(未申请)内存起始地址。
Memory[0x60:]     从 0x60 起为 solidity 程序可以动态申请的内存。
```



最常用的内存操作 opcode 为：[MLOAD、MSTORE](https://ethereum.org/en/developers/docs/evm/opcodes/)

MLOAD：从指定的内存地址读取 32bytes

MSTORE：向指定的内存地址写入 32bytes



写入以及读取一块内存的示例：

<div>
Code
<a style="float: right;" href="https://remix.ethereum.org/?#language=yul&amp;version=0.8.26&amp;code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IEdQTC0zLjAKcHJhZ21hIHNvbGlkaXR5IF4wLjguMjA7Cgpjb250cmFjdCBNZW1vcnkgewogICAgZnVuY3Rpb24gbWVtb3J5V3JpdGVSZWFkKCkgcHVibGljIHB1cmUgcmV0dXJucyAoYnl0ZXMzMiByZXQpIHsKICAgICAgICBieXRlczMyIGRhdGEgPSBieXRlczMyKHVpbnQyNTYoMSkpOwogICAgICAgIGFzc2VtYmx5ICgibWVtb3J5LXNhZmUiKSB7CiAgICAgICAgICAgIC8vIHN0b3JlIGRhdGEgdG8gbWVtWzB4MDA6MHgyMF0KICAgICAgICAgICAgbXN0b3JlKDB4MDAsIGRhdGEpCiAgICAgICAgICAgIC8vIHJlYWQgZGF0YSBmcm9tIG1lbVsweDAwOjB4MjBdCiAgICAgICAgICAgIHJldCA6PSBtbG9hZCgweDAwKQogICAgICAgIH0KICAgIH0KfQ==" >open in Remix</a>
</div>

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.20;

contract Memory {
    function memoryWriteRead() public pure returns (bytes32 ret) {
        bytes32 data = bytes32(uint256(1));
        assembly ("memory-safe") {
            // store data to mem[0x00:0x20]
            mstore(0x00, data)
            // read data from mem[0x00:0x20]
            ret := mload(0x00)
        }
    }
}
```

在上面的示例中我们通过`mstore`把data(0x1)存储到了临时存储`mem[0x00:0x20]`中，然后在通过`mload`读取这个临时存储中的数据并输出。



按照 Solidity 内存规范来申请一块新内存并完成上面的操作:

<div>
Code
<a style="float: right;" href="https://remix.ethereum.org/?#language=yul&amp;version=0.8.26&amp;code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IEdQTC0zLjAKcHJhZ21hIHNvbGlkaXR5IF4wLjguMjA7Cgpjb250cmFjdCBNZW1vcnkgewogICAgZnVuY3Rpb24gbWVtb3J5V3JpdGVSZWFkKCkgcHVibGljIHB1cmUgcmV0dXJucyAoYnl0ZXMzMiByZXQpIHsKICAgICAgICBieXRlczMyIGRhdGEgPSBieXRlczMyKHVpbnQyNTYoMSkpOwogICAgICAgIGFzc2VtYmx5ICgibWVtb3J5LXNhZmUiKSB7CiAgICAgICAgICAgIGZ1bmN0aW9uIGFsbG9jYXRlKCkgLT4gcG9zIHsKICAgICAgICAgICAgICAgIHBvcyA6PSBtbG9hZCgweDQwKQogICAgICAgICAgICAgICAgbXN0b3JlKDB4NDAsIGFkZChwb3MsIDMyKSkKICAgICAgICAgICAgfQogICAgICAgICAgICBsZXQgcHRyIDo9IGFsbG9jYXRlKCkKICAgICAgICAgICAgLy8gc3RvcmUgZGF0YSB0byBtZW1bcHRyOnB0cisweDIwXQogICAgICAgICAgICBtc3RvcmUocHRyLCBkYXRhKQogICAgICAgICAgICAvLyByZWFkIGRhdGEgZnJvbSBtZW1bcHRyOnB0cisweDIwXQogICAgICAgICAgICByZXQgOj0gbWxvYWQocHRyKQogICAgICAgIH0KICAgIH0KfQ==" >open in Remix</a>
</div>
```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.20;

contract Memory {
    function memoryWriteRead() public pure returns (bytes32 ret) {
        bytes32 data = bytes32(uint256(1));
        assembly ("memory-safe") {
            function allocate() -> pos {
                pos := mload(0x40)
                mstore(0x40, add(pos, 32))
            }
            let ptr := allocate()
            // store data to mem[ptr:ptr+0x20]
            mstore(ptr, data)
            // read data from mem[ptr:ptr+0x20]
            ret := mload(ptr)
        }
    }
}
```

在上面的示例中我们引入了 `inline assembly` 中的`function allocate()`，函数`allocate`的作用是从`mem[0x40:0x60]`读取出空闲内存的指针，并更新`mem[0x40:0x60]`中记录的空闲内存指针。此处`add(pos, 32)`中的32表示申请 32bytes 内存。



因为EVM使用了线性内存，意味着当申请新内存时使用了越高内存地址的内存则 gas 越高。我们通过下面的示例来测试写入 `mem[0x100:0x120]` 、 `mem[0x1000:0x1020]`、 `mem[0x10000:0x10020]`的成本区别：
<div>
Code
<a style="float: right;" href="https://remix.ethereum.org/?#language=yul&amp;version=0.8.26&amp;code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IEdQTC0zLjAKcHJhZ21hIHNvbGlkaXR5IF4wLjguMjA7Cgpjb250cmFjdCBNZW1vcnkgewogICAgZnVuY3Rpb24gbWVtb3J5V3JpdGUxKCkgcHVibGljIHZpZXcgcmV0dXJucyAodWludDI1NiBnYXNDb3N0KSB7CiAgICAgICAgdWludDI1NiBnYXMxID0gZ2FzbGVmdCgpOwogICAgICAgIGFzc2VtYmx5ICgibWVtb3J5LXNhZmUiKSB7CiAgICAgICAgICAgIG1zdG9yZSgweDEwMCwgMSkKICAgICAgICB9CiAgICAgICAgdWludDI1NiBnYXMyID0gZ2FzbGVmdCgpOwogICAgICAgIGdhc0Nvc3QgPSBnYXMxIC0gZ2FzMjsKICAgIH0KCiAgICBmdW5jdGlvbiBtZW1vcnlXcml0ZTIoKSBwdWJsaWMgdmlldyByZXR1cm5zICh1aW50MjU2IGdhc0Nvc3QpIHsKICAgICAgICB1aW50MjU2IGdhczEgPSBnYXNsZWZ0KCk7CiAgICAgICAgYXNzZW1ibHkgKCJtZW1vcnktc2FmZSIpIHsKICAgICAgICAgICAgbXN0b3JlKDB4MTAwMCwgMSkKICAgICAgICB9CiAgICAgICAgdWludDI1NiBnYXMyID0gZ2FzbGVmdCgpOwogICAgICAgIGdhc0Nvc3QgPSBnYXMxIC0gZ2FzMjsKICAgIH0KCiAgICBmdW5jdGlvbiBtZW1vcnlXcml0ZTMoKSBwdWJsaWMgdmlldyByZXR1cm5zICh1aW50MjU2IGdhc0Nvc3QpIHsKICAgICAgICB1aW50MjU2IGdhczEgPSBnYXNsZWZ0KCk7CiAgICAgICAgYXNzZW1ibHkgKCJtZW1vcnktc2FmZSIpIHsKICAgICAgICAgICAgbXN0b3JlKDB4MTAwMDAsIDEpCiAgICAgICAgfQogICAgICAgIHVpbnQyNTYgZ2FzMiA9IGdhc2xlZnQoKTsKICAgICAgICBnYXNDb3N0ID0gZ2FzMSAtIGdhczI7CiAgICB9Cn0K" >open in Remix</a>
</div>

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.20;

contract Memory {
    function memoryWrite1() public view returns (uint256 gasCost) {
        uint256 gas1 = gasleft();
        assembly ("memory-safe") {
            mstore(0x100, 1)
        }
        uint256 gas2 = gasleft();
        gasCost = gas1 - gas2;
    }

    function memoryWrite2() public view returns (uint256 gasCost) {
        uint256 gas1 = gasleft();
        assembly ("memory-safe") {
            mstore(0x1000, 1)
        }
        uint256 gas2 = gasleft();
        gasCost = gas1 - gas2;
    }

    function memoryWrite3() public view returns (uint256 gasCost) {
        uint256 gas1 = gasleft();
        assembly ("memory-safe") {
            mstore(0x10000, 1)
        }
        uint256 gas2 = gasleft();
        gasCost = gas1 - gas2;
    }
}
```

以下是上面三个示例的gas消耗：

```
┌─────────────────────┬────────────┐
│ Opcode              │  Gas Cost  │
├─────────────────────┼────────────┤
│ mstore(0x100,   1)  │  29   gas  │
├─────────────────────┼────────────┤
│ mstore(0x1000,  1)  │  421  gas  │
├─────────────────────┼────────────┤
│ mstore(0x10000, 1)  │  14349 gas │
└─────────────────────┴────────────┘
```

上面的这个例子能够非常直观地看到[内存扩展的成本](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#a0-1-memory-expansion)。（注意：仅仅执行 mload 并不会产生扩展内存的成本）



