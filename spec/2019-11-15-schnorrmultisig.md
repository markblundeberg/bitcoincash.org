---
layout: specification
title: 2019-NOV-15 Schnorr OP_CHECKMULTISIG specification
date: 2019-06-22
activation: x
version: 0.6 (DRAFT)
author: Mark B. Lundeberg
---

Discussion on Telegram workgroup: https://t.me/joinchat/Jw9WhUbcT1vEc6V2z-uH8Q

# Summary

OP_CHECKMULTISIG and OP_CHECKMULTISIGVERIFY will be upgraded to accept Schnorr signatures in a way that increases verification efficiency and is compatible with batch verification.

*note: this document assumes knowledge of [the prior Schnorr signature upgrade](2019-05-15-schnorr.md).*

# Motivation

In [the last upgrade](2019-05-15-upgrade.md), we added Schnorr support to OP_CHECKSIG and OP_CHECKDATASIG, but not OP_CHECKMULTISIG.

Although we could have added support to OP_CHECKMULTISIG as well (which would have been overall simpler), this would conflict with the desire to do batch verification in future: Currently with OP_CHECKMULTISIG validation, it is needed to check a signature against multiple public keys in order to find a possible match. In Schnorr batch verification however, it is required to know ahead of time, which signatures are supposed to match with which public keys. Without a clear path forward on how to resolve this, we postponed the issue and simply prevented Schnorr signatures from being used in OP_CHECKMULTISIG.

Schnorr aggregated signatures (with OP_CHECKSIG) are one way to do multisignatures, but they are technically limited and thus far from being a drop-in replacement for the familiar Bitcoin multisig. Besides that, it is also desirable that any existing coin can be spent using Schnorr signatures, and there are numerous OP_CHECKMULTISIG-based wallets and coins in existence that we want to be able to take advantage of Schnorr signatures.

# Specification

OP_CHECKMULTISIG and OP_CHECKMULTISIGVERIFY will be upgraded to allow *two* execution modes, based on the value of the dummy element.

Mode 1 (legacy ECDSA support, M-of-N; consumes N+M+3 items from stack):

    <dummy> <sig0> ... <sigM> M <pub0> ... <pubN> N OP_CHECKMULTISIG

The precise validation mechanics of this are complex and full of corner cases; the source code is the best reference. Most notably, for 2-of-3 (M=2, N=3), `sig0` may be a valid ECDSA transaction signature from `pub0` or from `pub1`; `sig1` may be from `pub1` (if `sig0` is from `pub0`) or `pub2`. Historical transactions (prior to FORKID, STRICTENC and NULLFAIL rules) had even more freedoms and [weirdness](https://decred.org/research/todd2014.pdf)). Upon activation, the `dummy` element must be null, i.e., an empty byte array.

Mode 2 (new Schnorr support, M-of-N; consumes N+M+3 items from stack):

    <checkbits> <sig0> ... <sigM> M <pub0> ... <pubN> N OP_CHECKMULTISIG

* The `dummy` element has now been repurposed as an integer that we call `checkbits`, whose bit representation indicates which signatures should be checked against which public keys.
* This mode activates when `dummy` (`checkbits`) is non-null, i.e., not an empty byte array.
* Crucially, each of the signature checks requested by `checkbits` *must* be valid, or else the script fails.
* In mode 2, ECDSA signatures are not allowed.

## Triggering and execution mechanism

Whether to execute in mode 1 or mode 2 is detemined by the value of the dummy / checkbits element.
* If the checkbits element is NULL, then Mode 1 is executed
* If the checkbits element is non-NULL, then Mode 2 is executed.

The new mode operates similar to legacy mode but only checks signatures as requested, according to the `checkbits` number. If the least significant bit of `checkbits` is set, then the last signature should be checked against the last public key, and so on. For a successful verification in the new mode, `checkbits` must have `M` bits set, and the signatures must be correctly ordered.

In pseudocode, the full OP_CHECKMULTISIG code is:

    Get N (number of pubkeys) from stack ; check bounds 0 <= N <= 20.
    Add N to nOpCount; if nOpCount exceeds 201 limit, fail script.
    Get M (number of signatures) from stack ; check bounds 0 <= M <= N.
    Set a cursor on the last signature.
    Set another cursor on the last public key.
    Calculate scriptCode.
    If activated, and the dummy element is not null, then:
        # New mode (2)
        Convert the dummy element to a number, called checkbits.
        If checkbits < 0, fail script.
        Loop while the signature and key cursors are not depleted:
            If the least significant bit of checkbits is 1, then:
                Check public key encoding.
                Check signature encoding; exclude non-Schnorr signatures.
                Validate the current signature against the current public key; if invalid, fail script.
                Move the signature cursor back one position.
            Bitshift checkbits down by one bit. (checkbits := checkbits >> 1)
            Move the public key cursor back one position.
        If the final checkbits value is nonzero, fail script.
        If the signature cursor has at least one remaining signature, then success=False; else success=True.
    Else:
        # Legacy mode (1)
        If pre-BCH-fork, then run findAndDelete on scriptCode.
        Loop while the signature cursor is not depleted:
            Check public key encoding.
            Check signature encoding; exclude Schnorr signatures (64+1 bytes).
            Validate the current signature against the current public key.
            If valid, then move signature cursor back one position.
            Move the public key cursor back one position.
            If more signatures remain than public keys, set success=False and abort loop early.
        If loop was not aborted, set success=True.
        [non-consensus] Check NULLDUMMY rule.

    If success is False, then ensure all signatures were null. (NULLFAIL rule)
    Clean up the used stack items.

    Push success onto stack
    If opcode is OP_CHECKMULTISIGVERIFY:
        Pop success from stack
        If not success:
            FAIL

## Notes

The mechanics of CHECKMULTISIG are complicated due to the order of signature checking, the timing of when key/signature encodings are checked, the ability to either hard-fail (fail script & invalidate transaction) or soft-fail (return False on stack), and the interaction with previously activated consensus rules.
Some features of the specification are worth emphasizing:

- The legacy mode has unaltered functionality, except being restricted to only use a null dummy element.
- In both modes, public keys only have their encoding checked just prior to performing a signature check. The unchecked public keys may be arbitrary data.
- Signature and pubkey iteration always starts at the last public key and signature (the last pushed on stack). In legacy mode, the precise order of checking is critical to obtaining a correct implementation, due to the public key encoding rule.
- Note that the numbers `N`, `M`, and `checkbits` will all require minimal encoding, per activation of the minimal number encoding rule (see XXXlinkXXX).
- Only the least significant N bits may be set in `checkbits`, i.e., if `checkbits` exceeds 2<sup>N</sup>-1 then the script will fail.
- If `checkbits` has more than `M` bits set, the script will fail.
- If `checkbits` is nonzero but has fewer than `M` bits set, then the script will fail either due to an invalid signature, or because of NULLFAIL. In other words, providing only some valid Schnorr signatures (but not enough of them) will cause a script failure.
- The new mode design means that `checkbits` should have only one valid value for a given set of signatures, and thus in normal circumstances cannot be third-party malleated.
- Likewise this design stops the malleation vector of the legacy mode, since the dummy element now must be null for it to execute. The non-consensus NULLDUMMY rule will thus be obsolete, after activation.
- Third-party malleation can still occur in some unusual cases. For example, if some public key points are repeated in the list of keys, then signatures can be reordered and/or the `checkbits` can be adjusted.
- The legacy mode can require up to N signature checks in order to complete. In the new mode, exactly M signature checks occur for a sucessful operation.
- A soft-failing CHECKMULTISIG (that returns False on stack) can only occur with all null signatures, due to NULLFAIL. Thus, a False-returning CHECKMULTISIG cannot normally occur in the new mode since it would require the checkbits element to be 0 (which minimally encodes to null).
- For M=0, the opcode always evaluates True. This also requires `checkbits=0` and can normally only occur in the legacy mode.
- The new mode is actually designed to function as a full multisignature --- even including the `checkbits=0` case, though it will not be accessible in real transactions. Implementations can internally test this by disabling the minimal number encoding flag/rule, and by passing the length-1 byte array `{0x00}` for the dummy element; this is a non-null element that will activate the new mode, and yet count as `checkbits=0`.

And, some clarifications:
- As usual, checking public key encoding means permitting only 65-long byte arrays starting with 0x04, or 33-long byte arrays starting with 0x02 or 0x03.
- As usual, checking signature encoding for either ECDSA or Schnorr involves permitting only recognized hashtype bytes; Schnorr signatures must have a given length, while ECDSA signatures must follow DER encoding and Low-S rules, and must not have the length allocated to Schnorr signatures. Null signatures (empty stack elements) are also treated as 'correctly encoded'.
- The findAndDelete operation only applies to old transactions prior to August 2017, and does not impact current transactions, not even in legacy mode.


# Wallet implementation guidelines

(Currently, the common multisig wallet uses P2SH-multisig, i.e., a redeemScript of the form `M <pub0> ... <pubN> N OP_CHECKMULTISIG`. We'll focus on this use case.)

In the new Schnorr mode, *all* signatures must be Schnorr; no mixing with ECDSA is supported. Multisig wallets that wish to use the new Schnorr signatures will need to update their co-signing pools infrastructure to support a new type of signing. If some parties are unable to generate a Schnorr signature, then it will not be possible to generate a successful transaction except by restarting to make an ECDSA multisig. This creates some problems in particular when some of the parties are a hardware wallet, which may only support ECDSA for the forseeable future.

We suggest the following for wallet software producers that wish to make Schnorr multisig spends:

* Add an optional marker to the initial setup process, such as appending `?schnorr=true` to the `xpub`.
* Add a new kind of non-backwards-compatible multisignature request that indicates schnorr signatures are needed.
* If it is not known that all parties can accept Schnorr requests, then only generate ECDSA multisignature requests.
* Have the ability to participate in either ECDSA or Schnorr multisignatures, as requested.

## Calculating and pushing checkbits

In order to complete a multisignature, whether in the new mode or legacy mode, wallets need to keep track of which signatures go with which public keys. In the new mode, wallets must not just correctly order the signatures, but must also correctly include the `checkbits` parameter. Besides calculating the bit pattern, `checkbits` must also be minimally encoded to bytes, and then the bytes must be minimally pushed in the scriptSig. The last two steps can be simplified to the following procedure:

* If `checkbits` <= 16, then use OP_1 through OP_16 (opcode `0x50 + checkbits`) or OP_0, otherwise:
* Encode checkbits as a bytestring in little-endian order.
* Trim off any trailing zero bytes.
* If the last byte has its high bit set, then append a single zero byte.
* Push the bytestring using a simple push.

## ScriptSig size

Wallets need to know ahead of time the maximum transaction size, in order to set the transaction fee.

Let `R` be the length of the redeemScript and its push opcode, combined.

The legacy mode scriptSig `<dummy> <sig0> ... <sigM>` can be as large as 73M + 1 + R bytes, which is the upper limit assuming all max-sized ECDSA signatures.

In the new mode scriptSig `<checkbits> <sig0> ... <sigN>`, the signatures will have a fixed size of 66 bytes (including push opcode), however the length of `checkbits` will vary somewhat. Wallets should allocate for fees based on the largest possible encoding, which gives a scriptSig size of:
* N <= 4: `checkbits` can always be pushed using OP_1 through OP_15, so always 66M + R + 1 bytes.
* 5 <= N <= 7: `checkbits` may need to be pushed as `0x01 0xnn` -- up to 66M + R + 2 bytes.
* 8 <= N <= 15: `checkbits` may need to be pushed as `0x02 0xnnnn` -- Up to 66M + R + 3 bytes.
* 16 <= N <= 20: `checkbits` may need to be pushed as `0x03 0xnnnnnn` -- Up to 66M + R + 4 bytes.

Though there are some special cases (like 1-of-5, or 0-of-N) which could be optimized slightly further, we suggest all multisig wallets use the above simple guidelines consistently, for privacy reasons.

## Pubkey Encoding

It is strongly recommended that wallets never create scripts with invalid pubkeys, even though this specification allows them to exist in the public key list as long as they are unused. It is possible that a future rule may stipulate that all pubkeys must be strictly encoded. If that were to happen, any outputs violating this rule would become unspendable.

# Rationale and commentary on design decisions

## Repurposing of dummy element

In an earlier edition it was proposed to require N signature items (either a signature or NULL) for new mode instead of M items and a dummy element. The following problems inspired a move away from that approach:

* Triggering mechanics for the new mode were somewhat of a kluge.
* Some scripts rely on a certain expected stack layout. This is particularly the case for recently introduced high-level smart contracting languages that compile down to script, which reach deep into the stack using OP_PICK and OP_ROLL.

That said, a scan of the blockchain only found about a hundred instances of scripts that would be impacted by stack layout changes. All were based on a template as seen in [this spend](https://blockchair.com/bitcoin-cash/transaction/612bd9fc5cb40501f8704028da76c4c64c02eb0ac80e756870dba5cf32650753), where OP_DEPTH was used to choose an OP_IF execution branch.

## No mixing ECSDA / Schnorr

Allowing mixed signature types might help alleviate the issue of supporting mixed wallet versions that do support / don't support Schnorr signatures.
However, this would mean that an all-ECDSA signature list could be easily converted to the new mode, unless extra complicated steps were taken to prevent that conversion. As this is an undesirable malleability mechanism, we opted to simply exclude ECDSA from the new mode, just as Schnorr are excluded from the legacy mode.

# Acknowledgements

Thanks to Tendo Pein, Rosco Kalis, Amaury Sechet, and Antony Zegers for valuable feedback.
