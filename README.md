# 5G New Authentication Version - Tamarin Protocol Verification

This repository contains a formal security analysis of a 5G authentication protocol using the **Tamarin** symbolic verification tool.

## Overview

This protocol defines a secure authentication mechanism for 5G networks, involving:
- **UE (User Equipment)**: The device being authenticated
- **gNB (gNodeB)**: The base station
- **HN (Home Network/HSS)**: The backend authentication server

The protocol aims to prove mutual authentication and protection against various attacks including replay attacks, forgery attacks, and key compromise scenarios.

## File Structure

- `5G_New_Authentication_Version.spthy`: Main Tamarin protocol specification file

## Key Components

### 1. **Builtins and Functions**

The protocol uses cryptographic primitives:
- **Diffie-Hellman (DH)**: For key agreement
- **Digital Signatures**: For message authentication
- **Hashing**: For data integrity

Custom functions defined:
- `kdf1/2`, `kdf2/2`: Key Derivation Functions
- `hkdf_sha256/1`: HMAC-based Key Derivation Function
- `hmac/2`: HMAC for message authentication codes
- `aes_enc/3`, `aes_dec/3`: AES encryption/decryption with ICV (Initialization Vector)

### 2. **Infrastructure Rules**

#### `Register_HN_Keys`
- Home Network generates and registers its long-term signing key (`skHN`)
- Publishes its public key for use by other parties

#### `Register_UE_HN_Shared`
- Establishes a pre-shared secret key (`k`) between UE and HN
- Associates it with a SUPI (Subscription Permanent Identifier)
- Stores the key in a database for later verification

#### `Reveal_K`
- Models a key compromise scenario where an attacker can learn the shared key
- Used for proving security even if keys are compromised

#### `UE_Initiates`
- UE starts the protocol by sending an identifier

### 3. **Protocol Execution Rules**

#### `gNB_Sends_Nonce`
- Base station generates a fresh random nonce (`ngNB`)
- Stores it locally and sends it to the UE along with the original ID

#### `UE_Registration_Request`
The core protocol step where the UE:
1. Generates a fresh Diffie-Hellman ephemeral key (`~ke`)
2. Computes the DH shared secret: `Z = pkHN^~ke`
3. Derives encryption and MAC keys from Z:
   - `Kenc = kdf1(Z, Re)`
   - `ICB = kdf2(Z, Re)`
   - `KMAC = hkdf_sha256(~k)` (using pre-shared key)
4. Creates a MAC of the SUPI: `M = hmac(KMAC, ~supi)`
5. Encrypts the SUPI and MAC: `C = aes_enc(Kenc, ICB, <~supi, M>)`
6. Signs the ciphertext and nonce: `sig = sign(<C, ngNB>, ~ke)`
7. Sends the ephemeral public key, ciphertext, signature, and ID to HN

#### `AlphaBox_Verify`
- An intermediate verification step (representing a security module)
- Verifies the signature on the received message
- Ensures the nonce matches what was sent
- Prevents replay attacks by checking the same (id, ngNB) pair only once

#### `HN_Deconceal`
- Home Network decrypts the message using its private key
- Recovers the original SUPI and verifies the HMAC
- Confirms successful authentication by checking the decrypted message matches the expected format

### 4. **Security Lemmas**

#### `executable`
Proves the protocol can run to completion without contradictions.

#### `alphabox_no_replay`
Ensures that AlphaBox rejects repeated uses of the same nonce-ID pair, preventing replay attacks.

#### `alphabox_rejects_invalid_signatures`
Verifies that messages with invalid signatures (using wrong keys or tampered messages) are rejected.

#### `hn_rejects_k_forgery`
Ensures that:
- The Home Network only accepts messages from legitimate UEs that know the correct shared key
- If a UE's key is compromised (`Reveal(k)`), the lemma allows security to be broken (explicit assumption)
- Without key compromise, the UE must have actually participated in the protocol

## Protocol Flow

```
UE                          gNB                          HN
|                            |                            |
|------ Send ID ------------->|                            |
|                            |                            |
|<---- ID + Nonce -----------|                            |
|                            |                            |
|--- Encrypted SUPI + Sig ----|---- Forward Message ----->|
|   (signed with DH key)       |  (verify signature)       |
|                            |                            |
|                            |<--- Decrypt & Verify -------|
|                            |    HMAC & SUPI             |
```

## Security Properties Verified

1. **Freshness**: The same nonce cannot be verified twice
2. **Signature Integrity**: Only messages signed with the correct key are accepted
3. **Mutual Authentication**: The HN confirms the UE possesses the correct shared key
4. **Replay Protection**: The AlphaBox module ensures nonces are not reused
5. **Key Confidentiality**: The encrypted SUPI protects user identity

## Usage

To verify this protocol with Tamarin:

```bash
tamarin-prover interactive 5G_New_Authentication_Version.spthy
```

Or to check all lemmas in batch mode:

```bash
tamarin-prover 5G_New_Authentication_Version.spthy
```

## Technical Details

- **Framework**: Tamarin Prover v1.6+
- **Logic**: Symbolic security verification with unbounded message analysis
- **Threat Model**: Dolev-Yao adversary (can intercept, modify, and replay messages)
- **Key Assumptions**: Perfect cryptography (no computational attacks modeled)

## Notes

- The protocol assumes secure channels for the pre-shared key establishment between UE and HN
- The `Reveal_K` rule allows modeling key compromise to test security under that assumption
- AlphaBox acts as a trusted verification module, potentially representing a security gateway or HSS module
# tamarin-5g-aka
