# Keythereum Security Audit<a id="heading-0"/>

* 1 - [Introduction](#heading-1)
    * 1.1 - [Authenticity](#heading-1.1)
    * 1.2 - [About Keythereum](#heading-1.2)
* 2 - [Overview](#heading-2)
    * 2.1 - [Source Code](#heading-2.1)
* 3 - [Audit Results](#heading-3)
	* 3.1.1 - [`str2buf`](#heading-3.1.1)
	* 3.1.2 - [`hex2utf16le`](#heading-3.1.2)
	* 3.1.3 - [`encrypt`](#heading-3.1.3)
	* 3.2.1 - [`deriveKey` silently fails for empty passwords](#heading-3.2.1)
	* 3.2.2 - [Remove elliptic](#heading-3.2.2)
	* 3.2.3 - [Simplify isCipherAvailable](#heading-3.2.3)
	* 3.2.4 - [Remove `ethereumjs-util`](#heading-3.2.4)
	* 3.2.5 - [Simplify `create`](#heading-3.2.5)
	* 3.2.6 - [Remove validator package](#heading-3.2.6)
	* 3.2.7 - [Use `process.browser` directly](#heading-3.2.7)
	* 3.2.8 - [Use `keccak` package](#heading-3.2.8)
* 4 - [Overall Feedback & Auditors](#heading-4)
	* 4.1 - [Kirill Fomichev](#heading-4.1)
  * 4.2 - [Maciej Hirsz](#heading-4.2)
  * 4.3 - [Gustav Simonsson](#heading-4.3)


# <a id="heading-1"/> Introduction

The Keythereum library has been audited by [Kirill Fomichev](https://github.com/fanatid), [Maciej Hirsz](https://github.com/maciejhirsz), and [Gustav Simonsson](https://github.com/gustav-simonsson) during March to April 2017. The findings of the audit are presented in this document.

## <a id="heading-1.1"/> Authenticity

This document has been cryptographically signed by the Forecast Foundation to ensure it hasn't been tampered with. The signature can be verified using our [PGP public key](https://augur.net/pgp.txt).

## <a id="heading-1.2"/> About Keythereum

Keythereum is a JavaScript tool to generate, import and export Ethereum keys.  This provides a simple way to use the same account locally and in web wallets.  It can be used for verifiable cold storage wallets.  Keythereum uses the same key derivation functions, symmetric ciphers, and message authentication codes as [geth](https://github.com/ethereum/go-ethereum).

# <a id="heading-2"/> Overview

## <a id="heading-2.1"/>Source Code

The [keythereum](https://github.com/ethereumjs/keythereum) repository lives in the [ethereumjs](https://github.com/ethereumjs) group on GitHub.  Source code is publicly available in the repository's `master` branch.

The audit covered the following source files:

- `./index.js`
- `./exports.js`
- `./test/keys.js`

# <a id="heading-3"/> Audit Results

## Summary

### <a id="heading-3.1.1"/> `str2buf`

Auditor: [Maciej Hirsz](github.com/maciejhirsz) and [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #25](https://github.com/ethereumjs/keythereum/pull/25)

#### Summary

This function has been simplified and added to keythereum's public API.  Detailed unit tests were added for `str2buf` in `test/keys.js`.

--------------------------------------------------

### <a id="heading-3.1.2"/> `hex2utf16le`

Auditor: [Maciej Hirsz](github.com/maciejhirsz) and [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #26](https://github.com/ethereumjs/keythereum/pull/26)

#### Summary

This function has been removed from keythereum, since it is now using a [keccak](https://github.com/cryptocoinjs/keccak) module that accepts a raw `Buffer` (byte array) input.

--------------------------------------------------

### <a id="heading-3.1.3"/> `encrypt`

Auditor: [Maciej Hirsz](github.com/maciejhirsz)

#### Input and output encoding:

Kind of following on `str2buf` issues described above. `plaintext` is described as "Text to be encrypted". The correct description should be "Data to be encrypted". The use of the word "text" would suggest that I can safely pass any JavaScript string and have it encrypted, however I have no guarantee how that encryption will proceed, will my text be converted to a buffer using a base64, hexadecimal or a utf8 codec?

What if my input is intended to be base64 encoded, but I made an error somewhere in my own code and my base64 is malformatted - I should get an error when trying to use such malformatted input, but instead my input can be converted a buffer as UTF-8 (or worse, hexadecimal), producing an incorrect result.

An option that would be idiomatic to Node.js' `crypto` module would be to allow the consumer of the API to define the encoding of the input as well as the output (more on that later). On the same note, instead of always returning a base64 encoded string, returning a `Buffer` by default, with an option to convert it to a String would be beneficial (or skipping that step altogether and delegating the user to use `.toString(encoding)`).

#### Unnecessary encoding and decoding back to back:

This is not the only function where it happens, but it is the first one - `cipher.update` handles `Buffers` by default, `plaintext` is already converted to a `Buffer`, encoding it as a hexadecimal string only to have it parsed back to a Buffer is completely unnecessary.

#### Concatenating base64 strings is unsafe:

This function will produce correct results only if the first part of `ciphertext` is encoded without base64 padding - that is, if the length of the buffer encoded to base64 is divisible by 3 (every subsequent 3 bytes can be tightly encoded as 4 base64 bytes without padding).

**Example:** 

ASCII `"fooba"` encoded to base64 produces `"Zm9vYmE="`. ASCII `"r"` encoded to base64 produces `"cg=="`.The concatenated product is thus `"Zm9vYmE=cg=="`, which is different from when ASCII `"foobar"` is concatenated before encoding, producing `"Zm9vYmFy"` (notice lack of padding since length is divisble by 3 here).

The results should be concatenated as buffers using `Buffer.concat`, and only then encoded, if necessary.

--------------------------------------------------

### <a id="heading-3.2.1"/> `deriveKey` silently fails for empty passwords

Auditor: [Maciej Hirsz](github.com/maciejhirsz)

Pull Request: [PR #19](https://github.com/ethereumjs/keythereum/issues/19)

`deriveKey` internally performs a `if (password && salt)` before proceeding. This works fine for most scenarios, but will fail when `password` is an empty string (which should still be a valid use case).

Additionally when it does fail, it does not throw an informative exception, it just returns undefined and tends to cause a confusing error somewhere down the line (calling `slice` of `undefined` and what not).

**Suggested tweaks:**

* Change the password check to `password != null` or `typeof password === 'string'`.

* When either `password` or `salt` are missing, throw an exception.

--------------------------------------------------

### <a id="heading-3.2.2"/> Remove elliptic

Auditor: [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #24](https://github.com/ethereumjs/keythereum/pull/24)

Remove the `elliptic` library as a dependency and throughout the codebase. 

--------------------------------------------------

### <a id="heading-3.2.3"/> Simplify isCipherAvailable

Auditor: [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #24](https://github.com/ethereumjs/keythereum/pull/28)

Simplyfing the isCipherAvailable function by utilzing the `crypto` library. 

--------------------------------------------------

### <a id="heading-3.2.4"/> Remove ethereumjs-util

Auditor: [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #29](https://github.com/ethereumjs/keythereum/pull/29)

Removing `ethereumjs-util` from `index.js` and `keys.js` as it is unneeded with the Keythereum library. 

--------------------------------------------------

### <a id="heading-3.2.5"/> Simplify `create`

Auditor: [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #30](https://github.com/ethereumjs/keythereum/pull/30)

Simplifying the creation of private keys and the generation of a random salt. 

--------------------------------------------------

### <a id="heading-3.2.6"/> Remove validator package

Auditor: [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #31](https://github.com/ethereumjs/keythereum/pull/31)

Removing the validator package and implimenting our own validators. 

--------------------------------------------------

### <a id="heading-3.2.7"/> Use `process.browser` directly

Auditor: [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #33](https://github.com/ethereumjs/keythereum/pull/33)

Use the `process.browser` call directly.

--------------------------------------------------

### <a id="heading-3.2.8"/> Use `keccak` package

Auditor: [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #35](https://github.com/ethereumjs/keythereum/pull/35)

Use the `keccak.js` package directly. 

--------------------------------------------------

# <a id="heading-4"/> Overall Feedback & Auditors

## <a id="heading-4.1"/> Gustav Simonsson

* [Github](https://github.com/Gustav-Simonsson)
* [Twitter](https://twitter.com/classygustav)

>Looks good, can’t find anything wrong on first pass through code. Maybe worth double checking by running tests also against these fixtures – if padding always works, e.g. if a generated key is less than 32 bytes (I’m guessing this would have been caught by the exhaustive tests though).

## <a id="heading-4.2"/> Aaron Davis

* [GitHub](https://github.com/kumavis)
* [Twitter](https://twitter.com/kumavis_)

> ANYTHING?

## <a id="heading-4.3"/> Maciej Hirsz

* [Github](https://github.com/maciejhirsz)
* [Twitter](https://twitter.com/maciejhirsz)

> ANYTHING?

## <a id="heading-4.4"/> Kirill Fomichev

* [Github](https://github.com/fanatid)
* [Twitter](https://twitter.com/_fanatid)

> ANYTHING?
