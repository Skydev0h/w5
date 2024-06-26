# Extensible Wallet V5

Author: Oleg Andreev <oleg@tonkeeper.com>

This is an extensible wallet specification aimed at replacing V4 and allowing arbitrary extensions.

* [Features](#features)
* [Overview](#overview)
* [Discussion](#discussion)
* [Wallet ID](#wallet-id)
* [Packed address](#packed-address)
* [TL-B definitions](#tl-b-definitions)
* [Source code](#source-code)


## Credits

Thanks to [Andrew Gutarev](https://github.com/pyAndr3w) for the idea to set c5 register to a [list of pre-composed actions](https://github.com/pyAndr3w/ton-preprocessed-wallet-v2).

Thanks to [@subden](https://t.me/subden), [@botpult](https://t.me/botpult) and [@tvorogme](https://t.me/tvorogme) for ideas and discussion.

Thanks to [Skydev](https://github.com/Skydev0h) for optimization and preparing the second revision of the contract.


## Features

* 25% smaller computation fees.
* Arbitrary amount of outgoing messages is supported via action list.
* Wallet code can be extended by anyone in a decentralized and conflict-free way: multiple feature extensions can co-exist.
* Extensions can perform the same operations as the signer: emit arbitrary messages on behalf of the owner, add and remove extensions.
* Signed requests can be delivered via internal message to allow 3rd party pay for gas.
* For consistency and ease of indexing, external messages also receive a 32-bit opcode.
* To lay foundation for support of scenarios like 2FA or access recovery it is possible to disable signature authentication.

## Overview

Wallet V5 supports **2 authentication modes** and **3 operations types**.

Authentication:
* by signature
* by extension

Operations:
* standard "send message" action (up to 255 messages at once),
* enable/disable signature authentication,
* install/remove extension.

Signed messages can be delivered both by external and internal messages.

All operations are available to all authentication modes.

## Discussion

### What is the job of the wallet?

The job of the wallet is to send messages to other apps in the TON network on behalf of a single user identified by a single public key.
User may delegate this job to other apps via extensions.

### The wallet is not for:

* multi-user operation: you should use a multisig or DAO solution instead.
* routing of incoming payments and messages: use a specialized contract instead.
* imposing limits on access to certain assets: put account restriction inside a jetton, or use a lockup contract instead.

### Extending the wallet

The best way to extend functionality of the wallet is to use the extensions mechanism that permit delegating access to the wallet to other contracts.

From the perspective of the wallet, every extension can perform the same actions as the owner of a private key. Therefore limits and capabilities can be embedded in such an extension with a custom storage scheme.

Extensions can co-exist simultaneously, so experimental capabilities can be deployed and tested independently from each other.

### Can the wallet outsource payment for gas fees?

Yes! You can deliver signed messages via an internal message from a 3rd party wallet. Also, the message is handled exactly like an external one: after the basic checks the wallet takes care of the fees itself, so that 3rd party does not need to overpay for users who actually do have TONs.

### Can plugins implement subscriptions that collect tokens?

Yes. Plugins can emit arbitrary messages, including token transfers, on behalf of the wallet.

### How can a plugin collect funds?

Plugin needs to send a request with a message to its own address.

### How can a plugin self-destruct?

Plugin can self-destroy by sending all TONs to the wallet with sendmode 128 and adding one more action that removes itself from the list.

### How can I deploy a plugin, install its code and send it a message in one go?

You need to put two requests in your message body:
1. add the extension address,
2. send a message with stateinit to that address.

### Does the wallet grow with number of plugins?

Yes. We have considered constant-size schemes where the wallet only stores trusted extension code. However, extension authentication becomes combursome and expensive: plugin needs to transmit additional data and each request needs to recompute plugin’s address. We estimate that for the reasonably sized wallets (less than 100 plugins) authentication via the dictionary lookup would not exceed costs of indirect address authentication.

### Why it can be useful to disallow authentication with signature?

Ability to disallow authentication with signature enables two related use-cases:

1. Two-factor authentication schemes: where control over wallet is fully delegated to an extension that checks two signatures: the user’s one and the signature from the auth service. Naturally, if the signature authentication in the wallet remains allowed, the second factor check is bypassed.

2. Account recovery: delegating full control to another wallet in case of key compromise or loss. Wallet may contain larger amount of assets and its address could be tied to long-term contracts, therefore delegation to another controlling account is preferred to simply transferring the assets.

### What is library on masterchain?

Library is a special code storage mechanism that allows to reduce storage cost for a new Wallet V5 contract instance. Wallet V5 contract code is stored into a masterchain library. 
When wallet contract is being deployed, original code hash is being used as the contract code.
Library contract itself data and code are empty cells. That leads to the inability to change the library code, delete the contract, or withdraw funds from it.
Therefore, any Wallet V5 user can top up the library contract balance if they are afraid that the library code of their wallet will be frozen.

## Wallet ID

Wallet ID disambiguates requests signed with the same public key to different wallet versions (V3/V4/V5) or wallets deployed on different chains.

For Wallet V5 we suggest using the following wallet ID:

```
wallet_id$_ global_id:int32 wc:int8 version:(## 8) subwallet_number:(## 32) = WalletID;
```

- `global_id` is a TON chain identifier. TON Mainnet `global_id = -239` and TON Testnet `global_id = -3`.
- `wc` is a Workchain. -1 for Masterchain and 0 for Basechain. 
- `version`: current version of wallet v5 is `0`.
- `subwallet_number` can be used to get multiple wallet contracts bound to the single keypair.

## Packed address

To make authorize extensions efficiently we compress 260-bit address (workchain + sha256 of stateinit) into a 256-bit integer:

```
int addr = addr_hash ^ (wc + 1)
```

Previously deployed wallet v4 was packing the address into a cell which costs ≈500 gas, while access to dictionary costs approximately `120*lg2(N)` in gas, that is serialization occupies more than half of the access cost for wallets with up to 16 extensions. This design makes packing cost around 50 gas and allows cutting the authentication cost 2-3x for reasonably sized wallets.

As of 2023 TON network consists of two workchains: -1 (master) and 0 (base). This means that the proposed address packing reduces second-preimage resistance of sha256 by 1 bit which we consider negligible. Even if the network is expanded with 254 more workchains in a distant future, our scheme would reduce security of extension authentication by only 8 bits down to 248 bits. Note that birthday attack is irrelevant in our setting as the user agent is not installing random extensions, although the security margin is plenty anyway (124 bits).



## TL-B definitions

Action types:

```tl-b
// Standard actions from block.tlb:
out_list_empty$_ = OutList 0;
out_list$_ {n:#} prev:^(OutList n) action:OutAction = OutList (n + 1);
action_send_msg#0ec3c86d mode:(## 8) out_msg:^(MessageRelaxed Any) = OutAction;

// Extended actions in W5:
action_list_basic$0 {n:#} actions:^(OutList n) = ActionList n 0;
action_list_extended$1 {m:#} {n:#} action:ExtendedAction prev:^(ActionList n m) = ActionList n (m+1);

action_add_ext#1c40db9f addr:MsgAddressInt = ExtendedAction;
action_delete_ext#5eaef4a4 addr:MsgAddressInt = ExtendedAction;
action_set_signature_auth_allowed#20cbb95a allowed:(## 1) = ExtendedAction;
```

Authentication modes:

```tl-b
signed_request$_             // 32 (opcode from outer)
  wallet_id:    WalletID     // 80
  valid_until:  #            // 32
  msg_seqno:    #            // 32
  inner:        InnerRequest // 1 .. (1 + 32 + 256) + ^Cell
  signature:    bits512      // 512
= SignedRequest;             // Total: 688 .. 976 + ^Cell

internal_signed#73696e74 signed:SignedRequest = InternalMsgBody;
internal_extension#6578746e inner:InnerRequest = InternalMsgBody;
external_signed#7369676e signed:SignedRequest = ExternalMsgBody;

actions$_ {m:#} {n:#} actions:(ActionList n m) = InnerRequest;
```

Contract state:
```tl-b
wallet_id$_ global_id:# wc:int8 version:(## 8) subwallet_number:# = WalletID;
contract_state$_ signature_auth_disabled:(## 1) seqno:# wallet_id:WalletID public_key:(## 256) extensions_dict:(HashmapE 256 int8) = ContractState;
```

## Source code

See [contracts/wallet_v5.fc](contracts/wallet_v5.fc).
