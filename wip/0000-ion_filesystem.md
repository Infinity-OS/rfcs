- Feature Name: ion_filesystem
- Start Date: 2017-01-12
- RFC PR: (leave this empty)
- Infinity Issue: (leave this empty)

# Summary
[summary]: #summary

iON is a next generation filesystem for flash devices, with support to snapshots, error correction , encryption, caching and more.

# Motivation
[motivation]: #motivation

With the rising of the new devices, way of use storage and the new types os storage we need a solution that can that advantage of the new types of storage, the flash memory.

iON is a modular, fast, and feature rich next-gen file system, built over the must modern techniques for high performance, high space efficiency, and high scalability. iON must be the default file system for the Infinity OS, inspired by ideas behind ZFS, but at the same time aims to be modular and easier to implement.

iON is designed with the following goals in mind:

## Modular

iON is highly modular, and is divided into various independent components. A significant amount of components are simple disk drivers without any semantic information. This makes iON relatively straight-forward to implement.

## Full-disk compression

iON is one of the first file systems to incorporate complete full--disk compression thought a scheme called RACC (random-access cluster compression). This means that every cluster is compressed only affecting performance slightly. Theoretical this will increases usable space in more than 50%.

## O(1) snapshots

Based in ZSF and on the new APFS (Apple File System), iON also allows full or partial disk revertible and writable snapshots in constant-time without clones or the alike.

## Copy-on-write semantics

Similar to Btrfs, ZSF and APFS, iON uses CoW semantics, meaning that no cluster is ever overwritten directly, but instead it is copied and written to a new cluster.

## Guaranteed atomicity

The system will never enter an inconsistent state (unless there is hardware failure), meaning that unexpected power-off at wort results is a 4KiB space leak. The system is never damaged by such shutdowns. The space can be recovered easily by running a _garbage collector_ tool.

## Improved caching

One of iON main focus is reduce the read times using a next generation cache to speed up the disk accesses. It uses a machine learning algorithm to learn patterns and predict future uses to reduce the number os cache misses.

## Concurrency

iON contains very few locks and aims to be as suitable for multithreaded systems as possible. It makes use of multiple truly concurrent structures to manage the data, and scales linearly by the number of cores.

## Better file monitoring

CoW is very suitable for high-performance, scalable file monitoring, but unfortunately only few file systems incorporate that. iON is one of those.

## All memory safe

One of the goals of making a new file system is make it safe, really safe. So for that iON only uses components written in Rust. As such, memory unsafety is only possible in code marked unsafe, which is checked extra carefully.

## Full coverage testing

iON aims to be full coverage with respect to testing. This gives relatively strong guarantees on correctness by instantly revealing large classes of bugs.

## SSD friendly

This new file system tries to avoid the write limitation in SSD by repositioning dead sectors.



Why are we doing this? What use cases does it support? What is the expected outcome?

# Detailed design
[design]: #detailed-design

In this part the iON file system design is presented in a such details that it can be implemented without ambiguity.

## Assumptions and guarantees

iON provides the two following guarantees. Unless data corruption happens, the disk should never be in an inconsistent state. iON archives this without using journaling or a transaction model. Poweroff or other unusual situation should not affect the system such that it enters an invalid or inconsistent state. And, any sector (assumed to be a power of two of at least 512 bytes) can be read and written atomically, i.e. it is never partially written, and interrupting the write will never render the disk in a state which it not already is written or retaining the old data.

## Disk header

The first 512 bytes are reserved for the "disk header" which contains unencrypted configuration and information about the state.

### Introducer (byte 0-64)

#### Magic number (byte 0-8)

The first 8 bytes are reserved for a magic number, which is used for determining if it does indeed store iON. It is specified to store the string "iON fmt " (note the space) in ASCII.

#### Version number (bytes 8-12)

This field stores a version number, in little-endian. By this revision, said number is 0.

> Breaking changes will increment the higher half of this number.

#### Bitwise negation version number (byte 12-16)

Repetition of the version number, bitwise negated. The reason for this negation is if one overwrites it with e.g. all zeros, it should be able to detect this.

### Encryption (byte 64-128)

#### Encryption algorithm (byte 64-66)

This field stores a number in little-endian defining encryption algorithm in use. It takes following values:

- `0`: No encryption (identity function)
- `1`: SPECK-128 (XEX mode) with scrypt key stretching (more bellow this is well described)

Any data not stored in the introducer is encrypted through the chosen method.

#### Bitwise negation of encryption algorithm (bytes 66-68)

Repetition of the "Encryption algorithm", bitwise negated.

#### Encryption parameters (byte 68-84)

There are parameters used differently based on the chosen encryption.

#### Bitwise negation of encryption parameters (byte 84-100)

Repetition of the "Encryption parameters", bitwise negated.

### State (byte 128-192)

#### State block address (byte 128-136)

This field stores a little-endian integer that takes the following values:

- `n=0`: state block uninitialized.
- `n≠0`: the `n`th cluster (starting at byte 512n) is used as the state block and abides the format set out in the "State block" section.

#### Bitwise negation of state block address (bytes 136-144)

Repletion of "State block address", bitwise negated.

#### State flag (byte 144)

This little-endian integer takes one of the following values:

- `0x00` - Disk I/O stream was properly closed.
- `0x01` - Disk I/O stream was not properly closed.
- `0x02` - The disk is in an inconsistent state.
- Any other value is considered invalid.

It takes two bytes instead of one to be able to detect error.

The `0x01` state does not make the state inconsistent, it simply serves to warn the user that the last writes might have been lost, e.g. the cache didn't flush.

The `0x02` state typically only happen if the user power-off the computer or remove the device while reformatting the disk.

## State block

At the address (cluster number) chosen in "State block address", a block defining the state of the file system is stored.

### Configuration

#### Checksum algorithm (byte 0-2)

This field stores a number in little-endian defining checksum algorithm in use.

- `0` - constant checksum as described in "Constant checksum"
- `1` - the SeaHash algorithm as described in "SeaHash"
- `≥2^15` - not used at the time of this specification

#### Compression algorithm (byte 2-4)

This field store a number in little-endian defining compression algorithm in use.

- `0` - No compression
- `1` - The LZ4 compressor as described in "LZ4"
- `≥2^15` - not used at the time of this specification

## State

#### Freelist head pointer (bytes 32-40)

This field store some number (in little-endian), which takes values:

- `n=0`: no free allocable cluster
- `n≠0`: the `n`th cluster is free and conforms to "Meta-cluster format"

#### Super-page pointer (byte 40-48)

This field stores some number (in little-endian), which takes values:

- `n=0`: super-page uninitialized
- `n≠0`: the `n`th page is the super-page as defined in "??"

### Integrity cheching

#### Checksum (byte 64-70)

This field stores a little-endian integer equal to the checksum of the state block preceding the checksum itself, calculated by the algorithm specified in "Checksum algorithm".

## Cluster management

### Cluster and pages

The disk is divided into clusters of 512 bytes each.

#### Data clusters

Data clusters has two bytes header in the start:

The **15 least-significant bits** is the **checksum** of the non-header data, stored in little-endian. The remaining **1-bit** is used to define the compression flag, if the bit is 1, the data following the header is compressed, otherwise, it read uncompressed.

A data cluster contain up to 256 of 510 bytes blocks called "pages". The decompressed data is the contained pages concatenated.

Pointers to pages uses the first 60 bits to define the cluster number, and the last four bits to define which page, it points to. In particular, if `c` is the cluster number and `p` is the number of the page in the cluster, then the pointer is given by `r = 256c+p`.

> If the cluster is uncompressed `p=0`

#### Meta-cluster format

The head of the freelist is a meta-cluster, which itself is a collection of other free clusters. It starts with a 8 byte checksum of the non-header part of the meta-cluster.

Following this, there is some number of 64-bit little-endian pointers to other free clusters.

If any, the first pointer must point to another meta-cluster. The rest of cluster is padded with zeros.

#### Allocation and deallocation

The algorithm for allocation and deallocation is implementation defined. It is generally done by inspecting the head of the freelist before popping it, to see if it has a sister cluster, where the page can be fit in. If it cannot, the freelist is popped. The way such clusters are paired is up to the implementation. Bijective maps are recommended for optimal performance.

### Checksums

#### Constant checksum

Constant checksum is independent of the input, yielding a contain value, `c=2^64-1`.

This is used as an alternative to storing no checksum, which would make implementation harder. Instead, this allows some – but weak – protection against failure with virtually no effort.

### SeaHash

???

## Compression algorithm

### LZ4

LZ4 compressed data is a series of blocks.

## References

- [LZ4](https://github.com/lz4/lz4)
- [SeaHash](https://github.com/ticki/tfs/tree/master/seahash)
- [SeaHash Explained](http://ticki.github.io/blog/seahash-explained/)
- [The SIMON and SPECK Families of Lightweight Block Ciphers](https://eprint.iacr.org/2013/404)
- [Cryptanalysis of the Speck Family of Block Ciphers](https://eprint.iacr.org/2013/568)
- [Improved Differential Cryptanalysis of Round-Reduced Speck](https://eprint.iacr.org/2014/320)

# Drawbacks
[drawbacks]: #drawbacks

This will take some years of refine and implementation, we could use an existent file system to reduce the planing time.

# Alternatives
[alternatives]: #alternatives

For start we could use an FAT32 implementation, just for testing, of course, and in parallel with the development of other components we could start developing this.

If we don't draw a new specification from the scratch the implementation can be hard, and we couldn't archive our main goal, security and performance.

# Unresolved questions
[unresolved]: #unresolved-questions

- Who the snapshots works?
- Can we implement multi-key encryption, which each file is encrypted with a separated key, and metadata encrypted with another one?
- Clones?! An mechanism that allow the OS make fast, power-efficient file copies on the same volume without occupying additional storage space. Modifications to the data write the new data elsewhere and continue to share the unmodified blocks. Changes to a file are saved as differences of the cloned file, reducing storage space required for document revisions and copies.
