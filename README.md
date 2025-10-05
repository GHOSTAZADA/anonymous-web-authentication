# **Anonymous Web Authentication (AWA) v1.0**
### *A Decentralized, Human-Centric Authentication and Authorization Protocol for the Modern Web*

---

## **Abstract**

Anonymous Web Authentication (AWA) is a deterministic, privacy-preserving web identity protocol designed to replace passwords, identity brokers, and rotating secrets with a single concept: **permanent proof of humanity without personal data**.  
AWA guarantees that every online service can reliably identify the *same human* across devices — without ever learning *who* that person is.

The protocol combines the assurance of **national eID systems** with the convenience of **local authenticators** (such as passkeys), producing a cryptographically verifiable, unlinkable pseudonym for each relying party (RP).  
AWA is built entirely on open web standards: WebCrypto, WebAuthn, and HTTPS.

---

## **1  Core Principle**

Each person possesses one **permanent, high-entropy seed** (`MASTER_SUB`) provisioned by a trusted identity authority (typically a national eID system).  
This seed is **not derived** from any personal data — it is a **random 1024-bit value**, created once (for example, at birth or at first eID issuance) and stored securely by national authorities under lawful audit conditions.

`MASTER_SUB`:
- Never leaves the user’s control unencrypted.  
- Has **no meaning outside AWA**.  
- Is used only to deterministically generate **pairwise pseudonyms** scoped to individual RPs (domains).

Every RP can thus assert *“this is the same human”* without knowing *which* human.

---

## **2  Motivation**

Conventional identity frameworks (OAuth, OIDC, SAML, FIDO2) expose personal data, require rotating credentials, or create friction.  
AWA eliminates these issues by ensuring:

1. **Permanent proof of humanity** via a stable national seed.  
2. **Deterministic trust** without rotation overhead.  
3. **Zero PII exchange.**  
4. **Client-driven decentralization** — no central identity broker.

---

## **3  Entities**

- **End User:** possesses `MASTER_SUB`.  
- **Authenticator:** browser / app storing the encrypted `MASTER_SUB` locally (e.g., IndexedDB + passkey encryption).  
- **Relying Party (RP):** any web domain implementing the `/session` endpoint.  
- **eID Provider (optional):** issues or renews the `MASTER_SUB`.

---

## **4  Data Model**

| Symbol | Description | Example |
|---------|--------------|----------|
| `MASTER_SUB` | Permanent 1024-bit random seed | `0xa34f...` |
| `PAIRWISE_SUB` | Derived per domain | `SHA-256(MASTER_SUB + TopDomain)` |
| `STATE` | One-time nonce for key lookup | Random 256-bit |
| `PK`,`SK` | RP-generated ephemeral key pair | 5 min TTL |
| `SESSION_COOKIE` | Encrypted session handle | `HTTPOnly; SameSite=Strict; Secure` |

---

## **5  Protocol Overview**

### **5.1 Initialization (Relying Party)**

1. RP generates three ephemeral values:  
    `STATE`, `PK`, `SK`.  
2. Stores `{STATE, SK}` temporarily (KV TTL ≈ 5 min).  
3. Redirects the user to the AWA Authenticator:

```
?state=<STATE>&public_key=<PK>
```

---

### **5.2 Authentication (Authenticator)**

1. Authenticator checks for a locally stored encrypted `MASTER_SUB`.  
    - If found → decrypts using passkey / biometric.  
    - **If not → initiates eID-based flow to refetches the MASTER_SUB and saves it locally.**  
2. Computes:
```
PAIRWISE_SUB = SHA-256(MASTER_SUB + RP_TopDomain)
```
3. Encrypts the pseudonym with the RP’s public key:
```
CIPHERTEXT = Encrypt(PK, PAIRWISE_SUB)
```
4. POSTs to the RP:
```
POST https://<RP_origin>/session
{
  state: <STATE>,
  payload: <CIPHERTEXT>
}
```

---

### **5.3 Verification (Relying Party)**

1. Retrieves `SK` using `STATE`.  
2. Decrypts `CIPHERTEXT` → `PAIRWISE_SUB`.  
3. Creates / updates session.  
4. Issues encrypted cookie:
```
cookie = AEAD_Encrypt(UniquePerCookieEncryptionKey, {
    noncePointer: RandomNonce,
    pseudonym: PAIRWISE_SUB
})
```

---

## **6  Security Properties**

| Property | Guarantee |
|-----------|------------|
| **Zero PII** | No names / emails ever leave the device. |
| **Deterministic Identity** | Same human → same pseudonym per RP. |
| **Unlinkable Between RPs** | Each RP sees unique pseudonym. |
| **Short-lived Crypto** | STATE + keys expire in minutes. |
| **Origin Isolation** | Authenticator posts only to original domain. |
| **No Rotations / Secrets** | Permanent trust without root key rotation. |

---

## **7  Local Storage and Security**

Authenticator stores `MASTER_SUB` only locally, encrypted with a key derived from the user’s passkey:
```
EncryptedMasterSub = AEAD_Encrypt(PasskeyDerivedKey, MASTER_SUB)
```
If missing or invalid, the eID flow refetches and resaves it.  
This design ensures offline capability and zero central storage.

---

## **8  Role of National Identity Authorities**

Each eID system issues one unique 1024-bit random seed per citizen — the `MASTER_SUB`.  
It is:
- Created once, immutable.  
- Meaningless outside AWA.  
- Shared only within the nation’s own systems, not across nations (good practice).  
- Accessible under lawful audit for recovery or accountability.

Thus a nation acts as a **seed broker**, not an identity broker.  
Trust is centralized in law, not in infrastructure.

---

## **9  Privacy Guarantees**

- No central login authority.  
- No cross-domain tracking.  
- No bot or duplicate accounts.  
- Deterministic re-derivation for lawful audit only.  
Humans stay private; services stay accountable.

---

## **10  Session Handling**

Sessions are RP-defined. Recommended pattern:
```
cookie = AEAD_Encrypt(MasterKey + EphemeralPepper, pseudonym)
```
`EphemeralPepper` lives in server-side KV; if missing → session invalid.  
No refresh tokens, no global state.

---

## **11  Authorized Data Exchange (AWA + OAuth Compatibility)**

AWA pseudonyms provide a trust anchor for temporary authorization between services — similar to OAuth but identity-free.

### **11.1 Flow**

1. **Service A** authenticates user via AWA:  
   ```
   pairwise_sub_A = SHA-256(MASTER_SUB + ServiceA_TopDomain)
   ```
2. **Service A** wishes to access data from **Service B**.  
   Service A redirects user to Service B’s AWA authenticator requesting specific scopes.  
3. **Service B** authenticates the user (standard AWA flow) and asks for consent.  
4. **Service B** issues a short-lived encrypted access token:
   ```
   token = AEAD_encrypt(pk_A, {
       iss: ServiceB_TopDomain,
       aud: ServiceA_TopDomain,
       sub: pairwise_sub_B,
       exp: now + 5 min,
       scope: ["profile.read","calendar.write"]
   })
   ```
5. **Service B → Service A:** returns token to the user’s browser → Service A receives it.  
6. **Service A → Service B:** uses `Authorization: Bearer <token>` for API calls within scoped rights.  
7. **Service B** decrypts token with its private key, verifies issuer and scope, and serves the requested data.

This flow is reciprocal: either party may act as issuer or recipient, and AWA never blocks such interaction.

---

### **11.2 Security**

- Ephemeral (minute-scale TTL).  
- Non-transferable (encrypted to recipient key).  
- Scope-restricted.  
- Rooted in verified human proof.  
- Drop-in compatible with OAuth / OIDC.

---

## **12  Summary**

> **Anonymous Web Authentication (AWA)** establishes permanent, verifiable humanity on the internet — without personal data, passwords, or rotating secrets.

Each eID system issues one random `MASTER_SUB` per citizen.  
Each RP derives a `PAIRWISE_SUB`.  
Each session is self-contained and zero-knowledge.  
Each interaction is proof of human authenticity, not identity.

---

## **Version**

**AWA Protocol Specification v1.0**  
Author – *Jori Lehtinen*  
Date – *October 2025*  
License – *CC BY 4.0 / Open Identity Standard Draft*
