|TEP: ????|Asymmetric Key Client Authentication|
|-:|:---|
|__Layer:__|Index|
|__Authors:__|Cameron Garnham|
|__Status:__|Draft|
|__Discussion:__||
|__Type:__|Standards Track|
|__Created:__|2022-08-22|
|__Updated:__||
|__Resolution:__||

# Introduction

## Abstract
This document proposes the use of asymmetric cryptography for user accounts and their authentication.

## Copyright
This document is licensed under the 2-clause BSD license. 

## Motivation
Currently Torrust uses traditional password-username based authentication:

* For every user the server stores a username, salt, and hashed password.
* To login, the client sends their username and password to the server.
* The server looks up the username and gets the salt, salts and hashes the supplied password, and compares it with the hashed password on record.
* If matching, the client is authenticated as that user.

Asymmetric authentication has a number of advantages when compared to password authentication, (assuming the client software is honest):
* No need to send any secret data to the server. In contrast, (hashed and salted) passwords that have to be protected from leakage and misuse.
* Synchronization of user-accounts in a group of federated servers becomes much safer: Instead of sharing hashed-passwords between the servers, (or using some centralized authentication service), the servers only need to synchronise the public-keys, without the risk of leaking secert-data.

The disadvantage:
* The user no longer picks their own password, instead a private key is generated for the user's client software, this key is 32-bytes long and impossible to remember.  If the user looses this code, they loose access to their account.

This disadvantage is mitigated by providing the user with a 'backup seed' when they generate their private key, that the user can use to recover their private key.

# Design
We first describe the database and user-structure, and general protocols, then we specify clients generating new private keys, and finally describe the public key authentication protocol.

## Overview
New Users:
1. Client generates secret keys, and derives public key.
2. Client and server preform the Diffie-Hellman authentication protocol.
3. Client creates username, and the public key is associated the username.

Existing Users:
1. Client and server preform the Diffie-Hellman authentication protocol.
2. Client selects username that it wishes to login with.

(There may be more than one username that is associated with the same public key).

## Users
A username is the unique key in the database retaining to the actions of a user. Public keys are associated with users, for a particular purpose. Every public-key-purpose combination has a set of timed status.

```
[user]
(PK. String) username

[purposed_public_key]
(PK. public_key || purpose)
(Enum.) purpose
(Binary.) 32-byte-public_key

[status_timed]
(PK. Unique Hashed)
(Enum.) status
(Date-Time. Optional.) beginning
(Date-Time. Optional.) end

[user] many-many [purposed_public_key]

[user][purposed_public_key] many [status_timed]
```

_"Users and Purposed-Public-Keys have a many-many association, each association may associate many timed statuses.":_
- Each _user_ may look up what _purposed public keys_ it is associated with.</br>
Likewise, Each _purposed public key_ may look up what _users_ it is associated with.
- Each _(user <> purposed public key)_ link can look up it's associated _timed statuses_.

### Purposes
In this document we only user the `A` for "Authentication".</br>In the future we may propose additional purposes.
```
Purposes (enum):

A: Authentication
```

### Status Validity
If revocation does not specify a beginning, it is for all time periods.</br>
`Active` has the lowest priority: The key must be `created`, and not `expired`, and not `revoked`.
```
Status (enum):

A: Active
C: Created
E: Expired
R: Revoked
```
#### Status Validity Table:
|status|beginning|end|
|:-|:-|:-|
|active|must|must|
|created|must|none|
|expired|must|none|
|revoked|optional|none|

# Specification
The server maintains a list of public keys that may be used for authentication for each user. A user may propose a new public key be added to their list of authentication keys, or may revoke or expire an existing key.

* To add, or expire, or increase the active period a public key, the user must first authenticate with this new key to prove ownership.
* To revoke, the user doesn't need to authenticate with that key. (However some other form of checks should be preformed, such as verification by a moderator.)

Users who do not have any public key associated, for example new users, or when transitioning from password based authentication should first authenticate with their public key, then preform the association with the user account.

## Mnemonic Codes
Clients generate a mnemonic code according to: [BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki).

The seed words are prefixed with an additional `"torrust"` word, and displayed to the user. The 'torrust' seed word is not included in the checksum calculation, however must be present when the user enters their seed in recovery.

Clients must provide an option for the user to give a seed-password in generation and recovery.

* The client should only need their seed mnemonic code in the case they have lost their other login credentials.

## Public and Private Key Derivation
Deriving the keys according to: [BIP-32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).

We use the following hardened derivation path:</br>The index is increased in the case the user has rotated their keys, for the moment we only use zero:
```
m / purpose' / version' / torrust-purpose' / index'

m / "torrust" / 0 / "authentication" / 0
```
The resulting node is a private-public secp256k1 key-pair.

## Login Credential Encoding
For encoding the secp256k1, we use [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) encoding, 32-bytes for public keys.</br>
For displaying the login credentials, we use [BIP-350](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki), 'Bech32m' encoding.

We use the follwoing "Human Readable Parts", or "hrp":

|hrp|description|
|:-|:-|
|"torrust"|for public keys|
|"torrust_secret"|for secret keys|

Upon generation the Public and Private keys are displayed to the user.</br>
_The user should be encouraged to save them in their password manager._

```
Login ID:   bech32m("torrust",32-byte-public-key)
Login Code: bech32m("torauth",32-byte-private-key)
```

While technically only the private key is needed, we require both as the public key acts as checksum for the private key. This makes it more comfortable for users that are used to traditional password-username based authentication.

## Authentication
The server and client preform Elliptical Curve Diffie-Hellman over the secp256k1 curve to authenticate:

_We use the `secp256k1_ecdh` as defined in [secp256k1_ecdh.h](https://github.com/bitcoin-core/secp256k1/blob/master/include/secp256k1_ecdh.h), see example: [examples/ecdh.c](https://github.com/bitcoin-core/secp256k1/blob/master/examples/ecdh.c)._

### Prerequisites
The server and client generate temporary ephemeral key pairs for every connection attempt. They should not be reused.
```
Ephemeral Keys:
(E_server, e_server)
(E_client, e_client)

Client User Key:
(U_client, u_server)
```

### Setup
The server transfers their ephemeral public key to the client.</br>
The client transfers their ephemeral public key and user public key to the server.
```
Server tx to Client:
(E_server)

Client tx to Server:
(E_client, U_client)
```

### Key Derive
The server and client both derive common secret keys.
```
Server:
ECDH(E_client, e_server) -> c_ephemeral
ECDH(U_client, e_server) -> c_user

Client:
ECDH(E_server, e_client) -> c_ephemeral
ECDH(E_server, u_client) -> c_user

Note: The resulting keys should be identical.
```

### Client Auth Token
The client includes their URI to help guard against man-in-the-middle attacks.
```
Shared RFC 3986 URI: format: "proto://[user@]host[:port][/path]"

HASH(uri_client || c_ephemeral || c_user) -> token_auth
```

### Authentication
The URI should match the expected domain of the server.
```
Client tx to Server:
(token_auth, uri_client)

Server Verify:
uri_client
token_auth

If matching, authorise user according to their user public key. (U_client).
```

# Compatibility

The updated client will support both login in with the new authentication scheme, and transferring existing users with password authentication to the new scheme.

# Reference Implementations
