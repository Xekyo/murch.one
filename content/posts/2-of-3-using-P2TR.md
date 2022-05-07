---
title: "2-of-3 inputs using Pay-to-Taproot"
date: 2020-08-18
submitted: true
---

*This article was amended on 2020–12–20 to improve the description of the P2TR output scripts. Thanks to Matthew Zipkin, for pointing out the prior imprecision.*

The Bitcoin community has been [abuzz for a few years](https://medium.com/hackernoon/excited-for-schnorr-signatures-a00ee467fc5f) about bringing Schnorr signatures to Bitcoin. Since then, the idea has evolved into three formal Bitcoin Improvement Proposals: ‘[BIP340 — Schnorr Signatures for secp256k1](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)’, ‘[BIP341 — Taproot: SegWit version 1 spending rules](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)’, and ‘[BIP342 — Validation of Taproot Scripts](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)’. Respectively, they define a standard for Schnorr signatures in Bitcoin, introduce the Taproot construction, and formalize the new Script v1 instruction set. Taproot also iterates on the previous proposal of Merklized Alternative Script Trees (MAST).

There are two ways of spending a pay-to-taproot (P2TR) output. The first way is called *keypath spending*. P2TR funds are locked to a single public key similarly to *pay-to-public-key* (P2PK) outputs. The output script appears in the transaction as `OP_1 <pubkey>`. This script can be satisfied by providing an empty input script and a Schnorr signature of the corresponding private key in the witness program. The keypath is the default spending path. It is used in single-sig and collaborative multi-party spending.

Under the hood, the public key is a composition of an *inner key* and the Merkle root of a Taproot tree (the inner key is tweaked with the Merkle root). The inner key can optionally be a composite itself, for example multiple public keys aggregated via the [MuSig](https://blockstream.com/2019/02/18/en-musig-a-new-multisignature-standard/) scheme. These compositions are possible due to Schnorr signatures being linear.

The second way of spending a P2TR output uses one of the Taproot leaves. We call this *scriptpath spending*. First, the existence of the leaf needs to be proven which is done by revealing the inner key, the Merkle path to the Taproot leaf, and the script encoded in the leaf. Then, the spending conditions of the leaf’s script are fulfilled.

The remainder of this article explores the input costs of 2-of-3 multisig Taproot constructions. Some familiarity with the details of the proposals is helpful. Check out the [Taproot overview](https://bitcoinops.org/en/topics/taproot/) and the excellent [summary of the three BIPs](https://bitcoinops.org/en/newsletters/2019/05/14/#overview-of-the-taproot--tapscript-proposed-bips) by the Bitcoin Optech Group if you are looking to refresh the details.

## The multisig user story

We are looking at a wallet construction using three keys held by two or three parties. The first key is held by the user, the second key is held by the service provider, and the third key is a backup key either held by the user or a key recovery service. We are assuming that the first two keys are hot and can be used in an interactive signing scheme as employed by MuSig. The backup key may be held cold e.g. on an air-gapped system in which case interactive signing should be avoided.

{{< figure src="/images/2-of-3-keys.png" class="center" caption="2-of-3 multisig with two hot keys and one (optionally) cold key" >}}

Classically, a 2-of-3 multisig address is constructed in the form of a *pay-to-scripthash* (P2SH) output of the form `2 <pubkey1> <pubkey2> <pubkey3> 3 OP_CHECKMULTISIG`. This construction can logically be expressed as ‘(A ∧ B) ∨(A ∧ C) ∨(B ∧ C)’ which could also be thought of as “one of three 2-of-2 multisig scripts”. The latter translates nicely to Taproot: we can encode the preferred condition of spending with the user and the service key (the two hot keys) in an aggregated pubkey for the keypath, and put the two spending conditions using the backup key in leaves of the Taproot tree. Two variants are explored: one where the backup key is capable of participating in the interactive MuSig signing scheme, another that falls back to a simpler multisig scheme where signing is non-interactive e.g. because the backup key is air-gapped and the multiple roundtrips required for MuSig are inconvenient.

{{< figure src="/images/tweak.png" class="center" caption="The most likely spending option with the two hot keys is encoded as an aggregated key in the keypath. The two backup spending options are put into leaves of the script tree." >}}

## Keypath spending costs

In the general case, all parties agree on a course of action and collaborate to use the keypath. This is the most cost-effective way to spend a P2TR output and allows multiparty spenders (and other complex spending conditions) to be indistinguishable from single-sig spenders.

### Costs of keypath input

An input commits to a specific *unspent transaction output* (UTXO) which is defined by the transaction that created it and the position in that transaction’s output list. The UTXO entry provides the spending conditions to be satisfied by the spender. An *nSequence* field allows encoding replaceability, and the witness provides the spender’s authentication. In the case of the keypath this is a single Schnorr signature which in some cases may actually consist of multiple aggregated signatures.

* outpoint (txid:vout): 32 vB+4 vB
* scriptSig size: 1 vB
* nSequence: 4 vB
* count witness items: 1 WU
* witness item size: 1 WU
* signature: 64 WU

32+4+1+4+(1+1+64)/4 = 57.5 vB

## Control Blocks

When a fallback to the backup key is necessary, the existence of the Taproot tree must be revealed. If there is only a single leaf, the spender provides only the inner key. Tweaking the inner key with the hashed leaf results in the public key. In sum, the data necessary to prove the existence of a scriptpath is called the *control block*.

### Depth 0 control block

* Length of control block: 1 WU
* Header byte (script version, sign of output key): 1 WU
* Inner key: 32 WU

1+1+32 = 34 WU

In case of two leaves, additionally the first hashing partner for the Merkle path must be revealed:

### Depth 1 control block

* Length of control block: 1 WU
* Header byte: 1 WU
* Inner key of root key: 32 WU
* Hashing partner in tree: 32 WU

1+ 1+ 32 + 32 = 66 WU

## Scriptpath spending cost

The below costs are in addition to the above costs of spending via the keypath.

### Scriptpath spend assuming 2-of-2 MuSig leaf (hot backup key)

When the backup key is on a networked system, e.g. an HSM, and can participate in a multi-roundtrip signing process, we can make use of MuSig to aggregate the two public keys.

* script size: 1 WU
* script `<pk> OP_CHECKSIG`: 33 WU+1 WU
* Depth 1 Control block: 66 WU

57.5+(1+34+66)/4 = 82.75 vB

### Construction with 2-of-2 OP_CHECKSIG (cold backup key, no MuSig)

In the case that the backup key is offline and a human would have to make multiple trips employing USB sticks or QR codes, saving roundtrips may take precedence over saving a few bytes. Instead of an aggregated public key, we use a non-interactive multisig construction.

* second signature: 1 WU+64 WU
* script size: 1 WU
* script `<pk1> OP_CHECKSIGVERIFY <pk2> OP_CHECKSIG`: 33+1+33+1=68 WU
* Depth 1 Control block: 66 WU

57.5+(1+64+1+68+66)/4 = 107.5 vB

### Discarded approach: single leaf with 2-of-3 script

It turns out that a single 2-of-3 leaf in lieu of the two 2-of-2 leaves is both more costly and less private.

* +2nd sig: 1+64 WU
* +1 empty witness item: 2 WU
* Length of script: 1 WU
* Script `<pk1> OP_CHECKSIG <pk2> OP_CHECKSIGADD <pk3> OP_CHECKSIGADD 2 OP_EQUAL`: 33+1+33+1+33+1+2=104 WU
* Depth 0 Control block: 34 WU

57.5+(1+64+2+1+104+34)/4 = 109 vB

## Conclusion

{{< figure src="/images/2-of-3-costs.png" class="center" caption="Upper bound of input and output sizes for single-sig and 2-of-3 multisig" >}}

The described 2-of-3 multisig scheme achieves input sizes of 57.5 vbytes for a keypath spend, 82.75 vbytes for the leaves using a hot backup key, and 107.5 vbytes for non-interactive backup spends. This results in a fee reduction by 45% for 2-of-3 inputs when switching from P2WSH to P2TR spending. In the uncommon case of a recovery transaction, the cost is negligibly increased for cold keys. Single-sig users are also incentivized to switch to P2TR as they save 11 vbytes on each input — the output cost is externalized on the sender.

*This article was [originally published](https://murchandamus.medium.com/2-of-3-multisig-inputs-using-pay-to-taproot-d5faf2312ba3) on medium.*
