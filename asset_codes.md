# Solis Network and Asset Codes

## Abstract

This document provides a centralized, recommended list of standardized codes for blockchain networks and crypto assets to be used within Solis. Its purpose is to ensure consistency and interoperability among different Solis implementations, minimizing ambiguity when referring to specific networks or assets.

## Status of This Document

This is an initial draft. This list is expected to evolve and be maintained by the Solis community. Future iterations might include an API for programmatic access to these codes.

## Introduction

To ensure that terms like "ETHEREUM" for the network or "USDC" for an asset mean the same thing across all Solis-compliant wallets and merchant systems, this document establishes a baseline set of codes. While the Solis specification defines the structure for exchanging payment information, these standardized codes define the *values* for common networks and assets.

Implementers should refer to this document when specifying `network` fields and `asset` fields in Solis API calls and responses.

## Network Codes

The following table lists the standardized codes for blockchain networks supported by Solis. The `Solis Network Code` should be used in the `network` field of Solis API messages.

| Network Name          | Solis Network Code | Notes                                       |
|-----------------------|------------------|---------------------------------------------|
| Bitcoin               | `BITCOIN`        |                                             |
| Ethereum              | `ETHEREUM`       |                                             |
| Base                  | `BASE`           |                                             |
| Polygon PoS           | `POLYGON`        |                                             |
| Arbitrum One          | `ARBITRUM`       |                                             |
| Optimism              | `OPTIMISM`       |                                             |
| Solana                | `SOLANA`         |                                             |
| Binance Smart Chain   | `BINANCE`        |                                             |
| Avalanche C-Chain     | `AVALANCHE`      |                                             |
| Tron                  | `TRON`           |                                             |
| Litecoin              | `LITECOIN`       |                                             |
| Dogecoin              | `DOGECOIN`       |                                             |
| Cardano               | `CARDANO`        |                                             |

## Asset Codes

The following table lists the standardized codes for crypto assets supported by Solis. The `Solis Asset Code` should be used in the `asset` field within Solis API messages. These codes generally correspond to the widely recognized ticker symbol for the asset.

It's important to note that the same asset (e.g., USDC) can exist on multiple networks. The Solis protocol distinguishes these by the combination of the `network` code and the `asset` code, along with the network-specific contract address when applicable.

| Asset Name        | Solis Asset Code | Official Website / Source (Informational) |
|-------------------|----------------|-------------------------------------------|
| Bitcoin           | `BTC`          | bitcoin.org                               |
| Ether             | `ETH`          | ethereum.org                              |
| USD Coin          | `USDC`         | circle.com/usdc                           |
| Tether            | `USDT`         | tether.to                                 |
| Binance Coin      | `BNB`          | bnbchain.org                              |
| Solana            | `SOL`          | solana.com                                |
| Cardano           | `ADA`          | cardano.org                               |
| XRP               | `XRP`          | xrpl.org                                  |
| Dogecoin          | `DOGE`         | dogecoin.com                              |
| Polkadot          | `DOT`          | polkadot.network                          |
| Avalanche         | `AVAX`         | avax.network                              |
| Polygon (Matic)   | `POL`          | polygon.technology                        |
| Dai Stablecoin    | `DAI`          | makerdao.com                              |
| Litecoin          | `LTC`          | litecoin.org                              |

## Using Network and Asset Codes

When a merchant specifies supported assets, they would use these codes. For example:
```json
{
  "network": "BASE",  // From Network Codes list
  "asset": "USDC",    // From Asset Codes list
  "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // Specific to USDC on Base
  "decimals": 6
}
```

To indicate support for Ether on Ethereum:
```json
{
  "network": "ETHEREUM", // From Network Codes list
  "asset": "ETH",        // From Asset Codes list
  "address": null,       // Native asset
  "decimals": 18
}
```

## Contribution and Maintenance

This list of codes is intended to be a community-maintained resource. To propose additions or changes:

1.  Open an issue in the Solis repository detailing the proposed network or asset code.
2.  Provide justification, including official sources or widespread community usage for the code.
3.  A consensus-driven approach will be used to update this document.

The goal is to maintain a practical and widely accepted set of codes that promotes clarity and reduces integration friction for Solis implementers. 