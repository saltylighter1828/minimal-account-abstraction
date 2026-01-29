# Account Abstraction Minimal Accounts (EIP-4337 + zkSync Era)

A small, “learn-by-building” Account Abstraction repo that implements:

- **`MinimalAccount` (EVM / EIP-4337)**: an `IAccount` smart wallet that validates `PackedUserOperation` signatures via an `EntryPoint`.
- **`ZkMinimalAccount` (zkSync Era / Type 113 AA)**: a zkSync-native AA account that validates + increments nonce via system contracts and executes transactions through the bootloader lifecycle.
- **Foundry scripts + tests** to deploy, sign ops, and exercise execution paths end-to-end.

> This is intentionally minimal: great for understanding the moving parts of AA, not intended as a production wallet.

---

## Repo Layout

```text
src/
  ethereum/
    MinimalAccount.sol
  zksync/
    ZkMinimalAccount.sol

script/
  DeployMinimal.s.sol
  HelperConfig.s.sol
  SendPackedUserOp.s.sol

test/
  ethereum/
    MinimalAccountTest.t.sol
  zksync/
    ZkMinimalAccountTest.t.sol
What’s Implemented
1) MinimalAccount (EIP-4337 style)
Core ideas:

Stores an immutable EntryPoint address (i_entryPoint).

Only EntryPoint or Owner can call execute().

Implements validateUserOp() required by IAccount.

Key flows:

execute(dest, value, functionData)
Performs a low-level call. Reverts with MinimalAccount__CallFailed(bytes) if the target call fails.

validateUserOp(userOp, userOpHash, missingAccountFunds)

Requires caller is the EntryPoint

Validates signature with ECDSA.recover over EIP-191 (toEthSignedMessageHash(userOpHash))

Prefunds EntryPoint when missingAccountFunds != 0

Signature logic:

If recovered signer != owner(), returns SIG_VALIDATION_FAILED

Otherwise returns SIG_VALIDATION_SUCCESS

2) ZkMinimalAccount (zkSync Era AA, txType 113 / 0x71)
Implements zkSync’s IAccount interface and follows the bootloader-driven lifecycle.

High level:

validateTransaction(...) (bootloader-only)

Increments nonce via NonceHolder system contract

Checks balance covers totalRequiredBalance()

Validates signature over zkSync’s encoded tx hash

Returns ACCOUNT_VALIDATION_SUCCESS_MAGIC on success

executeTransaction(...) (bootloader or owner)

Executes the tx:

If calling DEPLOYER_SYSTEM_CONTRACT, uses a system call

Else executes via low-level call

executeTransactionFromOutside(...)

Lets an EOA submit a tx directly (validates first, then executes)

payForTransaction(...)

Pays the bootloader via _transaction.payToTheBootloader()

Note: Some zkSync calls require Foundry --system-mode=true and is-system = true config when compiling/running.

Scripts
HelperConfig.s.sol
Network config helper that provides:

EntryPoint address + signer account for:

Ethereum Sepolia (11155111)

Arbitrum Sepolia (421614)

Local Anvil (31337): deploys a fresh EntryPoint mock and uses Anvil default account

DeployMinimal.s.sol
Deploys MinimalAccount using HelperConfig, then transfers ownership to the configured account.

SendPackedUserOp.s.sol
Builds + signs a PackedUserOperation and submits it to the EntryPoint via handleOps.

What it does:

Builds calldata for MinimalAccount.execute(...)

Creates unsigned userOp with:

accountGasLimits packed into bytes32((verificationGasLimit << 128) | callGasLimit)

gasFees packed into bytes32((maxPriorityFeePerGas << 128) | maxFeePerGas)

Hashes via EntryPoint.getUserOpHash(userOp)

Signs EIP-191 digest (toEthSignedMessageHash)

Calls handleOps

Tests
MinimalAccountTest
Covers:

✅ Owner can execute arbitrary calls

✅ Non-owner cannot call execute

✅ Recover signer from a signed userOp

✅ validateUserOp returns success when signed by owner

✅ EntryPoint can execute ops via handleOps (minting an ERC20 mock)

ZkMinimalAccountTest
Covers:

✅ Owner can execute a type-113 transaction via executeTransaction

✅ Bootloader can validate a signed transaction and returns ACCOUNT_VALIDATION_SUCCESS_MAGIC

Quickstart (Foundry)
Install / Build
forge --version
forge install
forge build
Run tests
forge test -vvv
Running the EIP-4337 Flow Locally (Anvil)
Start Anvil:

anvil
Deploy MinimalAccount + local EntryPoint (HelperConfig does this for 31337):

forge script script/DeployMinimal.s.sol:DeployMinimal --rpc-url http://127.0.0.1:8545 --broadcast
Send a signed UserOperation:

forge script script/SendPackedUserOp.s.sol:SendPackedUserOp --rpc-url http://127.0.0.1:8545 --broadcast
zkSync Notes
zkSync AA uses system contracts + bootloader rules, so your tooling setup matters:

You may need:

is-system = true in foundry.toml

--system-mode=true for compilation/runs depending on your setup

The account increments nonce using:

SystemContractsCaller.systemCallWithPropagatedRevert(...)

INonceHolder.incrementMinNonceIfEquals(...)

Security / Limitations
This repo is for learning and demos. Not production ready.

No session keys

No spending limits

No nonce scheme beyond what EntryPoint/zkSync provides

No paymaster support (zkSync: prepareForPaymaster stubbed)

Minimal validation (signature only)

If you fork this into a real wallet, you’ll want:

replay protections + robust nonce handling

upgrade strategy (or explicitly no-upgrades)

better error bubbling + event logging

permissions model (modules / guardians / 2FA / recovery)

audits 🙂

References
EIP-4337 Account Abstraction (EntryPoint + UserOperation)

OpenZeppelin ECDSA + MessageHashUtils

zkSync Era system contracts (NonceHolder, Bootloader, Deployer)

License
MIT# minimal-account-abstraction
