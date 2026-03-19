# VelixoMarket Backend API Documentation

**Base URL**: `https://velixov2-production.up.railway.app` (production)

**Chain**: FOGO (Solana fork) — native token is FOGO, not SOL.

**Program IDs**:

- V1: `EttADML4h179CtQhVywPaRS4YcbxmMrEWDgwmQ5EcsNe`
- V2: `3SWFm1HcBtpVyC8gcVHRCXFV5RRSEcxU8dJ7A5dgapGk`

---

## Table of Contents

- [Getting Started](#getting-started)
- [Authentication](#authentication)
- [Rate Limiting](#rate-limiting)
- [Response Format](#response-format)
- [V1 vs V2 Endpoints](#v1-vs-v2-endpoints)
- [V1 Endpoints](#v1-endpoints)
  - [V1 Listings](#v1-listings)
  - [V1 Metadata](#v1-metadata)
  - [V1 Users](#v1-users)
  - [V1 Collections](#v1-collections)
  - [V1 Offers](#v1-offers)
  - [V1 Stats](#v1-stats)
  - [V1 Transactions](#v1-transactions)
- [V2 Endpoints](#v2-endpoints)
  - [V2 Transactions](#v2-transactions)
  - [V2 Collections](#v2-collections)
  - [V2 Activity](#v2-activity)
  - [V2 Stats](#v2-stats)
  - [V2 Offers](#v2-offers)
  - [V2 NFT](#v2-nft)
  - [V2 Owner](#v2-owner)
- [Admin Endpoints](#admin-endpoints)
- [Environment Variables](#environment-variables)
- [Caching](#caching)
- [Error Codes](#error-codes)

---

## Getting Started

### Prerequisites

- Node.js 18+
- npm or yarn

### Installation

```bash
cd velixoMarket-backend
npm install
```

### Running

```bash
# Development
npm run dev

# Production
npm start
```

The server starts on port `3001` by default.

### Quick Test

```bash
curl http://localhost:3001/health
```

```json
{
  "status": "ok",
  "timestamp": "2026-01-01T00:00:00.000Z",
  "uptime": 123.45
}
```

---

## Authentication

Most endpoints are **public** and require no authentication.

**Admin endpoints** require an API key via one of:

- Header: `X-API-Key: your-api-key`
- Header: `Authorization: Bearer your-api-key`

The key is configured via the `ADMIN_API_KEY` environment variable.

---

## Rate Limiting

| Route Category | Limit               |
| -------------- | ------------------- |
| General API    | 100 requests/min/IP |
| Stats routes   | 60 requests/min/IP  |
| Admin routes   | 10 requests/min/IP  |

Rate limit headers are included in responses. When exceeded, a `429 Too Many Requests` response is returned.

---

## Response Format

### Success

```json
{
  "success": true,
  "data": { ... },
  "count": 10,
  "message": "optional message"
}
```

### Error

```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human-readable error description"
}
```

### Common HTTP Status Codes

| Code | Meaning                                   |
| ---- | ----------------------------------------- |
| 200  | Success                                   |
| 400  | Bad request (invalid params/body)         |
| 401  | Unauthorized (missing or invalid API key) |
| 404  | Resource not found                        |
| 429  | Rate limit exceeded                       |
| 500  | Internal server error                     |

---

## V1 vs V2 Endpoints

The API has two versions running side by side:

| Feature              | V1 (`/api/...`)                                | V2 (`/api/v2/...`)                             |
| -------------------- | ---------------------------------------------- | ---------------------------------------------- |
| Program ID           | `EttADML4h179CtQhVywPaRS4YcbxmMrEWDgwmQ5EcsNe` | `3SWFm1HcBtpVyC8gcVHRCXFV5RRSEcxU8dJ7A5dgapGk` |
| NFT Standards        | Token Metadata only                            | Token Metadata **and** Metaplex Core           |
| NFT identifier field | `nftMint`                                      | `nftAddress`                                   |
| Collection field     | `collectionMint`                               | `collectionAddress`                            |
| Royalty enforcement  | No                                             | Yes (optional `royaltyRecipient` field)        |
| Activity history     | Via `/api/stats/sales`                         | Dedicated `/api/v2/activity/` endpoints        |

**Recommendation:** Use V2 endpoints for new integrations. V1 endpoints remain fully functional.

---

## Health & Info

#### `GET /health`

Health check endpoint.

```bash
curl http://localhost:3001/health
```

```json
{
  "status": "ok",
  "timestamp": "2026-01-01T00:00:00.000Z",
  "uptime": 123.45
}
```

---

#### `GET /api`

Returns a JSON summary of all available V1 and V2 API endpoints with descriptions.

```bash
curl http://localhost:3001/api
```

---

## V1 Endpoints

### V1 Listings

Fetch and manage V1 NFT listings on the marketplace.

#### `GET /api/listings`

Get all active V1 marketplace listings.

**Example:**

```bash
curl http://localhost:3001/api/listings
```

**Response:**

```json
{
  "success": true,
  "count": 42,
  "data": [
    {
      "address": "listing-pda-address",
      "seller": "seller-wallet-address",
      "mint": "nft-mint-address",
      "price": 1500000000,
      "isActive": true,
      "bump": 255,
      "nftMetadata": {
        "name": "DINO #123",
        "imageUrl": "https://...",
        "attributes": [...]
      }
    }
  ]
}
```

> **Note:** `price` is in lamports (1 FOGO = 1,000,000,000 lamports).

---

#### `GET /api/listings/user/:address`

Get all V1 listings created by a specific wallet.

**Path Parameters:**

| Param     | Type   | Description             |
| --------- | ------ | ----------------------- |
| `address` | string | Seller's wallet address |

**Example:**

```bash
curl http://localhost:3001/api/listings/user/ABC123...
```

---

#### `POST /api/listings/refresh`

Force refresh the V1 listings cache. Useful after on-chain changes.

**Example:**

```bash
curl -X POST http://localhost:3001/api/listings/refresh
```

**Response:**

```json
{
  "success": true,
  "message": "Listings refreshed",
  "count": 42
}
```

---

### V1 Metadata

Fetch NFT metadata (name, image, attributes, etc.).

#### `GET /api/metadata/:mint`

Get metadata for a single NFT.

**Path Parameters:**

| Param  | Type   | Description      |
| ------ | ------ | ---------------- |
| `mint` | string | NFT mint address |

**Example:**

```bash
curl http://localhost:3001/api/metadata/NFTMintAddress123...
```

**Response:**

```json
{
  "success": true,
  "data": {
    "name": "DINO #123",
    "symbol": "DINO",
    "description": "A unique DINO NFT",
    "imageUrl": "https://arweave.net/...",
    "uri": "https://arweave.net/metadata.json",
    "creator": "creator-address",
    "collectionMint": "collection-mint-address",
    "collectionVerified": true,
    "royaltyRecipient": "royalty-address",
    "attributes": [
      { "trait_type": "Background", "value": "Blue" },
      { "trait_type": "Rarity", "value": "Legendary" }
    ]
  }
}
```

---

#### `POST /api/metadata/batch`

Get metadata for multiple NFTs in a single request.

**Body:**

```json
{
  "mints": ["mint1...", "mint2...", "mint3..."]
}
```

> Maximum 50 mints per request.

**Example:**

```bash
curl -X POST http://localhost:3001/api/metadata/batch \
  -H "Content-Type: application/json" \
  -d '{"mints": ["mint1...", "mint2..."]}'
```

**Response:**

```json
{
  "success": true,
  "count": 2,
  "data": [
    { "mint": "mint1...", "metadata": { ... } },
    { "mint": "mint2...", "metadata": { ... }, "error": null }
  ]
}
```

---

### V1 Users

Fetch user profiles, NFTs, and listings.

#### `GET /api/users/:address/nfts`

Get all NFTs owned by a wallet.

**Path Parameters:**

| Param     | Type   | Description    |
| --------- | ------ | -------------- |
| `address` | string | Wallet address |

**Example:**

```bash
curl http://localhost:3001/api/users/WalletAddress.../nfts
```

**Response:**

```json
{
  "success": true,
  "count": 5,
  "data": [
    {
      "mint": "nft-mint-address",
      "name": "DINO #123",
      "imageUrl": "https://...",
      "collectionMint": "collection-address",
      "attributes": [...]
    }
  ]
}
```

---

#### `GET /api/users/:address/listings`

Get a user's active V1 listings. Same response format as `GET /api/listings`.

---

#### `GET /api/users/:address/profile`

Get a complete user profile combining NFTs and listings.

**Response:**

```json
{
  "success": true,
  "data": {
    "address": "wallet-address",
    "nfts": {
      "count": 5,
      "items": [...]
    },
    "listings": {
      "count": 2,
      "items": [...]
    }
  }
}
```

---

### V1 Collections

Fetch collection data, stats, listings, and offers.

#### `GET /api/collections`

Get all V1 collections.

**Query Parameters:**

| Param          | Type    | Default | Description                  |
| -------------- | ------- | ------- | ---------------------------- |
| `includeStats` | boolean | false   | Include volume/sales metrics |

**Example:**

```bash
curl "http://localhost:3001/api/collections?includeStats=true"
```

**Response:**

```json
{
  "success": true,
  "count": 3,
  "data": [
    {
      "collectionMint": "collection-address",
      "name": "DINO",
      "symbol": "DINO",
      "description": "The DINO collection by VelixoMarket",
      "imageUrl": "https://...",
      "verified": true,
      "creator": "creator-address",
      "volume24h": 150.5,
      "volume7d": 820.3,
      "sales24h": 12,
      "sales7d": 67
    }
  ]
}
```

---

#### `GET /api/collections/highest-bids`

Get the highest collection offer for every V1 collection.

**Response:**

```json
{
  "success": true,
  "data": {
    "dino-collection-mint": 2.5,
    "other-collection-mint": 1.8
  }
}
```

---

#### `GET /api/collections/:collectionMint`

Get a single V1 collection with stats.

**Path Parameters:**

| Param            | Type   | Description             |
| ---------------- | ------ | ----------------------- |
| `collectionMint` | string | Collection mint address |

**Response:**

```json
{
  "success": true,
  "data": {
    "collectionMint": "...",
    "name": "DINO",
    "floorPrice": 1.5,
    "topBid": 1.2,
    "volume24h": 150.5,
    "sales24h": 12,
    "listingsCount": 42,
    "updatedAt": "2026-01-01T00:00:00.000Z"
  }
}
```

---

#### `GET /api/collections/:collectionMint/listings`

Get all V1 listings in a collection.

---

#### `GET /api/collections/:collectionMint/offers`

Get all V1 collection-level offers.

**Response:**

```json
{
  "success": true,
  "count": 5,
  "data": [
    {
      "address": "offer-pda",
      "offerer": "wallet-address",
      "collectionMint": "...",
      "amountPerNft": 1500000000,
      "maxNfts": 3,
      "nftsPurchased": 1,
      "isActive": true,
      "isFulfilled": false,
      "expiresAt": 1700000000,
      "createdAt": 1699900000
    }
  ]
}
```

---

#### `GET /api/collections/:collectionMint/stats`

Get comprehensive V1 collection statistics.

**Response:**

```json
{
  "success": true,
  "data": {
    "collectionMint": "...",
    "name": "DINO",
    "symbol": "DINO",
    "floorPrice": 1.5,
    "floorPriceLamports": 1500000000,
    "topBid": 1.2,
    "volume24h": 150.5,
    "volume24hLamports": 150500000000,
    "sales24h": 12,
    "volume7d": 820.3,
    "volume7dLamports": 820300000000,
    "sales7d": 67,
    "floorChange1d": -0.05,
    "floorChange7d": 0.12,
    "owners": 234,
    "supply": 500,
    "updatedAt": "2026-01-01T00:00:00.000Z"
  }
}
```

---

#### `GET /api/collections/:collectionMint/floor`

Get V1 collection floor price.

**Response:**

```json
{
  "collectionMint": "...",
  "floorPrice": 1.5,
  "floorPriceLamports": 1500000000,
  "listingsCount": 42
}
```

---

#### `GET /api/collections/:collectionMint/volume`

Get V1 collection trading volume for 24h and 7d windows.

**Response:**

```json
{
  "collectionMint": "...",
  "volume24h": {
    "sol": 150.5,
    "lamports": 150500000000,
    "salesCount": 12
  },
  "volume7d": {
    "sol": 820.3,
    "lamports": 820300000000,
    "salesCount": 67
  }
}
```

---

#### `GET /api/collections/:collectionMint/activity`

Get recent V1 sales and activity for a collection.

**Query Parameters:**

| Param   | Type   | Default | Max | Description       |
| ------- | ------ | ------- | --- | ----------------- |
| `limit` | number | 50      | 100 | Number of results |

---

#### `GET /api/collections/:collectionMint/nfts`

Get all NFTs in a V1 collection (listed and optionally unlisted).

**Query Parameters:**

| Param             | Type    | Default | Description             |
| ----------------- | ------- | ------- | ----------------------- |
| `includeUnlisted` | boolean | false   | Include non-listed NFTs |

**Response:**

```json
{
  "success": true,
  "count": 10,
  "data": [
    {
      "mint": "nft-mint",
      "name": "DINO #123",
      "imageUrl": "https://...",
      "attributes": [...],
      "price": 1500000000,
      "priceSol": 1.5,
      "seller": "seller-wallet",
      "isListed": true,
      "listingAddress": "listing-pda"
    }
  ]
}
```

---

#### `GET /api/collections/:collectionMint/metadata`

Get V1 collection metadata only (no stats).

---

#### `GET /api/collections/:collectionMint/summary`

**Recommended.** Get complete V1 collection data in a single call — eliminates multiple round trips.

**Response:**

```json
{
  "collectionMint": "...",
  "name": "DINO",
  "symbol": "DINO",
  "description": "The DINO collection by VelixoMarket",
  "imageUrl": "https://...",
  "verified": true,
  "creator": "...",
  "stats": {
    "floorPrice": 1.5,
    "topBid": 1.2,
    "volume24h": 150.5,
    "sales24h": 12,
    "owners": 234,
    "supply": 500
  },
  "listings": [...],
  "offers": [...],
  "recentActivity": [...]
}
```

---

### V1 Offers

Fetch offers on individual NFTs and collections.

#### `GET /api/offers/nft/:mint`

Get all V1 offers on a specific NFT.

**Path Parameters:**

| Param  | Type   | Description      |
| ------ | ------ | ---------------- |
| `mint` | string | NFT mint address |

**Response:**

```json
{
  "success": true,
  "count": 3,
  "data": [
    {
      "address": "offer-pda",
      "offerer": "wallet-address",
      "nftMint": "nft-mint",
      "nftOwner": "owner-wallet",
      "amount": 1200000000,
      "isActive": true,
      "expiresAt": 1700000000,
      "createdAt": 1699900000
    }
  ]
}
```

---

#### `GET /api/offers/collection/:collectionMint`

Get all V1 collection-level offers.

---

#### `GET /api/offers/collection/:collectionMint/highest`

Get the highest active V1 collection offer.

**Response:**

```json
{
  "success": true,
  "data": { ... },
  "highestBid": 2.5
}
```

---

#### `GET /api/offers/user/:userAddress`

Get all V1 offers made by a user.

**Path Parameters:**

| Param         | Type   | Description    |
| ------------- | ------ | -------------- |
| `userAddress` | string | Wallet address |

---

#### `GET /api/offers/received/:userAddress`

Get all V1 offers received by a user (offers on NFTs they own).

---

#### `GET /api/offers/collection/user/:userAddress`

Get all V1 collection offers made by a user.

---

### V1 Stats

Marketplace-wide and per-collection statistics.

#### `GET /api/stats/marketplace`

Get V1 marketplace-wide statistics.

**Response:**

```json
{
  "success": true,
  "data": {
    "totalListings": 150,
    "activeListings": 120,
    "totalCollections": 5,
    "volume24h": 450.2,
    "volume24hLamports": 450200000000,
    "sales24h": 35,
    "volume7d": 2100.8,
    "volume7dLamports": 2100800000000,
    "sales7d": 189,
    "updatedAt": "2026-01-01T00:00:00.000Z"
  }
}
```

---

#### `GET /api/stats/collection/:collectionMint`

Get comprehensive V1 collection stats. Same response as `GET /api/collections/:collectionMint/stats`.

---

#### `GET /api/stats/collection/:collectionMint/floor`

Get V1 collection floor price.

---

#### `GET /api/stats/collection/:collectionMint/volume/24h`

Get V1 24-hour trading volume for a collection.

**Response:**

```json
{
  "period": "24h",
  "volume": 150.5,
  "volumeLamports": 150500000000,
  "salesCount": 12,
  "startTime": "2026-01-01T00:00:00.000Z",
  "endTime": "2026-01-02T00:00:00.000Z"
}
```

---

#### `GET /api/stats/collection/:collectionMint/volume/7d`

Get V1 7-day trading volume for a collection.

---

#### `GET /api/stats/volume/24h`

Get V1 marketplace-wide 24h volume.

---

#### `GET /api/stats/volume/7d`

Get V1 marketplace-wide 7d volume.

---

#### `GET /api/stats/sales`

Get V1 recent sales history.

**Query Parameters:**

| Param        | Type   | Default | Max | Description               |
| ------------ | ------ | ------- | --- | ------------------------- |
| `limit`      | number | 50      | 100 | Number of results         |
| `collection` | string | —       | —   | Filter by collection mint |

**Response:**

```json
{
  "success": true,
  "count": 10,
  "data": [
    {
      "nftMint": "...",
      "seller": "...",
      "buyer": "...",
      "price": 1500000000,
      "priceSol": 1.5,
      "collectionMint": "...",
      "timestamp": 1699900000,
      "type": "NftPurchased"
    }
  ]
}
```

---

### V1 Transactions

All V1 transaction endpoints return **unsigned, serialized transactions** (base64-encoded) for the client to sign with a wallet adapter and submit to the chain.

> **Note:** V1 transactions use `nftMint` and `collectionMint` fields. For dual-standard NFT support (Token Metadata + Metaplex Core), use [V2 Transactions](#v2-transactions) instead.

**Typical client flow:**

```js
// 1. Request transaction from backend
const res = await fetch("/api/transactions/purchase", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ buyer: walletAddress, nftMint, seller }),
});
const { transaction } = await res.json();

// 2. Deserialize
const tx = Transaction.from(Buffer.from(transaction, "base64"));

// 3. Sign with wallet
const signed = await wallet.signTransaction(tx);

// 4. Send to chain
const signature = await connection.sendRawTransaction(signed.serialize());
```

---

#### `POST /api/transactions/list` (V1)

List an NFT for sale on the marketplace.

**Body:**

| Field            | Type   | Required | Description                        |
| ---------------- | ------ | -------- | ---------------------------------- |
| `seller`         | string | yes      | Seller's wallet address            |
| `nftMint`        | string | yes      | NFT mint address                   |
| `price`          | number | yes      | Price in lamports                  |
| `collectionMint` | string | yes      | Collection mint for categorization |

**Example:**

```bash
curl -X POST http://localhost:3001/api/transactions/list \
  -H "Content-Type: application/json" \
  -d '{
    "seller": "SellerWallet...",
    "nftMint": "DINOMintAddress...",
    "price": 1500000000,
    "collectionMint": "DINOCollectionMint..."
  }'
```

**Response:**

```json
{
  "success": true,
  "transaction": "base64-encoded-transaction...",
  "message": "List transaction created"
}
```

---

#### `POST /api/transactions/purchase` (V1)

Purchase a listed NFT.

**Body:**

| Field     | Type   | Required | Description             |
| --------- | ------ | -------- | ----------------------- |
| `buyer`   | string | yes      | Buyer's wallet address  |
| `nftMint` | string | yes      | NFT mint address        |
| `seller`  | string | yes      | Seller's wallet address |

---

#### `POST /api/transactions/delist` (V1)

Remove an NFT listing from the marketplace.

**Body:**

| Field     | Type   | Required | Description             |
| --------- | ------ | -------- | ----------------------- |
| `seller`  | string | yes      | Seller's wallet address |
| `nftMint` | string | yes      | NFT mint address        |

---

#### `POST /api/transactions/make-offer` (V1)

Make an offer on a specific NFT.

**Body:**

| Field       | Type   | Required | Description                     |
| ----------- | ------ | -------- | ------------------------------- |
| `offerer`   | string | yes      | Offerer's wallet address        |
| `nftMint`   | string | yes      | NFT mint address                |
| `nftOwner`  | string | yes      | Current owner's wallet address  |
| `amount`    | number | yes      | Offer amount in lamports        |
| `expiresAt` | number | no       | Unix timestamp for offer expiry |

**Example:**

```bash
curl -X POST http://localhost:3001/api/transactions/make-offer \
  -H "Content-Type: application/json" \
  -d '{
    "offerer": "BuyerWallet...",
    "nftMint": "DINOMintAddress...",
    "nftOwner": "OwnerWallet...",
    "amount": 1200000000,
    "expiresAt": 1700000000
  }'
```

---

#### `POST /api/transactions/accept-offer` (V1)

Accept an offer on your NFT.

**Body:**

| Field      | Type   | Required | Description                |
| ---------- | ------ | -------- | -------------------------- |
| `nftOwner` | string | yes      | NFT owner's wallet address |
| `offerer`  | string | yes      | Offerer's wallet address   |
| `nftMint`  | string | yes      | NFT mint address           |

---

#### `POST /api/transactions/cancel-offer` (V1)

Cancel an offer you made.

**Body:**

| Field      | Type   | Required | Description              |
| ---------- | ------ | -------- | ------------------------ |
| `offerer`  | string | yes      | Offerer's wallet address |
| `nftMint`  | string | yes      | NFT mint address         |
| `nftOwner` | string | yes      | Owner's wallet address   |

---

#### `POST /api/transactions/make-collection-offer` (V1)

Make an offer on any NFT in a collection (e.g., bid on any DINO NFT).

**Body:**

| Field            | Type   | Required | Description                     |
| ---------------- | ------ | -------- | ------------------------------- |
| `offerer`        | string | yes      | Offerer's wallet address        |
| `collectionMint` | string | yes      | Collection mint address         |
| `amountPerNft`   | number | yes      | Offer amount per NFT (lamports) |
| `maxNfts`        | number | yes      | Max NFTs to purchase            |
| `expiresAt`      | number | no       | Unix timestamp for expiry       |

**Example:**

```bash
curl -X POST http://localhost:3001/api/transactions/make-collection-offer \
  -H "Content-Type: application/json" \
  -d '{
    "offerer": "BuyerWallet...",
    "collectionMint": "DINOCollectionMint...",
    "amountPerNft": 1000000000,
    "maxNfts": 5
  }'
```

---

#### `POST /api/transactions/accept-collection-offer` (V1)

Accept a collection offer with your NFT.

**Body:**

| Field            | Type   | Required | Description                |
| ---------------- | ------ | -------- | -------------------------- |
| `nftOwner`       | string | yes      | NFT owner's wallet address |
| `offerer`        | string | yes      | Offerer's wallet address   |
| `nftMint`        | string | yes      | NFT mint to sell           |
| `collectionMint` | string | yes      | Collection mint address    |

---

#### `POST /api/transactions/cancel-collection-offer` (V1)

Cancel a collection offer you made.

**Body:**

| Field            | Type   | Required | Description              |
| ---------------- | ------ | -------- | ------------------------ |
| `offerer`        | string | yes      | Offerer's wallet address |
| `collectionMint` | string | yes      | Collection mint address  |

---

## V2 Endpoints

V2 endpoints support **both Token Metadata and Metaplex Core NFT standards**. This is the recommended integration path for new projects.

### Key Differences from V1

| V1 Field         | V2 Field            | Notes                                  |
| ---------------- | ------------------- | -------------------------------------- |
| `nftMint`        | `nftAddress`        | Works with both Token Metadata & Core  |
| `collectionMint` | `collectionAddress` | Works with both Token Metadata & Core  |
| —                | `royaltyRecipient`  | Optional field for creator royalties   |
| —                | `nftStandard`       | Optional field to specify NFT standard |

---

### V2 Transactions

All V2 transaction endpoints return **unsigned, serialized transactions** (base64-encoded), same as V1. The client signing flow is identical.

**Typical client flow (V2):**

```js
// 1. Request transaction from backend
const res = await fetch("/api/v2/transactions/purchase", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    buyer: walletAddress,
    nftAddress: dinoNftAddress,
    seller,
  }),
});
const { transaction } = await res.json();

// 2. Deserialize
const tx = Transaction.from(Buffer.from(transaction, "base64"));

// 3. Sign with wallet
const signed = await wallet.signTransaction(tx);

// 4. Send to chain
const signature = await connection.sendRawTransaction(signed.serialize());
```

---

#### `POST /api/v2/transactions/list` (V2)

List an NFT for sale. Supports both Token Metadata and Metaplex Core NFTs.

**Body:**

| Field               | Type   | Required | Description                           |
| ------------------- | ------ | -------- | ------------------------------------- |
| `seller`            | string | yes      | Seller's wallet address               |
| `nftAddress`        | string | yes      | NFT address (mint or asset)           |
| `price`             | number | yes      | Price in lamports                     |
| `collectionAddress` | string | no       | Collection address for categorization |

**Example:**

```bash
curl -X POST http://localhost:3001/api/v2/transactions/list \
  -H "Content-Type: application/json" \
  -d '{
    "seller": "SellerWallet...",
    "nftAddress": "DINONftAddress...",
    "price": 1500000000,
    "collectionAddress": "DINOCollectionAddress..."
  }'
```

**Response:**

```json
{
  "success": true,
  "transaction": "base64-encoded-transaction...",
  "message": "List transaction created"
}
```

---

#### `POST /api/v2/transactions/purchase` (V2)

Purchase a listed NFT with optional royalty enforcement.

**Body:**

| Field              | Type   | Required | Description                         |
| ------------------ | ------ | -------- | ----------------------------------- |
| `buyer`            | string | yes      | Buyer's wallet address              |
| `nftAddress`       | string | yes      | NFT address                         |
| `seller`           | string | yes      | Seller's wallet address             |
| `royaltyRecipient` | string | no       | Creator address for royalty payment |

---

#### `POST /api/v2/transactions/delist` (V2)

Remove an NFT listing.

**Body:**

| Field        | Type   | Required | Description             |
| ------------ | ------ | -------- | ----------------------- |
| `seller`     | string | yes      | Seller's wallet address |
| `nftAddress` | string | yes      | NFT address             |

---

#### `POST /api/v2/transactions/make-offer` (V2)

Make an offer on a specific NFT.

**Body:**

| Field        | Type   | Required | Description                     |
| ------------ | ------ | -------- | ------------------------------- |
| `offerer`    | string | yes      | Offerer's wallet address        |
| `nftAddress` | string | yes      | NFT address                     |
| `nftOwner`   | string | yes      | Current owner's wallet address  |
| `amount`     | number | yes      | Offer amount in lamports        |
| `expiresAt`  | number | no       | Unix timestamp for offer expiry |

---

#### `POST /api/v2/transactions/accept-offer` (V2)

Accept an offer on your NFT with optional royalty enforcement.

**Body:**

| Field              | Type   | Required | Description                         |
| ------------------ | ------ | -------- | ----------------------------------- |
| `nftOwner`         | string | yes      | NFT owner's wallet address          |
| `offerer`          | string | yes      | Offerer's wallet address            |
| `nftAddress`       | string | yes      | NFT address                         |
| `royaltyRecipient` | string | no       | Creator address for royalty payment |

---

#### `POST /api/v2/transactions/cancel-offer` (V2)

Cancel an offer you made.

**Body:**

| Field        | Type   | Required | Description              |
| ------------ | ------ | -------- | ------------------------ |
| `offerer`    | string | yes      | Offerer's wallet address |
| `nftAddress` | string | yes      | NFT address              |

---

#### `POST /api/v2/transactions/make-collection-offer` (V2)

Make an offer on any NFT in a collection (e.g., bid on any DINO NFT).

**Body:**

| Field               | Type   | Required | Description                     |
| ------------------- | ------ | -------- | ------------------------------- |
| `offerer`           | string | yes      | Offerer's wallet address        |
| `collectionAddress` | string | yes      | Collection address              |
| `amountPerNft`      | number | yes      | Offer amount per NFT (lamports) |
| `maxNfts`           | number | yes      | Max NFTs to purchase            |
| `expiresAt`         | number | no       | Unix timestamp for expiry       |
| `nftStandard`       | string | no       | NFT standard (if known)         |

**Example:**

```bash
curl -X POST http://localhost:3001/api/v2/transactions/make-collection-offer \
  -H "Content-Type: application/json" \
  -d '{
    "offerer": "BuyerWallet...",
    "collectionAddress": "DINOCollectionAddress...",
    "amountPerNft": 1000000000,
    "maxNfts": 5
  }'
```

---

#### `POST /api/v2/transactions/accept-collection-offer` (V2)

Accept a collection offer with your NFT.

**Body:**

| Field               | Type   | Required | Description                         |
| ------------------- | ------ | -------- | ----------------------------------- |
| `nftOwner`          | string | yes      | NFT owner's wallet address          |
| `offerer`           | string | yes      | Offerer's wallet address            |
| `nftAddress`        | string | yes      | NFT address to sell                 |
| `collectionAddress` | string | yes      | Collection address                  |
| `royaltyRecipient`  | string | no       | Creator address for royalty payment |

---

#### `POST /api/v2/transactions/cancel-collection-offer` (V2)

Cancel a collection offer you made.

**Body:**

| Field               | Type   | Required | Description              |
| ------------------- | ------ | -------- | ------------------------ |
| `offerer`           | string | yes      | Offerer's wallet address |
| `collectionAddress` | string | yes      | Collection address       |

---

#### `POST /api/v2/transactions/sell-now` (V2)

Convenience endpoint for "Sell Now" — auto-resolves the highest active collection offer and builds an accept-collection-offer transaction in one call.

**Body:**

| Field               | Type   | Required | Description                        |
| ------------------- | ------ | -------- | ---------------------------------- |
| `nftOwner`          | string | yes      | Seller's wallet address            |
| `nftAddress`        | string | yes      | NFT to sell                        |
| `collectionAddress` | string | yes      | Collection address                 |
| `royaltyRecipient`  | string | no       | Optional royalty recipient override |

**Response:**

```json
{
  "success": true,
  "transaction": "base64-encoded-transaction...",
  "offer": {
    "offerer": "OffererWallet...",
    "amountPerNft": "1500000000",
    "collectionOfferPda": "OfferPDA..."
  }
}
```

Returns `404` if no active collection offers exist.

---

### V2 Collections

Fetch collection data with V2 dual-standard support.

#### `GET /api/v2/collections`

Get all V2 collections.

**Example:**

```bash
curl http://localhost:3001/api/v2/collections
```

**Response:**

```json
{
  "collections": [
    {
      "collectionMint": "DINOCollectionAddress...",
      "name": "DINO",
      "symbol": "DINO",
      "imageUrl": "https://..."
    }
  ]
}
```

---

#### `GET /api/v2/collections/with-stats`

**Recommended.** Get all V2 collections with stats in a single call — avoids multiple requests.

**Example:**

```bash
curl http://localhost:3001/api/v2/collections/with-stats
```

**Response:**

```json
[
  {
    "collectionMint": "DINOCollectionAddress...",
    "name": "DINO",
    "symbol": "DINO",
    "imageUrl": "https://...",
    "stats": {
      "floorPrice": 1.5,
      "volume24h": 150.5,
      "sales24h": 12,
      "owners": 234,
      "supply": 500
    }
  }
]
```

---

#### `GET /api/v2/collections/:collectionMint/stats`

Get V2 stats for a specific collection.

**Path Parameters:**

| Param            | Type   | Description             |
| ---------------- | ------ | ----------------------- |
| `collectionMint` | string | Collection mint address |

---

#### `GET /api/v2/collections/:collectionMint/listings`

Get all V2 listings in a collection.

---

#### `GET /api/v2/collections/:collectionMint/holders`

Get full holder list with percentages, top holders, and distribution buckets.

**Path Parameters:**

| Param            | Type   | Description             |
| ---------------- | ------ | ----------------------- |
| `collectionMint` | string | Collection mint address |

**Response:**

```json
{
  "success": true,
  "uniqueHolders": 234,
  "activeSupply": 500,
  "topHolders": [
    { "address": "Wallet...", "count": 15, "percentage": 3.0 }
  ],
  "holderDistribution": {
    "1": 150,
    "2-4": 50,
    "5-9": 20,
    "10-19": 10,
    "20-49": 3,
    "50-99": 1,
    "100+": 0
  },
  "data": [
    { "address": "Wallet...", "count": 15, "percentage": 3.0 }
  ]
}
```

---

#### `GET /api/v2/collections/:collectionMint/nfts`

Get NFTs in a collection from on-chain listings and activity data.

**Path Parameters:**

| Param            | Type   | Description             |
| ---------------- | ------ | ----------------------- |
| `collectionMint` | string | Collection mint address |

**Query Parameters:**

| Param    | Type   | Default | Max | Description          |
| -------- | ------ | ------- | --- | -------------------- |
| `limit`  | number | 100     | 500 | Number of results    |
| `offset` | number | 0       | —   | Pagination offset    |

**Response:**

```json
{
  "success": true,
  "pagination": {
    "total": 500,
    "limit": 100,
    "offset": 0,
    "hasMore": true
  },
  "data": [
    {
      "mint": "NFTMintAddress...",
      "collectionMint": "CollectionAddress...",
      "isListed": true,
      "price": "1500000000",
      "seller": "SellerWallet..."
    }
  ]
}
```

---

### V2 Activity

Dedicated activity history endpoints (V2 only — no V1 equivalent with this level of detail).

#### `GET /api/v2/activity/collection/:collectionAddress`

Get V2 activity history for a collection.

**Path Parameters:**

| Param               | Type   | Description        |
| ------------------- | ------ | ------------------ |
| `collectionAddress` | string | Collection address |

**Query Parameters:**

| Param   | Type   | Default | Max | Description             |
| ------- | ------ | ------- | --- | ----------------------- |
| `limit` | number | 50      | 200 | Number of results       |
| `sort`  | string | `desc`  | —   | `asc` or `desc` by time |

**Example:**

```bash
curl "http://localhost:3001/api/v2/activity/collection/DINOCollectionAddress...?limit=10&sort=desc"
```

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "type": "NftPurchased",
      "nftMint": "DINOMintAddress...",
      "seller": "SellerWallet...",
      "buyer": "BuyerWallet...",
      "price": 1500000000,
      "collectionMint": "DINOCollectionAddress...",
      "timestamp": 1699900000,
      "signature": "tx-signature..."
    }
  ],
  "count": 10,
  "sort": "desc"
}
```

**Activity types:**

| Type                       | Description                         |
| -------------------------- | ----------------------------------- |
| `NftListed`                | NFT was listed for sale             |
| `NftDelisted`              | NFT listing was removed             |
| `NftPurchased`             | NFT was bought                      |
| `OfferCreated`             | Offer was made on an NFT            |
| `OfferAccepted`            | Offer on an NFT was accepted        |
| `OfferCancelled`           | Offer on an NFT was cancelled       |
| `CollectionOfferCreated`   | Collection-wide offer was made      |
| `CollectionOfferAccepted`  | Collection-wide offer was accepted  |
| `CollectionOfferCancelled` | Collection-wide offer was cancelled |

---

#### `GET /api/v2/activity/user/:userAddress`

Get V2 activity history for a specific user (all their buys, sells, offers, etc.).

**Path Parameters:**

| Param         | Type   | Description    |
| ------------- | ------ | -------------- |
| `userAddress` | string | Wallet address |

**Query Parameters:** Same as collection activity (`limit`, `sort`).

**Example:**

```bash
curl "http://localhost:3001/api/v2/activity/user/WalletAddress...?limit=20"
```

---

#### `GET /api/v2/activity/recent`

Get V2 global recent marketplace activity across all collections.

**Query Parameters:** Same as collection activity (`limit`, `sort`).

**Example:**

```bash
curl "http://localhost:3001/api/v2/activity/recent?limit=50&sort=desc"
```

---

### V2 Stats

Marketplace-wide and per-collection statistics with multi-period volume breakdowns.

#### `GET /api/v2/stats/marketplace`

Get V2 marketplace-wide statistics including listings, collections, and volume across all time periods.

**Example:**

```bash
curl http://localhost:3001/api/v2/stats/marketplace
```

**Response:**

```json
{
  "success": true,
  "data": {
    "totalListings": 150,
    "activeListings": 120,
    "totalCollections": 5,
    "volume": {
      "24h": { "fogo": 45.2, "lamports": "45200000000", "sales": 8 },
      "3d": { "fogo": 120.5, "lamports": "120500000000", "sales": 22 },
      "7d": { "fogo": 310.8, "lamports": "310800000000", "sales": 55 },
      "30d": { "fogo": 1200.3, "lamports": "1200300000000", "sales": 210 },
      "allTime": { "fogo": 5400.0, "lamports": "5400000000000", "sales": 890 }
    },
    "updatedAt": 1700000000000
  }
}
```

---

#### `GET /api/v2/stats/collection/:collectionMint`

Get comprehensive V2 stats for a specific collection with multi-period volumes.

**Path Parameters:**

| Param            | Type   | Description             |
| ---------------- | ------ | ----------------------- |
| `collectionMint` | string | Collection mint address |

**Example:**

```bash
curl http://localhost:3001/api/v2/stats/collection/DINOCollectionAddress...
```

**Response:**

```json
{
  "success": true,
  "data": {
    "floorPrice": 1.5,
    "topBid": 1.2,
    "listed": 42,
    "volumeChange24h": 12.5,
    "floorChange1d": -0.05,
    "floorChange7d": 0.12,
    "owners": 234,
    "supply": 42,
    "collectionMint": "DINOCollectionAddress...",
    "volume": {
      "24h": { "fogo": 15.2, "sales": 3 },
      "3d": { "fogo": 40.5, "lamports": "40500000000", "sales": 8 },
      "7d": { "fogo": 95.8, "sales": 18 },
      "30d": { "fogo": 350.3, "lamports": "350300000000", "sales": 65 },
      "allTime": { "fogo": 1200.0, "lamports": "1200000000000", "sales": 220 }
    }
  }
}
```

---

#### `GET /api/v2/stats/volume`

Get all marketplace volumes in a single call (24h, 3d, 7d, 30d, allTime).

**Example:**

```bash
curl http://localhost:3001/api/v2/stats/volume
```

**Response:**

```json
{
  "success": true,
  "data": {
    "24h": { "fogo": 45.2, "lamports": "45200000000", "sales": 8 },
    "3d": { "fogo": 120.5, "lamports": "120500000000", "sales": 22 },
    "7d": { "fogo": 310.8, "lamports": "310800000000", "sales": 55 },
    "30d": { "fogo": 1200.3, "lamports": "1200300000000", "sales": 210 },
    "allTime": { "fogo": 5400.0, "lamports": "5400000000000", "sales": 890 }
  }
}
```

---

### V2 Offers

Fetch individual NFT offers, collection offers, and user offer data from on-chain V2 program accounts.

#### `GET /api/v2/offers/nft/:mint`

Get all active offers for a specific NFT, sorted by amount descending.

**Path Parameters:**

| Param  | Type   | Description      |
| ------ | ------ | ---------------- |
| `mint` | string | NFT mint address |

**Response:**

```json
{
  "success": true,
  "count": 2,
  "data": [
    {
      "address": "OfferPDA...",
      "offerer": "OffererWallet...",
      "nftOwner": "OwnerWallet...",
      "nftAddress": "NFTMintAddress...",
      "amount": "1500000000",
      "isActive": true,
      "expiresAt": "1700000000",
      "createdAt": "1699900000"
    }
  ]
}
```

---

#### `GET /api/v2/offers/collection/:collectionMint`

Get all active collection offers for a specific collection.

**Path Parameters:**

| Param            | Type   | Description             |
| ---------------- | ------ | ----------------------- |
| `collectionMint` | string | Collection mint address |

---

#### `GET /api/v2/offers/collection/:collectionMint/highest`

Get the highest collection offer for a specific collection.

**Response:**

```json
{
  "success": true,
  "data": { "...offer object..." },
  "highestBid": 1.5
}
```

---

#### `GET /api/v2/offers/user/:userAddress`

Get all individual NFT offers made by a user.

**Path Parameters:**

| Param         | Type   | Description    |
| ------------- | ------ | -------------- |
| `userAddress` | string | Wallet address |

---

#### `GET /api/v2/offers/received/:userAddress`

Get all offers received by a user (on NFTs they own).

---

#### `GET /api/v2/offers/collection/user/:userAddress`

Get all collection offers made by a user.

---

### V2 NFT

Individual NFT lookup with metadata, listing status, and activity history.

#### `GET /api/v2/nft/:mint`

Get full details for a single NFT.

**Path Parameters:**

| Param  | Type   | Description      |
| ------ | ------ | ---------------- |
| `mint` | string | NFT mint address |

**Response:**

```json
{
  "success": true,
  "data": {
    "mint": "NFTMintAddress...",
    "name": "DINO #123",
    "symbol": "DINO",
    "imageUrl": "https://...",
    "description": "...",
    "collectionAddress": "CollectionAddress...",
    "collectionVerified": true,
    "nftStandard": "token_metadata",
    "owner": "OwnerWallet...",
    "isListed": true,
    "listing": {
      "price": "1500000000",
      "seller": "SellerWallet...",
      "listingPda": "ListingPDA..."
    },
    "activity": [
      {
        "type": "NftListed",
        "from": "SellerWallet...",
        "to": null,
        "price": "1500000000",
        "signature": "tx-signature...",
        "timestamp": 1699900000
      }
    ]
  }
}
```

---

### V2 Owner

Fetch all NFTs owned by a wallet address.

#### `GET /api/v2/owner/:address`

Get all NFTs owned by an address (fetched from on-chain), with listing status.

**Path Parameters:**

| Param     | Type   | Description    |
| --------- | ------ | -------------- |
| `address` | string | Wallet address |

**Response:**

```json
{
  "success": true,
  "count": 5,
  "data": [
    {
      "mint": "NFTMintAddress...",
      "name": "DINO #123",
      "imageUrl": "https://...",
      "collectionAddress": "CollectionAddress...",
      "nftStandard": "token_metadata",
      "isListed": true,
      "listingPrice": "1500000000"
    }
  ]
}
```

---

## Admin Endpoints

Server management endpoints. All admin endpoints require authentication via `X-API-Key` header.

#### `GET /api/admin/status`

Get server and indexer status. **Requires authentication.**

**Response:**

```json
{
  "success": true,
  "data": {
    "server": {
      "uptime": 12345,
      "memory": { "rss": "...", "heapUsed": "..." },
      "nodeEnv": "production"
    },
    "cache": {
      "stats": { "keys": 150, "hits": 5000, "misses": 200 }
    },
    "indexer": {
      "status": "running"
    }
  }
}
```

---

#### `POST /api/admin/cache/clear`

Clear server cache. **Requires API key.**

**Body:**

| Field  | Type   | Values                                                 |
| ------ | ------ | ------------------------------------------------------ |
| `type` | string | `all`, `listings`, `metadata`, `offers`, `collections` |

**Example:**

```bash
curl -X POST http://localhost:3001/api/admin/cache/clear \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-key" \
  -d '{"type": "all"}'
```

---

#### `POST /api/admin/indexer/run`

Force-run the blockchain event indexer. **Requires API key.**

**Example:**

```bash
curl -X POST http://localhost:3001/api/admin/indexer/run \
  -H "X-API-Key: your-key"
```

---

## Environment Variables

| Variable                      | Default                             | Description                                   |
| ----------------------------- | ----------------------------------- | --------------------------------------------- |
| `PORT`                        | `3001`                              | Server port                                   |
| `NODE_ENV`                    | `development`                       | Environment mode                              |
| `FOGO_RPC_URL`                | `https://mainnet.fogo.io`           | Primary FOGO RPC endpoint                     |
| `FOGO_RPC_URL_BACKUP`         | `https://mainnet.fogo.io`           | Backup RPC endpoint                           |
| `MARKETPLACE_PROGRAM_ID`      | See [V1 vs V2](#v1-vs-v2-endpoints) | On-chain program ID                           |
| `REDIS_URL`                   | —                                   | Redis URL (optional, uses in-memory if unset) |
| `CACHE_TTL_METADATA`          | `3600`                              | Metadata cache TTL (seconds)                  |
| `CACHE_TTL_LISTINGS`          | `30`                                | Listings cache TTL (seconds)                  |
| `CACHE_TTL_USER_NFTS`         | `60`                                | User NFTs cache TTL (seconds)                 |
| `CACHE_TTL_COLLECTION_OFFERS` | `30`                                | Collection offers cache TTL (seconds)         |
| `FRONTEND_URL`                | `http://localhost:3000`             | Frontend URL for CORS                         |
| `ALLOWED_ORIGINS`             | `http://localhost:3000`             | Comma-separated allowed CORS origins          |
| `RATE_LIMIT_WINDOW_MS`        | `60000`                             | Rate limit window (ms)                        |
| `RATE_LIMIT_MAX_REQUESTS`     | `100`                               | Max requests per window                       |
| `INDEXER_INTERVAL_MS`         | `30000`                             | Event indexer poll interval (ms)              |
| `ADMIN_API_KEY`               | —                                   | API key for admin endpoints                   |
| `DB_HOST`                     | `localhost`                         | MySQL database host                           |
| `DB_PORT`                     | `3306`                              | MySQL database port                           |
| `DB_USER`                     | `root`                              | MySQL database user                           |
| `DB_PASSWORD`                 | —                                   | MySQL database password                       |
| `DB_NAME`                     | `velixomarket`                      | MySQL database name                           |
| `DB_CONNECTION_LIMIT`         | `10`                                | MySQL connection pool size                    |

---

## Caching

The backend uses an in-memory cache (or Redis if `REDIS_URL` is set) with the following TTLs:

| Data Type         | TTL        |
| ----------------- | ---------- |
| NFT Metadata      | 1 hour     |
| Listings          | 30 seconds |
| User NFTs         | 1 minute   |
| Collection Offers | 30 seconds |
| Stats             | 1 minute   |
| Volume            | 1 minute   |

Cache can be manually cleared via `POST /api/admin/cache/clear`.

---

## Error Codes

| Code                   | Description                                         |
| ---------------------- | --------------------------------------------------- |
| `INVALID_ADDRESS`      | Provided address is not a valid Solana/FOGO address |
| `INVALID_PRICE`        | Price must be a positive number                     |
| `INVALID_AMOUNT`       | Amount must be a positive number                    |
| `NOT_FOUND`            | Requested resource not found                        |
| `LISTING_NOT_FOUND`    | NFT listing does not exist                          |
| `OFFER_NOT_FOUND`      | Offer does not exist                                |
| `COLLECTION_NOT_FOUND` | Collection does not exist                           |
| `BATCH_LIMIT_EXCEEDED` | Too many items in batch request (max 50)            |
| `UNAUTHORIZED`         | Missing or invalid API key                          |
| `RATE_LIMIT_EXCEEDED`  | Too many requests — wait and retry                  |
| `TRANSACTION_ERROR`    | Failed to build transaction                         |
| `RPC_ERROR`            | FOGO chain RPC connection error                     |
| `INTERNAL_ERROR`       | Unexpected server error                             |

---

## Notes

- All prices/amounts on-chain are in **lamports** (1 FOGO = 1,000,000,000 lamports). Some response fields include both lamport and FOGO-denominated values for convenience.
- The backend runs a **blockchain event indexer** that polls the FOGO chain every 15 seconds for marketplace events (sales, listings, offers). This powers the activity and stats endpoints.
- Sales, activity history, floor price snapshots, and collection metadata are persisted in a **MySQL database**. Data survives server restarts with no entry limits.
- The database schema is auto-created on startup (5 tables: `sales`, `activities`, `floor_price_snapshots`, `collections`, `indexer_locks`).
- **V2 endpoints are the recommended integration path** as they support both Token Metadata and Metaplex Core NFT standards.
- The DINO collection is VelixoMarket's flagship collection on the FOGO chain.
