# 📦 Tc3_Blake3 Library

A Blake3 Hash calculation library written in Structured Text (ST)
---

## ✅ Features
Based upon official BLAKE3 reference implementation, https://github.com/BLAKE3-team/BLAKE3

Specifically https://github.com/oconnor663/blake3_reference_impl_c by Jack O'Connor

Supports the three defined use cases
* Scenario 1: Basic hashing (no key, no derive key)
* Scenario 2: Keyed hashing
* Scenario 3: Derive key

Scenarios are tested with the official test vectors published here https://github.com/BLAKE3-team/BLAKE3/blob/master/test_vectors/test_vectors.json in the unit test program included.


## 🧠 Remarks and limitations

- The maximum number of blocks has been limited to 32bit
  - It does not really make sense for industrial controllers because of their internal limitations / use cases
  - It would limit the library to 64bit targets
- Compiled with TwinCAT 4024.67 but tested on TwinCAT 4026.17
- No parallellization implemented so expect no drastic better performance compared to other hashing algorithms
- TO BE INVESTIGATED: speed improvements could be achieved by using TwinCAT job tasks
- Will disturbe the realtime behavior of the task if too large blobs are hashed, monitor for task overruns


## 📦 Dependencies
* This library relies on [Tc3_AlgoInterfaces](https://www.github.com/kimro/Tc3_AlgoInterfaces) for the use of the generic IDigest interface.


## 🧪 Example

```iecst
VAR
    fbBlake3    : BLAKE3();
    dataShort	: ARRAY[0..14] OF BYTE := [72, 101, 108, 108, 111, 44, 32, 66, 76, 65, 
                                           75, 69, 51, 33, 0]; // "Hello, BLAKE3!" 
    dataLong	: STRING[2000] := 'some very long string';
    dataLong1	: STRING[2000] := 'some very ';
    dataLong2	: STRING[2000] := 'long string';
    result      : XWORD;
END_VAR

// Scenario 1: Basic hashing (no key, no derive key)
// Initialize, hash and finalize
hasher.init();
hasher.update(ADR(dataShort), len2(ADR(dataShort)));
hasher.finalize(hasher.self, hash);

// Initialize, hash and finalize
hasher.init();
hasher.update(ADR(dataLong), len2(ADR(dataLong)));
hasher.finalize(hasher.self, hash);

hasher.init();
hasher.update(ADR(dataLong01), len2(ADR(dataLong01)));
hasher.update(ADR(dataLong02), len2(ADR(dataLong02)));
hasher.finalize(hasher.self, hash);

// Scenario 2: Keyed hashing
// see RefVectorTests
hasher.init_keyed(Blake3TestKeyHex);
hasher.update(ADR(Blake3TestInput), Blake3TestVectors[i].input_len);
hasher.finalize(hasher.self, hash);

// Scenario 3: Derive key
// see RefVectorTests
hasher.init_derive_key(Blake3ContextString);
hasher.update(ADR(Blake3TestInput), Blake3TestVectors[i].input_len);
hasher.finalize(hasher.self, hash);
```

## 🔢 Understanding the BLAKE3 Algorithm

BLAKE3 is a modern cryptographic hash function known for its speed and parallelism. It builds upon the BLAKE2 algorithm and introduces a Merkel-tree-like structure (specifically, a "Bao tree") to achieve its high performance, especially on multi-core processors.

Here's a step-by-step explanation of how the BLAKE3 algorithm works:

### 1. Input Segmentation (Chunking)

The first step is to break the entire input message into fixed-size chunks of 1024 bytes (1 KiB). The last chunk might be shorter than 1024 bytes, but it cannot be empty unless the entire input is empty.

### 2. Binary Tree Structure (Bao Tree)

These 1024-byte chunks become the leaf nodes of a binary tree. BLAKE3 processes these chunks in a highly parallel fashion. Each chunk is processed independently.

The tree structure is built bottom-up:

* **Leaf Node Hashing:** Each 1 KiB chunk is fed into a modified BLAKE2s-based compression function. This function produces a 32-byte (256-bit) chaining value (hash output) for that specific chunk. This process involves multiple rounds (7 rounds in BLAKE3, compared to 10 in BLAKE2s) of internal permutations and mixing operations on a 512-bit internal state.
* **Parent Node Hashing:** Once two child nodes (either two hashed chunks or two hashed parent nodes) are processed, their 32-byte chaining values are combined and fed into the same compression function to produce the 32-byte chaining value for their parent node. This continues recursively up the tree.
* **Tree Structure Rules:**
    * Left subtrees are full (a complete binary tree with chunks at the same depth and a power-of-2 number of chunks).
    * Left subtrees are "big" (contain more chunks than or equal to their right sibling). This ensures a predictable tree structure.

### 3. Compression Function Details

The core of BLAKE3's hashing is its compression function, which takes several inputs:

* **Chaining Value (previous hash):** An 8-word (256-bit) input chaining value. For leaf nodes, this starts with an initial vector (IV). For parent nodes, it's the combined output of its children.
* **Message Block:** A 16-word (512-bit) message block (a portion of the 1 KiB chunk or the combined child hashes).
* **Counter (t):** A 64-bit counter that helps in domain separation and preventing identical hashes for identical blocks at different positions.
* **Block Length (b):** The number of input bytes in the current block (32 bits).
* **Flags (d):** A set of domain separation flags (32 bits) that indicate the type of input being processed (e.g., `CHUNK_START`, `CHUNK_END`, `PARENT`, `ROOT`). These flags are crucial for security, preventing certain attacks by ensuring different types of data produce distinct hashes.

The compression function proceeds in 7 rounds, each applying a series of "G" functions (quarter-rounds) to its 16-word internal state. These G functions involve additions, XORs, and bitwise rotations, mixing the state words with message words in a carefully designed pattern.

### 4. Root Node and Final Hash

This tree construction continues until a single root node is formed. The 32-byte chaining value produced by the compression of the root node (with a specific `ROOT` flag) is the default 32-byte BLAKE3 hash of the entire input message.

## Comparison Summary

| Feature           | Hashing (Standard)                   | Keyed Hashing (MAC/PRF)                | Key Derivation Function (KDF)               |
| :---------------- | :----------------------------------- | :------------------------------------- | :------------------------------------------ |
| **Primary Goal** | Data Integrity                       | Data Integrity & Authenticity          | Securely derive diverse keys from input     |
| **Input Key?** | No (uses fixed IV)                   | Yes (32-byte secret key)               | No (uses context and keying material)       |
| **Domain Separ.** | Implicit (via input)                 | Implicit (via key)                     | Explicit (via context string)               |
| **Output Type** | Message Digest / Arbitrary XOF       | Message Authentication Code / Arbitrary XOF | Derived Key(s) / Arbitrary XOF              |
| **Core Usage** | File checksums, digital signatures   | Authenticating messages, secure comms | Generating encryption/signing keys, session keys |
| **Flags Used** | `CHUNK_START`, `CHUNK_END`, `PARENT`, `ROOT` | `KEYED_HASH` (in addition to standard flags) | `DERIVE_KEY_CONTEXT`, `DERIVE_KEY_MATERIAL` (in addition to standard flags) |

## ⚖️ License-related information

Attribution to the work of Jack O'Connor (oconnor663).
This library is derived work based on https://github.com/oconnor663/blake3_reference_impl_c by Jack O'Connor that was released under CC0 1.0 Universal. 
