---
layout: specification
title: 2019-NOV-15 Schnorr OP_CHECKMULTISIG specification
date: 2019-03-13
activation: x
version: 0.1 (DRAFT)
author: Mark B. Lundeberg
---

# Summary

OP_CHECKMULTISIG and OP_CHECKMULTISIGVERIFY will be upgraded to accept Schnorr signatures in a way that is compatible with batch verification.

# Motivation

In the last upgrade, we added Schnorr support to OP_CHECKSIG and OP_CHECKDATASIG, but not OP_CHECKMULTISIG.

Although we could have added support to OP_CHECKMULTISIG as well (which would have been overall simpler), this would conflict with the desire to do batch verification in future: Currently with OP_CHECKMULTISIG validation, you need to sometimes check a signature against multiple public keys in order to find a possible match. In Schnorr batch verification however, it is required to know ahead of time, which signatures are supposed to match with which public keys. Without a clear path forward on how to resolve this, we postponed the issue and simply prevented Schnorr signatures from being used in OP_CHECKMULTISIG.

Schnorr aggregated signatures (with OP_CHECKSIG) are one way to do multisignatures, but they are technically limited and thus far from being a drop-in replacement for the familiar Bitcoin multisig. Besides that, it is also desirable that any existing coin can be spent using Schnorr signatures, and there are numerous OP_CHECKMULTISIG wallets and coins in existence that deserve ongoing support.

# Specification

OP_CHECKMULTISIG and OP_CHECKMULTISIGVERIFY will be upgraded to allow *two* execution modes, based on the values of consumed stack items.

Mode 1 (legacy ECDSA support, M-of-N; consumes N+M+3 items from stack):

    <dummy> <sig0> ... <sigM> M <pub0> ... <pubN> N OP_CHECKMULTISIG

The precise validation mechanics of this are complex and full of corner cases; the source code is the best reference. Most notably, for 2-of-3 (M=2, N=3), `sig0` may be a valid ECDSA transaction signature from `pub0` or from `pub1`; `sig1` may be from `pub1` (if `sig0` is from `pub0`) or `pub2`. Also, `sig0` and `sig1` may be NULL in which case OP_CHECKMULTISIG returns FALSE. The `dummy` element can assume any value but the NULLDUMMY policy rule (soon to be made consensus) restricts it to be NULL. Historical transactions (prior to FORKID, STRICTENC and NULLFAIL rules) had even more freedoms and [weirdness](https://decred.org/research/todd2014.pdf)).

Mode 2 (new Schnorr support, M-of-N: consumes 2N+3 items from stack):

    <sig0> ... <sigN> <sentinel> M <pub0> ... <pubN> N OP_CHECKMULTISIG

* For each `sigI` for all values of `I`, its value must be either NULL (bytelength 0) or a valid Schnorr transaction signature (bytelength 65) from the corresponding key `pubI`.
* If any non-null *invalid* signatures are present, the script fails (hardcoded NULLFAIL rule)
* If the number of valid signatures is equal to:
  * M, return True;
  * 0, return False (or fail script if OP_CHECKMULTISIGVERIFY)
  * other, fail script.

Whereas execution of the original OP_CHECKMULTISIG counts as N+1 opcodes (towards the 201 opcode limit), the new mode will count as M+1 opcodes (towards the 201 opcode limit). TODO: mention sigops is unchanged.

## Sentinel value

The sentinel value is to be determined by an examination of the historical blockchain on both mainnet and testnet. Under current consensus rules (with NULLFAIL), it is no longer possible to put arbitrary values for `sigM`, however historically it was possible to put anything there, provided the script would not fail after OP_CHECKMULTISIG returned false. An examination of the past blockchain will be able to recover a sentinel value such that simply introducing the new-mode code will not affect validity of old transactions. The sentinel value could be the 1-byte array [0x01], which can be pushed onto the stack with the 1-byte opcode 0x51 (OP_1).

## Execution mechanism

In pseudocode:

    Pop N number from stack ; check bounds 0 <= N <= 20
    Pop N pubkeys from stack [pub0...pubN]
    Pop M number from stack ; check bounds 0 <= M <= N
    If top stack item is not equal to sentinel value, then:
        # Mode 1
        Add N to nOpCount
        ... (see existing code) ...
    Else:
        # Mode 2
        Add M to nOpCount
        Pop N signatures from stack [sig0...sigN]
        Let validsigs=0
        For I in 0...N-1:
            If len(sigI) == 0:
                continue
            If sigI is not valid tx Schnorr signature from pubI:
                FAIL
            validsigs += 1
        If validsigs == 0:
            Let success = False
        Elif validsigs == M:
            Let success = True
        Else:
            FAIL
    Push success onto stack
    If opcode is OP_CHECKMULTISIGVERIFY:
        Pop success from stack
        If not success:
            FAIL

(Non-consensus rules like MINIMALDATA have been omitted here, for simplicity).

# Wallet implementation guidelines

Currently, the common multisig wallet uses P2SH-multisig, i.e., a redeemScript of the form `M <pub0> ... <pubN> N OP_CHECKMULTISIG`. We'll focus on this use case.

In the new Schnorr mode, *all* signatures must be Schnorr; no mixing with ECDSA is supported. Multisig wallets that wish to use the new Schnorr signatures will need to update their co-signing pools infrastructure to support a new type of signing. If some parties are unable to generate a Schnorr signature, then it will not be possible to generate a successful transaction.

This creates some problems in particular when some of the parties are hardware wallet, which may only support ECDSA for the forseeable future. Thus, we recommend the following for wallet software producers:

* Add the ability to participate in Schnorr multisignatures, if requested.
* By default however, only generate ECDSA multisignature requests.
* Years in the future when all infrastructure supports

## ScriptSig size

Wallets need to know ahead of time the maximum transaction size, in order to set the transaction fee.

Let `R` be the length of the redeemScript and its push opcode, combined.

The legacy mode scriptSig `<dummy> <sig0> ... <sigM>` can be as large as 73M + 1 + R bytes, which is the upper limit assuming all max-sized ECDSA signatures.

The new mode scriptSig `<sig0> ... <sigN> <sentinel>` will be 66M + 1 + N - M + R bytes. This usually shorter than legacy mode, but not always (consider a 1-of-9, 1-of-10, etc.).

# Rationale and commentary on design decisions

## Sentinel value

Without some kind of sentinel value, it turns out to be impossible to make an unambiguous triggering mechanism that distinguishes the new and old modes. Consider the unusual case of a 2-of-6 CHECKMULTISIG where a FALSE result is desired, i.e., only NULL signatures are passed in. Legacy would have `<dummy> <sig0> ... <sigM> == NULL NULL NULL`; how to distinguish this from the new mode where *six* NULL values would need to be popped from stack?

## No mixing ECSDA / Schnorr

The Mode 2 mechanism could be easily designed to support both ECDSA and Schnorr signatures. This would solve the issue of supporting mixed wallets that do support / don't support Schnorr signatures. However, it would introduce a new malleability mechanism: any legacy mode (all-ECDSA) scriptSig could be trivially converted to a new mode scriptSig. To fix this we would have to introduce another rule that in the new mode, it's not allowed to have *only* ECDSA signatures (i.e., if not all NULL, then at least one signature must be 65 bytes in length).

# Acknowledgements

None so far.
