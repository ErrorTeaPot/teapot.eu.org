# Password-less LUKS is safer

Note : generative AI have been used to give advices on the content of the
article. No generated text have been used in the article.

## LUKS 101

LUKSv2 (Linux Unified Key Setup version 2) is the go-to standard for full-disk
encryption on Linux.
It ensures that the data stored on the disk stays confidential as long as you
don't have a valid key to decrypt it.

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

## TPM 101
