# Memory-safe

在之前的例子中我们在每个`inline-assembly`块中都加入了`("memory-safe")`标识:

```solidity
function memoryWriteRead() public pure returns (bytes32 ret) {
     bytes32 data = bytes32(uint256(1));
     assembly ("memory-safe") {
         mstore(0x00, data)
         ret := mload(0x00)
     }
}
```

`memory-safe`关键字指示当前的`inline-assembly`遵守[solidity的内存模型](1-memory.md)，在这种情况下solidity编译器能够更好的优化编译的过程，尤其当`viaIR=true`时尤其重要。

注意：当`inline-assembly`内没有执行`MLOAD`、`MSTORE`等内存操作时，编译器会自动标记代码为`memory-safe`，而无论您是否使用`memory-safe`标识。



在启用了`memory-safe`后您只能按照以下的方式来访问内存:

1. 任意使用`mem[0x00:0x40]`之间的内存（临时内存）。
2. 按照 solidity 的方式申请内存并把最新的空闲内存指针写入到`mem[0x40:0x60]`中。
3. 按照 solidity 的方式申请内存但仅仅用作临时空间，而不更新内存指针到`mem[0x40:0x60]`。

通常来讲使用`memory-safe`能够编译出更加gas高效的程序，但是在少数的情况下可以通过不遵守solidity内存模型来获得更高的性能。



以下代码不是`memory-safe`的，原因：

因为`returndatasize()`可能会超过64（mem[0x00:0x40]）而导致使用的内存超出临时内存的范围

<div>
Code 
</div>

```solidity
assembly {
  returndatacopy(0, 0, returndatasize())
  revert(0, returndatasize())
}
```



以下代码是`memory-safe`的，原因：

因为按照solidity的方式来使用内存，因为最终执行了`revert`而不需要更新内存指针到`mem[0x40:0x60]`。

<div>
Code
</div>

```solidity
assembly ("memory-safe") {
  let p := mload(0x40)
  returndatacopy(p, 0, returndatasize())
  revert(p, returndatasize())
}
```



虽然启用`memory-safe`在多数情况下都能提升gas效率，但是在下面的示例中我们通过不遵守solidity的内存模型来提高性能:

<div>
Code
<a style="float: right;" href="https://remix.ethereum.org/?#language=yul&amp;version=0.8.26&amp;code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IEdQTC0zLjAKcHJhZ21hIHNvbGlkaXR5IF4wLjguMjA7Cgpjb250cmFjdCBNZW1vcnlTYWZlVGVzdCB7CiAgICBmYWxsYmFjaygpIGV4dGVybmFsIHBheWFibGUgewogICAgICAgIGFkZHJlc3MgYWRkciA9IGFkZHJlc3MoMTApOwogICAgICAgIGFzc2VtYmx5IHsKICAgICAgICAgICAgY2FsbGRhdGFjb3B5KDAsIDAsIGNhbGxkYXRhc2l6ZSgpKQogICAgICAgICAgICBsZXQgc3VjY2VzcyA6PSBkZWxlZ2F0ZWNhbGwoZ2FzKCksIGFkZHIsIDAsIGNhbGxkYXRhc2l6ZSgpLCAwLCAwKQogICAgICAgICAgICByZXR1cm5kYXRhY29weSgwLCAwLCByZXR1cm5kYXRhc2l6ZSgpKQogICAgICAgICAgICBpZiBpc3plcm8oc3VjY2VzcykgewogICAgICAgICAgICAgICAgcmV2ZXJ0KDAsIHJldHVybmRhdGFzaXplKCkpCiAgICAgICAgICAgIH0KICAgICAgICAgICAgcmV0dXJuKDAsIHJldHVybmRhdGFzaXplKCkpCiAgICAgICAgfQogICAgfQoKICAgIHJlY2VpdmUoKSBleHRlcm5hbCBwYXlhYmxlIHt9Cn0K" >open in Remix</a>
</div>

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.20;

contract MemorySafeTest {
    fallback() external payable {
        address addr = address(10);
        assembly {
            calldatacopy(0, 0, calldatasize())
            let success := delegatecall(gas(), addr, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            if iszero(success) {
                revert(0, returndatasize())
            }
            return(0, returndatasize())
        }
    }

    receive() external payable {}
}
```

在上面的例子中我们通过不遵守solidity内存模型（直接使用超过临时内存区域的内存）来避免内存扩展的费用以及可以执行更少的OpCode来降低执行成本。
