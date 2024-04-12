# via-IR

在 Solidity compiler 中启用`viaIR`参数能够使用经过 IR(intermediate representation) 优化的编译。

具体使用体现：

1. 大多数情况下编译后的字节码更加 gas 高效。
2. 可以编译更加复杂的代码, 可以解决部分`Stack too deep`的问题。

但是如果启用了 IR 则程序中有些默认的行为会产生变化。以下是一个例子：

<div>
Code
<a style="float: right;" href="https://remix.ethereum.org/?#language=yul&amp;version=0.8.26&amp;code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IEdQTC0zLjAKcHJhZ21hIHNvbGlkaXR5IF4wLjguMjA7Cgpjb250cmFjdCBJUlRlc3QgewogICAgc3RydWN0IERBVEEgewogICAgICAgIHVpbnQxMjggYTsKICAgIH0KCiAgICBEQVRBIGRhdGE7CgogICAgZnVuY3Rpb24gc2V0KCkgcHVibGljIHsKICAgICAgICBkYXRhLmEgPSAxMDsKICAgIH0KCiAgICBmdW5jdGlvbiBnZXQoKSBwdWJsaWMgdmlldyByZXR1cm5zICh1aW50MTI4KSB7CiAgICAgICAgcmV0dXJuIGRhdGEuYTsKICAgIH0KCiAgICBmdW5jdGlvbiBnZXRSYXcoKSBwdWJsaWMgdmlldyByZXR1cm5zIChieXRlczMyIF9kYXRhKSB7CiAgICAgICAgYXNzZW1ibHkgKCJtZW1vcnktc2FmZSIpIHsKICAgICAgICAgICAgX2RhdGEgOj0gc2xvYWQoZGF0YS5zbG90KQogICAgICAgIH0KICAgIH0KCiAgICBmdW5jdGlvbiBzZXRFeHQoKSBwdWJsaWMgewogICAgICAgIHVpbnQxMjggZXh0ID0gMHhmOwogICAgICAgIGFzc2VtYmx5ICgibWVtb3J5LXNhZmUiKSB7CiAgICAgICAgICAgIGxldCBfZGF0YSA6PSBzbG9hZChkYXRhLnNsb3QpCiAgICAgICAgICAgIF9kYXRhIDo9IGFuZChfZGF0YSwgMHhmZmZmZmZmZmZmZmZmZmZmZmZmZmZmZmZmZmZmZmZmZikKICAgICAgICAgICAgX2RhdGEgOj0gb3IoX2RhdGEsIHNobChtdWwoMTYsIDgpLCBleHQpKQogICAgICAgICAgICBzc3RvcmUoZGF0YS5zbG90LCBfZGF0YSkKICAgICAgICB9CiAgICB9CgogICAgZnVuY3Rpb24gZ2V0RXh0KCkgcHVibGljIHZpZXcgcmV0dXJucyAodWludDEyOCBfZGF0YSkgewogICAgICAgIGFzc2VtYmx5ICgibWVtb3J5LXNhZmUiKSB7CiAgICAgICAgICAgIF9kYXRhIDo9IHNsb2FkKGRhdGEuc2xvdCkKICAgICAgICAgICAgX2RhdGEgOj0gc2hyKG11bCg4LCAxNiksIF9kYXRhKQogICAgICAgIH0KICAgIH0KCiAgICBmdW5jdGlvbiBkZWwoKSBwdWJsaWMgewogICAgICAgIGRlbGV0ZSBkYXRhOwogICAgfQp9Cg==" >open in Remix</a>
</div>

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.20;

contract IRTest {
    struct DATA {
        uint128 a;
    }

    DATA data;

    function set() public {
        data.a = 10;
    }

    function get() public view returns (uint128) {
        return data.a;
    }

    function getRaw() public view returns (bytes32 _data) {
        assembly ("memory-safe") {
            _data := sload(data.slot)
        }
    }

    function setExt() public {
        uint128 ext = 0xf;
        assembly ("memory-safe") {
            let _data := sload(data.slot)
            _data := and(_data, 0xffffffffffffffffffffffffffffffff)
            _data := or(_data, shl(mul(16, 8), ext))
            sstore(data.slot, _data)
        }
    }

    function getExt() public view returns (uint128 _data) {
        assembly ("memory-safe") {
            _data := sload(data.slot)
            _data := shr(mul(8, 16), _data)
        }
    }

    function del() public {
        delete data;
    }
}

```

在上面的示例中`data`中的元素只占用了 16bytes，在 `viaIR=false`时执行 `del()`仅仅会清除`data`所使用的16bytes，但是在`viaIR=true`时执行`del()`则会删除整个slot，从而改变`getExt()`的结果。
