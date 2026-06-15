+++
title = "Implementing ChaCha20 in C"
date = 2026-06-15
[taxonomies]
tags = ["c", "crypto", "chillssh"]
+++
# Implementing ChaCha20 from Scratch in C — devblog

## Why ChaCha20?

Modern SSH connections use `chacha20-poly1305` as their AEAD cipher. ChaCha20 is the stream cipher half. fast in software, no hardware AES required, and fully specified in RFC 8439 with test vectors at every level. It was the obvious first crypto primitive to implement.

---

## The State Is Just 16 Words

A ChaCha20 instance is 16 `uint32_t` values — 512 bits — arranged as a 4×4 matrix:

```
constant  constant  constant  constant   // "expand 32-byte k"
key       key       key       key
key       key       key       key
counter   nonce     nonce     nonce
```

The constants are the ASCII string `"expand 32-byte k"` split into four little-endian 32-bit values.

---

## Everything Is ARX

The entire cipher reduces to one operation repeated 80 times: the quarter round.

```c
s[a] += s[b]; s[d] ^= s[a]; s[d] = ROTL32(s[d], 16);
s[c] += s[d]; s[b] ^= s[c]; s[b] = ROTL32(s[b], 12);
s[a] += s[b]; s[d] ^= s[a]; s[d] = ROTL32(s[d],  8);
s[c] += s[d]; s[b] ^= s[c]; s[b] = ROTL32(s[b],  7);
```

Add, Rotate, XOR. that's it. The rotation amounts (16, 12, 8, 7) are chosen to maximize diffusion across the 32-bit words.

---

## The Add-Back Is the Clever Part

After 20 rounds of mixing, the block function adds the original state back into the scrambled working copy word-by-word. That final addition is what makes ChaCha20 a secure. You can't reverse the 20 rounds to recover the key, because both the scrambled state and the original are baked into the output.

---

## Scrubbing Keystream from Memory

After each block is XOR'd into the output, the keystream buffer gets zeroed through a `volatile` pointer:

```c
static void chacha20_scrub(void *p, size_t n) {
    volatile uint8_t *vp = (volatile uint8_t *)p;
    for (size_t i = 0; i < n; i++) vp[i] = 0;
}
```

Without `volatile`, the compiler sees a dead write and optimizes it away. Keystream bytes are sensitive and this is the right habit to build early.

---

## Test Vectors All the Way Down

RFC 8439 ships test vectors at every level: quarter round, full state, block output, and a complete encryption. I test all four with `assert()` at startup.

---

Next up: Poly1305, the MAC half of `chacha20-poly1305`. Once that's done I can wire up the full AEAD construction.
