---
layout: specification
title: 2019-NOV-15 Schnorr OP_CHECKMULTISIG specification
date: 2019-05-14
activation: x
version: 0.3 (DRAFT)
author: Mark B. Lundeberg
---

Discussion on Telegram workgroup: https://t.me/joinchat/Jw9WhUbcT1vEc6V2z-uH8Q

# Summary

OP_CHECKMULTISIG and OP_CHECKMULTISIGVERIFY will be upgraded to accept Schnorr signatures in a way that is compatible with batch verification.

*note: this document assumes knowledge of [the prior Schnorr signature upgrade](2019-05-15-schnorr.md).*

# Motivation

In [the last upgrade](2019-05-15-upgrade.md), we added Schnorr support to OP_CHECKSIG and OP_CHECKDATASIG, but not OP_CHECKMULTISIG.

Although we could have added support to OP_CHECKMULTISIG as well (which would have been overall simpler), this would conflict with the desire to do batch verification in future: Currently with OP_CHECKMULTISIG validation, it is needed to check a signature against multiple public keys in order to find a possible match. In Schnorr batch verification however, it is required to know ahead of time, which signatures are supposed to match with which public keys. Without a clear path forward on how to resolve this, we postponed the issue and simply prevented Schnorr signatures from being used in OP_CHECKMULTISIG.

Schnorr aggregated signatures (with OP_CHECKSIG) are one way to do multisignatures, but they are technically limited and thus far from being a drop-in replacement for the familiar Bitcoin multisig. Besides that, it is also desirable that any existing coin can be spent using Schnorr signatures, and there are numerous OP_CHECKMULTISIG wallets and coins in existence that deserve ongoing support.

# Specification

OP_CHECKMULTISIG and OP_CHECKMULTISIGVERIFY will be upgraded to allow *two* execution modes, based on the values of consumed stack items.

Mode 1 (legacy ECDSA support, M-of-N; consumes N+M+3 items from stack):

    <dummy> <sig0> ... <sigM> M <pub0> ... <pubN> N OP_CHECKMULTISIG

The precise validation mechanics of this are complex and full of corner cases; the source code is the best reference. Most notably, for 2-of-3 (M=2, N=3), `sig0` may be a valid ECDSA transaction signature from `pub0` or from `pub1`; `sig1` may be from `pub1` (if `sig0` is from `pub0`) or `pub2`. The `dummy` element can assume any value but the NULLDUMMY policy rule (soon to be made consensus) restricts it to be NULL. Historical transactions (prior to FORKID, STRICTENC and NULLFAIL rules) had even more freedoms and [weirdness](https://decred.org/research/todd2014.pdf)).

Mode 2 (new Schnorr support, M-of-N: consumes 2N+2 items from stack):

    <sig0> ... <sigN> M <pub0> ... <pubN> N OP_CHECKMULTISIG

* For each `sigI` for all values of `I`, its value must be either NULL (bytelength 0) or a valid Schnorr transaction signature (bytelength 65) from the corresponding key `pubI`.
* If any non-null *invalid* signatures are present, the script fails (hardcoded NULLFAIL rule).
* If the number of valid signatures is not 0 or M, fail script.
* If the number of valid signatures is equal to M, return True.
* If the number of valid signatures is equal to 0, return False. (must be checked after previous)

Whereas execution of the original OP_CHECKMULTISIG counts as N+1 opcodes (towards the 201 opcode limit), the new mode will count as M+1 opcodes (towards the 201 opcode limit). Note that the current sigops counting will be unaffected in this mechanism -- i.e., it will count as 20 sigops [except in some P2SH scripts where it will count as N sigops](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki). However, I recommend that in a new SigOps2 accounting, the number of actual signature checks be used for both mechanism, with NULL signatures contributing 0 sigops (can vary for old mode, but is either 0 or M for new mode).

## Triggering and execution mechanism

Note that the above description leaves some ambiguity -- how does one know whether to execute in mode 1 or mode 2?
Execution of the new Schnorr mode 2 is triggered based on the last signature item (the item on stack just before M). If it has size 0 or size 65, then mode 2 is activated. However in the special case of M=0, this stack item is *not* examined and mode 2 executes by default.

In pseudocode:

    Pop N number from stack ; check bounds 0 <= N <= 20
    Pop N items from stack [pub0...pubN]
    Pop M number from stack ; check bounds 0 <= M <= N
    If N > 0:
        Let L = length of top stack item
    Else:
        Let L = 0
    If activated and L == 0 or L == 65:
        # Mode 2
        Add M to nOpCount; if nOpCount exceeds 201 limit, fail script.
        Pop N signatures from stack [sig0...sigN]
        Let validsigs=0
        For I in 0...N-1:
            If len(sigI) == 0:
                continue
            If len(sigI) != 65:
                FAIL
            If sigI or pubI have bad encoding per STRICTENC:
                FAIL
            If sigI is not valid tx Schnorr signature from pubI:
                FAIL
            validsigs += 1
        If validsigs == M
            Let success = True
        Elif validsigs == 0:
            Let success = False
        Else:
            FAIL
    Else:
        # Mode 1:
        Add N to nOpCount; if nOpCount exceeds 201 limit, fail script.
        ...
        ... (see existing code.)
        ...
    Push success onto stack
    If opcode is OP_CHECKMULTISIGVERIFY:
        Pop success from stack
        If not success:
            FAIL

(Non-consensus rules like MINIMALDATA have been omitted here, for simplicity).

# Wallet implementation guidelines

Currently, the common multisig wallet uses P2SH-multisig, i.e., a redeemScript of the form `M <pub0> ... <pubN> N OP_CHECKMULTISIG`. We'll focus on this use case.

In the new Schnorr mode, *all* signatures must be Schnorr; no mixing with ECDSA is supported. Multisig wallets that wish to use the new Schnorr signatures will need to update their co-signing pools infrastructure to support a new type of signing. If some parties are unable to generate a Schnorr signature, then it will not be possible to generate a successful transaction except by restarting to make an ECDSA multisig.

This creates some problems in particular when some of the parties are hardware wallet, which may only support ECDSA for the forseeable future. Thus, we suggest the following for wallet software producers:

* Add an optional marker to the initial setup process, such as appending `?schnorr=true` to the `xpub`.
* Add a new kind of non-backwards-compatible multisignature request that indicates schnorr signatures are needed.
* If it is not known that all parties can accept Schnorr requests, then only generate ECDSA multisignature requests.
* Have the ability to participate either ECDSA Schnorr multisignatures, if requested.

## ScriptSig size

Wallets need to know ahead of time the maximum transaction size, in order to set the transaction fee.

Let `R` be the length of the redeemScript and its push opcode, combined.

The legacy mode scriptSig `<dummy> <sig0> ... <sigM>` can be as large as 73M + 1 + R bytes, which is the upper limit assuming all max-sized ECDSA signatures.

The new mode scriptSig `<sig0> ... <sigN>` will be 65M + N + R bytes. This usually shorter than legacy mode, but not always (consider a 1-of-9, 1-of-10, etc., or the degenerate case of 0-of-N).

# Subtleties for unusual scripts

## Interaction with stack manipulation

Since the number of pushed items is no longer a predictable constant, smart contract designers need to be careful about combining heavy stack manipulation with OP_CHECKMULTISIG. In particular they should avoid relying on operations (OP_PICK, OP_ROLL) that reach deep into the stack and access items before the OP_CHECKMULTISIG signature list. The ability of spends to shift the stack by choosing between ECDSA / Schnorr mode may create very sublte vulnerabilities in such scripts.

As far as we are aware, such a strange combination of OP_CHECKMULTISIG and heavy stack manipulation does not occur in any current smart contract systems of note.

## NULL-ed multisignatures

In some smart contracts it is useful to rely on a False-returning OP_CHECKMULTISIG operation, which occurs when all signatures are NULL. Prior to this upgrade, this would happen by providing M+1 null items in scriptsig (dummy element then M null signatures). The new mechanics however overrides this, so that NULL-ed signature lists *must* be provided in the new mode, i.e., N null items in the scriptsig.

## Garbage public keys

In legacy verification, it is permitted to have some of public keys be garbage data, provided that the signature verification finishes before those public keys are reached. In the new mode the same idea applies: the contents of public key stack items are not examined at all if the corresponding signature is NULL.

## 0-of-0 multisignatures

A 0-of-0 multisignature is currently valid, and remains valid under current mechanics. However, the spending mechanics have been altered. Previously it would be necessary to push a dummy element, while under the new mechanics, no item is pushed before M.

Currently, 0-of-0 multisignatures result in success (just like 0-of-N signatures); this is maintained in the new mode.

# Rationale and commentary on design decisions

## No sentinel value

In an earlier edition it was proposed to use an additional sentinel item on stack just before M, to trigger between the two modes. It was thought that a sentinel-less mechanism would be too ambiguous. However, this turns out not to be true.

## No mixing ECSDA / Schnorr

Allowing mixed signature types would help alleviate the issue of supporting mixed wallet versions that do support / don't support Schnorr signatures.

However, it is not possible to so simply discriminate between the mode 1 / mode 2 divide if mixing of types is allowed. Not only that, but it would introduce a new third-party malleability mechanism: any legacy mode (all-ECDSA) scriptSig could be trivially converted to a new mode scriptSig.

## Preference to new mechanics

Although preserving legacy OP_CHECKMULTISIG mechanics for the standard case (a multisignature wallet), we have assigned unusual cases like NULL-ed signatures and 0-of-N multisignatures to only use the new preferred mechanism. The rationale is that the new mechanics are "cleaner", and they set the stage for clean integration of possible future signature algorithms. In future, the legacy mechanism will be ECDSA-only and the new mechanism will be for all other signature algorithms.

# Acknowledgements

None so far.
