---
sip: <to be assigned>
title: EIP-7702 Compatibility Standard
description: Defines how Seaport contracts handle EIP-7702 delegated EOAs for activation guards, signature verification, and zone interactions.
author: Ridwan Nurudeen (@Ridwannurudeen)
discussions-to: https://github.com/ProjectOpenSea/seaport/discussions/1394
status: Draft
type: Standards Track
category: Core
created: 2026-03-15
---

## Abstract

[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702), introduced in Ethereum's Pectra upgrade, allows Externally Owned Accounts (EOAs) to temporarily delegate code execution to smart contracts via `SET_CODE_TX` transactions. This fundamentally breaks the invariant that `tx.origin == msg.sender` implies no code execution context, which the Seaport ecosystem relies on in the standalone `tstorish` library's `__activateTstore()` guard. This SIP specifies: (1) a replacement EOA detection pattern for activation guards, (2) a standard utility interface for detecting EIP-7702 delegation designators, (3) updated guidance for zones and contract offerers that use `extcodesize`-based account classification, and (4) signature verification considerations for delegated EOAs.

## Motivation

The Pectra upgrade (live on Ethereum mainnet) introduced EIP-7702, which sets an account's code to a 23-byte delegation designator (`0xef0100 || delegate_address`). This breaks several assumptions in the Seaport ecosystem:

### 1. `__activateTstore()` Guard Bypass in Standalone tstorish

The [`tstorish`](https://github.com/ProjectOpenSea/tstorish/blob/main/src/Tstorish.sol) library uses `msg.sender != tx.origin` as a guard to ensure `__activateTstore()` is only callable by pure EOAs:

```solidity
function __activateTstore() external {
    if (msg.sender != tx.origin) {
        revert OnlyDirectCalls();
    }
    // ... activates TSTORE support
}
```

Under EIP-7702, a delegated EOA can execute arbitrary code while maintaining `msg.sender == tx.origin`. This enables the following attack:

1. An EOA delegates to a malicious contract via `SET_CODE_TX`.
2. The EOA sends a transaction that enters a `tstorish`-inheriting contract, setting the reentrancy guard via SSTORE.
3. During execution, the delegated code calls `__activateTstore()`. The `msg.sender != tx.origin` check passes because both are the EOA address.
4. `_tstoreSupport` is set to `true`, switching all subsequent guard reads to TSTORE.
5. The reentrancy guard was written via SSTORE but is now read via TLOAD (which returns 0/unset).
6. The reentrancy guard is effectively bypassed, enabling reentrant calls into protected functions.

Note: The `seaport-core` [`ReentrancyGuard.sol`](https://github.com/ProjectOpenSea/seaport-core/blob/main/src/lib/ReentrancyGuard.sol) uses a guard-state check (`sload(_REENTRANCY_GUARD_SLOT) == _NOT_ENTERED_SSTORE`) instead of `msg.sender != tx.origin`, making it resilient to this specific vector. However, any contract inheriting the standalone `tstorish` library remains vulnerable.

### 2. `extcodesize` Account Classification

Seaport's signature verification in [`SignatureVerification.sol`](https://github.com/ProjectOpenSea/seaport-core/blob/main/src/lib/SignatureVerification.sol) uses `extcodesize(signer)` in its error-handling path to distinguish between EOA signature failures and contract signature failures:

```assembly
if extcodesize(signer) {
    // Bad contract signature
    mstore(0, BadContractSignature_error_selector)
    revert(...)
}
```

For a 7702-delegated EOA, `extcodesize` returns 23 (the delegation designator size), causing valid ECDSA signature failures to be misclassified as `BadContractSignature` instead of the appropriate ECDSA-specific error (`InvalidSignature`, `InvalidSigner`, or `BadSignatureV`).

### 3. Zone and Contract Offerer Assumptions

Custom zones implementing SIP-7 (Server-Signed Orders) or SIP-15 (Dynamic Traits Enforcement) may use `extcodesize`-based checks to determine whether an offerer or fulfiller is an EOA or contract. Under EIP-7702, these checks misclassify delegated EOAs, potentially causing:

- Incorrect signature verification path selection (ECDSA vs. ERC-1271).
- Zone authorization failures for legitimate delegated EOA users.
- Incorrect fee calculations or permission grants based on account type.

### 4. Broken Invariants

EIP-7702 explicitly breaks three invariants that Seaport-adjacent code may rely on:

| Invariant | Pre-7702 | Post-7702 |
|---|---|---|
| `tx.origin == msg.sender` implies top-level EOA call | True | False — delegated EOA can make nested calls while maintaining equality |
| An account's balance can only decrease from txs originating from it | True | False — delegated code can transfer funds |
| EOA nonce cannot increase after tx execution begins | True | False — delegated code can send txs |

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

### 1. EOA Detection for Activation Guards

Contracts using EOA-only guards (such as `__activateTstore()`) MUST NOT rely on `msg.sender != tx.origin` to enforce EOA-only access. Instead, contracts MUST use a code-length check:

```solidity
function __activateTstore() external {
    // Ensure the caller has no code (pure EOA, not delegated).
    if (msg.sender.code.length != 0) {
        revert OnlyDirectCalls();
    }
    // ... remainder of activation logic
}
```

This correctly rejects 7702-delegated EOAs because their `code.length` is 23 (the delegation designator), not 0.

Contracts that intentionally wish to allow delegated EOAs while blocking regular contracts MAY use the delegation detection pattern specified in Section 2.

### 2. EIP-7702 Delegation Detection Interface

A standard utility for detecting and parsing EIP-7702 delegation designators SHALL be defined as follows:

```solidity
/// @notice Utilities for detecting EIP-7702 delegated EOAs.
interface ISIP_EIP7702Utils {
    /// @notice The 3-byte prefix of an EIP-7702 delegation designator.
    /// @dev Equal to 0xef0100 as specified in EIP-7702.
    // bytes3 constant EIP7702_DELEGATION_PREFIX = 0xef0100;

    /// @notice Returns the delegate address if the account has an active
    ///         EIP-7702 delegation, or address(0) if it does not.
    /// @param account The address to check.
    /// @return delegate The delegate address, or address(0).
    function fetchDelegate(address account) external view returns (address delegate);

    /// @notice Returns true if the account has an active EIP-7702 delegation.
    /// @param account The address to check.
    /// @return True if the account has a delegation designator.
    function isDelegatedEOA(address account) external view returns (bool);

    /// @notice Returns true if the account is a pure EOA (no code, no delegation).
    /// @param account The address to check.
    /// @return True if the account has no code.
    function isPureEOA(address account) external view returns (bool);
}
```

The reference implementation of these functions SHALL be:

```solidity
library SIP_EIP7702Utils {
    bytes3 internal constant EIP7702_DELEGATION_PREFIX = 0xef0100;
    uint256 internal constant DELEGATION_DESIGNATOR_LENGTH = 23;

    /// @notice Returns the delegate address for a 7702-delegated account,
    ///         or address(0) if the account is not delegated.
    function fetchDelegate(address account) internal view returns (address) {
        if (account.code.length != DELEGATION_DESIGNATOR_LENGTH) {
            return address(0);
        }
        bytes23 designator = bytes23(account.code);
        if (bytes3(designator) != EIP7702_DELEGATION_PREFIX) {
            return address(0);
        }
        return address(bytes20(designator << 24));
    }

    /// @notice Returns true if the account has an active EIP-7702 delegation.
    function isDelegatedEOA(address account) internal view returns (bool) {
        return fetchDelegate(account) != address(0);
    }

    /// @notice Returns true if the account is a pure EOA with no code.
    function isPureEOA(address account) internal view returns (bool) {
        return account.code.length == 0;
    }
}
```

### 3. Signature Verification Guidance

Seaport's existing signature verification strategy — attempting ECDSA recovery first, then falling back to ERC-1271 — is compatible with EIP-7702 for the primary verification path. A delegated EOA's private key still produces valid ECDSA signatures regardless of delegation status.

However, the error classification path SHOULD be updated to account for delegated EOAs. When ECDSA recovery fails and ERC-1271 returns an invalid value, the error classification SHOULD distinguish between:

- **Pure contracts** (`code.length > 23` or `code.length > 0 && !isDelegatedEOA`): Revert with `BadContractSignature`.
- **Delegated EOAs** (`isDelegatedEOA == true`): Revert with `InvalidSigner` or a new `BadDelegatedEOASignature` error, since the account is fundamentally an EOA with a failed ECDSA check.
- **Pure EOAs** (`code.length == 0`): Revert with the existing ECDSA-specific errors (`InvalidSignature`, `InvalidSigner`, `BadSignatureV`).

Off-chain tooling that selects between ECDSA and ERC-1271 signing paths based on `extcodesize` MUST be updated to check for the delegation designator prefix. Delegated EOAs SHOULD use ECDSA signing (the underlying EOA key), not ERC-1271, unless the delegated contract explicitly implements `isValidSignature`.

### 4. Zone and Contract Offerer Guidance

Zones and contract offerers implementing SIP-5, SIP-6, SIP-7, or SIP-15 that perform account-type classification MUST NOT use `extcodesize(account) == 0` as the sole indicator of an EOA. The following patterns are affected:

#### 4a. Zones Using `isContract()` Checks

```solidity
// INCORRECT — misclassifies delegated EOAs as contracts:
function isContract(address account) internal view returns (bool) {
    return account.code.length > 0;
}

// CORRECT — accounts for 7702 delegation:
function isContract(address account) internal view returns (bool) {
    return account.code.length > 0
        && !SIP_EIP7702Utils.isDelegatedEOA(account);
}
```

#### 4b. Zones Selecting Signature Verification Paths

Zones that decide between ECDSA and ERC-1271 verification based on `extcodesize` MUST treat delegated EOAs as ECDSA-capable:

```solidity
// INCORRECT:
if (account.code.length > 0) {
    // ERC-1271 path
} else {
    // ECDSA path
}

// CORRECT:
if (account.code.length > 0 && !SIP_EIP7702Utils.isDelegatedEOA(account)) {
    // ERC-1271 path (true contract)
} else {
    // ECDSA path (pure EOA or delegated EOA)
}
```

#### 4c. Zones Restricting Access to EOAs

Zones that restrict certain operations to EOAs (e.g., to prevent contract-based MEV or sandwich attacks) SHOULD consider their policy for delegated EOAs explicitly. A delegated EOA is technically an EOA (it has a private key, can sign ECDSA) but can execute arbitrary code. Zones MUST document whether their EOA restriction intends to:

- **Restrict to keyholders only** (allow delegated EOAs): Use `isPureEOA(account) || isDelegatedEOA(account)`.
- **Restrict to non-programmable accounts** (block delegated EOAs): Use `isPureEOA(account)` only.

### 5. Reentrancy Guard Best Practices

Contracts in the Seaport ecosystem MUST NOT use `require(tx.origin == msg.sender)` as a reentrancy guard. Contracts MUST use mutex-based reentrancy guards (SSTORE or TSTORE) as implemented in `seaport-core`'s [`ReentrancyGuard.sol`](https://github.com/ProjectOpenSea/seaport-core/blob/main/src/lib/ReentrancyGuard.sol).

The `seaport-core` reentrancy guard uses a storage-slot state machine with explicit states (`_NOT_ENTERED_SSTORE`, `_ENTERED_SSTORE`, `_ENTERED_AND_ACCEPTING_NATIVE_TOKENS_SSTORE`, `_TSTORE_ENABLED_SSTORE`) and is already resilient to EIP-7702 because:

1. `__activateTstore()` checks `sload(_REENTRANCY_GUARD_SLOT) == _NOT_ENTERED_SSTORE`, which is false during execution (the slot contains `_ENTERED_SSTORE` or `_ENTERED_AND_ACCEPTING_NATIVE_TOKENS_SSTORE`).
2. No `tx.origin` or `msg.sender` identity checks are used.

Contracts using the standalone `tstorish` library MUST upgrade to either:
- The `seaport-core` `ReentrancyGuard.sol` pattern, or
- The patched `tstorish` with `msg.sender.code.length != 0` as specified in Section 1.

### 6. SIP-5 Metadata Extension

Contracts implementing SIP-5's `getSeaportMetadata()` SHOULD declare EIP-7702 compatibility awareness by including this SIP's number in their supported SIPs array. This signals to marketplaces and tooling that the contract has been audited for 7702 compatibility.

## Rationale

### Why `msg.sender.code.length != 0` over `msg.sender != tx.origin`

The `msg.sender != tx.origin` pattern was designed when EOAs were guaranteed to have no code. EIP-7702 breaks this guarantee by allowing EOAs to carry a 23-byte delegation designator. The code-length check directly tests the property we care about (presence of executable code) rather than a proxy property (call origin).

We chose `msg.sender.code.length != 0` over `extcodesize(msg.sender) > 0` because `address.code.length` is the idiomatic Solidity 0.8+ pattern and avoids inline assembly.

### Why a Utility Library

EIP-7702 detection requires reading the first 23 bytes of an account's code and checking for the `0xef0100` prefix. This logic is non-trivial and error-prone to implement inline. A standard utility library ensures consistent detection across the Seaport ecosystem. The interface follows the pattern established by [OpenZeppelin's `EIP7702Utils.sol`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/account/utils/EIP7702Utils.sol).

### Why Delegated EOAs Should Default to ECDSA

A delegated EOA retains its private key and can produce valid ECDSA signatures. The delegation is a code execution property, not a signing property. Forcing delegated EOAs through the ERC-1271 path would require the delegated contract to implement `isValidSignature`, adding unnecessary complexity. The ECDSA-first approach (which Seaport already uses) remains correct.

### Why Separate `isPureEOA` and `isDelegatedEOA`

Zones and offerers have different security requirements. An anti-MEV zone may want to block all programmatic accounts (including delegated EOAs), while a zone enforcing KYC may only care that the account has a keyholder. Providing both functions lets zone developers make explicit policy decisions.

### Alternative Considered: Checking the Delegation Prefix Only

An alternative is to check only for the `0xef0100` prefix without validating code length. We rejected this because:
1. A hypothetical future contract could coincidentally start with `0xef0100` (unlikely but possible with CREATE2 mining).
2. Checking both prefix and length (== 23) is more precise and costs minimal additional gas.

## Backwards Compatibility

This SIP introduces no breaking changes to the Seaport protocol's on-chain behavior. It specifies updates to:

1. **Standalone tstorish**: Replacing the `msg.sender != tx.origin` guard with `msg.sender.code.length != 0`. This is a strictly tighter check (rejects everything `tx.origin` check rejects, plus delegated EOAs). Existing callers that are pure EOAs are unaffected.

2. **Zone and offerer guidance**: Zones using `extcodesize` checks will need updates to handle delegated EOAs. This is an additive change — zones that do not use account-type classification are unaffected.

3. **Error classification**: Signature verification error types may change for delegated EOA signers (from `BadContractSignature` to `InvalidSigner`). This affects off-chain error handling only, not on-chain settlement.

Contracts deployed before this SIP that use `msg.sender != tx.origin` remain vulnerable to the attack described in the Motivation section. This SIP cannot retroactively fix deployed contracts but provides the standard for new deployments and upgradeable contracts.

## Test Cases

### Test 1: `__activateTstore()` Rejects Delegated EOA

```solidity
function test_activateTstore_rejectsDelegatedEOA() public {
    // Deploy a contract inheriting patched tstorish
    PatchedTstorish target = new PatchedTstorish();

    // Simulate a 7702-delegated EOA (code.length == 23)
    // by deploying a mock with the delegation designator
    address delegatedEOA = _deployDelegationDesignator(address(0xBEEF));

    vm.prank(delegatedEOA, delegatedEOA); // msg.sender == tx.origin
    vm.expectRevert(Tstorish.OnlyDirectCalls.selector);
    target.__activateTstore();
}
```

### Test 2: `__activateTstore()` Accepts Pure EOA

```solidity
function test_activateTstore_acceptsPureEOA() public {
    PatchedTstorish target = new PatchedTstorish();
    address pureEOA = makeAddr("pureEOA");

    vm.prank(pureEOA, pureEOA);
    // Should not revert (assuming TSTORE is supported on the test chain)
    target.__activateTstore();
}
```

### Test 3: Delegation Detection

```solidity
function test_fetchDelegate_returnsDelegateAddress() public {
    address delegate = address(0xBEEF);
    address delegatedEOA = _deployDelegationDesignator(delegate);

    assertEq(SIP_EIP7702Utils.fetchDelegate(delegatedEOA), delegate);
    assertTrue(SIP_EIP7702Utils.isDelegatedEOA(delegatedEOA));
    assertFalse(SIP_EIP7702Utils.isPureEOA(delegatedEOA));
}

function test_fetchDelegate_returnsZeroForPureEOA() public {
    address pureEOA = makeAddr("pureEOA");

    assertEq(SIP_EIP7702Utils.fetchDelegate(pureEOA), address(0));
    assertFalse(SIP_EIP7702Utils.isDelegatedEOA(pureEOA));
    assertTrue(SIP_EIP7702Utils.isPureEOA(pureEOA));
}

function test_fetchDelegate_returnsZeroForContract() public {
    address contractAddr = address(new SomeContract());

    assertEq(SIP_EIP7702Utils.fetchDelegate(contractAddr), address(0));
    assertFalse(SIP_EIP7702Utils.isDelegatedEOA(contractAddr));
    assertFalse(SIP_EIP7702Utils.isPureEOA(contractAddr));
}
```

### Test 4: Zone Account Classification

```solidity
function test_zone_classifiesDelegatedEOACorrectly() public {
    address delegate = address(0xBEEF);
    address delegatedEOA = _deployDelegationDesignator(delegate);

    // Delegated EOA should NOT be classified as a contract
    assertFalse(zone.isContract(delegatedEOA));

    // Regular contract SHOULD be classified as a contract
    assertTrue(zone.isContract(address(new SomeContract())));
}
```

## Reference Implementation

A reference implementation of `SIP_EIP7702Utils` is provided in Section 2 of the Specification.

A reference patch for the standalone `tstorish` library's `__activateTstore()` function:

```diff
  function __activateTstore() external {
-     // Ensure this function is triggered from an externally-owned account.
-     if (msg.sender != tx.origin) {
+     // Ensure the caller is a pure EOA with no code (rejects both
+     // regular contracts and EIP-7702 delegated EOAs).
+     if (msg.sender.code.length != 0) {
          revert OnlyDirectCalls();
      }
```

## Security Considerations

### Reentrancy via Mid-Execution Activation

The primary security concern is the reentrancy guard bypass described in the Motivation section. An attacker with a delegated EOA can activate TSTORE support during execution, causing the guard to read from uninitialized transient storage (value 0) instead of the SSTORE-written guard value. This effectively disables reentrancy protection.

**Mitigation**: The `msg.sender.code.length != 0` check prevents this attack because delegated EOAs have `code.length == 23`.

**Scope**: This vulnerability affects contracts using the standalone `tstorish` library. The `seaport-core` `ReentrancyGuard.sol` is not vulnerable because it uses guard-state checks instead of identity checks. However, any third-party contract that inherits `tstorish` directly (not via `seaport-core`) is at risk.

### Delegation Volatility

EIP-7702 delegations can be added and removed between transactions. An account that is a delegated EOA in one block may be a pure EOA in the next. This has implications for:

- **Order validity**: An order signed by a delegated EOA via ERC-1271 may become unverifiable after the delegation is revoked. Zones SHOULD NOT cache account-type classifications across transactions.
- **Zone authorization**: Zones that authorize based on account type MUST re-check on every call, not rely on prior state.

### Gas Overhead

The `msg.sender.code.length` check costs approximately 2,600 gas (EXTCODESIZE opcode). The full delegation detection (`fetchDelegate`) costs approximately 5,200 gas (EXTCODESIZE + EXTCODECOPY). Zones performing frequent account-type checks SHOULD consider gas implications and may cache results within a single transaction using TSTORE.

### Delegation Designator Spoofing

A malicious actor could deploy a contract with exactly 23 bytes of code starting with `0xef0100` using CREATE2. However, [EIP-3541](https://eips.ethereum.org/EIPS/eip-3541) prohibits contract creation with code starting with `0xef`, making this infeasible. The `0xef` prefix is reserved for EIP-3540 (EOF) and EIP-7702 delegation designators.

### Smart Contract Wallet Interaction

ERC-4337 smart accounts and other smart contract wallets may coexist with EIP-7702 delegated EOAs. Zones MUST NOT assume that an account with code is necessarily a smart contract wallet — it may be a delegated EOA. The `isDelegatedEOA` function SHOULD be used to disambiguate before applying smart-wallet-specific logic.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
