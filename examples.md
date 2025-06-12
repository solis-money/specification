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
    "decimals": 6
  },
  {
    "network": "BASE",
    "asset": "USDC",
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "decimals": 6
  },
  {
    "network": "POLYGON",
    "asset": "USDC",
    "address": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
    "decimals": 6
  },
  {
    "network": "ARBITRUM",
    "asset": "USDC",
    "address": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "decimals": 6
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
  "expiresAt": 1698765932,
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
  "expiresAt": 1698765932,
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
    "decimals": 6
  },
  {
    "network": "BASE",
    "asset": "USDC",
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "decimals": 6
  },
  {
    "network": "POLYGON",
    "asset": "USDC",
    "address": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
    "decimals": 6
  },
  {
    "network": "ARBITRUM",
    "asset": "USDC",
    "address": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "decimals": 6
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
  "expiresAt": 1698765432
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
  "expiresAt": 1698765432,
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