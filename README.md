
# Introduction

Mainnet-beta, testnet, devnet gateway program address: 
```
ZETAjseVjuFsxdRxo6MmTCvqFwb3ZHUx56Co3vCmGis
```

This repository hosts the smart contracts (program)
on Solana network to support ZetaChain cross-chain
functionality. Specifically, it consists of a single
program deployed
which allows the following two actions: 

1. Users on Solana network can send SOL or selected
SPL tokens to the program to deposit into ZetaChain
and optionally invoke a ZetaChain EVM contract. 
2. Allows contracts on ZetaChain EVM to withdraw
SOL and SPL tokens to users on Solana;
3. (TO DO) In the withdraw above, optionally allow
a contract on ZetaChain EVM to withdraw SOL/SPL tokens
and call a user specified contract (program) with
parameters. 

# Authentication and Authorization

Anyone can deposit and remote invoke ZetaChain contracts. 

Only ZetaChain TSS account can make withdraw or withdrawAndCall
transactions on the program. The ZetaChain TSS account
is a collection of Observer/KeySigners which uses
ECDSA TSS (Threshold Signature Scheme) to sign 
outbound transactions. The TSS address will appear in this program
a single ECDSA secp256k1 address; but it does
not have a single private key, rather its private
key consists of multiple key shares and they collectively
sign a message in a KeySign MPC ceremony. 
The program authenticates
via verifying the TSS signature and is authorized
by ZetaChain to perform outbound transactions as
part of ZetaChain cross-chain machinery. 

The ZetaChain TSS is on ECDSA secp256k1 curve; 
But Solana native digital signature scheme is
EdDSA Ed25519 curve.  Therefore the program uses
custom logic to verify the TSS ECDSA signature
(like alternative authentication in smart contract wallet); 
the native transaction signer (fee payer on Solana)
does not carry authorization and it's only used
to build the transaction and pay tx fees. There
are no restrictions on who the native transaction
signer/fee payer is. The following code excerpt is
for authenticating TSS signature in the contract itself, 
using the [Rust secp256k1 library bundled with solana](https://docs.rs/solana-program/latest/solana_program/secp256k1_recover/index.html):
https://github.com/zeta-chain/protocol-contracts-solana/blob/01eeb9733a00b6e972de0578b0e07ebc5837ec54/programs/protocol-contracts-solana/src/lib.rs#L116-L121

The function `recover_eth_address` is implemented in the gateway program: 
https://github.com/zeta-chain/protocol-contracts-solana/blob/01eeb9733a00b6e972de0578b0e07ebc5837ec54/programs/protocol-contracts-solana/src/lib.rs#L180-L196

The TSS signature is a ECDSA secp256k1 signature; its public key therefore address
(Ethereum compatible hashing from pubkey) is therefore verifiable using the `secp256k1_recover`
function. Alternatively, Solana runtime also provides a program to provide this verification service
via CPI; see [proposal 48](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0048-native-program-for-secp256r1-sigverify.md)
which might be more cost efficient. 

In both `withdraw` and `withdrawAndCall` instructions, 
the ECDSA signed message_hash must commit to the 
`nonce`, `amount`, and `to` address. See the 
check in these instructions like: 
https://github.com/zeta-chain/protocol-contracts-solana/blob/01eeb9733a00b6e972de0578b0e07ebc5837ec54/programs/protocol-contracts-solana/src/lib.rs#L110-L114
The commitment of `nonce` in the signature prevents replay of the TSS ECDSA signed message. 
Also (to be added in https://github.com/zeta-chain/protocol-contracts-solana/issues/6) a chain id of Solana
should be added to the commitment of the message hash, so that the signature cannot be replayed on *other blockchains*
that potentially uses similar authentication (say in TON). 

# Build and Test Instructions

Prerequisites: a recent version of `rust` compiler
and `cargo` package manger must be installed. The program
is built with the `anchor` framework so it needs to be
installed as well; see [installation](https://www.anchor-lang.com/docs/installation)

Please install compatible Solana tools and anchor-cli before build, otherwise the program will not be built successfully
```bash
$ solana-install init 1.18.15

$ cargo install --git https://github.com/coral-xyz/anchor --tag v0.30.0 anchor-cli --locked
```

To show the installed versions of the tools
```bash
$ cargo-build-sbf --version
solana-cargo-build-sbf 1.18.15
platform-tools v1.41
rustc 1.75.0

$ anchor --version
anchor-cli 0.30.0
```

To build (this will require at least 2GB disk space)
```bash
$ anchor build
```

To run the tests
```bash
$ anchor test
```

# Relevant Account and Addresses

The Gateway program derive a PDA (Program Derived Address)
with seeds `b"meta"` and canonical bump. 
This PDA account/address actually holds the SOL
balance of the Gateway program. 
For SPL tokens, the program stores the SPL token
in PDA derived ATAs. For each SPL token (different mint
account), the program creates ATA from PDA and the Mint
(standard way of deriving ATA in Solana SPL program).

The PDA account itself is a data account that holds
Gateway program state, namely the following data types
https://github.com/zeta-chain/protocol-contracts-solana/blob/01eeb9733a00b6e972de0578b0e07ebc5837ec54/programs/protocol-contracts-solana/src/lib.rs#L271-L275

The `nonce` is incremented on each successful withdraw transaction,
and it's used to prevent replay of signed ECDSA messages. 
The `tss_address` is the TSS address of ZetaChain (20Bytes,
Ethereum style). `authority` is the one who can update
the TSS address stored in PDA account. 

The `initialize` instruction sets nonce to 0. 

# Troubleshooting

## MacOS error when running `anchor test` or `solana-test-validator`

If you see errors like
```
Unable to get latest blockhash. Test validator does not look started. Check ".anchor/test-ledger/test-ledger-log.txt" for errors. Consider increasing [test.startup_wait] in Anchor.toml.
```

or 
```bash
% solana-test-validator --reset
Ledger location: test-ledger
Log: test-ledger/validator.log
Error: failed to start validator: Failed to create ledger at test-ledger: io error: Error checking to unpack genesis archive: Archive error: extra entry found: "._genesis.bin" Regular
```

This is because the BSD tar program is not compatible with the GNU tar program.

FIX: install GNU tar program using homebrew and export it's executable path in your .zshrc file.

## Mac with Apple Silicon

```bash
brew install gnu-tar
# Put this in ~/.zshrc 
export PATH="/opt/homebrew/opt/gnu-tar/libexec/gnubin:$PATH"
```

## Intel-based Mac

```bash
brew install gnu-tar
# Put this in ~/.zshrc 
export PATH="/usr/local/opt/gnu-tar/libexec/gnubin:$PATH"
```
see https://solana.stackexchange.com/questions/4499/blockstore-error-when-starting-solana-test-validator-on-macos-13-0-1/16319#16319