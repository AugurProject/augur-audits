# Keythereum Security Audit<a id="heading-0"/>

* 1 - [Introduction](#heading-1)
    * 1.1 - [Authenticity](#heading-1.1)
    * 1.2 - [About Keythereum](#heading-1.2)
* 2 - [Overview](#heading-2)
    * 2.1 - [Source Code](#heading-2.1)
* 3 - [Audit Results](#heading-3)
    * 3.1 - [Detailed Findings](#heading-3.1)
		* 3.1.1 - [`str2buf`](#heading-3.1.1)
		* 3.1.2 - [`hex2utf16le`](#heading-3.1.2)
		* 3.1.3 - [`encrypt`](#heading-3.1.3)
	* 3.2 - [General Findings](#heading-3.2)
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

This document has been cryptographically signed by the Forecast Foundation to ensure it hasn't been tampered with. The signature can be verified using our public key, linked on Augur.net below.

[Forecast Foundations PGP Key](https://augur.net/pgp.txt)


## <a id="heading-1.2"/> About Keythereum

The primary repository for [Keythereum](https://github.com/ethereumjs/keythereum) can be found in the [EthereumJS](https://github.com/ethereumjs) team repository on GitHub.

> Keythereum is a JavaScript tool to generate, import and export Ethereum keys. This provides a simple way to use the same account locally and in web wallets. It can be used for verifiable cold storage wallets.

Keythereum uses the same key derivation functions (PBKDF2-SHA256 or scrypt), symmetric ciphers (AES-128-CTR or AES-128-CBC), and message authentication codes as geth. You can export your generated key to file, copy it to your data directory's keystore, and immediately start using it in your local Ethereum client.

# <a id="heading-2"/> Overview

## <a id="heading-2.1"/>Source Code

The ENS source code is publicly available in the `keythereum/master` branch of the Keythereum Github
repository.

[https://github.com/ethereumjs/keythereum](https://github.com/ethereumjs/keythereum)

This audit covers the following source files:

- `./index.js`
- `./exports.js`
- `./test/keys.js`

# <a id="heading-3"/> Audit Results

This section contains the issues, improvements, and fixes that have been addressed and outlined by the Keytheruem auditors. 


## <a id="heading-3.1"/> Detailed Findings

Outlined below are detailed findings of the Keythereum audit: 

--------------------------------------------------

### <a id="heading-3.1.1"/> `str2buf`

Auditor: [Maciej Hirsz](github.com/maciejhirsz) and [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #25](https://github.com/ethereumjs/keythereum/pull/25)

#### No fast-fail mechanism: 

The function will only process Strings, and otherwise return any value passed unchanged. If I misuse the function and do something along the lines of:

```js
const notAString = 5;

doSomethingWithABuffer(str2buf(notAString));
```

I will, hopefully, get an exception that something is amiss down in the guts of `doSomethingWithABuffer`. The problem here is that the actual error - passing a number where a string was expected - isn't detected until much later in the execution of the program. Any situation where the execution continues while operating on erroneous data is far worse than throwing an error. An informative error here would helping the developer using the function find the fault at the level where it occurs in shorter time.

In case a situation where a `Buffer` is passed into the function should be allowed, and that `Buffer` should then be returned unchanged, such a situation should be handled explicitly.

#### Potential correctness issues when the format isn't easy to deduct:

Consider a string `deadbeef` - is it a buffer serialized as hexadecimal or base64? Turns out, it can be either. The probability of a situation where a buffer is encoded as base64 and then incorrectly parsed as hexadecimal is insanely low for buffers of any reasonable length (32 bytes for private keys should virtually be unaffected), yet still - a chance of an error equal to 0 would be preferable.

This might be good for the end user who has to deal with the implementation, but the checks made are completely unnecessary in the inner-workings of the library where the encoding is (should be) known beforehand, more on that later.

#### No unit tests:

This is a pure function, which is a unit test heaven. It's however not exposed in the API, and thus not tested.

--------------------------------------------------

### <a id="heading-3.1.2"/> `hex2utf16le`

Auditor: [Maciej Hirsz](github.com/maciejhirsz) and [Kirill Fomichev](https://github.com/fanatid)

Pull Request: [PR #26](https://github.com/ethereumjs/keythereum/pull/26)

#### Summary

This function has been removed from keythereum, since it is now using a [keccak](https://github.com/cryptocoinjs/keccak) module that accepts a raw `Buffer` (byte array) input.

--------------------------------------------------

### <a id="heading-3.1.3"/> `encrypt`

Auditor: [Maciej Hirsz](github.com/maciejhirsz)

```js
  /**
   * Symmetric private key encryption using secret (derived) key.
   * @param {buffer|string} plaintext Text to be encrypted.
   * @param {buffer|string} key Secret key.
   * @param {buffer|string} iv Initialization vector.
   * @param {string=} algo Encryption algorithm (default: constants.cipher).
   * @return {string} Base64 encrypted text.
   */
  encrypt: function (plaintext, key, iv, algo) {
    var cipher, ciphertext;

    if (plaintext.constructor === String) plaintext = str2buf(plaintext);
    if (key.constructor === String) key = str2buf(key);
    if (iv.constructor === String) iv = str2buf(iv);

    cipher = crypto.createCipheriv(algo || this.constants.cipher, key, iv);
    ciphertext = cipher.update(plaintext.toString("hex"), "hex", "base64");

    return ciphertext + cipher.final("base64");
  },
 ```
 
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

## <a id="heading-3.2"/> General Findings

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
