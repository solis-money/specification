# Solis

## Abstract

This document proposes Solis, a standardized protocol to simplify and secure cryptocurrency payment experiences. Solis aims to eliminate common errors in crypto payments such as network mismatch, asset mismatch, and incorrect payment amounts through a universally adoptable URI scheme and API specifications.

## Status of This Document

This document is a draft proposal.

## Introduction

Cryptocurrency payments currently suffer from poor user experience and high error rates due to the complexity of blockchain networks, asset varieties, and the technical knowledge required to complete transactions. Users often make costly mistakes by:

1. Sending the correct asset on the wrong network (e.g., USDC on Ethereum instead of Base)
2. Sending the wrong asset on the correct network (e.g., USDT instead of USDC)
3. Sending incorrect payment amounts
4. Manually entering recipient addresses incorrectly

Solis aims to resolve these issues by creating a standardized, protocol-agnostic solution that allows for dynamic, context-aware payment requests with minimal user input.

## Goals

- Provide a universal URI scheme for cross-chain cryptocurrency payments
- Reduce user friction and error rates in crypto payment flows
- Protect user privacy
- Ensure simple implementation for merchants and payment processors
- Support various interaction methods (QR codes, NFC, deeplinks, etc.)
- Allow for dynamic pricing and asset conversion
- Maintain backward compatibility with existing wallet infrastructure

## Protocol Design

### Solis URI Scheme

Solis defines a new URI scheme `solis://` that serves as the foundation for payment requests:

```
solis://[payment-id]?endpoint=[api-endpoint]&options=[options]
```

Where:
- `payment-id`: A unique identifier for the payment request
- `api-endpoint`: Base URL for the Solis API endpoints 
- `options`: Optional parameters (version, expiry, merchant-data, etc.)

Example:
```
solis://pay_7dZ4MGal82bx?endpoint=https://merchant.com/solis&options=version=1.0,expiry=1698765432
```

### API Endpoints

The protocol requires merchants/receivers to implement the following REST API endpoints:

#### 1. Payment Information Endpoint

`GET [api-endpoint]/[payment-id]`

Returns basic information about the payment request:

```json
{
  "id": "pay_7dZ4MGal82bx",
  "merchant": "Example Store",
  "description": "Order #12345",
  "value": {
    "amount": "100.01",
    "asset": "USDC"
  },
  "expiresAt": 1698765432000,
  "status": "PENDING"
}
```

#### 2. Supported Assets Endpoint

`GET [api-endpoint]/assets/[payment-id]`

Returns an array of supported assets for payment:

```json
[
  {
    "network": "ETHEREUM",
    "asset": "ETH",
    "address": null,
    "decimals": 18,
    "time": 60000
  },
  {
    "network": "ETHEREUM",
    "asset": "USDC",
    "address": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    "decimals": 6,
    "time": 60000
  },
  {
    "network": "BASE",
    "asset": "USDC",
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "decimals": 6,
    "time": 2000
  }
]
```

#### 3. Quote Endpoint

`POST [api-endpoint]/quote/[payment-id]`

Request body:
```json
{
  "network": "BASE",
  "asset": "USDC"
}
```

Response:
```json
{
  "id": "quote_8dZ4MGal82bx",
  "expiresAt": 1698765932000,
  "network": "BASE",
  "value": {
    "amount": "100.01",
    "asset": "USDC"
  }
}
```

#### 4. Payment Intent Creation Endpoint

`POST [api-endpoint]/intent/[payment-id]`

Request body:
```json
{
  "network": "BASE",
  "asset": "USDC",
  "quoteId": "quote_8dZ4MGal82bx",
  "compliance": {
    "format": "PLAIN",
    "type": "INDIVIDUAL", 
    "details": {
      "firstName": "Satoshi",
      "lastName": "Nakamoto",
      "country": "US"
    }
  }
}
```

The `compliance` object is optional and may be included for travel rule compliance. If present, it MUST contain the `format`, `type`, and `details` object with the payer's `firstName`, `lastName`, and `country` (ISO 3166-1 alpha-2 code).

Example request without compliance info:
```json
{
  "network": "BASE",
  "asset": "USDC",
  "quoteId": "quote_8dZ4MGal82bx"
}
```

Response:
```json
{
  "network": "BASE",
  "value": {
    "amount": "100.01",
    "asset": "USDC"
  },
  "destination": {
    "type": "CRYPTO",
    "details": {
      "address": "0xB180731051bb0116DA91Ab7F1bf3cE1bD7C02Fd4",
      "reference": "5431263"
    }
  }
}
```

#### 5. Transaction Notification Endpoint 

`POST [api-endpoint]/notify/[payment-id]`

This endpoint allows wallets to notify merchants about a transaction submission before it's confirmed on the blockchain, enabling faster feedback to users. All fields are optional. The reference field is if a network such as Ripple supports a source reference (like source tag).

Request body:
```json
{
  "hash": "0x10fe14ae0a6ea8a4e69827b2a72a2a24fd151d2f017ac62943b94351a2e8d935",
  "transaction": "0x...",
  "sender": {
    "address": "0xB180731051bb0116DA91Ab7F1bf3cE1bD7C02Fd4",
    "reference": "5431263"
  }
}
```

Response:
```json
{}
```

### Payment Flow

1. **Initiate Payment**: The merchant generates a Solis URI containing a unique payment ID and embeds it in a QR code, makes it available via NFC (for tap-to-pay in Point-of-Sale scenarios), or shares it through other means.

2. **User Scans Code/Taps Device**: The user's wallet app recognizes the Solis protocol, either by scanning the QR code or receiving the URI via NFC, and parses the URI.

3. **Fetch Payment Info**: The wallet fetches payment details from the `/{payment-id}` endpoint.

4. **Get Supported Assets**: The wallet queries the `/assets/{payment-id}` endpoint to determine supported assets and networks.

5. **User Asset Selection**: The user selects which asset they want to pay with from supported options presented by their wallet.

6. **Get Payment Quote**: The wallet calls the `/quote/{payment-id}` endpoint with the selected asset to retrieve the exact amount required and other payment details.

7. **Create Payment Intent**: The wallet calls the `/intent/{payment-id}` endpoint with the selected asset, quote information, and optionally compliance information.

8. **Execute Payment**: The wallet initiates the transaction using the details received, including exact amount and recipient address.

9. **Transaction Notification**: The wallet notifies the merchant about the submitted transaction via the `/notify/{payment-id}` endpoint, including transaction hash and details if available.

10. **Verify Completion**: The wallet can check payment completion through blockchain monitoring or merchant-specific status endpoints.

## Error Handling

The protocol defines standard error codes and handling mechanisms:

- `400` - Bad Request: Invalid parameters
- `404` - Not Found: Payment ID doesn't exist
- `402` - Payment Required: Payment expired or invalid
- `409` - Conflict: Asset or network not supported
- `500` - Server Error: Processing error

All error responses should include a standardized error object:

```json
{
  "error": {
    "code": "UNSUPPORTED_ASSET",
    "message": "The requested asset is not supported for this payment",
    "details": {}
  }
}
```

**Standard Error Codes:**
- `PAYMENT_NOT_FOUND`
- `PAYMENT_EXPIRED` 
- `UNSUPPORTED_ASSET`
- `UNSUPPORTED_NETWORK`
- `INVALID_QUOTE`
- `INVALID_AMOUNT`
- `MISSING_REQUIRED_FIELD`
- `INVALID_PAYMENT_ID`
- `INTERNAL_SERVER_ERROR`

## Privacy Considerations

1. **Minimal Data Collection**: The protocol is designed to collect only necessary information to complete a payment.

2. **No Personal Data**: The protocol doesn't require personal information beyond optional compliance data provided at the user's discretion.

3. **Optional Compliance Info**: Compliance information in the payment intent creation is optional, allowing users to provide it only when required.

4. **Local Processing**: Wallets can process asset selection locally before making API calls.

## Security Considerations

1. **No Private Keys**: The protocol never requires the sharing of private keys.

2. **HTTPS Required**: All API communications must use HTTPS.

3. **Unique Payment IDs**: Each payment gets a unique, one-time ID to prevent replay attacks.

4. **Time-Limited**: Payment intents can include expiry timestamps.

5. **Authenticated Endpoints**: Implementations may add additional authentication requirements.

6. **Domain Verification**: Wallet implementations should verify that API endpoints match the domain in the QR code.

## Future Extensions

1. **Cross-Chain Swaps**: Built-in asset swapping for unsupported assets.
2. **Payment Proofs**: Standardized receipt format for payments.
3. **Recurring Payments**: Support for subscription and recurring payment models.
4. **Fiat On-Ramps**: Integration with fiat on-ramp services.
5. **Message Signing**: Support for signed payment requests.

## POS Mode (BLE HTTP Proxy)

In Point-of-Sale scenarios, customers often have poor or no cellular connectivity inside stores while the POS terminal has a stable internet connection. POS Mode enables the wallet to operate completely offline using BLE, with the POS terminal acting as an HTTP proxy.

This design uses a minimal "HTTP Proxy over GATT" model with only 4 message types. After session establishment, the wallet sends standard HTTP-like requests which the POS proxies to the backend API. This means:
- **Single source of truth**: The HTTP API schemas are authoritative
- **No schema duplication**: BLE messages carry the same JSON bodies as HTTP
- **Simpler wallet implementation**: Reuse existing HTTP request/response handling

### Architecture

```
┌─────────────────┐                    ┌─────────────────┐                    ┌──────────┐
│     WALLET      │        BLE         │       POS       │       HTTPS        │ Backend  │
│   (Peripheral)  │◄──────────────────►│  (HTTP Proxy)   │◄──────────────────►│          │
│    [offline]    │                    │    [online]     │                    │          │
└─────────────────┘                    └─────────────────┘                    └──────────┘
```

**Key Design Decisions:**
- **POS is BLE Central** (scanner/initiator) - has power, can scan aggressively
- **Wallet is BLE Peripheral** (advertiser) - low power, always ready
- **POS acts as HTTP proxy** - forwards wallet requests to backend API unchanged
- **Only 4 message types** - CONNECT, CLAIM, REQUEST, RESPONSE (plus ERROR)
- **User confirms on wallet** to claim the payment (resolves multi-wallet ambiguity)

### Payment Flow

```
TIME        WALLET                              POS                         BACKEND
─────────────────────────────────────────────────────────────────────────────────────────
            [Advertising: "Solis Wallet"]

    0ms                                         Cashier enters $5.00
                                                Presses CHARGE

   30ms     ◄─────────── BLE CONNECT ──────────

   50ms     ◄───────── SOLIS_CONNECT ──────────
            {paymentId, posPubkey,
             merchant, value, assets}

  100ms     [Display: "Pay $5 to Store?"]
            [User confirms + selects asset]

  150ms     ────────── SOLIS_CLAIM ───────────►
            {walletPubkey}

            ═══════ Session Encrypted ═══════

  180ms     ─── SOLIS_REQUEST ────────────────►  ─── POST /quote/{id} ──────►
            {method:"POST", path:"/quote/...",
             body:{network, asset}}
  210ms     ◄── SOLIS_RESPONSE ───────────────  ◄── {quoteId, value...} ────
            {status:200, body:{...}}

  230ms     ─── SOLIS_REQUEST ────────────────►  ─── POST /intent/{id} ─────►
            {method:"POST", path:"/intent/...",
             body:{quoteId}}
  260ms     ◄── SOLIS_RESPONSE ───────────────  ◄── {destination...} ───────
            {status:200, body:{...}}

  280ms     [Sign transaction locally]

  320ms     ─── SOLIS_REQUEST ────────────────►  ─── POST /notify/{id} ─────►
            {method:"POST", path:"/notify/...",   ─── Broadcast to chain ───►
             body:{signedTransaction}}
  360ms     ◄── SOLIS_RESPONSE ───────────────  ◄── {hash, status} ─────────
            {status:200, body:{hash, status}}

            [Display: "Payment Complete"]        [Display: "Paid"]
─────────────────────────────────────────────────────────────────────────────────────────
            Total: ~360ms from CHARGE to confirmation
```

### Resolving Multiple Wallets (Broadcast + Claim)

When multiple wallet-enabled devices are in BLE range:

1. **POS connects** to ALL nearby Solis wallets and sends SOLIS_CONNECT
2. **All wallets** in range display the payment prompt
3. **Only the paying customer** taps "Confirm" and sends SOLIS_CLAIM
4. **POS accepts** first claim, disconnects other wallets

```
POS ────SOLIS_CONNECT────► Wallet A (sees prompt, ignores - not their purchase)
    ────SOLIS_CONNECT────► Wallet B (sees prompt, CONFIRMS) ──SOLIS_CLAIM──►
    ────SOLIS_CONNECT────► Wallet C (sees prompt, ignores - not their purchase)
```

Additionally, RSSI (signal strength) filtering ensures only nearby devices receive requests:
- RSSI > -45 dBm: Very close (~0.5m) - auto-connect
- RSSI -45 to -65 dBm: Near (0.5-2m) - include in broadcast
- RSSI < -65 dBm: Far (>2m) - ignore

### BLE Message Types (4 Types Only)

All messages are JSON-encoded. After SOLIS_CLAIM, messages are encrypted with AES-128-GCM.

| Type | Direction | Encrypted | Purpose |
|------|-----------|-----------|---------|
| `SOLIS_CONNECT` | POS → Wallet | No | Initial payment info + POS session key |
| `SOLIS_CLAIM` | Wallet → POS | No | Wallet claims payment + wallet session key |
| `SOLIS_REQUEST` | Wallet → POS | Yes | HTTP request proxy (method, path, body) |
| `SOLIS_RESPONSE` | POS → Wallet | Yes | HTTP response proxy (status, body) |
| `SOLIS_ERROR` | Either | Conditional | Error notification |

#### SOLIS_CONNECT (POS → Wallet)

Sent immediately after BLE connection. Contains all info needed for user decision plus POS session key:

```json
{
  "type": "SOLIS_CONNECT",
  "version": "1.1",
  "paymentId": "pos_123456",
  "posPubkey": "02a1b2c3d4e5f6...",
  "merchant": "Coffee Shop",
  "payment": {
    "description": "Order #789",
    "value": { "amount": "5.00", "asset": "USD" },
    "expiresAt": 1698765432000
  },
  "assets": [
    { "network": "BASE", "asset": "USDC", "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "decimals": 6, "time": 2000 },
    { "network": "ETHEREUM", "asset": "USDC", "address": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48", "decimals": 6, "time": 60000 }
  ]
}
```

#### SOLIS_CLAIM (Wallet → POS)

Sent when user confirms. Establishes encrypted session:

```json
{
  "type": "SOLIS_CLAIM",
  "walletPubkey": "03f9e8d7c6b5a4..."
}
```

After this message, both parties derive the shared secret and all subsequent messages are encrypted.

#### SOLIS_REQUEST (Wallet → POS) [Encrypted]

Generic HTTP request proxy. The POS forwards the request to the backend API:

```json
{
  "type": "SOLIS_REQUEST",
  "id": "req_001",
  "method": "POST",
  "path": "/quote/pos_123456",
  "body": {
    "network": "BASE",
    "asset": "USDC"
  }
}
```

The POS translates this to:
```
POST https://api.merchant.com/solis/quote/pos_123456
Content-Type: application/json

{"network": "BASE", "asset": "USDC"}
```

#### SOLIS_RESPONSE (POS → Wallet) [Encrypted]

HTTP response from the backend:

```json
{
  "type": "SOLIS_RESPONSE",
  "id": "req_001",
  "status": 200,
  "body": {
    "id": "quote_abc123",
    "expiresAt": 1698765532000,
    "network": "BASE",
    "value": { "amount": "5.00", "asset": "USDC" }
  }
}
```

#### SOLIS_ERROR

Error notification (encrypted if session established):

```json
{
  "type": "SOLIS_ERROR",
  "code": "PAYMENT_CLAIMED",
  "message": "This payment has already been claimed by another wallet"
}
```

Error codes:
- `PAYMENT_NOT_FOUND` - Payment ID does not exist
- `PAYMENT_EXPIRED` - Payment has expired
- `PAYMENT_CLAIMED` - Another wallet already claimed this payment
- `SESSION_EXPIRED` - BLE session has expired
- `PROXY_ERROR` - POS failed to reach backend
- `BROADCAST_FAILED` - Transaction broadcast failed

### Complete Payment Example

**Step 1: Quote Request**
```json
// Wallet → POS (encrypted)
{
  "type": "SOLIS_REQUEST",
  "id": "1",
  "method": "POST",
  "path": "/quote/pos_123456",
  "body": { "network": "BASE", "asset": "USDC" }
}

// POS → Wallet (encrypted)
{
  "type": "SOLIS_RESPONSE",
  "id": "1",
  "status": 200,
  "body": {
    "id": "quote_abc",
    "expiresAt": 1698765532000,
    "network": "BASE",
    "value": { "amount": "5.00", "asset": "USDC" }
  }
}
```

**Step 2: Intent Request**
```json
// Wallet → POS (encrypted)
{
  "type": "SOLIS_REQUEST",
  "id": "2",
  "method": "POST",
  "path": "/intent/pos_123456",
  "body": { "quoteId": "quote_abc" }
}

// POS → Wallet (encrypted)
{
  "type": "SOLIS_RESPONSE",
  "id": "2",
  "status": 200,
  "body": {
    "network": "BASE",
    "value": { "amount": "5.00", "asset": "USDC" },
    "destination": {
      "type": "CRYPTO",
      "details": { "address": "0xB180731051bb0116DA91Ab7F1bf3cE1bD7C02Fd4" }
    }
  }
}
```

**Step 3: Transaction Broadcast**
```json
// Wallet → POS (encrypted)
{
  "type": "SOLIS_REQUEST",
  "id": "3",
  "method": "POST",
  "path": "/notify/pos_123456",
  "body": {
    "hash": "0x...",
    "transaction": "0x02f87083...",
    "sender": { "address": "0xWalletAddress..." }
  }
}

// POS → Wallet (encrypted)
{
  "type": "SOLIS_RESPONSE",
  "id": "3",
  "status": 200,
  "body": {}
}
```

Note: For transaction broadcast, the POS also broadcasts the signed transaction to the blockchain RPC before returning the response.

### Session Security

1. **Key Exchange**: POS includes `posPubkey` in SOLIS_CONNECT, wallet includes `walletPubkey` in SOLIS_CLAIM
2. **Shared Secret**: Both derive via ECDH (secp256k1) after receiving peer's public key
3. **Key Derivation**: `sessionKey = HKDF-SHA256(sharedSecret, salt="solis-ble-v1.1", info=paymentId)`
4. **Encryption**: AES-128-GCM for all SOLIS_REQUEST/SOLIS_RESPONSE messages
5. **Nonces**: Counter-based (8 bytes counter + 4 bytes random per session)
6. **Session Timeout**: 2 minutes maximum session duration

### BLE Technical Specifications

#### Wallet GATT Service (Peripheral)

```
Service: Solis Wallet
UUID: 0000FFF0-0000-1000-8000-00805F9B34FB

Characteristics:
  Request (POS → Wallet)
    UUID: 0000FFF1-0000-1000-8000-00805F9B34FB
    Properties: Write, Write Without Response

  Response (Wallet → POS)
    UUID: 0000FFF2-0000-1000-8000-00805F9B34FB
    Properties: Notify, Indicate

Advertisement Data:
  - Flags (3 bytes)
  - Service UUID (4 bytes)
  - Local Name: "Solis-XXXXXXXX" (rotating hash)
```

#### Connection Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Advertising Interval (Wallet) | 20-30ms | Fast discovery when paying |
| Connection Interval | 7.5-15ms | Low latency data exchange |
| MTU | 512 bytes | Minimize fragmentation |
| Supervision Timeout | 4 seconds | Quick disconnect detection |

### Platform Compatibility

| Platform | BLE Peripheral | BLE Central | Background | Notes |
|----------|----------------|-------------|------------|-------|
| iOS 13+ | ✅ | ✅ | Limited | Background advertising throttled to ~1s interval |
| Android 5+ | ✅ | ✅ | ✅ | Full background support with foreground service |

**iOS Background Limitations:**
- Advertising interval forced to ~1000ms when backgrounded
- Service UUIDs moved to overflow area (slower discovery)
- Mitigation: User opens app or uses widget when ready to pay

### Security Considerations

| Threat | Mitigation |
|--------|------------|
| Malicious POS provides fake address | Wallet displays destination address prominently for user verification |
| POS withholds signed transaction | Wallet retains signed tx locally; can resubmit when online |
| Eavesdropping on BLE | AES-128-GCM encryption after ECDH key exchange |
| Replay attacks | Counter-based nonces + payment ID uniqueness |
| Rogue wallet claims payment | POS accepts first claim only; RSSI filtering limits range |
| Tracking via BLE address | Rotating BLE addresses; rotating wallet ID hash |
| Multiple POS confusion | Wallet displays merchant name; user confirms before payment |

### Privacy Considerations

1. **Rotating Addresses**: Wallet BLE address changes every 15 minutes
2. **Rotating Wallet ID**: Hash in advertisement changes per session
3. **Opt-in Advertising**: User must enable "ready to pay" mode
4. **Auto-disable**: Advertising stops after 5 minutes of inactivity
5. **No Persistent Identifiers**: Payment IDs are one-time use

### Implementation Notes

**For POS Terminals:**
- Act as BLE Central (scanner)
- Advertise POS presence for wallet discovery
- Scan for Solis wallets when payment initiated
- Connect to wallets with strong RSSI (>-65 dBm)
- Broadcast payment request to multiple wallets if needed
- Accept first valid claim response
- Relay API calls to backend
- Broadcast signed transactions to blockchain

**For Wallets:**
- Act as BLE Peripheral (advertiser)
- Advertise Solis Wallet service when ready to pay
- Accept connections from Solis POS devices
- Display payment details prominently for user verification
- Require explicit user confirmation before signing
- Sign transactions locally (never send private keys)
- Retain signed transactions until confirmation received
- Support background mode where platform allows

## Implementation Recommendations

Wallets should:
- Implement the Solis URI handler
- Support visual confirmation of payment details before transaction
- Handle network switching gracefully
- Save frequent merchant information for faster processing
- Implement transaction notification when possible to improve user experience

Merchants should:
- Generate unique payment IDs for each transaction
- Implement all required API endpoints
- Support multiple popular assets/networks
- Monitor payment status through blockchain APIs
- Ensure quote validity times are reasonable for market volatility
- Implement the transaction notification endpoint for faster user feedback

## Backwards Compatibility

Wallets that don't support Solis can use fallback mechanisms:
- QR codes can include a standard crypto address as a fallback
- Deeplinks can redirect to basic payment pages
- NFC implementations can default to standard crypto address sharing

## Reference Implementation

A reference implementation specification is available at: [/openapi/solis-openapi-spec.yaml](/openapi/solis-openapi-spec.yaml)

## Conclusion

Solis aims to dramatically improve the user experience of cryptocurrency payments by standardizing payment flows across wallets and merchants. By preventing common mistakes and reducing friction, Solis brings crypto payments closer to the seamless experience of traditional payment methods while preserving the decentralized nature and privacy benefits of cryptocurrencies. 