---
layout: specification
title: 2019-NOV-15 Schnorr OP_CHECKMULTISIG specification
date: 2019-05-14
activation: x
version: 0.4 (DRAFT)
author: Mark B. Lundeberg
---

Discussion on Telegram workgroup: https://t.me/joinchat/Jw9WhUbcT1vEc6V2z-uH8Q

# Summary

OP_CHECKMULTISIG and OP_CHECKMULTISIGVERIFY will be upgraded to accept Schnorr signatures in a way that is compatible with batch verification.

*note: this document assumes knowledge of [the prior Schnorr signature upgrade](2019-05-15-schnorr.md).*

# Motivation

In [the last upgrade](2019-05-15-upgrade.md), we added Schnorr support to OP_CHECKSIG and OP_CHECKDATASIG, but not OP_CHECKMULTISIG.

Although we could have added support to OP_CHECKMULTISIG as well (which would have been overall simpler), this would conflict with the desire to do batch verification in future: Currently with OP_CHECKMULTISIG validation, it is needed to check a signature against multiple public keys in order to find a possible match. In Schnorr batch verification however, it is required to know ahead of time, which signatures are supposed to match with which public keys. Without a clear path forward on how to resolve this, we postponed the issue and simply prevented Schnorr signatures from being used in OP_CHECKMULTISIG.

Schnorr aggregated signatures (with OP_CHECKSIG) are one way to do multisignatures, but they are technically limited and thus far from being a drop-in replacement for the familiar Bitcoin multisig. Besides that, it is also desirable that any existing coin can be spent using Schnorr signatures, and there are numerous OP_CHECKMULTISIG wallets and coins in existence that we want to be able to take advantage of Schnorr signatures.

# Specification

OP_CHECKMULTISIG and OP_CHECKMULTISIGVERIFY will be upgraded to allow *two* execution modes, based on the values of consumed stack items.

Mode 1 (legacy ECDSA support, M-of-N; consumes N+M+3 items from stack):

    <dummy> <sig0> ... <sigM> M <pub0> ... <pubN> N OP_CHECKMULTISIG

The precise validation mechanics of this are complex and full of corner cases; the source code is the best reference. Most notably, for 2-of-3 (M=2, N=3), `sig0` may be a valid ECDSA transaction signature from `pub0` or from `pub1`; `sig1` may be from `pub1` (if `sig0` is from `pub0`) or `pub2`. The `dummy` element can assume any value but the NULLDUMMY policy rule (soon to be made consensus) restricts it to be NULL. Historical transactions (prior to FORKID, STRICTENC and NULLFAIL rules) had even more freedoms and [weirdness](https://decred.org/research/todd2014.pdf)).

Mode 2 (new Schnorr support, M-of-N; consumes N+M+3 items from stack):

    <checkbits> <sig0> ... <sigM> M <pub0> ... <pubN> N OP_CHECKMULTISIG

* The `dummy` element has now been repurposed as an integer that we call `checkbits`, whose bit representation indicates which signatures should be checked against which public keys.
* Crucially, each of the signature checks requested by `checkbits` *must* be valid, or else the script fails.
* In mode 2, it is necessary that all public keys are validly encoded.
* In mode 2, ECDSA signatures are not allowed.

## Triggering and execution mechanism

Note that the above description leaves some ambiguity -- how does one know whether to execute in mode 1 or mode 2?
Execution of the new Schnorr mode 2 is triggered by first doing a quick scan of the content of public keys and signatures. The new mode executes when:

- The upgrade is active based on MTP; and
- Every public key is encoded properly under current encoding rules: 33 byte compressed key starting with 0x02 or 0x03, or 65 byte uncompressed key starting with 0x04; and
- Every signature is null (0-length) or correctly encoded Schnorr (65 bytes long, with a valid hashtype byte under current encoding rules).

If any of the above conditions is not satisfied, then execution proceeds in the legacy mode where behaviour is unchanged from before.

The new mode operates by checking public keys against signatures, according to the `checkbits` number. In pseudocode, the full OP_CHECKMULTISIG code is:

    Pop N number from stack ; check bounds 0 <= N <= 20.
    Add N to nOpCount; if nOpCount exceeds 201 limit, fail script.
    Pop N items from stack [pub0...pubN]
    Pop M number from stack ; check bounds 0 <= M <= N.
    Set a cursor on the last signature.
    Set another cursor on the last public key.
    Calculate scriptCode.
    If activated, and all pubkeys are correctly encoded, and all signatures are either null or correctly encoded Schnorr, then:
        # New mode (2)
        Convert the dummy element to integer, called checkbits; if it was not minimally encoded, fail script.
        If checkbits < 0, fail script.
        Loop while the signature and key cursors are not depleted:
            If the least significant bit of checkbits is 1, then:
                Validate the current signature against the current public key; if invalid, fail script.
                Move the signature cursor back one position.
            Shift checkbits right by one bit. (checkbits := checkbits >> 1)
            Move the public key cursor back one position.
        If remaining checkbits is nonzero, fail script.
        If the signature cursor has remaining signatures, then success=False; if depleted, then success=True.
    Else:
        # Legacy mode (1)
        If pre-BCH-fork, then run findAndDelete on scriptCode.
        Loop while the signature cursor is not depleted:
            Check public key encoding.
            Check signature encoding; exclude 65-byte signatures (Schnorr).
            Validate the current signature against the current public key.
            If valid, then move signature cursor back one position.
            Move the public key cursor back one position.
            If more signatures remain than public keys, set success = False and abort loop early.
        If loop was not aborted, success = True.
        If NULLDUMMY rule is active, then ensure the dummy element is null.

    If success is False and NULLFAIL rule is active, then ensure all signatures were null.

    Push success onto stack
    If opcode is OP_CHECKMULTISIGVERIFY:
        Pop success from stack
        If not success:
            FAIL

Some features to note:

- In the legacy mode, some of the public keys may be garbage data, even if strict encoding rules are being applied. This is possible since the loop ends before all public keys have been examined.
- In the new mode, it is not necessary to run findAndDelete ever, since it applies only to transactions well after the BCH fork.
- In the new mode, `checkbits` must be minimally encoded, regardless of MINIMALDATA flag (as the latter not a consensus rule).
- The new mode design means that `checkbits` has only one valid representation for a given set of signatures, provided that all public keys are distinct. Thus ordinarily, it is not malleable. Note however that if some public keys are repeated, however, then it may be possible to reorder the signatures and/or flip bits to apply the signature checks to different public keys.
- For a failing CHECKMULTISIG (that returns False on stack), the previously adopted NULLFAIL consensus rule means that all signatures must be null. However, only the last public key needs to be correctly encoded. If all pubkeys are correctly encoded, then the new mode executes and the checkbits element must be zero (null). If any keys are not correctly encoded, then legacy mode executes and the dummy element need only be null if the NULLDUMMY flag is adopted.
- In the legacy mode, it can require up to N signature checks in order to complete. In new mode, at most M signature checks occur.
- For M=0, the opcode always evaluates True.

# Wallet implementation guidelines

Currently, the common multisig wallet uses P2SH-multisig, i.e., a redeemScript of the form `M <pub0> ... <pubN> N OP_CHECKMULTISIG`. We'll focus on this use case.

In the new Schnorr mode, *all* signatures must be Schnorr; no mixing with ECDSA is supported. Multisig wallets that wish to use the new Schnorr signatures will need to update their co-signing pools infrastructure to support a new type of signing. If some parties are unable to generate a Schnorr signature, then it will not be possible to generate a successful transaction except by restarting to make an ECDSA multisig.

This creates some problems in particular when some of the parties are hardware wallet, which may only support ECDSA for the forseeable future. Thus, we suggest the following for wallet software producers:

* Add an optional marker to the initial setup process, such as appending `?schnorr=true` to the `xpub`.
* Add a new kind of non-backwards-compatible multisignature request that indicates schnorr signatures are needed.
* If it is not known that all parties can accept Schnorr requests, then only generate ECDSA multisignature requests.
* Have the ability to participate either ECDSA Schnorr multisignatures, if requested.

## Calculating and pushing checkbits

Note that the new mechanism requires keeping track of which signatures go with which public keys, which ordinarily wallets may not be careful of (but should really be doing, in order to validate and correctly order the signatures from co-signers). Wallets must also correctly include the `checkbits` parameter, which means a) minimally encoding `checkbits` into bytes, and b) minimally pushing this bytestring in the scriptSig. This process can be abbreviated as:

* If `checkbits` <= 16, then use OP_1 through OP_16 (opcode `0x50 + checkbits`), otherwise:
* Encode checkbits as a bytestring in little-endian order; trim off any trailing zero bytes.
* If the last byte has its high bit set, then append a single zero byte.
* Push the bytestring using a simple push.

## ScriptSig size

Wallets need to know ahead of time the maximum transaction size, in order to set the transaction fee.

Let `R` be the length of the redeemScript and its push opcode, combined.

The legacy mode scriptSig `<dummy> <sig0> ... <sigM>` can be as large as 73M + 1 + R bytes, which is the upper limit assuming all max-sized ECDSA signatures.

The new mode scriptSig `<sig0> ... <sigN>` will have a length that varies depending on minimal encoding of `checkbits`. Wallets should allocate for fees based on the largest possible encoding, which gives a scriptSig size of:
* Always 66M + R + 1 bytes, for N <= 4.
* Up to 66M + R + 2 bytes, for 5 <= N <= 7.
* Up to 66M + R + 3 bytes, for 8 <= N <= 15.
* Up to 66M + R + 4 bytes, for 16 <= N <= 20.

# Rationale and commentary on design decisions

## Repurposing of dummy element

In an earlier edition it was proposed to require N signature items (either a signature or NULL) for new mode instead of M items and a dummy element. The following problems inspired a move away from that approach:

* Triggering mechanics for the new mode were somewhat of a kluge.
* Some scripts rely on a certain expected stack layout. This is particularly the case for recently introduced high-level smart contracting languages that compile down to script, which reach deep into the stack.

That said, a scan of the blockchain only found about a hundred instances of scripts that would be impacted by stack layout changes. All were based on a template as seen in [this spend](https://blockchair.com/bitcoin-cash/transaction/612bd9fc5cb40501f8704028da76c4c64c02eb0ac80e756870dba5cf32650753), where OP_DEPTH was used to choose an OP_IF execution branch.

## No mixing ECSDA / Schnorr

Allowing mixed signature types might help alleviate the issue of supporting mixed wallet versions that do support / don't support Schnorr signatures.
However, this would mean that an all-ECDSA signature list could be easily converted to the new mode, unless extra complicated steps were taken to prevent that conversion. As this is an undesirable malleability mechanism, we opted to simply exclude ECDSA from the new mode, just as Schnorr are excluded from the legacy mode.

## Triggering on pubkey and signature encoding validity

An alternative, slightly simpler trigger mechanism would be to trigger on the dummy element being non-null, which cou.d make things overall simpler as it automatically integrates the 'nulldummy' rule.

The choice to trigger on public key and signature validity is done so that the new mode is able to support both true- and false- returning executions of checkmultisig; for the false-returning execution, it is required for the dummy element to be null.

In addition, it is valuable if the new mode has been implemented with strict public key encodings for all public keys, as it is expected that this strict context would help simplify future introductions of alternative signature schemes (with different pubkey encodings).

# Acknowledgements

Thanks to Tendo Pein, Rosco Kalis, and Amaury Sechet for valuable feedback.
