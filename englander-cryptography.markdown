Cryptography In Depth
---------------------

_June 2, 2017 - Adam Englander_

### Introduction

Adam is a Senior Engineer at iovation, a security and fraud company.

"We use cryptography to obscure data in such a way that it is difficult, and
therefore costly, for an adversary to duplicate and reverse."

- **How Difficult is Difficult?**

_Algorithmic complexity and computational expense_

In the past, MD5 and SHA1 were sufficient; today, it is more efficient to
calculate an MD5 hash than to use a rainbow table.

- **How Random is Random?** (Entropy)
- **How Large is Large?** (Key Size)
- **How Secret is Secret?** (Protecting Crypto Secrets)

All of these questions are important, and security is inevitably about
compromises.  There is a give-and-take between usability, cost, and access.
How much response time is acceptable?  How about interoperability?  What
setup in PHP is commonly available to users and will be widely adopted?

Make sure to factor in the cost of a breach due to poor security.  How many
users will you lose?  Would it cause your business to be devalued or to fail
completely?

### How Difficult is Difficult?

Difficulty often refers to a complex algorithm and associated elements that
make reverse-engineering expensive.  Difficulty can also refer to duplication
cost for brute force (mainly for password hashing).  Libraries like bcrypt and
argon2 continue to be developed to beat the hash-breakers.

### How Random is Random?

Entropy is the measure of randomness in the system (this has to do with common
initialization vectors).  This is increased via the use of randomized keys; 
random init vectors / salts / nonces; modes (as in AES) to reduce the
determinism inherent in the encryption; and additional random padding of
values.

[Good Entropy / Bad Entropy example: Tux and his encrypted image]

### How Large is Large?

In cryptography, it's usually YUGE.  Larger keys increase the computational
power (and often the memory) necessary to perform the same algorithm.  They
increase the time / power / cost to brute force.  Larger hash sizes reduce
the potential for _collision_ (duplication).  In general, use the largest size
that will work for your application stack / business needs.

### How Secret is Secret?

Asymmetric encryption is intrinsically more secure than symmetric encryption
because the parties involved do not possess the same information.  Neither party
should be required to trust one another with their own secret information.

### Libraries

Which library should I use?  Which elements are better than others?

- _Forget_ mcrypt
- OpenSSL
    - RSA for asymmetric encryption; **only use PKCS1 OAEP padding**
      (OPENSSL_PKCS1_OAEP_PADDING), key sizes >= 2048
    - AES for symmetric encryption with CBC modes, key size 256 bit. (minimum)
    - PBKDF2 can be used as a last resort, many better options available
- Cryptography-Hash
    - hash for digests (sha256, sha384, sha512)
    - hash_hmac for HMACs
    - hash_pbkdf2 as a last resort
- libsodium
    - AES-CGM or ChaCha20-Poly1305 for symmetric encryption including auth tag
    - XSalsa20-Poly1305 for asymmetric encryption
    - Ed25519 for asym digital signatures
    - Blake2b for hashing
    - Argon2 and Scrypt KDFs for password hashing
- Password Hashing library
    - Provides easy access to proper KDFs
    - Allows for determining if a password hash needs upgrading
    - Available from PHP 5.5 on
    - BCRYPT at the moment, but Argon2 in PHP 7.2
    - Stick with PASSWORD_DEFAULT for the algorithm
- CSPRNG
    - Built into PHP 7
    - paragonie/random_compat as polyfill
    - Generates properly pseudorandom ints and bytes
    - **JUST USE IT**

### Preferred Algorithms, Modes, and Sizes

- Encryption - _prefer asymmetric_
    1. XSalsa20-Poly1305 (libsodium)
    2. RSA-OAEP 2048+ key (OpenSSL)
    3. ChaCha20-Poly1305 (libsodium)
    4. AES256-GCM (libsodium)
    5. AES256-CBC (OpenSSL)
- Digital Signatures
    1. Ed25519 (libsodium)
    2. RSA/SHA(512/384/256) (OpenSSL)
    3. HMAC/SHA(ditto above) (Cryptography-Hashing)
- Key Derivation Functions (KDFs)
    1. Argon2i (Password - currently better than libsodium)
    2. Argon2 (libsodium)
    3. Scrypt (libsodium)
    4. Bcrypt (Password)
    5. PBKDF2 (OpenSSL / Cryptography-Hashing)
- Hashing
    1. Blake2b (libsodium)
    2. SHA2/MD5 (Cryptography-Hashing)
    3. SHA2-512 (Cryptography-Hashing)
    4. SHA2-384 (Cryptography-Hashing)
    5. SHA2-256 (Cryptography-Hashing)

### Resources

[https://en.wikipedia.org/wiki/Encryption](https://en.wikipedia.org/wiki/Encryption)
[https://download.libsodium.org/doc/](https://download.libsodium.org/doc/)
[https://wiki.php.net/rfc/argon2_password_hash](https://wiki.php.net/rfc/argon2_password_hash)
[http://php.net/manual/en/refs/crypto.php](http://php.net/manual/en/refs/crypto.php)
[http://csrc.nist.gov/groups/ST/toolkit](http://csrc.nist.gov/groups/ST/toolkit)

### Questions

_When working with multiple algorithms, is the hash only as strong as the weakest
link?_

Adam is talking about storing both.

_Are you concerned that switching from BCrypt to Argon2i might overflow the DB
field size?_

A good question we don't have an answer for yet (check the RFC).

_What is a correct field size to hold a password hash?_

255 characters is a good choice (via Scott A).
