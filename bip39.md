# BIP0039
This BIP refer to the standard for the generation of phrase. Refer to original guide for official description [here](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki).

# Method
Generation of phrases goes over the following steps

## Choosing Entropy Size
Following are the valid size of entropy in bits

 - 128 bits
 - 160 bits
 - 192 bits
 - 224 bits
 - 256 bits

## Generation of Random Entropy
After the selection of size, we generate random bits of the same size. For the sake of exampe, I will select 128 bits and the random entropy I got from [this site](https://www.random.org/bytes/) are (in hex):

    2d b3 a4 57 f1 d8 9b ce b9 42 ae 3f e2 cf b6 f5


## SHA256 of the Entropy
Then we will get [SHA256](https://datatracker.ietf.org/doc/html/rfc6234) of the above entropy. SHA256 of the above example is:

 > 73 75 8c 0c b8 c1 51 7a 4f bb 26 02 e9 a1 83 da 86 c7 33 e1 43 fe 30 ba 08 90 f6 7d 9c 86 f7 23

## Appending SHA256 to the Entropy
We will append `size of entropy bits / 32` bits of the SHA256 to the entropy. `128 / 32 = 4` bits will be appended to the entropy. Total number of bits are now 132. Here 4 bits are half a byte, so I am selecting the left nibble of the hash. If you select for example, selected 160 bits of entropy, you have to select 5 bits. For that, we use the following formula. `(byte >> (8 - num)) & ((1 << num) - 1)` where num is the number of bits to select.

 > 2d b3 a4 57 f1 d8 9b ce b9 42 ae 3f e2 cf b6 f5 7

## Splitting into Group of 11 Bits
To view this, we will first convert the bits from hex to binary form:

  > 00101101 10110011 10100100 01010111 11110001 11011000 10011011 11001110 10111001 01000010 10101110 00111111 11100010 11001111 10110110 11110101 0111

Splitted into group of 11 bits, we get

 > 00101101101 10011101001 00010101111 11100011101 10001001101 11100111010 11100101000 01010101110 00111111111 00010110011 11101101101 11101010111

The reason we did this that the [wordlist](https://github.com/bitcoin/bips/blob/master/bip-0039/bip-0039-wordlists.md) are exact 2048 words, and we got groups of 11 bits, `2^11 = 2048`.

## Selecting Language:
Wordlist are defined in many languages in the standard, you can select any one but we will go for the most common used, [English](https://github.com/bitcoin/bips/blob/master/bip-0039/english.txt).

## Converting Group of Bits to Words
We will use every group as an index to the wordlist (read in the same order) and use the replace those by the word found at that index.

    color outdoor bicycle together meadow trap topic fiction divide biology unit turtle

## Converting to Binary Form
The binary form of this will be used as seed while generating wallet address. We will select a pass phrase, which can be empty and prepend "mnemonic" to it. I will also be selecting empty pass phrase so, the prepended form will be "mnemonic". If you had selected "phrase" as pass phrase, prepended form would have been "mnemonicphrase".

We will then use the UTF-8 form of the above generated group of both of these.

 > 63 6f 6c 6f 72 20 6f 75 74 64 6f 6f 72 20 62 69 63 79 63 6c 65 20 74 6f 67 65 74 68 65 72 20 6d 65 61 64 6f 77 20 74 72 61 70 20 74 6f 70 69 63 20 66 69 63 74 69 6f 6e 20 64 69 76 69 64 65 20 62 69 6f 6c 6f 67 79 20 75 6e 69 74 20 74 75 72 74 6c 65

and

 > 6d 6e 65 6d 6f 6e 69 63

and pass them through [PBKDF2](https://www.ietf.org/rfc/rfc2898.txt) using `2048` iterations and [HMAC](https://tools.ietf.org/html/rfc2104)-[SHA512](https://datatracker.ietf.org/doc/html/rfc6234) using generated words as data and pass phrase as salt. 

The result is

  > 1d 87 7b 08 17 9a d7 7d bc cf b6 30 db a3 8e 86 f6 1f 8d 58 1d 31 3a 7f fe d1 f0 a3 a4 ef f4 50 21 77 ee 51 7d d9 df ed d9 b6 bb dc c3 ea 41 1a 21 73 d9 b4 6b 0c 63 4f 47 68 67 05 4c d5 d8 ec


# Conclusion
As a result we got a seed which will be used to derive private and public of a bitcoin wallet.

# P.S.
The above entropy and its pass-phrase is totally random.