# Prerequisites

## Technologies
To implement SEA, each of the two involved parties must be able to support certain technologies:
* JSON encoding/decoding
* AES-256-CBC encryption/decryption
* ECDH using the X25519 curve
* SHA-1 hashing
* OpenSSL signature generation (service) and verification (client)

[//]: # (TODO check the curve)

## Service Verification
To support service verification, the [Service Verification Pair](Terms-Entities.md#service-verification-pair) OpenSSL pair must be generated – a Service Verification Secret Key (SVSK) and Service Verification Public Key (SVPK). The secret should be stored only on the service, and the public key shared with the client. SEA does not enforce whether this key pair should be unique to the client, or whether a single pair per service is acceptable. However, private keys should never be shared between services.