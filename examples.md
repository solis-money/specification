# Solis Implementation Examples

This document provides practical examples of Solis implementation.

## Example 1: E-commerce Checkout

### Scenario
A user is checking out on an e-commerce site to purchase an item for $100 USD.

### Implementation Steps

1. **Generate Payment Request**

The merchant backend creates a payment request with a unique ID:

```json
// Internal payment record
{
  "paymentId": "pay_AB12CD34EF56",
  "orderId": "order_98765",
  "amount": "100.00",
  "currency": "USD",
  "customer": "customer_12345",
  "expiresAt": 1698765432
}
```

2. **Create Solis URI**

The merchant generates a Solis URI:

```
solis://pay_AB12CD34EF56?endpoint=https://store.example.com/api/payments&options=version=1.0,expiry=1698765432
```

3. **Display QR Code**

The URI is encoded into a QR code and displayed on the checkout page.

4. **User Scans QR Code**

The user scans the QR code with their wallet app, which recognizes the Solis protocol.

5. **Query Payment Information**

The wallet makes an API call to get payment details:

```
GET https://store.example.com/api/payments/pay_AB12CD34EF56
```

Response:
```json
{
  "id": "pay_AB12CD34EF56",
  "merchant": "Example Store",
  "description": "Order #98765",
  "value": {
    "amount": "100.00",
    "asset": "USDC"
  },
  "expiresAt": 1698765432,
  "status": "PENDING"
}
```

6. **Query Supported Assets**

The wallet queries for supported payment methods:

```
GET https://store.example.com/api/payments/assets/pay_AB12CD34EF56
```

Response:
```json
[
  {
    "network": "ETHEREUM",
    "asset": "USDC",
    "address": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    "decimals": 6,
    "time": 2000
  },
  {
    "network": "BASE",
    "asset": "USDC",
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "decimals": 6,
    "time": 2000
  },
  {
    "network": "POLYGON",
    "asset": "USDC",
    "address": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
    "decimals": 6,
    "time": 5000
  },
  {
    "network": "ARBITRUM",
    "asset": "USDC",
    "address": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "decimals": 6,
    "time": 5000
  }
]
```

7. **User Selects Payment Option**

The user selects to pay with USDC on Base network.

8. **Request Payment Quote**

The wallet requests a quote for the selected asset:

```
POST https://store.example.com/api/payments/quote/pay_AB12CD34EF56
```

Request:
```json
{
  "network": "BASE",
  "asset": "USDC"
}
```

Response:
```json
{
  "id": "quote_XYZ123",
  "expiresAt": 1698765932000,
  "network": "BASE",
  "value": {
    "amount": "100.00",
    "asset": "USDC"
  }
}
```

9. **Create Payment Intent**

The wallet creates a payment intent with the quote information:

```
POST https://store.example.com/api/payments/intent/pay_AB12CD34EF56
```

Request:
```json
{
  "network": "BASE",
  "asset": "USDC",
  "quoteId": "quote_XYZ123",
  "compliance": {
    "format": "PLAIN",
    "type": "INDIVIDUAL",
    "details": {
      "firstName": "Alice",
      "lastName": "Smith",
      "country": "US"
    }
  }
}
```

Response:
```json
{
  "network": "BASE",
  "value": {
    "amount": "100.00",
    "asset": "USDC"
  },
  "destination": {
    "type": "CRYPTO",
    "details": {
      "address": "0x7a16fF8270133F063aAb6C9977183D9e72835428",
      "reference": "Payment for Order #98765"
    }
  }
}
```

10. **Execute Transaction**

The wallet prepares and executes the transaction with the exact parameters provided.

```
Transaction Hash: 0x7b65b75d204abed71587c9e519a89277766ee1d0
```

10.1. ** Transaction Notification**

Immediately after the transaction is submitted to the blockchain, the wallet notifies the merchant:

```
POST https://store.example.com/api/payments/notify/pay_AB12CD34EF56
```

Request:
```json
{
  "hash": "0x7b65b75d204abed71587c9e519a89277766ee1d0",
  "transaction": "0x...",
  "sender": {
    "address": "0xae2Fc483527B8EF99EB5D9B44875F005ba1FaE13"
  }
}
```

Response:
```json
{}
```

The merchant can now start watching for this specific transaction on the blockchain and provide immediate feedback to the user that the payment is in progress, without waiting for blockchain confirmations.

11. **Complete Checkout**

Once the payment is confirmed on the blockchain, the checkout process is complete.

## Example 2: Physical Point-of-Sale

### Scenario
A user is paying at a coffee shop with cryptocurrency.

### Implementation Steps

1. **Create Payment Request**

The point-of-sale system creates a payment request for a $5 coffee:

```json
{
  "paymentId": "pos_123456",
  "amount": "5.00",
  "currency": "USD",
  "merchantId": "coffee_shop_downtown",
  "expiresAt": 1698765432
}
```

2. **Generate and Display QR Code**

The POS terminal displays a QR code containing:

```
solis://pos_123456?endpoint=https://payments.coffeeshop.com/api&options=version=1.0,expiry=1698765432
```

3. **Workflow Continues**

The payment flow continues as in Example 1, with the wallet retrieving payment information, supported assets, requesting a quote for the selected asset, creating a payment intent, and executing the transaction.

4. **In-Person Confirmation**

Once the payment is confirmed, the POS terminal displays a confirmation message, and the barista prepares the coffee.

## Example 3: Error Handling

### Scenario
A user attempts to pay with an unsupported asset.

### Implementation Flow

1-6. **Same as Example 1**

7. **User Selects Unsupported Asset**

The user attempts to pay with an asset not listed in the supported assets.

8. **Error Response from Quote Endpoint**

```
POST https://store.example.com/api/payments/quote/pay_AB12CD34EF56
```

Request:
```json
{
  "network": "ETHEREUM",
  "asset": "SHIB"
}
```

Response (HTTP 409 Conflict):
```json
{
  "error": {
    "code": "UNSUPPORTED_ASSET",
    "message": "The requested asset is not supported for this payment",
    "details": {}
  }
}
```

9. **Wallet Handles Error**

The wallet app displays an error message to the user and suggests selecting from the list of supported assets.

## Example 4: Privacy-Enhanced Flow

### Scenario
A user wishes to pay for an item without providing compliance information.

### Implementation Flow

1-7. **Same as Example 1**

8. **Request Payment Quote**

The wallet requests a quote for the selected asset:

```
POST https://store.example.com/api/payments/quote/pay_AB12CD34EF56
```

Request:
```json
{
  "network": "BASE",
  "asset": "USDC"
}
```

Response:
```json
{
  "id": "quote_XYZ123",
  "expiresAt": 1698765932000,
  "network": "BASE",
  "value": {
    "amount": "100.00",
    "asset": "USDC"
  }
}
```

9. **Create Payment Intent Without Compliance Information**

The wallet creates a payment intent without compliance information:

```
POST https://store.example.com/api/payments/intent/pay_AB12CD34EF56
```

Request:
```json
{
  "network": "BASE",
  "asset": "USDC",
  "quoteId": "quote_XYZ123"
}
```

Response:
```json
{
  "network": "BASE",
  "value": {
    "amount": "100.00",
    "asset": "USDC"
  },
  "destination": {
    "type": "CRYPTO",
    "details": {
      "address": "0x7a16fF8270133F063aAb6C9977183D9e72835428",
      "reference": "Payment for Order #98765"
    }
  }
}
```

10. **Execute Transaction**

The wallet prepares and executes the transaction with the exact parameters provided.

11. **Verify Payment**

The payment verification proceeds through blockchain monitoring or merchant-specific mechanisms.

### Privacy Benefits

This flow provides several privacy advantages:

1. **No Personal Information**: No personal information is shared with the merchant.

2. **Optional Compliance**: Compliance information is only provided when explicitly required.

3. **Minimal Data**: Only essential payment information is exchanged.

## Example 5: Multi-Network Payment Options

### Scenario
A merchant accepts the same asset on multiple networks, and the user can choose their preferred network.

### Implementation Flow

1-5. **Same as Example 1**

6. **Query Supported Assets**

```
GET https://store.example.com/api/payments/assets/pay_AB12CD34EF56
```

Response showing USDC on multiple networks:
```json
[
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
  },
  {
    "network": "POLYGON",
    "asset": "USDC",
    "address": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
    "decimals": 6,
    "time": 5000
  },
  {
    "network": "ARBITRUM",
    "asset": "USDC",
    "address": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "decimals": 6,
    "time": 5000
  }
]
```

7. **User Selects Preferred Network**

The user chooses to pay with USDC on Polygon (perhaps for lower fees).

8-11. **Continue as in Example 1**

The payment flow continues with the user's selected network/asset combination.

These examples demonstrate the flexibility and error-handling capabilities of the Solis protocol across different payment scenarios.

## Example 6: WalletConnect Integration

### Scenario
A user is shopping on a desktop e-commerce site and wants to pay with cryptocurrency using their mobile wallet without scanning a QR code.

### Implementation Steps

1. **Generate Payment Request**

The merchant backend creates a payment request as usual:

```json
{
  "paymentId": "pay_WC78CD90EF12",
  "orderId": "order_54321",
  "amount": "75.50",
  "currency": "USD",
  "customer": "customer_67890",
  "expiresAt": 1698765432000
}
```

2. **Display Payment Options**

The checkout page displays both QR code and "Connect Wallet" options:

```html
<div class="payment-methods">
  <div class="qr-payment">
    <h3>Scan to Pay</h3>
    <qr-code data="solis://pay_WC78CD90EF12?endpoint=https://store.example.com/api/payments&options=version=1.0,expiry=1698765432"></qr-code>
  </div>
  
  <div class="wallet-connect">
    <h3>Connect Wallet</h3>
    <button id="connect-wallet">Connect Wallet</button>
  </div>
</div>
```

3. **User Initiates WalletConnect**

User clicks "Connect Wallet" button, triggering WalletConnect session request:

```javascript
// Frontend code
import { createWeb3Modal, defaultWagmiConfig } from '@web3modal/wagmi/react'

const connectWallet = async () => {
  try {
    // Initialize WalletConnect session
    const session = await web3Modal.open()
    
    if (session) {
      // Session established, proceed with Solis payment
      initiateSolisPayment()
    }
  } catch (error) {
    console.error('WalletConnect failed:', error)
  }
}
```

4. **Wallet Connection Established**

Once connected, the merchant can identify the user's wallet address and proceed with payment:

```javascript
const initiateSolisPayment = async () => {
  const walletAddress = await getAccount()
  
  // Send Solis payment request to connected wallet
  const paymentRequest = {
    protocol: 'solis',
    version: '1.0',
    paymentId: 'pay_WC78CD90EF12',
    endpoint: 'https://store.example.com/api/payments',
    walletAddress: walletAddress
  }
  
  // Send request through WalletConnect
  await sendCustomRequest({
    method: 'solis_paymentRequest',
    params: [paymentRequest]
  })
}
```

5. **Wallet Receives Solis Payment Request**

The connected wallet receives the Solis payment request through WalletConnect and automatically fetches payment details:

```
GET https://store.example.com/api/payments/pay_WC78CD90EF12
```

Response:
```json
{
  "id": "pay_WC78CD90EF12",
  "merchant": "Example Store",
  "description": "Order #54321",
  "value": {
    "amount": "75.50",
    "asset": "USDC"
  },
  "expiresAt": 1698765432000,
  "status": "PENDING"
}
```

6. **Streamlined Asset Selection**

Since the wallet is connected, it can pre-filter supported assets based on the user's available balances:

```
GET https://store.example.com/api/payments/assets/pay_WC78CD90EF12
```

The wallet automatically shows only assets the user actually holds:

```json
[
  {
    "network": "BASE",
    "asset": "USDC",
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "decimals": 6,
    "userBalance": "150.25"
  },
  {
    "network": "ETHEREUM",
    "asset": "ETH",
    "address": null,
    "decimals": 18,
    "userBalance": "0.084"
  }
]
```

7. **Enhanced User Experience**

The wallet displays a streamlined payment interface with pre-selected optimal options:

```javascript
// Wallet automatically suggests best option based on:
// - User's available balances
// - Network fees
// - Transaction speed
const suggestedPayment = {
  network: "BASE",
  asset: "USDC",
  estimatedFee: "$0.02",
  estimatedTime: "2-5 seconds"
}
```

8. **Payment Authorization**

User approves payment in wallet. The wallet sends confirmation back to merchant through WalletConnect:

```javascript
// Wallet sends response back through WalletConnect
const response = {
  method: 'solis_paymentResponse',
  result: {
    approved: true,
    network: 'BASE',
    asset: 'USDC',
    fromAddress: walletAddress
  }
}

await sendCustomResponse(response)
```

9. **Continue with Standard Solis Flow**

The payment now continues with the standard Solis flow:
- Quote request
- Payment intent creation  
- Transaction execution
- Confirmation

10. **Real-time Updates**

Since WalletConnect maintains the connection, the merchant can provide real-time updates:

```javascript
// Merchant sends transaction status updates
await sendCustomRequest({
  method: 'solis_paymentStatus',
  params: [{
    paymentId: 'pay_WC78CD90EF12',
    status: 'CONFIRMED',
    transactionHash: '0x7b65b75d204abed71587c9e519a89277766ee1d0'
  }]
})
```

### Benefits of WalletConnect Integration

1. **Seamless Desktop Experience**: No need to switch between devices or scan QR codes
2. **Pre-filtered Options**: Wallet can show only available payment methods
3. **Faster Setup**: Immediate connection without manual scanning
4. **Real-time Communication**: Bidirectional updates between merchant and wallet
5. **Enhanced Security**: Maintains WalletConnect's security model
6. **Better UX**: Wallet can pre-populate transaction details
7. **Session Persistence**: Connection can persist across multiple payments

### Implementation Considerations

- **Fallback Support**: Always provide QR code option for mobile-first users
- **Session Management**: Handle connection timeouts and reconnection gracefully  
- **Error Handling**: Manage WalletConnect failures with appropriate fallbacks
- **Protocol Extensions**: Define custom WalletConnect methods for Solis integration
- **Cross-Chain**: WalletConnect naturally supports multi-chain wallet selection

This integration combines the best of both protocols: WalletConnect's seamless connectivity with Solis's structured payment flow.

## Example 7: POS Mode with BLE (Offline Wallet)

### Scenario
A customer is paying at a retail store. Their phone has poor or no cellular connectivity inside the store, but the POS terminal has a stable internet connection. The payment is completed entirely via Bluetooth Low Energy (BLE) without the wallet needing internet access.

### Prerequisites
- POS terminal with BLE capability (acts as BLE Central)
- Wallet app with BLE support (acts as BLE Peripheral) - works on both iOS and Android
- POS terminal connected to internet

### Architecture: HTTP Proxy over GATT

The POS terminal acts as an HTTP proxy - the wallet sends standard Solis API requests over BLE, and the POS forwards them to the backend. This ensures the HTTP API schemas remain the single source of truth.

```
┌─────────────────┐                    ┌─────────────────┐                    ┌──────────┐
│     WALLET      │        BLE         │       POS       │       HTTPS        │ Backend  │
│   (Peripheral)  │◄──────────────────►│    (Central)    │◄──────────────────►│          │
│    [offline]    │   HTTP Proxy       │    [online]     │   Same API calls   │          │
└─────────────────┘                    └─────────────────┘                    └──────────┘
```

**Key Design Principle**: Only 4 message types are needed. `SOLIS_REQUEST` and `SOLIS_RESPONSE` carry the same JSON bodies as the HTTP API endpoints.

| Message Type | Direction | Purpose |
|--------------|-----------|---------|
| `SOLIS_CONNECT` | POS → Wallet | Initial payment info + assets (bundled) |
| `SOLIS_CLAIM` | Wallet → POS | Wallet claims payment, establishes encryption |
| `SOLIS_REQUEST` | Wallet → POS | Generic HTTP request proxy |
| `SOLIS_RESPONSE` | POS → Wallet | Generic HTTP response proxy |
| `SOLIS_ERROR` | Either | Error notification |

### Implementation Steps

1. **POS Creates Payment Request**

The POS terminal creates a payment request for a $25 purchase:

```json
{
  "paymentId": "pos_BLE_789",
  "amount": "25.00",
  "currency": "USD",
  "merchantId": "retail_store_001",
  "expiresAt": 1698765432000
}
```

2. **Wallet Enters Payment Mode**

User opens wallet app and enables "Pay at Store" mode. The wallet starts advertising as a BLE Peripheral with the Solis service UUID:

```javascript
// Wallet advertises Solis payment service
const SOLIS_SERVICE_UUID = "0000FFF0-0000-1000-8000-00805F9B34FB"

await BLE.startAdvertising({
  serviceUUIDs: [SOLIS_SERVICE_UUID],
  localName: "Solis-" + walletIdHash.substring(0, 8)
})
```

3. **POS Scans and Connects**

POS scans for nearby Solis-enabled wallets. Multiple wallets may be advertising:

```javascript
// POS scans for Solis peripherals
const nearbyWallets = await BLE.scan({
  serviceUUIDs: [SOLIS_SERVICE_UUID],
  rssiThreshold: -65  // Only devices within ~1-2 meters
})

// Connect to all nearby wallets
for (const wallet of nearbyWallets) {
  await BLE.connect(wallet.id)
}
```

4. **POS Sends SOLIS_CONNECT**

POS sends the bundled payment information to all connected wallets. This includes payment details and supported assets (pre-fetched from backend):

```json
{
  "type": "SOLIS_CONNECT",
  "version": "1.1",
  "paymentId": "pos_BLE_789",
  "posPubkey": "02a1b2c3d4e5f6789abcdef0123456789abcdef01234567",
  "merchant": "Retail Store",
  "payment": {
    "description": "In-store purchase",
    "value": {
      "amount": "25.00",
      "asset": "USD"
    },
    "expiresAt": 1698765432000
  },
  "assets": [
    {
      "network": "BASE",
      "asset": "USDC",
      "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "decimals": 6,
      "time": 2000
    },
    {
      "network": "POLYGON",
      "asset": "USDC",
      "address": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
      "decimals": 6,
      "time": 5000
    }
  ]
}
```

5. **User Self-Selects (Broadcast + Claim)**

All nearby wallets receive the payment info and display it. **Only the actual customer** taps "Pay" on their device. This resolves the multi-wallet ambiguity:

```
┌─────────────────────────────────────┐
│         Retail Store                │
│         $25.00 USD                  │
│         In-store purchase           │
├─────────────────────────────────────┤
│  Pay with:                          │
│  ○ USDC on Base (~2s, $0.01 fee)    │
│  ○ USDC on Polygon (~5s, $0.02)     │
├─────────────────────────────────────┤
│         [ PAY $25.00 ]              │
└─────────────────────────────────────┘
```

6. **Wallet Sends SOLIS_CLAIM**

The paying customer's wallet sends a claim message with its ECDH public key:

```json
{
  "type": "SOLIS_CLAIM",
  "walletPubkey": "03fedcba9876543210fedcba9876543210fedcba98765432"
}
```

Both devices derive a shared secret using ECDH (secp256k1). All subsequent messages (`SOLIS_REQUEST`/`SOLIS_RESPONSE`) are encrypted with AES-128-GCM.

POS disconnects from other wallets and displays:
```
Payment claimed
Processing...
```

7. **Quote Request via SOLIS_REQUEST (Encrypted)**

Wallet sends a quote request using the generic HTTP proxy message:

```json
{
  "type": "SOLIS_REQUEST",
  "id": "req_001",
  "method": "POST",
  "path": "/quote/pos_BLE_789",
  "body": {
    "network": "BASE",
    "asset": "USDC"
  }
}
```

POS forwards to backend (`POST /quote/pos_BLE_789`) and returns:

```json
{
  "type": "SOLIS_RESPONSE",
  "id": "req_001",
  "status": 200,
  "body": {
    "id": "quote_BLE_456",
    "expiresAt": 1698765532000,
    "network": "BASE",
    "value": {
      "amount": "25.00",
      "asset": "USDC"
    }
  }
}
```

8. **Intent Request via SOLIS_REQUEST (Encrypted)**

Wallet requests payment intent:

```json
{
  "type": "SOLIS_REQUEST",
  "id": "req_002",
  "method": "POST",
  "path": "/intent/pos_BLE_789",
  "body": {
    "network": "BASE",
    "asset": "USDC",
    "quoteId": "quote_BLE_456"
  }
}
```

POS returns destination address:

```json
{
  "type": "SOLIS_RESPONSE",
  "id": "req_002",
  "status": 200,
  "body": {
    "network": "BASE",
    "value": {
      "amount": "25.00",
      "asset": "USDC"
    },
    "destination": {
      "type": "CRYPTO",
      "details": {
        "address": "0xB180731051bb0116DA91Ab7F1bf3cE1bD7C02Fd4"
      }
    }
  }
}
```

9. **Wallet Signs Transaction Locally**

The wallet constructs and signs the transaction locally (no internet required):

```javascript
// Wallet constructs transaction
const tx = {
  to: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC contract
  data: encodeTransfer(
    "0xB180731051bb0116DA91Ab7F1bf3cE1bD7C02Fd4", // destination
    parseUnits("25.00", 6) // amount in smallest unit
  ),
  chainId: 8453, // Base
  // ... gas parameters
}

// User approves, wallet signs
const signedTx = await wallet.signTransaction(tx)
```

10. **Wallet Notifies via SOLIS_REQUEST (Encrypted)**

Wallet sends signed transaction to POS for broadcast:

```json
{
  "type": "SOLIS_REQUEST",
  "id": "req_003",
  "method": "POST",
  "path": "/notify/pos_BLE_789",
  "body": {
    "hash": "0x7b65b75d204abed71587c9e519a89277766ee1d0",
    "transaction": "0x02f87382210580843b9aca00852e90edd00082520894...",
    "sender": {
      "address": "0xae2Fc483527B8EF99EB5D9B44875F005ba1FaE13"
    }
  }
}
```

11. **POS Broadcasts and Responds**

POS broadcasts to blockchain and sends confirmation:

```json
{
  "type": "SOLIS_RESPONSE",
  "id": "req_003",
  "status": 200,
  "body": {}
}
```

12. **Payment Complete**

- Wallet displays success with transaction hash
- POS terminal shows "Payment Received"
- Receipt printed/emailed to customer
- Total time: ~360ms from claim to broadcast

### Timing Breakdown

| Phase | Duration | Cumulative |
|-------|----------|------------|
| BLE Advertisement → POS Detection | ~50ms | 50ms |
| POS Connect + SOLIS_CONNECT | ~100ms | 150ms |
| User Reviews + Taps "Pay" | User action | - |
| SOLIS_CLAIM + ECDH Key Exchange | ~30ms | 180ms |
| Quote Request/Response | ~60ms | 240ms |
| Intent Request/Response | ~60ms | 300ms |
| Sign Transaction (local) | ~20ms | 320ms |
| Notify + Broadcast | ~40ms | 360ms |

**Total protocol overhead: ~360ms** (excluding user review time)

### Sequence Diagram

```
Customer Wallet                    POS Terminal                     Backend
(BLE Peripheral)                   (BLE Central)
     │                                  │                              │
     │◄═══ Advertise Solis Service ════►│                              │
     │                                  │                              │
     │◄──────── BLE Connect ───────────►│                              │
     │                                  │                              │
     │◄── SOLIS_CONNECT ────────────────│  (payment + assets bundled)  │
     │                                  │                              │
     │   [User reviews, taps "Pay"]     │                              │
     │                                  │                              │
     │─── SOLIS_CLAIM ─────────────────►│                              │
     │      (+ walletPubkey)            │   [POS disconnects others]   │
     │                                  │                              │
     │═══════ Encrypted Channel ════════│                              │
     │                                  │                              │
     │─── SOLIS_REQUEST ───────────────►│──── POST /quote/... ────────►│
     │     {method: POST, path: /quote} │                              │
     │◄── SOLIS_RESPONSE ───────────────│◄─── Quote ───────────────────│
     │     {status: 200, body: {...}}   │                              │
     │                                  │                              │
     │─── SOLIS_REQUEST ───────────────►│──── POST /intent/... ───────►│
     │     {method: POST, path: /intent}│                              │
     │◄── SOLIS_RESPONSE ───────────────│◄─── Destination Address ─────│
     │     {status: 200, body: {...}}   │                              │
     │                                  │                              │
     │   [Wallet signs tx locally]      │                              │
     │                                  │                              │
     │─── SOLIS_REQUEST ───────────────►│──── Broadcast + Notify ─────►│
     │     {method: POST, path: /notify}│                              │
     │◄── SOLIS_RESPONSE ───────────────│◄─── Success ─────────────────│
     │     {status: 200, body: {}}      │                              │
     │                                  │                              │
     ▼                                  ▼                              ▼
```

### Error Handling

Errors are returned via `SOLIS_RESPONSE` with appropriate HTTP status codes, or via `SOLIS_ERROR` for transport-level issues:

**HTTP-level error (in SOLIS_RESPONSE):**
```json
{
  "type": "SOLIS_RESPONSE",
  "id": "req_001",
  "status": 409,
  "body": {
    "error": {
      "code": "UNSUPPORTED_ASSET",
      "message": "The requested asset is not supported for this payment"
    }
  }
}
```

**Transport-level error (SOLIS_ERROR):**
```json
{
  "type": "SOLIS_ERROR",
  "code": "SESSION_EXPIRED",
  "message": "BLE session has expired. Please reconnect."
}
```

BLE-specific error codes:
- `PAYMENT_CLAIMED` - Another wallet already claimed this payment
- `SESSION_EXPIRED` - BLE session timed out
- `PROXY_ERROR` - POS failed to reach backend
- `BROADCAST_FAILED` - Transaction broadcast to chain failed

### Security Notes

1. **ECDH Key Exchange**: Session encryption established after SOLIS_CLAIM prevents eavesdropping
2. **Broadcast + Claim**: User self-selection resolves multi-wallet ambiguity naturally
3. **RSSI Filtering**: POS only connects to devices within ~1-2 meters (-65 dBm threshold)
4. **Signed Transaction**: POS only receives signed tx - cannot modify or forge
5. **Wallet Retains Tx**: Wallet keeps signed transaction for resubmission if needed
6. **Address Verification**: Wallets should display destination address for user verification

### Privacy Considerations

1. **Rotating BLE Addresses**: Wallets use rotating private addresses (standard BLE privacy)
2. **Rotating Wallet ID Hash**: The advertised wallet ID hash rotates periodically
3. **Opt-in Advertising**: Wallet only advertises when user explicitly enables "Pay at Store"
4. **No Tracking**: POS cannot correlate payments across sessions

### Platform Compatibility

| Platform | BLE Peripheral Support | Notes |
|----------|------------------------|-------|
| iOS 13+ | ✅ Full support | CoreBluetooth CBPeripheralManager |
| Android 5.0+ | ✅ Full support | BluetoothLeAdvertiser |
| Android (no BLE 5.0) | ⚠️ Limited | May have shorter advertising range |

### Benefits

- **Zero Internet Required**: Wallet operates completely offline
- **Universal Compatibility**: Works on both iOS and Android (no HCE required)
- **Fast**: ~360ms protocol overhead, sub-second total transaction time
- **Single Source of Truth**: HTTP API schemas are authoritative - no schema duplication
- **Familiar UX**: Similar to Apple Pay / Google Pay experience
- **Secure**: Encrypted channel + signed transactions + RSSI proximity filtering
- **Natural Multi-Wallet Handling**: Broadcast + Claim lets users self-select