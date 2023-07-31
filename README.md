# CHIP-2023-07-MBC-standard: Minting Baton Covenant (MBC) Standard for Fungible Tokens

        Title: Minting Baton Covenant (MBC) Standard for Fungible Tokens
        Type: Standards
        Layer: Applications
        Owners: Mathieu Geukens 
        Status: Draft
        Initial Publication Date: 2023-07-10
        Latest Revision Date: 2023-07-31

## Introduction

This standard uses a covenant to achieve the same functionality as a classic minting baton that controls issuance.
The minting covenant is tracked using an authchain starting at the covenant's creation making it easy to track the minting history.
The authority to mint tokens is held by the minting baton NFT making it easy to track changes to the minting setup.

By keeping the design simple, this specification aims to be a practical solution which can be used immediately.

## Deployment

This proposal does not require coordinated deployment.

## Motivation

This specification is a standard to create minting baton functionality for CashTokens. Bitcoin Cash does not have 
the concept of a minting baton for fungible CashTokens natively because then the token amount could overflow 64-bit integer size.
Other token protocols, such as SLP and ERC20, allow for issuing supply on an ongoing basis with a minting token.
The CashTokens and BCMR specifications propose how to mark supply as reserved, held by the trusted entity and not in circulation but this is not exaclty the minting baton functionality and has drawbacks of its own.

This standard uses a covenant holding the reserved supply to create a minting baton NFT for fungible CashTokens.
An extension to the BCMR metadata schema is specified for tokens following the MBC token-standard, this way applications such as wallets and
blockexplorers can be correctly display the NFT as a MBC minting baton and can easily show the issued supply.

## Benefits

### Simple

By using only a single covenant to manage to reserved supply and a single minting baton, the MBC token-standard discerns itself with the focus on simplicity.
The minting covenant managing the reserved supply is very minimal and simple with only 3 checks (8 opcodes).

### Practical

By integrating with the BCMR standard from the start and by using an authchain to track the covenant instead of needing specific indexed tokens,
this specification focuses on being a practical solution.
The MBC standard makes it easy to track the minting history, the issued and unissued supply and any transfers of the minting authority.

### Known Method

The concept of a minting baton is familiar, it is a known method from other token standards. The issuer pre-minting the maximum supply to his own address and marking it as reserved is different from this. Having the maximum supply already in the wallet of the issuer feels very different from just having the option to mint more.

## Technical Specification

### The Genesis Transaction

The genesis transaction has 5 outputs in the following structure:

- output index 0: BCMR authchain for metadata updates
- output index 1: minting covenant, holding the unreleased tokens (and starting the covenant authchain)
- output index 2: minting baton NFT
- output index 3: released tokens
- output index 4: OP_RETURN with self-cerified BCMR info

The first and last output are for the BCMR on-chain metadata.
The second output is the simple minting covenant specified in `mintingCovenant.cash` which holds the reserved token supply.
The 'minting baton' is a one-off immutable NFT with a zero commitment (`0x00`) and of the same tokenId as the created fungible tokens.
The third output is the minting baton, an immutable one-off NFT with the same tokenId as the fungible tokens created.
The fourth output are the tokens released into circulation right away.

### Metadata

Tokens created according to the MBC standard are required to link self-certified BCMR on-chain in the token genesis transaction.
To show from the BCMR json file that the token follows the MBC standard the extensions object should be extended in the following way:
```
{
  "extensions": {
    "tokenStandard": "MBC"
  }
}
```

When wallets or blockexplorers detect this in the metadata file, the applications can label the NFT as a MBC minting baton.
Applications can choose whether to display the icon of the fungible tokens for the baton or to display a generic icon for all MBC minting batons.

### Tracking the Reserved Supply

Tokens using the MBC standard make an important distinction between genesis supply, total supply, circulating supply and reserved supply.
These concepts are well defined in the [CashTokens specification](https://github.com/cashtokens/cashtokens).
Wallets, blockexplorers and other applications might want to display this information to users.

Calculating the **reserved supply** is somewhat different from the CashTokens specification as there is only one reserved supply covenant and
this covenant is not carrying a `mutable` or `minting` NFT. Instead, the covenant is tracked using an authchain from its creation in the token genesis.
This means the authchain holds the full history of all minting events and the authHead holds the current unissued supply.
Tracking an authchain works the same as explained in the [BCMR metadata standard](https://github.com/bitjson/chip-bcmr#zeroth-descendant-transaction-chains)
with a nice graphic.

Additionally, this specification introduces the concept of **issued supply** which is simply the `genesisSupply - reservedSupply`.
The advantage of issued supply over the definition of circulating supply is that for calculating the circulating supply all token UTXOs should be checked
so the tokens which are burned by sending them to an OP_RETURN output can be removed from the circulating supply. 
This requires specialized indexers to keep track of the circulating supply, whereas the issued supply would always be easy to calculate.

In user-interfaces for a token following the MBC standard, the token's genesis supply may be labeled as **open-ended** when the maxint 
(`9223372036854775807`) is created in the genesis transaction. This shows users that there is no hard limit on the amount of tokens without confusing
users with this very large number in user interfaces.

### Verifying the Covenant Contract

Verifying the covenant contract is not strictly necessary as the issuer is trusted with the minting baton so could release the whole locked supply anyway.
If applications do want to check this anyway they need to generate P2SH lockingbytecode for a minting covenant for that specific tokenId and 
check it with the actual P2SH lockingbytecode. Equivalently the generated p2sh address from the tokenId could be checked against the actual p2sh address.

## Rationale

This section documents design decisions made in this specification.

### Single Minting Baton & Covenant

Having a single minting baton and a single covenant simplifies the specification and the usecase for having multiple batons/covenants is unclear. 
The minting baton NFT itself can be governed by arbitrary rules such as a 1-of-3 multisig already allowing to give multiple people the
authority to mint new CashTokens of the same kind.
Having only one covenant with reserved supply simplifies implementation for applications querying the issued supply.

### Simultaneous Creation of Tokens & Baton

The simultaneous creation of the fungible tokens, the minting covenant and the minting baton is so the whole setup can happen in one transaction.
Creating the covenant right away also makes it possible to track the minting covenant authchain for the token genesis.

## Evaluation of Alternatives

### CashTokens spec

The [CashTokens specification](https://github.com/cashtokens/cashtokens) proposes how reserved supply of fungible tokens should be marked in the [fungible-token-supply-definitions](https://github.com/cashtokens/cashtokens#fungible-token-supply-definitions) section.
The specification suggests keeping the unissued supply in covenants holding `minting` or `mutable` NFT of the same tokenId.
This is with the usecase of multi-threaded covenant design in mind so there is a need for multiple markers.

Instead, the MBC standard keeps the reserved supply in a single covenant so the full minting history can be tracked with just the authchain 
as to enable validation of the reserved supply by by light clients.
Compared to the CashTokens spec, this simplifies getting the minting history, it also make a key rotation/transfer of the authority to mint 
new tokens separate from the minting history itself, as those are now done by sending the minting baton.
Lastly, this standard makes it clear through the BCMR metadata that the token follows the MBC specification so applications can correctly display 
the minting baton NFT and information about issued supply, reserved supply, etc.

### BCMR spec

The [BCMR specification](https://github.com/bitjson/chip-bcmr#providing-for-continued-issuance-of-fungible-tokens) further details how reserved supply can be verified by light clients in [continued-issuance-of-fungible-tokens](https://github.com/bitjson/chip-bcmr#providing-for-continued-issuance-of-fungible-tokens) section. 
This is achieved by keeping the reserved supply in the identity output and marked by a `mutable` NFT.

Using an authchain to track the reserved/unissued supply is similar to the current proposal, but using the token's authchain for both metadata updates and issuance makes it harder to separate both events. 
If you want to control the issuance with a covenant like the current proposal, the covenant would also have to allow for metadata updates.

For light clients it is not clear when they should check the identity output for reserved supply and when there are multiple reserved supply markers.
That is why the current proposal proposes a `BCMR` extension to tell clients when they can check the reserved supply by following the authchain.

## Implementations

The following software is known to support the MBC-standard:

*(pending initial implementations)*

## Feedback & Reviews

## Acknowledgements

Thank you to Imaginary_username, Joemar Taganna, Dagur Valberg and bitcoincashautist for providing early feedback.

## Changelog
- **v0.1.2 – 2022-7-31**
  - added Known Method section to the benefits
  - added BCMR spec to evaluated alternatives
- **v0.1.1 – 2022-7-10**
  - fixed issue empty commit in covenant
- **v0.1.0 – 2022-7-10**
  - first published draft version

## Copyright

This document is placed in the public domain.
