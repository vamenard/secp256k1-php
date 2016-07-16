# secp256k1-php

[![Build Status](https://travis-ci.org/Bit-Wasp/secp256k1-php.svg?branch=master)](https://travis-ci.org/Bit-Wasp/secp256k1-php)
[![Join the chat at https://gitter.im/Bit-Wasp/secp256k1-php](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/Bit-Wasp/secp256k1-php?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

PHP bindings for https://github.com/bitcoin/secp256k1

Please note that the upstream library, [[libsecp256k1](https://github.com/bitcoin/secp256k1)] is considered 
experimental by it's authors, and has not yet been formally released. For this reason, it's use should be 
discouraged. For consensus systems this warning is critical.

The library supports the EcDH, Schnorr, and signature recovery modules - these libraries are required for installation.

### Requirements
PHP 5.* versions are supported in the v0.0.x release.
PHP 7 is supported in the v0.1.x series. 

### About the extension
  - Runs against latest libsecp256k1 (until upstream produces versioned releases)
  - Tests are present for all currently added functions. The C library also has it's own tests, with some useful edge case tests, which will be ported soon.
  - This extension only supports deterministic signatures at present. In fact, no RNG is utilized in this extension - private keys must be generated elsewhere. 
  - The extension exposes the same raw API of libsecp256k1. As such, you must ensure you are passing the binary representations of each value.   
  - In keeping with libsecp256k1, this extension returns certain data to the user by writing to a variable reference, and returning a code indicating the failure/success.
  
### To Install:
```
    git clone git@github.com:Bit-Wasp/secp256k1-php
    git clone git@github.com:bitcoin/secp256k1
    cd secp256k1
    ./autogen.sh && ./configure --enable-experimental --enable-module-{schnorr,ecdh,recovery} && make && sudo make install
    cd ../secp256k1-php/secp256k1
    phpize && ./configure --with-secp256k1 && make && sudo make install
```

### (Optional) - Enable extension by default!
If you're a heavy user, you can add this line to your php.ini files for php-cli, apache2, or php-fpm. 
```
extension=secp256k1.so
```

### Run Benchmarks
```
    time php -dextension=secp256k1.so ../benchmark.php > /dev/null
```
Yes - it is FAST!

### Run Tests
```
    php -dextension=secp256k1.so vendor/bin/phpunit tests/
```

### Examples
#### Convert a private key to a public key.
`secp256k1_pubkey_create(string $privateKey32, integer $compressKey, string & $publicKey)` takes `$privateKey32`, 
and writes the public key to `$publicKey`. Depending on whether `$compressKey` is 0 or 1, the public key will be
uncompressed or compressed respectively. 

```php
$context = secp256k1_context_create(SECP256K1_CONTEXT_SIGN | SECP256K1_CONTEXT_VERIFY);
 
// Private keys are never generated by secp256k1. 
$privateKey = pack("H*", "abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789");

/** @var resource $publicKey */
$publicKey = '';
$result = secp256k1_ec_pubkey_create($context, $publicKey, $privateKey);
if ($result === 1) {
    $compress = true;

    $serialized = '';
    if (1 !== secp256k1_ec_pubkey_serialize($context, $serialized, $publicKey, $compress)) {
        throw new \Exception('secp256k1_ec_pubkey_serialize: failed to serialize public key');
    }
    
    echo bin2hex($serialized);
} else {
    throw new \Exception('secp256k1_pubkey_create: secret key was invalid');
}

``` 

#### Decompress a public key
This is carried out by parsing and reserializing the public key.

```php
$context = secp256k1_context_create(SECP256K1_CONTEXT_SIGN | SECP256K1_CONTEXT_VERIFY);
 
$publicKeyBin = pack("H*", '04ae1a62fe09c5f51b13905f07f06b99a2f7159b2225f374cd378d71302fa28414e7aab37397f554a7df5f142c21c1b7303b8a0626f1baded5c72a704f7e6cd84c');

/** @var resource $publicKey */
$publicKey = '';
if (1 !== secp256k1_ec_pubkey_parse($context, $publicKey, $publicKeyBin)) {
    throw new \RuntimeException('Failed to parse public key');
}

$decompressed = '';
secp256k1_ec_pubkey_serialize($context, $decompressed, $publicKey, true /* whether to compress */);

echo bin2hex($decompressed) . PHP_EOL;
```

#### Verify a private key
`secp256k1_seckey_verify(resource $context, string $privateKey)` validate the given `$privateKey` and returns 0 or 1 as the result.

```php
$context = secp256k1_context_create(SECP256K1_CONTEXT_SIGN | SECP256K1_CONTEXT_VERIFY);
// A compressed public key starts with 02/03. Compressed keys can be created from secp256k1_pubkey_create
$secretKey = pack("H*", "a34b99f22c790c4e36b2b3c2c35a36db06226e41c692fc82b8b56ac1c540c5bd");
if (secp256k1_ec_seckey_verify($context, $secretKey) !== 1) {
    throw new \Exception("Private key was invalid");
}

echo "Private key was valid\n";
```

#### Private Key tweak by addition
`secp256k1_ec_privkey_tweak_add(resource $context, string & $privateKey32, string $tweak32)` takes the given `$tweak` value and adds it to the private key. 
The result is written to the provided `$privateKey` memory location. 

This function is useful for deterministic key derivation. 
```php
$context = secp256k1_context_create(SECP256K1_CONTEXT_SIGN | SECP256K1_CONTEXT_VERIFY);

$privateKey = pack("H*", "88b59280e39997e49ebd47ecc9e3850faff5d7df1e2a22248c136cbdd0d60aae");
$tweak = pack("H*", "0000000000000000000000000000000000000000000000000000000000000001");

$result = secp256k1_ec_privkey_tweak_add($context, $privateKey, $tweak);
if ($result == 1) {
    echo sprintf("Tweaked private key: %s\n", bin2hex($privateKey));
} else {
    throw new \Exception("Invalid private key or augend value");
}
```

#### Public Key tweak by addition
`secp256k1_ec_pubkey_tweak_add(resource $context, resource $publicKey, string $tweak32)` takes the given `$tweak` value, converts it to a point, and
adds the point to the `$publicKey` point. The result is written to the provided $publicKey memory location.

This function is useful for deterministic key derivation. 
```php
$context = secp256k1_context_create(SECP256K1_CONTEXT_SIGN | SECP256K1_CONTEXT_VERIFY);

$publicKey = hex2bin("03fae8f5e64c9997749ef65c5db9f0ec3e121dc6901096c30da0f105a13212b6db");
$tweak = pack("H*", "0000000000000000000000000000000000000000000000000000000000000001");

$result = secp256k1_ec_pubkey_tweak_add($context, $publicKey, $tweak);
if ($result == 1) {
    echo sprintf("Tweaked public key: %s\n", bin2hex($publicKey));
} else {
    throw new \Exception("Invalid public key or augend value");
}
```

#### Public Key tweak by multiplication
`secp256k1_ec_pubkey_tweak_mul(resource $context, resource $publicKey, string $tweak32)` takes the given `$tweak` value, and
performs scalar multiplication of against the provided `$publicKey` point. Note - the length of the supplied public key must be supplied.

```php
$compressed = true;
$publicKeyBin = pack("H*", "03fae8f5e64c9997749ef65c5db9f0ec3e121dc6901096c30da0f105a13212b6db");
$tweak = pack("H*", "0000000000000000000000000000000000000000000000000000000000000002");

/** @var resource $publicKey */
$publicKey = '';
secp256k1_ec_pubkey_parse($context, $publicKey, $publicKeyBin);

$result = secp256k1_ec_pubkey_tweak_mul($context, $publicKey, $tweak);
if ($result == 1) {
    $serialized = '';
    secp256k1_ec_pubkey_serialize($context, $serialized, $publicKey, $compressed);
    echo bin2hex($serialized);
} else {
    throw new \Exception("Invalid public key or multiplicand value");
}
```

#### Sign a message
`secp256k1_ecdsa_sign(resource $context, resource $signature, string $msg32, string $privateKey)` takes the given `$msg32` and signs it with
`$privateKey`. If successful, the signature is written to the memory location of `$signature`. 

```php
$context = secp256k1_context_create(SECP256K1_CONTEXT_SIGN | SECP256K1_CONTEXT_VERIFY);

$msg32 = hash('sha256', 'this is a message!', true);
$privateKey = pack("H*", "88b59280e39997e49ebd47ecc9e3850faff5d7df1e2a22248c136cbdd0d60aae");
/** @var resource $signature */
$signature = '';

if (secp256k1_ecdsa_sign($context, $signature, $msg32, $privateKey) != 1) {
    throw new \Exception("Failed to create signature");
}

$serialized = '';
secp256k1_ecdsa_signature_serialize_der($context, $serialized, $signature);
echo sprintf("Produced signature: %s \n", bin2hex($serialized));
```

#### Verify a signature
`secp256k1_ecdsa_verify(resource $context, resource $signature, string $msg32, resource $publicKey)` verifies the provided `$msg32` and `$signature` against the provided `$publicKey`. Returns for valid signature, 0 for incorrect signature. 

```php
$context = secp256k1_context_create(SECP256K1_CONTEXT_SIGN | SECP256K1_CONTEXT_VERIFY);
$msg32 = hash('sha256', 'this is a message!', true);

$signatureRaw = pack("H*", "3044022055ef6953afd139d917d947ba7823ab5dfb9239ba8a26295a218cad88fb7299ef022057147cf4233ff3b87fa64d82a0b9a327e9b6d5d0070ab3f671b795934c4f2074");
$publicKeyRaw = pack("H*", '04fae8f5e64c9997749ef65c5db9f0ec3e121dc6901096c30da0f105a13212b6db4315e65a2d63cc667c034fac05cdb3c7bc1abfc2ad90f7f97321613f901758c9');

// Load up the public key from its bytes (into $publicKey):
/** @var resource $publicKey */
$publicKey = '';;
secp256k1_ec_pubkey_parse($context, $publicKey, $publicKeyRaw);

// Load up the signature from its bytes (into $signature):
/** @var resource $signature */
$signature = '';
secp256k1_ecdsa_signature_parse_der($context,$signature, $signatureRaw);

// Verify:
$result = secp256k1_ecdsa_verify($context, $signature, $msg32, $publicKey);
if ($result == 1) {
    echo "Signature was verified\n";
} else {
    echo "Signature was NOT VERIFIED\n";
}

```
