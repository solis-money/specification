# UPP POS Mode: BLE HTTP Proxy Technical Specification

This document provides the complete technical specification for implementing POS Mode using Bluetooth Low Energy (BLE) as an HTTP proxy, enabling UPP payments when the user's wallet has no internet connectivity.

## Overview

POS Mode allows a wallet application to complete UPP payments entirely offline by using the POS terminal as an HTTP proxy. The wallet sends standard UPP API requests over BLE, and the POS forwards them to the backend. This design ensures the HTTP API schemas remain the single source of truth.

```
┌─────────────────┐                    ┌─────────────────┐                    ┌──────────┐
│     WALLET      │        BLE         │       POS       │       HTTPS        │ Backend  │
│   (Peripheral)  │◄──────────────────►│    (Central)    │◄──────────────────►│          │
│    [offline]    │   HTTP Proxy       │    [online]     │   Same API calls   │          │
└─────────────────┘                    └─────────────────┘                    └──────────┘
```

**Key Architecture Decisions**:
1. **Wallet as BLE Peripheral**: Ensures iOS and Android compatibility (no HCE required)
2. **HTTP Proxy Model**: Only 4 message types needed; HTTP API is the single source of truth
3. **Broadcast + Claim**: Resolves multi-wallet ambiguity through user self-selection

## BLE Role Assignment

| Device | BLE Role | Responsibilities |
|--------|----------|------------------|
| **Wallet** | Peripheral | Advertises UPP service, receives connections, sends HTTP requests |
| **POS** | Central | Scans for wallets, initiates connections, proxies HTTP requests to backend |

This role assignment is critical for iOS compatibility. iOS does not support NFC Host Card Emulation (HCE), but fully supports BLE Peripheral mode via CoreBluetooth.

## Message Types

The HTTP Proxy model uses only 4 message types (plus error):

| Type | Direction | Encrypted | Purpose |
|------|-----------|-----------|---------|
| `UPP_CONNECT` | POS → Wallet | No | Initial payment info + POS session key |
| `UPP_CLAIM` | Wallet → POS | No | Wallet claims payment + wallet session key |
| `UPP_REQUEST` | Wallet → POS | Yes | HTTP request proxy (method, path, body) |
| `UPP_RESPONSE` | POS → Wallet | Yes | HTTP response proxy (status, body) |
| `UPP_ERROR` | Either | Conditional | Transport-level error notification |

**Key Benefit**: `UPP_REQUEST` and `UPP_RESPONSE` carry the same JSON bodies as the HTTP API endpoints. No schema duplication.

## Transport Layer

### BLE Specifications

| Property | Value |
|----------|-------|
| GATT Service UUID | `0000FFF0-0000-1000-8000-00805F9B34FB` |
| Request Characteristic | `0000FFF1-0000-1000-8000-00805F9B34FB` |
| Response Characteristic | `0000FFF2-0000-1000-8000-00805F9B34FB` |
| MTU | Request 512 bytes minimum |
| Advertising Interval | 20-30ms (fast mode during payment) |
| Connection Interval | 7.5-15ms (optimized for low latency) |
| Supervision Timeout | 4 seconds |

### GATT Service Definition

```
Service: UPP Payment Service
UUID: 0000FFF0-0000-1000-8000-00805F9B34FB

Characteristics:

  1. Request Characteristic (POS → Wallet)
     UUID: 0000FFF1-0000-1000-8000-00805F9B34FB
     Properties: Write, Write Without Response
     Permissions: Open
     Description: POS writes UPP_CONNECT and UPP_RESPONSE messages here

  2. Response Characteristic (Wallet → POS)
     UUID: 0000FFF2-0000-1000-8000-00805F9B34FB
     Properties: Notify, Indicate
     Permissions: Open
     CCCD: Must be enabled after connection
     Description: Wallet sends UPP_CLAIM and UPP_REQUEST messages here
```

### Advertising Configuration

**Wallet Advertising Data:**
```
Flags: 0x06 (LE General Discoverable, BR/EDR Not Supported)
Complete List of 128-bit Service UUIDs: 0000FFF0-0000-1000-8000-00805F9B34FB
Local Name: "UPP-XXXXXXXX" (where XXXXXXXX is first 8 chars of rotating wallet ID hash)
```

**Scan Response (optional):**
```
Manufacturer Specific Data: Version info, capabilities
```

### RSSI-Based Proximity Filtering

POS terminals should use RSSI (Received Signal Strength Indicator) to filter connections:

| RSSI Range | Distance | Action |
|------------|----------|--------|
| -45 dBm or stronger | ~0.5m | High confidence, auto-connect |
| -45 to -65 dBm | ~1-2m | Normal payment range |
| -65 to -80 dBm | ~2-5m | Low confidence, may require user confirmation |
| Weaker than -80 dBm | >5m | Ignore, too far |

### Message Framing

For messages larger than MTU, use length-prefixed framing:

```
┌──────────┬──────────┬──────────┬───────────────────┐
│  Length  │ Total    │  Seq #   │      Payload      │
│ (2 bytes)│ Fragments│ (1 byte) │   (up to MTU-4)   │
└──────────┴──────────┴──────────┴───────────────────┘
```

- **Length**: Total message length across all fragments (big-endian uint16)
- **Total Fragments**: Number of fragments (1 byte, 0 = single fragment)
- **Seq #**: Fragment sequence number (0-indexed)
- **Payload**: Message fragment

## Session Security

### Key Exchange (Broadcast + Claim Model)

1. POS generates ephemeral ECDH key pair (secp256k1) per payment
2. POS includes public key in `UPP_CONNECT`
3. Wallet includes its public key in `UPP_CLAIM`
4. Both derive shared secret using ECDH
5. All `UPP_REQUEST` and `UPP_RESPONSE` messages are encrypted

```
// Key derivation
sharedSecret = ECDH(myPrivateKey, theirPublicKey)
sessionKey = HKDF-SHA256(
  salt: "upp-ble-v1.1",
  ikm: sharedSecret,
  info: paymentId || posPubkey || walletPubkey,
  length: 32
)
```

### Encryption Specification

All messages after `UPP_CLAIM` are encrypted with AES-128-GCM:

```
┌──────────┬──────────────┬──────────┐
│  Nonce   │  Ciphertext  │   Tag    │
│(12 bytes)│  (variable)  │(16 bytes)│
└──────────┴──────────────┴──────────┘
```

- **Nonce**: Counter-based (8 bytes counter + 4 bytes random per session)
- **Tag**: AES-GCM authentication tag

### Replay Prevention

1. Nonce counter increments monotonically
2. Reject messages with nonce ≤ last received
3. Session expires after 2 minutes of inactivity
4. Each payment ID can only be claimed once

## Message Protocol

### Message Format

All messages are JSON-encoded with a `type` field:

```json
{
  "type": "UPP_MESSAGE_TYPE",
  ...message fields...
}
```

### UPP_CONNECT

Sent by POS to wallet after BLE connection. Bundles payment info and assets to avoid extra round-trips.

```typescript
interface UPPConnect {
  type: "UPP_CONNECT";
  version: string;              // Protocol version (e.g., "1.1")
  paymentId: string;            // Unique payment identifier
  posPubkey: string;            // ECDH public key (secp256k1, compressed, hex)
  merchant: string;             // Merchant display name
  payment: {
    description: string;        // Payment description
    value: {
      amount: string;           // Amount
      asset: string;            // Currency (e.g., "USD", "USDC")
    };
    expiresAt: number;          // Expiration (ms since epoch)
  };
  assets: Array<{               // Supported payment options
    network: string;
    asset: string;
    address: string | null;
    decimals: number;
    time: number;               // Estimated confirmation time (ms)
  }>;
}
```

**Example:**
```json
{
  "type": "UPP_CONNECT",
  "version": "1.1",
  "paymentId": "pos_abc123",
  "posPubkey": "02a1b2c3d4e5f6789abcdef0123456789abcdef01234567",
  "merchant": "Coffee Shop",
  "payment": {
    "description": "Order #789 - 2x Latte",
    "value": {
      "amount": "12.50",
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

### UPP_CLAIM

Sent by wallet to claim the payment and establish encrypted session.

```typescript
interface UPPClaim {
  type: "UPP_CLAIM";
  walletPubkey: string;         // Wallet's ECDH public key (compressed, hex)
}
```

**Example:**
```json
{
  "type": "UPP_CLAIM",
  "walletPubkey": "03fedcba9876543210fedcba9876543210fedcba98765432"
}
```

### UPP_REQUEST

Generic HTTP request proxy from wallet to POS. Encrypted after claim.

```typescript
interface UPPRequest {
  type: "UPP_REQUEST";
  id: string;                   // Request ID for matching responses
  method: "GET" | "POST";       // HTTP method
  path: string;                 // API path (e.g., "/quote/pos_123")
  body?: object;                // Request body (same as HTTP API)
}
```

**Example (Quote Request):**
```json
{
  "type": "UPP_REQUEST",
  "id": "req_001",
  "method": "POST",
  "path": "/quote/pos_abc123",
  "body": {
    "network": "BASE",
    "asset": "USDC"
  }
}
```

**Example (Intent Request):**
```json
{
  "type": "UPP_REQUEST",
  "id": "req_002",
  "method": "POST",
  "path": "/intent/pos_abc123",
  "body": {
    "network": "BASE",
    "asset": "USDC",
    "quoteId": "quote_xyz789"
  }
}
```

**Example (Notify with Transaction):**
```json
{
  "type": "UPP_REQUEST",
  "id": "req_003",
  "method": "POST",
  "path": "/notify/pos_abc123",
  "body": {
    "hash": "0x7b65b75d204abed71587c9e519a89277766ee1d0",
    "transaction": "0x02f87382210580843b9aca00...",
    "sender": {
      "address": "0xae2Fc483527B8EF99EB5D9B44875F005ba1FaE13"
    }
  }
}
```

### UPP_RESPONSE

Generic HTTP response proxy from POS to wallet. Encrypted.

```typescript
interface UPPResponse {
  type: "UPP_RESPONSE";
  id: string;                   // Matches request ID
  status: number;               // HTTP status code
  body?: object;                // Response body (same as HTTP API)
}
```

**Example (Quote Response):**
```json
{
  "type": "UPP_RESPONSE",
  "id": "req_001",
  "status": 200,
  "body": {
    "id": "quote_xyz789",
    "expiresAt": 1698765532000,
    "network": "BASE",
    "value": {
      "amount": "12.50",
      "asset": "USDC"
    }
  }
}
```

**Example (Error Response):**
```json
{
  "type": "UPP_RESPONSE",
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

### UPP_ERROR

Transport-level error (BLE session issues, proxy failures).

```typescript
interface UPPError {
  type: "UPP_ERROR";
  code: string;
  message: string;
}
```

**BLE Error Codes:**

| Code | Description |
|------|-------------|
| `PAYMENT_NOT_FOUND` | Payment ID does not exist |
| `PAYMENT_EXPIRED` | Payment has expired |
| `PAYMENT_CLAIMED` | Payment already claimed by another wallet |
| `SESSION_EXPIRED` | BLE session has expired |
| `PROXY_ERROR` | POS failed to reach backend |
| `BROADCAST_FAILED` | Transaction broadcast to chain failed |

**Example:**
```json
{
  "type": "UPP_ERROR",
  "code": "PAYMENT_CLAIMED",
  "message": "This payment has already been claimed by another wallet"
}
```

## Implementation Guide

### POS Terminal Implementation

```
┌─────────────────────────────────────────────────────────────────┐
│                        POS Terminal                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │  BLE Central  │  │  HTTP Client  │  │  Blockchain   │       │
│  │   (Scanner)   │  │               │  │    Client     │       │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘       │
│          │                  │                  │                │
│          └──────────────────┼──────────────────┘                │
│                             │                                    │
│                    ┌────────▼────────┐                          │
│                    │   HTTP Proxy    │                          │
│                    │   Controller    │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│                    ┌────────▼────────┐                          │
│                    │  Session State  │                          │
│                    │    Manager      │                          │
│                    └─────────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

**Required Components:**
1. BLE Central capability (scanning + connection)
2. HTTPS client for backend communication
3. Blockchain RPC client for transaction broadcast (optional - can use backend)
4. Secure session state management
5. ECDH key generation and AES-GCM encryption

**Pseudocode:**

```python
class UPPPOSController:
    def __init__(self):
        self.ble = BLECentral()
        self.http = HTTPClient()
        self.blockchain = BlockchainClient()
        self.sessions = {}
        self.current_payment = None

    def start_payment(self, payment_id: str):
        # Fetch payment details and assets from backend
        payment_info = self.http.get(f"/{payment_id}")
        assets = self.http.get(f"/assets/{payment_id}")

        # Generate ECDH keypair
        keypair = generate_ecdh_keypair()

        self.current_payment = Payment(
            id=payment_id,
            info=payment_info,
            assets=assets,
            keypair=keypair
        )

        # Start scanning for UPP wallets
        self.ble.start_scan(
            service_uuid=UPP_SERVICE_UUID,
            rssi_threshold=-65
        )

    def on_wallet_discovered(self, peripheral):
        # Connect to the wallet
        connection = self.ble.connect(peripheral)

        # Subscribe to response characteristic
        connection.subscribe(RESPONSE_CHAR_UUID, self.on_wallet_message)

        # Send UPP_CONNECT (bundled payment + assets)
        connect_msg = {
            "type": "UPP_CONNECT",
            "version": "1.1",
            "paymentId": self.current_payment.id,
            "posPubkey": self.current_payment.keypair.public_hex,
            "merchant": self.current_payment.info["merchant"],
            "payment": {
                "description": self.current_payment.info["description"],
                "value": self.current_payment.info["value"],
                "expiresAt": self.current_payment.info["expiresAt"]
            },
            "assets": self.current_payment.assets
        }
        connection.write(REQUEST_CHAR_UUID, json.dumps(connect_msg))

        # Track connection
        self.sessions[peripheral.id] = Session(
            connection=connection,
            payment=self.current_payment,
            state="AWAITING_CLAIM"
        )

    def on_wallet_message(self, peripheral_id: str, data: bytes):
        session = self.sessions.get(peripheral_id)
        if not session:
            return

        # Decrypt if session is established
        if session.shared_secret:
            message = decrypt_aes_gcm(session.shared_secret, data)
        else:
            message = json.loads(data)

        if message["type"] == "UPP_CLAIM":
            self.handle_claim(session, message)
        elif message["type"] == "UPP_REQUEST":
            self.handle_request(session, message)

    def handle_claim(self, session: Session, message: dict):
        # Derive shared secret
        session.wallet_pubkey = message["walletPubkey"]
        session.shared_secret = derive_shared_secret(
            session.payment.keypair.private,
            session.wallet_pubkey,
            session.payment.id
        )
        session.state = "CLAIMED"

        # Disconnect other wallets for this payment
        for sid, s in list(self.sessions.items()):
            if s.payment.id == session.payment.id and sid != peripheral_id:
                self.send_error(s, "PAYMENT_CLAIMED", "Payment claimed by another wallet")
                s.connection.disconnect()
                del self.sessions[sid]

        # Display on POS
        self.ui.show_claimed()

    def handle_request(self, session: Session, message: dict):
        request_id = message["id"]
        method = message["method"]
        path = message["path"]
        body = message.get("body")

        try:
            # Proxy to backend
            if method == "GET":
                result = self.http.get(path)
            else:
                result = self.http.post(path, body)

            # Special handling for /notify - broadcast transaction if provided
            if "/notify/" in path and body and body.get("transaction"):
                tx_hash = self.blockchain.broadcast(
                    session.payment.info.get("network", "BASE"),
                    body["transaction"]
                )
                # Update hash in response if broadcast succeeded
                if tx_hash and not body.get("hash"):
                    body["hash"] = tx_hash

            response = {
                "type": "UPP_RESPONSE",
                "id": request_id,
                "status": 200,
                "body": result
            }

        except HTTPError as e:
            response = {
                "type": "UPP_RESPONSE",
                "id": request_id,
                "status": e.status_code,
                "body": e.body
            }

        except Exception as e:
            response = {
                "type": "UPP_RESPONSE",
                "id": request_id,
                "status": 500,
                "body": {
                    "error": {
                        "code": "INTERNAL_SERVER_ERROR",
                        "message": str(e)
                    }
                }
            }

        self.send_encrypted(session, response)

    def send_encrypted(self, session: Session, message: dict):
        encrypted = encrypt_aes_gcm(session.shared_secret, json.dumps(message))
        session.connection.write(REQUEST_CHAR_UUID, encrypted)

    def send_error(self, session: Session, code: str, message: str):
        error = {
            "type": "UPP_ERROR",
            "code": code,
            "message": message
        }
        # Send unencrypted if session not established
        if session.shared_secret:
            self.send_encrypted(session, error)
        else:
            session.connection.write(REQUEST_CHAR_UUID, json.dumps(error))
```

### Wallet Implementation

**Required Components:**
1. BLE Peripheral capability (advertising + GATT server)
2. Local transaction construction and signing
3. Secure key storage
4. ECDH key generation and AES-GCM encryption

**Platform Support:**

| Platform | BLE Peripheral API | Notes |
|----------|-------------------|-------|
| iOS 13+ | CoreBluetooth CBPeripheralManager | Full support |
| Android 5.0+ | BluetoothLeAdvertiser | Full support |
| Android 4.3-4.4 | Limited | Central only, no Peripheral |

**Pseudocode:**

```python
class UPPWallet:
    def __init__(self):
        self.ble = BLEPeripheral()
        self.keychain = SecureKeychain()
        self.current_session = None
        self.request_counter = 0

    def enable_pos_payment_mode(self):
        # Generate rotating wallet ID hash
        wallet_id_hash = sha256(self.keychain.primary_address)[:8]

        # Set up GATT service
        self.ble.add_service(UPP_SERVICE_UUID)
        self.ble.add_characteristic(
            REQUEST_CHAR_UUID,
            properties=["write", "writeWithoutResponse"],
            on_write=self.on_request_write
        )
        self.ble.add_characteristic(
            RESPONSE_CHAR_UUID,
            properties=["notify", "indicate"]
        )

        # Start advertising
        self.ble.start_advertising(
            service_uuids=[UPP_SERVICE_UUID],
            local_name=f"UPP-{wallet_id_hash}"
        )

        self.ui.show_waiting_for_pos()

    def on_request_write(self, central_id: str, data: bytes):
        # Decrypt if session established
        if self.current_session and self.current_session.shared_secret:
            message = decrypt_aes_gcm(self.current_session.shared_secret, data)
        else:
            message = json.loads(data)

        if message["type"] == "UPP_CONNECT":
            self.handle_connect(central_id, message)
        elif message["type"] == "UPP_RESPONSE":
            self.handle_response(message)
        elif message["type"] == "UPP_ERROR":
            self.handle_error(message)

    def handle_connect(self, central_id: str, message: dict):
        # Store session info
        self.current_session = Session(
            central_id=central_id,
            payment_id=message["paymentId"],
            pos_pubkey=message["posPubkey"],
            merchant=message["merchant"],
            payment=message["payment"],
            assets=message["assets"]
        )

        # Display payment to user
        self.ui.show_payment_request(
            merchant=message["merchant"],
            description=message["payment"]["description"],
            amount=message["payment"]["value"]["amount"],
            currency=message["payment"]["value"]["asset"],
            options=message["assets"]
        )

    def on_user_approve(self, selected_network: str, selected_asset: str):
        # Generate ECDH keypair and derive shared secret
        keypair = generate_ecdh_keypair()
        self.current_session.keypair = keypair
        self.current_session.shared_secret = derive_shared_secret(
            keypair.private,
            self.current_session.pos_pubkey,
            self.current_session.payment_id
        )
        self.current_session.selected_network = selected_network
        self.current_session.selected_asset = selected_asset

        # Send claim (unencrypted - establishes session)
        claim = {
            "type": "UPP_CLAIM",
            "walletPubkey": keypair.public_hex
        }
        self.ble.notify(RESPONSE_CHAR_UUID, json.dumps(claim))

        # Request quote after brief delay
        self.schedule_quote_request()

    def schedule_quote_request(self):
        sleep(50)  # 50ms for claim to be processed
        self.send_request("POST", f"/quote/{self.current_session.payment_id}", {
            "network": self.current_session.selected_network,
            "asset": self.current_session.selected_asset
        })

    def handle_response(self, message: dict):
        request_id = message["id"]
        status = message["status"]
        body = message.get("body", {})

        if status >= 400:
            # Handle error
            error = body.get("error", {})
            self.ui.show_error(error.get("message", "Request failed"))
            return

        # Determine what type of response this is based on body content
        if "destination" in body:
            # Intent response - show confirmation
            self.current_session.intent = body
            self.ui.show_confirmation(
                address=body["destination"]["details"]["address"],
                amount=body["value"]["amount"],
                asset=body["value"]["asset"],
                network=body["network"]
            )
        elif "id" in body and request_id.startswith("quote"):
            # Quote response - request intent
            self.current_session.quote_id = body["id"]
            self.send_request("POST", f"/intent/{self.current_session.payment_id}", {
                "network": self.current_session.selected_network,
                "asset": self.current_session.selected_asset,
                "quoteId": body["id"]
            })
        else:
            # Notify response - payment complete
            self.ui.show_success(tx_hash=self.current_session.pending_tx_hash)
            self.cleanup_session()

    def on_user_confirm_transaction(self):
        intent = self.current_session.intent

        # Build and sign transaction locally
        tx = self.build_transaction(
            network=intent["network"],
            to=intent["destination"]["details"]["address"],
            amount=intent["value"]["amount"],
            asset=intent["value"]["asset"]
        )
        signed_tx = self.keychain.sign(tx)
        tx_hash = compute_tx_hash(signed_tx)

        # Store for display
        self.current_session.pending_tx_hash = tx_hash

        # Send notify with signed transaction
        self.send_request("POST", f"/notify/{self.current_session.payment_id}", {
            "hash": tx_hash,
            "transaction": signed_tx.hex(),
            "sender": {
                "address": self.keychain.primary_address
            }
        })

        self.ui.show_submitting()

    def send_request(self, method: str, path: str, body: dict = None):
        self.request_counter += 1
        request = {
            "type": "UPP_REQUEST",
            "id": f"req_{self.request_counter:03d}",
            "method": method,
            "path": path
        }
        if body:
            request["body"] = body

        encrypted = encrypt_aes_gcm(
            self.current_session.shared_secret,
            json.dumps(request)
        )
        self.ble.notify(RESPONSE_CHAR_UUID, encrypted)

    def handle_error(self, message: dict):
        self.ui.show_error(message["message"], code=message["code"])
        if message["code"] == "PAYMENT_CLAIMED":
            self.cleanup_session()

    def cleanup_session(self):
        self.current_session = None
        self.request_counter = 0
        # Resume advertising for next payment
        self.enable_pos_payment_mode()
```

## Security Considerations

### Threat Model

| Threat | Attack Vector | Mitigation |
|--------|---------------|------------|
| Eavesdropping | Passive BLE interception | AES-128-GCM encryption after claim |
| MITM | Rogue device intercepts | ECDH key exchange, user verification of address |
| Replay | Reuse captured messages | Nonce-based encryption, session binding |
| Address Substitution | POS provides fake address | Wallet displays address for user verification |
| Transaction Withholding | POS doesn't broadcast | Wallet retains signed tx for later submission |
| Multi-Claim | Multiple wallets claim | POS accepts first claim, rejects subsequent |
| Tracking | BLE address correlation | Rotating BLE addresses, rotating wallet ID hash |

### Security Requirements

1. **Broadcast + Claim Model**: User self-selection resolves multi-wallet ambiguity
2. **RSSI Filtering**: POS only accepts connections from nearby devices
3. **Forward Secrecy**: Ephemeral ECDH keys per payment
4. **Authenticated Encryption**: AES-128-GCM for all post-claim messages
5. **Replay Prevention**: Nonce-based encryption with monotonic counter
6. **User Verification**: Display destination address before signing
7. **Transaction Retention**: Wallet keeps signed tx until confirmed

### Privacy Requirements

1. **Rotating BLE Addresses**: Use BLE privacy features (RPA)
2. **Rotating Wallet ID**: Hash rotates periodically (e.g., daily)
3. **Opt-in Advertising**: Wallet only advertises when user enables payment mode
4. **No Persistent Identifiers**: POS cannot correlate payments across sessions

### Best Practices

**For POS Terminals:**
- Accept only the first claim per payment
- Disconnect non-claiming wallets after claim received
- Clear session state after completion or timeout (2 minutes)
- Log transactions for audit (without sensitive data)
- Implement rate limiting on payment creation
- Validate all message formats before processing

**For Wallets:**
- Never auto-approve transactions
- Display full destination address
- Show network and asset clearly
- Require explicit user confirmation before signing
- Store signed transactions securely for retry
- Implement session timeout (2 minutes)
- Clear session on disconnect

## Timing Optimization

### Target Latency Budget

| Phase | Target | Notes |
|-------|--------|-------|
| BLE Discovery | 50ms | 20-30ms advertising interval |
| Connection | 50ms | Optimized connection parameters |
| UPP_CONNECT Delivery | 50ms | Pre-fetched data bundled |
| User Review | Variable | User-dependent |
| UPP_CLAIM + Key Exchange | 30ms | Local computation |
| Quote Request/Response | 60ms | Backend latency included |
| Intent Request/Response | 60ms | Backend latency included |
| Signing | 20ms | Local computation |
| Notify + Broadcast | 40ms | Chain broadcast included |
| **Total (protocol)** | **~360ms** | Excluding user review |

### Optimization Techniques

1. **Pre-fetch + Bundle**: POS fetches payment info and assets before any wallet connects, bundles in UPP_CONNECT
2. **Fast Advertising**: 20-30ms interval during active payment
3. **Connection Parameters**: 7.5-15ms connection interval, 0 slave latency
4. **Message Pipelining**: Wallet sends quote request immediately after claim

## Testing

### Test Scenarios

1. **Happy Path**: Complete payment with single wallet
2. **Multi-Wallet**: Multiple wallets receive UPP_CONNECT, one claims
3. **Claim Race**: Two wallets attempt to claim simultaneously
4. **Connection Loss**: BLE disconnect during flow
5. **Session Timeout**: 2+ minutes between steps
6. **Invalid Messages**: Malformed or out-of-sequence messages
7. **Encryption Errors**: Tampered or replayed messages
8. **Backend Errors**: Various HTTP error codes proxied correctly
9. **Broadcast Failure**: Transaction rejected by chain

### Platform Testing Matrix

| Platform | Peripheral Mode | Central Mode | Notes |
|----------|-----------------|--------------|-------|
| iOS 13+ | ✅ | ✅ | Full support |
| iOS 12 | ✅ | ✅ | Background limitations |
| Android 5.0+ | ✅ | ✅ | Full support |
| Android 4.3-4.4 | ❌ | ✅ | No Peripheral support |

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-01 | Initial NFC/BLE specification |
| 1.1 | 2024-12 | BLE-only "Broadcast + Claim" model for iOS compatibility |
| 1.2 | 2024-12 | HTTP Proxy model - reduced to 4 message types, HTTP API as single source of truth |
