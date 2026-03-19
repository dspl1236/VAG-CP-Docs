# GEKO DMS Trust Store Analysis

## Files
- `dms_trust.keystore` — Java KeyStore (JKS), 2248 bytes, 2 entries
- `dms_client.keystore` — Java KeyStore (JKS), 32 bytes, **0 entries** (empty)

## What these are

These keystores are from a GEKO DMS (Dealer Management System) client
installation. GEKO uses mutual TLS (mTLS) for all dealer communication:

- `dms_trust.keystore` = what the dealer tool *trusts* (server-side certs)
- `dms_client.keystore` = the dealer's own certificate + private key (missing)

The client keystore being empty means either:
1. The client certificate was never provisioned into this installation
2. It was removed/expired
3. This is a partial installation (trust store shipped without dealer-specific credentials)

## Certificate 1 — GEKO Server (`www.vwgeko.net`)

| Field | Value |
|-------|-------|
| Subject | OU=www.vwgeko.net |
| Issuer | OU=www.vwgeko.net (self-signed) |
| Key | 2048-bit RSA |
| Signature | SHA256WithRSAEncryption |
| Valid | 2023-05-10 → 2033-05-10 |
| Serial | 0x48B6C3083205D69A |
| SHA256 | 11:4A:11:A4:DA:6E:E0:C7:69:11:64:0C:8D:1B:FF:D3:E9:81:CC:E4:AB:7C:84:61:11:49:21:AB:FA:92:BF:B8 |

This is the TLS certificate presented by the GEKO server. ODIS verifies
this cert before trusting any response from GEKO.

## Certificate 2 — VW CA Root (`VW-CA-ROOT-08 / VW-CA-ROOT-00`)

| Field | Value |
|-------|-------|
| Subject | OU=VW-CA-ROOT-08, CN=VW-CA-ROOT-00 |
| Issuer | OU=VW-CA-ROOT-08, CN=VW-CA-ROOT-00 (self-signed root) |
| Key | **4096-bit RSA** |
| Signature | SHA384WithRSAEncryption |
| Valid | 2023-03-17 → 2033-03-17 |
| Serial | 0x1F53B9650DCABAAB |
| SHA256 | B3:04:6D:36:95:AA:D4:B2:17:66:32:23:6D:4C:3E:7F:75:85:D8:FE:9B:10:21:91:B1:7A:20:63:07:86:D5:9A |

This is VW's CA root certificate. All VW-signed artifacts (ODIS packages,
firmware, dealer tool updates, potentially IKA key blobs) chain back to this
CA. Having this public key allows verification of any data VW signs.

## What this tells us about CP token generation

The IKA key blob that GEKO generates and ODIS writes to `DID 0x00BE` is:
- Generated server-side at GEKO
- Signed with an **internal GEKO signing key** (NOT these TLS certificates)
- The signing key may chain back to `VW-CA-ROOT-00` but is a different leaf cert

The mTLS flow:
```
Dealer tool → presents dms_client cert → GEKO server
GEKO server → presents www.vwgeko.net cert → Dealer tool verifies via dms_trust
GEKO server → generates IKA key signed with internal key → returns blob
Dealer tool → writes blob to DID 0x00BE on J533/J255
J533 → verifies blob against embedded verification key in MCU firmware
```

## What's missing for offline derivation

1. The **GEKO internal signing key** — held by GEKO server, never sent to clients
2. The **J533 verification key** — embedded in Renesas D70F3433 MCU firmware
3. A **populated `dms_client.keystore`** — would contain a dealer private key,
   allowing direct GEKO API calls if/when the server returns

## Path forward

The VW CA Root public key can be used to:
- Verify VW-signed ODIS packages and firmware updates
- Identify any other certificates in the VW PKI chain
- Potentially verify the format of IKA key blobs if they carry a cert chain

The J533 firmware extraction path remains the most direct route to the
verification key. If the verification key chains to VW-CA-ROOT-00, we would
recognize it when found.

## Files

- `vwgeko_server.pem` — GEKO server TLS certificate (PEM format)
- `vw_ca_root.pem` — VW CA Root certificate (PEM format)
