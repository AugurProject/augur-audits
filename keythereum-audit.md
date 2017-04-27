-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

# Keythereum Security Audit<a id="heading-0"/>

* 1 - [Introduction](#heading-1)
    * 1.1 - [Authenticity](#heading-1.1)
    * 1.2 - [About Keythereum](#heading-1.2)
* 2 - [Overview](#heading-2)
* 3 - [Audit Results](#heading-3)
    * 3.1 - [Changes to dependencies](#heading-3.1)
        * 3.1.1 - [Dependencies added](#heading-3.1.1)
            * 3.1.1.1 - [`cryptocoinjs/keccak`](#heading-3.1.1.1)
        * 3.1.2 - [Dependencies removed](#heading-3.1.2)
            * 3.1.2.1 - [`indutny/elliptic`](#heading-3.1.2.1)
            * 3.1.2.2 - [`ethereumjs/ethereumjs-util`](#heading-3.1.2.2)
            * 3.1.2.3 - [`chriso/validator`](#heading-3.1.2.3)
            * 3.1.2.4 - [`drostie/sha3-js`](#heading-3.1.2.4)
    * 3.2 - [Changes to functions](#heading-3.2)
        * 3.2.1 - [`create`](#heading-3.2.1)
        * 3.2.2 - [`decrypt`](#heading-3.2.2)
        * 3.2.3 - [`deriveKey`](#heading-3.2.3)
        * 3.2.4 - [`encrypt`](#heading-3.2.4)
        * 3.2.5 - [`hex2utf16le`](#heading-3.2.5)
        * 3.2.6 - [`isCipherAvailable`](#heading-3.2.6)
        * 3.2.7 - [`privateKeyToAddress`](#heading-3.2.7)
        * 3.2.8 - [`recover`](#heading-3.2.8)
        * 3.2.9 - [`str2buf`](#heading-3.2.9)
* 4 - [Auditors](#heading-4)
    * 4.1 - [Kirill Fomichev](#heading-4.1)
    * 4.2 - [Maciej Hirsz](#heading-4.2)
    * 4.3 - [Gustav Simonsson](#heading-4.3)


# <a id="heading-1"/> Introduction

The Keythereum library has been audited by [Kirill Fomichev](https://github.com/fanatid), [Maciej Hirsz](https://github.com/maciejhirsz), and [Gustav Simonsson](https://github.com/gustav-simonsson) during March-April 2017. The findings of the audit are presented in this document. This document will be updated if additional changes are made to Keythereum from the auditors findings.

## <a id="heading-1.1"/> Authenticity

This document has been cryptographically signed by the Forecast Foundation to ensure it hasn't been tampered with. The signature can be verified using our [PGP public key](https://augur.net/pgp.txt).

## <a id="heading-1.2"/> About Keythereum

Keythereum is a JavaScript tool to generate, import and export Ethereum keys.  This provides a simple way to use the same account locally and in web wallets.  It can be used for verifiable cold storage wallets.  Keythereum uses the same key derivation functions, symmetric ciphers, and message authentication codes as [geth](https://github.com/ethereum/go-ethereum).

# <a id="heading-2"/> Overview

The [keythereum](https://github.com/ethereumjs/keythereum) repository lives in the [ethereumjs](https://github.com/ethereumjs) group on GitHub.  Source code is publicly available in the repository's `master` branch.

The audit covered the following source files:

- - `./index.js`
- - `./exports.js`
- - `./test/keys.js`

# <a id="heading-3"/> Audit Results

## <a id="heading-3.1"/> Changes to dependencies

### <a id="heading-3.1.1"/> Dependencies added

- - <a id="heading-3.1.1.1"/> [cryptocoinjs/keccak](https://github.com/cryptocoinjs/keccak): [PR #35](https://github.com/ethereumjs/keythereum/pull/35)

### <a id="heading-3.1.2"/> Dependencies removed

- - <a id="heading-3.1.2.1"/> [indutny/elliptic](https://github.com/indutny/elliptic): [PR #24](https://github.com/ethereumjs/keythereum/pull/24)
- - <a id="heading-3.1.2.2"/> [ethereumjs/ethereumjs-util](https://github.com/ethereumjs/ethereumjs-util): [PR #29](https://github.com/ethereumjs/keythereum/pull/29)
- - <a id="heading-3.1.2.3"/> [chriso/validator](https://github.com/chriso/validator.js): [PR #31](https://github.com/ethereumjs/keythereum/pull/31)
- - <a id="heading-3.1.2.4"/> [drostie/sha3-js](https://github.com/drostie/sha3-js): [PR #35](https://github.com/ethereumjs/keythereum/pull/35)

## <a id="heading-3.2"/> Changes to functions

### <a id="heading-3.2.1"/> `create`

- - [PR #30](https://github.com/ethereumjs/keythereum/pull/30): Simplify create

This function was simplified and edited to improve its readability.

- --------------------------------------------------

### <a id="heading-3.2.2"/> `decrypt`

- - [commit 016e0d1](https://github.com/ethereumjs/keythereum/commit/016e0d12da24af53063b8688bc6621a3413b8807): Use buffer concat in encrypt/decrypt; encrypt/decrypt now return buffers

Unnecessary encoding/decoding steps have been removed from this function.  This function now returns a `Buffer` instead of a hexadecimal string.

- --------------------------------------------------

### <a id="heading-3.2.3"/> `deriveKey`

- - [PR #20](https://github.com/ethereumjs/keythereum/issues/20) deriveKey silently fails for empty passwords

An initial check for `undefined` or `null` has been added to `deriveKey`.  This function now throws an `"Must provide password and salt to derive a key"` error if password or salt are not provided.

- --------------------------------------------------

### <a id="heading-3.2.4"/> `encrypt`

- - [commit 016e0d1](https://github.com/ethereumjs/keythereum/commit/016e0d12da24af53063b8688bc6621a3413b8807): Use buffer concat in encrypt/decrypt; encrypt/decrypt now return buffers

Unnecessary encoding/decoding steps have been removed from this function.  String concatenation has been replaced with buffer concatenation.  This function now returns a `Buffer` instead of a base64 string.

- --------------------------------------------------

### <a id="heading-3.2.5"/> `hex2utf16le`

- - [PR #26](https://github.com/ethereumjs/keythereum/pull/26): Simplify hex2utf16le
- - [commit cdfece3](https://github.com/ethereumjs/keythereum/commit/cdfece32c721c10334b5e6bce3c88149a6eaeafb): Removed unused hex2utf16le function

This function has been removed, as it is not needed for the [keccak](https://github.com/cryptocoinjs/keccak) module's input.

- --------------------------------------------------

### <a id="heading-3.2.6"/> `isCipherAvailable`

- - [PR #24](https://github.com/ethereumjs/keythereum/pull/28): Simplify isCipherAvailable

This function was edited to improve its readability.

- --------------------------------------------------

### <a id="heading-3.2.7"/> `privateKeyToAddress`

- - [commit 6557352](https://github.com/ethereumjs/keythereum/commit/65573528e55860d6e1f0f1729d0a75cd93cfe477) Left-pad private keys to 32 bytes; added more privateKeyToAddress test cases

Test vectors from [go-ethereum](https://github.com/ethereum/go-ethereum)'s test data have been added to this function's tests.

- --------------------------------------------------

### <a id="heading-3.2.8"/> `recover`

- - [commit 7c52909](https://github.com/ethereumjs/keythereum/commit/7c52909aca9a6a913a06c461dbe740284507cd6e): Version 1 private keys can now be recovered; added isCipherAvailable check prior to encryption/decryption
- - [commit 495d0dd](https://github.com/ethereumjs/keythereum/commit/495d0ddaeacfd00232342aa91459a414e7fb638c): `.recover` should not affect `.constants`
- - [commit 6557352](https://github.com/ethereumjs/keythereum/commit/65573528e55860d6e1f0f1729d0a75cd93cfe477) Left-pad private keys to 32 bytes; added more privateKeyToAddress test cases
- - [commit 2b696be](https://github.com/ethereumjs/keythereum/commit/2b696bed35d4dbbce3470879ed3d7652fac6d2f0): Added test vectors from go-ethereum testdata

This function now supports recovery (decryption) of version 1 private keys, the original encrypted private key format used for Ethereum keys before Olympic.  `recover` no longer modifies `keythereum.constants`.  Private keys that are less than 32 bytes now work correctly with this function.  Test vectors from [go-ethereum](https://github.com/ethereum/go-ethereum)'s test data have been added to this function's test cases.

- --------------------------------------------------

### <a id="heading-3.2.9"/> `str2buf`

- - [PR #25](https://github.com/ethereumjs/keythereum/pull/25): Simplify code with checks
- - [commit 2bdf6c4](https://github.com/ethereumjs/keythereum/commit/2bdf6c433b92a5bc77e334c085bf0ed388ab6e4f): Export and unit test str2buf and hex2utf16le; use Buffer.from instead of new Buffer

This function has been simplified, added to keythereum's public API, and included in unit tests.

- --------------------------------------------------

# <a id="heading-4"/> Auditors

## <a id="heading-4.4"/> Kirill Fomichev

* [Github](https://github.com/fanatid)
* [Twitter](https://twitter.com/_fanatid)

## <a id="heading-4.3"/> Maciej Hirsz

* [Github](https://github.com/maciejhirsz)
* [Twitter](https://twitter.com/maciejhirsz)

## <a id="heading-4.1"/> Gustav Simonsson

* [Github](https://github.com/Gustav-Simonsson)
* [Twitter](https://twitter.com/classygustav)

-----BEGIN PGP SIGNATURE-----
Comment: GPGTools - https://gpgtools.org

iQIcBAEBCgAGBQJZAUhJAAoJEEJOD01Ku7vgkioP/ieDChkGQ8C6Gl25OMcv1Wk9
jDi9EjldDO7UQFgWFAnFd+0ZHFG5Yupr+7BBm7FA/g2tDekwwvWWFptBLHmhLbdW
2KXJlRTFw7d7zyxA+6hWAaYBuj4/czzHumCBR8ysQZ0xEfKjDy+o5qIIrKGsh9DM
jG+SCi3oe5p8u4FyswtGQ2z6SJneLpOT6l29nLR1/GMMjCSpk0+MdxsYUBzyOg1u
Fgnyta0B+TuH5io6MoqmB9djveYF72iE42tvMuf4E37bVnCoDY+irCKDhDkZtoEv
Z+TFQCccHR90tWpY+4LXvDWlhQNUru/R3GQ7Qi6qplqZ9Y42TV5sS9pqkIG8hljV
FcGZxIAlg5/YYX+m7RTWqAWwuweS8H3kRQvdTXniC4X3h8PtMoDXq27cz/qY8Cso
+onAHvCdUvzOzjyjPLsz1ZuYk/cmCBVFbCHpOFtcd1VKLmg2C6yBuNsut0FxJLyM
sN0x1TgTgBGWC6+uk/2o+8eKsrcYvMZrWmjXOZJ+XKdsuir5bh+JDpjpiH5wHcJ+
i03vxUiyrnfhpHPk0GaWDrBg8r+E9o2Qkcap48PV/MRHfE/FEl0mnqzLLO4rxTFQ
JUe0JbmAO0zcBX3/v91Z409DiO0FfJq6pWhrJ/nq4l1kG7DfEBLtElEWUF2BIklh
vO4d1zM2/Ze4EqHi5hua
=Xe58
-----END PGP SIGNATURE-----
