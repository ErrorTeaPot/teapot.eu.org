# Password-less LUKS is safer

Note : generative AI have been used to give advices on the content of the
article. No generated text have been used in the article.

## LUKS 101

LUKSv2 (Linux Unified Key Setup version 2) is the go-to standard for full-disk
encryption on Linux.
It ensures that the data stored on the disk stays confidential as long as you
don't have a valid key to decrypt it.

### Multi-layer approach

This technology uses a multi-layer hierarchical encryption approach that
combines a master key with user keys, that they can change if desired without
reencrypting the all drive.
The key that encrypts/decrypts the disk is the master key, which is itself
encrypted by user keys.

An encrypted disk will contain two things :

- A **unencrypted header** that contains metadata and the encrypted master key.
- The **encrypted data** on the rest of the disk

User keys are derived from passphrases, FIDO2 security keys, TPMs or other
authentication factors and stored in the keyslots in the LUKS header. There is
one copy of the encrypted master key stored per existing user key.
When a passphrase is asked to unlock the disk, providing a valid one will
decrypt the master key copy corresponding to the used user key.

### On the fly operations

Now that we have in mind how keys are overall managed (I am not a crypto expert)
, how are operations performed when the OS is running ?

First we need to understand what **block devices** are.
Quoting from [Wikipedia's device file page](https://en.wikipedia.org/wiki/Device_file#BLOCKDEV) :

> Block special files or block devices provide buffered access to hardware
> devices, and provide some abstraction from their specifics. [...] block
> devices will always allow the programmer to read or write a block of any size
> (including single characters/bytes) and any alignment.

So it is a provided abstraction that makes the interractions with hardware
easier.
Device files in general are interfaces exposed by the kernel that apperears as
ordinary files in a filesystem.

## TPM 101
