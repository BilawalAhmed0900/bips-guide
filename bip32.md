# BIP0032
This BIP refer to the standard for generation of high-determinitic (HD) wallet which can shared partially or entirely with different system with or without ability to spend coins. Refer to official documentation [here](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).

# Method
Generation of wallet address goes over the following steps

## Selection of a Random Number
Initially we select a random number of size between 128 and 512 bits. We can also select the binary seed derived from words from [BIP0039](bip39.md) if we want to derive address from those words. I have selected the same from the example over at BIP0039.

> 1d 87 7b 08 17 9a d7 7d bc cf b6 30 db a3 8e 86 f6 1f 8d 58 1d 31 3a 7f fe d1 f0 a3 a4 ef f4 50 21 77 ee 51 7d d9 df ed d9 b6 bb dc c3 ea 41 1a 21 73 d9 b4 6b 0c 63 4f 47 68 67 05 4c d5 d8 ec

## Generation of Master Key
We will use the the above random number and pass it through [HMAC](https://tools.ietf.org/html/rfc2104)-[SHA512](https://datatracker.ietf.org/doc/html/rfc6234) using `Bitcoin seed` (string as UTF-8) as key and that number as data. The result of above is

> 53 e9 b9 46 7d 4f 71 3e 32 37 c8 68 80 4f 10 34 87 bb 45 ae ac 1c 3d 98 1b ec d6 89 94 ca da a6 d2 cc 78 58 c7 33 4e 89 44 02 ae 55 52 63 0c b4 c0 c5 3e c6 77 b1 9a 71 dd a6 f1 23 4c ed ff 33

The first 32 bytes from the above generated will be our `Master Secret Key` and second 32 bytes as `Master Chain Code`

### Restrictions
There are few restriction from the `Master Secret Key`

 - It should not be all `0`
 - It should be less than a specific number  `n` (we will define it at a later stage)

If any one of the above fails, we will call our key as invalid and a new random number will be selected and the whole process will repeat. My `Master Secret Key` satisfies both of these conditions, so we can proceed.

### Naming Convention
The `Master Secret Key` is also called `Private Key` since it is private. We will derive child keys from it, they will also be private, until we derive public key from that.

## Derivation of Child Key
It takes two parameter, a pair of `Private Key` and `Chain Code` and an unsigned integer of 32-bit size (will be called `i` here). We will call the function `CKDpriv` same as the official document. So, its signature will be same too.

> CKDpriv((k_par, c_par), i) -> (K, C)

Where k_par is the private key and c_par is the chain code.

### Process
 - Generation of `I`
   - If `i` >= 2^31^
     - I = [HMAC](https://tools.ietf.org/html/rfc2104)-[SHA512](https://datatracker.ietf.org/doc/html/rfc6234) with `key = c_par` and data = `0x00 || k_par || i`
   - else
     - I = [HMAC](https://tools.ietf.org/html/rfc2104)-[SHA512](https://datatracker.ietf.org/doc/html/rfc6234) with `key = c_par` and data = `point(ser_p(k_par)) || i` (the function `ser_p`, and `point` will be defined at the later stage)
 - Split `I` into 2 32-byte sequences `Il` and `Ir`
 - Let K = `(Il + k_par) mod n` (n will be defined later)
 - Let C = c_par
 - return `(K, C)` where K will be the child private key and C will be the child chain code

#### Note
 - `||` means concatination two sequence. For example, 0x00 || 0x123456 is equal to 0x00123456, works with arbitrary length sequence
 - The addition `+` is done by first converting the byte-sequence to a big-integer and then adding. Conversion will be most-significant byte first.

## Derivation of Child Public Key
There are two ways to derive Public Key. From private key (same level) and from public parent key. We will go through only derivation from same level private key. Go through the documentation for the method.

The parameters given is (k, c), the private key and the chain code. The name for this function will be same as in the official documentation `N` so the signature is

> N((k, c)) -> (K, C)

- return `(point(k), c)`

## The Function `point`
This is usually the most time-consuming fuction and most-hard to implement you own. I recommend using a library for this. I will also show you the code for Openssl only for this one.

It uses an Elliptic-curve named `secp256k1` and multiplies the incoming point with its generator number (`G`) and defines the number  `n` (finally). 

>  n  = FFFFFFFF FFFFFFFF FFFFFFFF FFFFFFFE BAAEDCE6 AF48A03B BFD25E8C D0364141

The signature of the function will be
> point(k) -> K

### Process
  - return `k * G` (EC multiplication under secp256k1)

#### Openssl Code
Below is the C++ code for the above method using openssl API

    int point(const EC_GROUP* group,
	  const std::array<uint8_t, (SHA512_DIGEST_LENGTH >> 1u)>& p,
	  std::array<uint8_t, (SHA512_DIGEST_LENGTH >> 1u)>& output,
	  bool& is_y_odd)
	{
	  BIGNUM* p_num = BN_bin2bn(p.data(), p.size(), nullptr);
	  if (p_num == nullptr)
	  {
	    return -1;
	  }

	  EC_POINT* result_point = EC_POINT_new(group);
	  if (result_point == nullptr)
	  {
	    BN_free(p_num);
	    return -1;
	  }

	  int result_EC_POINT_mul = EC_POINT_mul(group, result_point, p_num, nullptr, nullptr, nullptr);
	  if (result_EC_POINT_mul != 1)
	  {
	    BN_free(p_num);
	    EC_POINT_free(result_point);
	    return -1;
	  }

	  BIGNUM* x = BN_new(), *y = BN_new();
	  if (x == nullptr || y == nullptr)
	  {
	    BN_free(p_num);
	    EC_POINT_free(result_point);

	    if (x) BN_free(x);
	    if (y) BN_free(y);
	    return -1;
	  }

	  int result_get_coordinates = EC_POINT_get_affine_coordinates(group, result_point, x, y, nullptr);
	  if (result_get_coordinates != 1)
	  {
	    BN_free(p_num);
	    EC_POINT_free(result_point);
	    BN_free(x);
	    BN_free(y);

	    return -1;
	  }

	  is_y_odd = BN_is_odd(y) == 1 ? true : false;
	  if (BN_num_bytes(x) > (SHA512_DIGEST_LENGTH >> 1u))
	  {
	    BN_free(p_num);
	    EC_POINT_free(result_point);
	    BN_free(x);
	    BN_free(y);

	    return -1;
	  }
	  BN_bn2bin(x, output.data());

	  BN_free(p_num);
	  EC_POINT_free(result_point);
	  BN_free(x);
	  BN_free(y);
	  return 0;
	}

`SHA512_DIGEST_LENGTH ` was used because we will mostly deal with SHA-512 and half of SHA-512 will be `SHA512_DIGEST_LENGTH  >> 1u` refered to as 256-bit integer.

The variable `group`  is initialized as
> group = EC_GROUP_new_by_curve_name(NID_secp256k1);


# Convention
You will deal with lot of m/1/2'/3'/4/... type convention while working near this BIP. We will usually keep deriving private child key until we need a public key. But it is upto the developer to do what he/she needs.

A single tick `'` above a name means `2^31 + number`. This is refered to as harderned-key. Refer to official documentation for more details.

The `m/1/2'/3'/4` , means to take a private key pair `(k, c)` as `m` and do the following

> CKDpriv(CKDpriv(CKDpriv(CKDpriv(m, 1), 2^31^ + 2),  2^31^ + 3), 4)