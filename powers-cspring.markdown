Let's Get Random: Under the Hood of PHP 7's CSPRNG
--------------------------------------------------

_June 2, 2017 - Sammy K. Powers_

### Introduction

Sammy Powers runs the PHP Round Table podcast and is a PHP core committer and
'Johnny Appleseed' for the PHP core community.  :)

"I don't consider myself an INFOSEC developer, but security is important to
all kinds of developers in all contexts."

### What is Random?

"That's so random." Is it really?

Example:

- A number between 1 and 100, both digits odd (37 very likely)

If no pattern emerges, it is difficult to figure out the next value. Otherwise,
we would call it **deterministic**.

"True Random" is non-deterministic (measuring atmospheric noise, number of
electrons coming off of radioactive material).

Computer architecture _is_ deterministic, therefore random numbers in the
context of computer hardware are practically impossible.  We call the numbers
we generate via computation 'pseudorandom' because the results can be
reproduced if we know the inputs and the algorithms that are executed.

Example: LCG (Linear Congruential Generator)

```
const LCG_M = 9;
const LCG_A = 2;
const LCG_C = 0;
// const LCG_SEED = 1;
define('LCG_SEED', time()); // is this now 'safe'?

{
    static $seed =  LCG_SEED;
    $seed = (LCG_A * $seed + LCG_C) % LCG_M;
    return $seed;
}

for ($x = 0; $x < 40; $X++) {  // nasty pattern emerges
    echo lcg() . ", ";
}
```

**PHP has its own LCG**

lcg_value() returns a value between 0 and 1, but is far too predictable.  Is
there a better PRNG?

mt_rand() (Mersenne Twister) is - or was - a better version of a PRNG, but it
had a bug and wasn't running the MT to spec.

In PHP 7.1 and beyond, rand() is an alias for a _fixed_ implementation of
mt_rand().

```
for ($x = 0; $x < 40; $X++) {  // is this OK? At 2e19937-1 it repeats.
    echo mt_rand() . ", ";
}

for ($x = 0; $x < 624; $X++) {   // Not repeating, within the range...
    echo mt_rand(0, 99) . ", ";  // But each state was fed from the state before
}                                // and at 624 repetitions it's broken

echo mt_rand(0, 99);
> 11
echo mt_rand(0, 99);
> 9
echo mt_rand(0, 99);
> 60

// Now, add mt_srand()
mt_srand(10);           // SEED: same input, same output
echo mt_rand(0,99);
> 21
echo mt_rand(0,99);
> 21
echo mt_rand(0,99);     // is this totally random? This is expected behavior.
> 21
```

### Now, Let's Look at CSRF Tokens

CSRF (Cross-site request forgery) tokens have to be unique and unpredictable in
order to prevent malicious requests.

```
$csrf = md5(mt_rand());
echo $csrf;
> [Output is a unique 32-character hash]

$mt_srand(42);   // NASTY
$csrf = md5(mt_rand());
echo $csrf;
> [Output is the same value every time]

// What if we let PHP seed this for us?
#define GENERATE_SEED() ( \
((zend_long) (time(0) * getpid())) ^ \
((zend_long) (1000000.0 * php_combined_lcg())) \
)
```

In the HTTP header, you can easily get the time from the server.  It is also
possible to get the PID if you're clever.  And the PHP LCG is predictable.
**So this is a bad idea.** (Bad idea jeans!)  Use UUIDs for uniques, by the way.

What can we use instead?

### CSPRNG - The Magical Unicorns

A CSPRNG (Cryptographically Secure) uses seeds that are RIDICULOUSLY hard to
guess.  In practice, they are impossible to predict.  These are the _only_
random numbers to use in cryptographic contexts.

**PHP Options:**

```
openssl_random_pseudo_bytes() // CAN produce the same values across processes
    // OpenSSL cannot fix the fork-safety problem...
    // Don't use RAND_bytes - use /dev/urandom, etc.

mcrypt_create_iv() // deprecated in PHP 7.1, will be REMOVED in 7.2
    // After 7.2, can only be installed via PECL
    // The C library used for encryption is unmaintained

/dev/urandom
    // Non-Windows, open_basedir can potentially break this
```

So there actually have not been safe options built-in to PHP up to this point.

**Enter CSPRNG in PHP 7**

```
random_int();
random_bytes();

bin2hex(random_bytes(16)); // produces a 32-character hexadecimal string
```

### Under the Hood

**TL/DR**:

- On Windows: CryptGenRandom
- On BSD: arc4random_buf()
- On Linux: getrandom(2) syscall
- Or read directly from /dev/urandom

If PHP 7 CSPRNG cannot find a 'true random' source, it will FAIL - Closed.

_Note that the last three all pull from /dev/urandom._

So what _is_ /dev/urandom?

/dev/urandom gathers 'environmental noise' from your system as it is running.
Things like device drivers, inter-keyboard timings, inter-interrupt timings,
and other non-deterministic events that are difficult for an outside observer
to measure.

On PHP 5.x this is still a problem.  If you cannot upgrade to 7, there is a
Polyfill out there:

```
paragonie/random_compat
```

And by the way, get to PHP 5.6 or better _now_, anything prior is definitely
insecure.

### TL/DR

- Never use LCG, MT, or any other PRNG in crypto contexts.
- Only use a CSPRNG like /dev/urandom
- In PHP use random_bytes() or random_int(), or the Polyfill on 5.x.

Check out NIST SP 800-90B (on Entropy sources for Random bit gen).

Also, there are now updated man pages for /dev/urandom.

### Questions

Why use bin2hex vs base64encode? (The latter generates non-URL-safe characters)

