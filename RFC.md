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
  "expiresAt": 1698765432,
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
    "decimals": 18
  },
  {
    "network": "ETHEREUM",
    "asset": "USDC",
    "address": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    "decimals": 6
  },
  {
    "network": "BASE",
    "asset": "USDC",
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "decimals": 6
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
  "expiresAt": 1698765932,
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