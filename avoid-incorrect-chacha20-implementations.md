from https://insanecoding.blogspot.com/2014/06/avoid-incorrect-chacha20-implementations.html

## Sunday, June 22, 2014

---

### Avoid incorrect ChaCha20 implementations

Posted by [insane coder](https://www.blogger.com/profile/06901386115570670209 "author profile") at [Sunday, June 22, 2014](https://insanecoding.blogspot.com/2014/06/avoid-incorrect-chacha20-implementations.html "permanent link")

---



[ChaCha20](http://cr.yp.to/chacha.html) is a [stream cipher](http://en.wikipedia.org/wiki/Stream_cipher) which is gaining a lot of popularity of late. Practically every library today which provides ciphers seems to have it as an addition in their latest releases.

In cryptography, there are two kinds of ciphers, [block ciphers](http://en.wikipedia.org/wiki/Block_cipher) and stream ciphers. Block ciphers are where the underlying algorithm works with data with a certain fixed chunk size (or block). Popular blocks sizes are 16 and 64 bytes. Stream ciphers are effectively block ciphers where the chunk size is a single byte.

Classical stream ciphers, such as [RC4](http://en.wikipedia.org/wiki/RC4), can work with data of arbitrary size, although every single byte is dependent on every previous byte. Which means encryption/decryption cannot begin in the middle of some data, and maintain compatibility where some other starting point was used. Block ciphers generally can have their blocks encrypted and decrypted arbitrarily, with none dependent upon any other, however, they cannot work with data of arbitrary size.

In order to allow block ciphers to work with data of arbitrary size, one needs to [pad the data](http://en.wikipedia.org/wiki/Padding_%28cryptography%29#Block_cipher_mode_of_operation) to be encrypted to a multiple of the block size. However, a clever alternative is [counter mode](http://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_.28CTR.29).

Different modes for working with block ciphers exist. Some try to improve security by making each block depend on every other, some utilize various interesting techniques for other properties.  Counter mode does not encrypt the desired data (the _plaintext)_ directly, rather, an ever incrementing counter is encrypted. The result of this encryption is then [xored](http://en.wikipedia.org/wiki/XOR_cipher) with the desired data.

Counter mode effectively turns a block cipher into a stream cipher, as the _plaintext_ is never actually passed to the block cipher. Rather, a counter which is a multiple of the block size is used. One can always xor bytes with an arbitrary size, and since that is the only step in counter mode against the plain text, it is effectively a stream cipher. Since the underlying cipher can be a block cipher with no dependency between blocks, this kind of stream cipher also allows one to jump ahead to any particular multiple of the block size in the data, and begin encryption/decryption from there.

Now while ChaCha20 is advertised as a stream cipher, it's actually designed as a block cipher in counter mode. The internal design mostly mirrors that of typical counter mode design, except that the counter components are directly fused with a large but simple block cipher. Since it's really a block cipher, it has an internal block size, and also allows one to jump ahead to some multiple of it.

Since ChaCha20 is considered to have a great level of security, and all these other wonderful properties, it's starting to see a lot of use. However, practically every implementation I'm seeing is either utterly broken, or has some ridiculous API.

Common ChaCha20 implementation mistakes:

- Implemented as a typical block cipher, not allowing usage with arbitrary amounts of bytes, or worse, the API allows for it, but produces incorrect results.
- Implemented as a typical stream cipher with no way to jump ahead.
- Failing on [Big-Endian](http://en.wikipedia.org/wiki/Endianness) systems.


The first mistake I listed is the most common. If some software is only using ChaCha20 internally, and always using it in a multiple of its block size (or it's all the crummy API offers), then things are fine. But if it's a library which is inviting others to use it, and it can be used incorrectly, expect disaster to ensue.

The reference implementation of ChaCha20 was designed that an arbitrary amount of data can be encrypted, as long as all but the last usage of the API was a multiple of the block size. This was also mentioned in its documentation. However, practically every other implementation out there copies this design in some way, but makes no note of it. Worse yet, some libraries are offering ChaCha20 with this implementation flaw alongside other stream ciphers with an identical API whereas those can be used arbitrarily throughout.

Essentially, this means if you're using ChaCha20 right now in a continuous fashion with chunks of various sizes, your data is being encrypted incorrectly, and won't be interoperable with other implementations. These broken implementations are able to output exactly one chunk correctly which is not a multiple of the block size, which destroys their internal buffers, and screws up every output thereafter.

I noticed a [similar situation](http://insanecoding.blogspot.com/2007/05/hashing-out-hashing-and-hash-libraries.html) with hash algorithm implementations several years back. However, most hash implementations are fine. Yet with ChaCha20, practically every implementation I looked at recently was broken.

Since this situation cannot stand, especially with ChaCha20 gaining speed, I am providing a _**[simple implementation](http://chacha20.insanecoding.org/)**_ without these flaws. This implementation is designed to be correct, portable, and simple. (Those wanting an optimized version of this should consider paying for more optimized routines)

Usage of the [C99](http://en.wikipedia.org/wiki/C99) API I designed is as follows:

Custom type: _chacha20\_ctx_
This type is used as a context for a state of encryption.

To initialize:

_void_ chacha20\_setup(_chacha20\_ctx \*_ctx, _const uint8\_t \*_key, _size\_t_ length, _uint8\_t_ nonce\[8\]);


The encryption key is passed via a pointer to a byte array and its length in bytes. The key can be 16 or 32 bytes. The [nonce](http://en.wikipedia.org/wiki/Cryptographic_nonce) is always 8 bytes.

Once initialized, to encrypt data:

_void_ chacha20\_encrypt(_chacha20\_ctx \*_ctx, _const uint8\_t \*_in, _uint8\_t \*_out, _size\_t_ length);


You can pass an arbitrary amount of data to be encrypted, just ensure the output buffer is always at least as large as the input buffer. This function can be called repeatedly, and it doesn't matter what was done with it previously.

To decrypt data, initialize, and then call the decryption function:

_void_ chacha20\_decrypt(_chacha20\_ctx \*_ctx, _const uint8\_t \*_in, _uint8\_t \*_out, _size\_t_ length);


For encryption or decryption, if you want to jump ahead to a particular block:

_void_ chacha20\_counter\_set(_chacha20\_ctx \*_ctx, _uint64\_t_ counter);


Counter is essentially the number of the next block to encrypt/decrypt. ChaCha20's internal block size is 64 bytes, so to calculate how many bytes are skipped by a particular counter value, multiply it by 64.

In addition to just providing a library, I gathered the test vectors that were submitted for various [RFCs](http://en.wikipedia.org/wiki/Request_for_Comments), and included a series of unit tests to test it for correctness.

For fun, since I'm also playing around a bit with [LibreSSL](http://insanecoding.blogspot.com/2014/04/libressl-good-and-bad.html) these days, I wrapped its API up in the API I described above. The wrapper is included in my package with the rest of the code, however it is currently not designed for serious usage outside of the included test cases.

Since I already whipped up some unit tests that anyone can use, I'll leave it as an exercise to the reader to determine which libraries are and aren't implemented correctly.

I tried to ensure my library is bug free, but I am only human. If you find a mistake, please report it.
