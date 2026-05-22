# Kiến Trúc Tổng Quan — ArtVerse NFT Marketplace

## 1. Tổng Quan Hệ Thống

ArtVerse là một ứng dụng **NFT Marketplace** phi tập trung (decentralized) xây dựng trên mạng **Ethereum Sepolia Testnet**. Toàn bộ dữ liệu giao dịch (mua/bán/đấu giá) được ghi trực tiếp vào **blockchain** thông qua các **Smart Contract**, và ứng dụng web (frontend) đọc hiển thị qua blockchain RPC.

```
┌─────────────────────────────────────────────────────────────────┐
│                        SEPOLIA TESTNET                          │
│                                                                 │
│  NFTMarketplace.sol   NFTCollection.sol                        │
│  ┌───────────────┐   ┌───────────────┐                        │
│  │ Listing       │   │ Mint NFT      │                        │
│  │ Auction       │   │ TokenURI      │                        │
│  │ Offer         │   │ Royalty       │                        │
│  └───────┬───────┘   └───────┬───────┘                        │
│          │                   │                                 │
│   emit ItemSold              │                                 │
│   emit AuctionFinalized       │                                 │
│   emit OfferAccepted          │                                 │
└──────────┼───────────────────┼──────────────────────────────────┘
           │                   │
           │ eth_getLogs       │ eth_call / eth_sendTransaction
           │ (polling)         │ (user signed tx)
           ▼                   ▼
┌──────────────────┐    ┌──────────────────────────────────────┐
│    BACKEND       │    │           FRONTEND (React)            │
│  Spring Boot     │    │                                     │
│                  │    │  wagmi / ethers.js                  │
│  - Event Indexer │    │  MetaMask / WalletConnect           │
│  - TradeHistory  │    │                                     │
│  - REST API      │    │  - Browse listings/auctions         │
│                  │    │  - Buy / Bid / Make Offer           │
│  MySQL DB        │    │  - Wallet connect                  │
│  (trade_history) │    │  - IPFS upload (Pinata)           │
└──────────────────┘    └──────────────────────────────────────┘
```

---

## 2. Ba Thành Phần Chính

### 2.1. Smart Contracts (`/contracts`)

- **NFTMarketplace.sol** — Hợp đồng marketplace: tạo listing, mua bán, đấu giá, offer.
- **NFTCollection.sol** — Hợp đồng NFT ERC-721: mint, tokenURI, royalty.
- Được compile bằng **Foundry** (`forge build`), deploy lên Sepolia bằng `forge create`.
- Source code trên Etherscan: marketplace tại `0x42aD78a0B2763688eBab29DD3260BCE64098352b`, collection tại `0xEA995b3df84552A177CC56D2dda6e508Ffd5da92`.

### 2.2. Backend (`/backend`)

- **Spring Boot 3.3.5** + **Java 17**.
- Giao tiếp blockchain qua **Web3j** (thư viện Java Ethereum).
- **MySQL** lưu lịch sử giao dịch (`trade_history` table).
- **Event Indexer** — polling blockchain mỗi 15 giây, đọc `ItemSold`, `AuctionFinalized`, `OfferAccepted` events → ghi vào MySQL.
- Cung cấp **REST API** cho frontend: `/api/v1/trades/*`.

### 2.3. Frontend (`/frontend`)

- **React 18** + **Vite** + **TypeScript-like JavaScript**.
- Giao tiếp blockchain qua **wagmi v2** + **viem** + **ethers.js**.
- Tích hợp ví qua **wagmi connectors** (MetaMask, Coinbase Wallet, Injected).
- Upload metadata lên **IPFS** qua **Pinata**.
- Gọi REST API backend để hiển thị lịch sử giao dịch.

---

## 3. Các Chức Năng Chính

### 3.1. Mua NFT — Direct Listing

1. Seller tạo **Direct Listing** (fixed price) bằng `createDirectListing()`.
2. Buyer gọi `buyItem(listingId)` với `value = price` → ETH chuyển cho marketplace.
3. Contract tự động chia tiền: `platformFee` (2.5%) + `royalty` (nếu có) + `sellerProceeds`.
4. Contract emit `ItemSold` event.
5. Backend indexer đọc event → ghi vào MySQL.
6. Frontend hiển thị danh sách listing từ blockchain và lịch sử từ backend.

### 3.2. Đấu Giá — Auction

1. Seller tạo **Auction Listing** với `startingBid`, `buyoutPrice`, `startTime`, `endTime`.
2. Buyers gọi `placeBid(listingId)` với `value = bidAmount`.
3. Nếu bid >= `buyoutPrice` → mua ngay lập tức (buyout).
4. Khi `block.timestamp > endTime`, ai cũng gọi được `finalizeAuction(listingId)` để settle:
   - NFT transfer cho highest bidder.
   - ETH cho seller (trừ platform fee + royalty).
5. Contract emit `AuctionFinalized` event.

### 3.3. Offer — Chào Mua Phi Custodial

1. Buyer gọi `makeOffer()` với `value = price` → ETH được **giữ trong contract** (escrow).
2. Seller gọi `acceptOffer(offerId)` → NFT transfer cho buyer, ETH cho seller (trừ fees).
3. Buyer có thể `cancelOffer()` để hoàn tiền.
4. Contract emit `OfferAccepted` event.

### 3.4. Mint NFT

1. Owner upload ảnh + metadata lên **IPFS** qua Pinata.
2. Gọi `mintNFT(to, tokenURI, royaltyFee)` → mint ERC-721 với tokenURI trỏ đến IPFS.
3. NFT hiển thị trên giao diện với metadata từ IPFS.

---

## 4. Sơ Đồ Dữ Liệu

```
Block   Contract Emit Event
#12345  ItemSold(txHash: 0xabc...)   ──→  Backend Indexer  ──→  MySQL trade_history
        AuctionFinalized(...)                    │
        OfferAccepted(...)                      ▼
                                          Frontend API
                                                │
                                                ▼
                                          React Components
                                                │
              Blockchain RPC ←──────────────────┘
              (listings, auctions, offers, balances)
```

---

## 5. Blockchain RPC — Ai Dùng Cái Gì?

| Mục đích | Ai gọi | Thư viện | Giao thức |
|----------|--------|-----------|-----------|
| Đọc contract state (view functions) | Frontend + Backend | wagmi / viem / Web3j | `eth_call` |
| Gọi contract state-changing (signed tx) | Frontend (user) | wagmi → MetaMask | `eth_sendTransaction` |
| Đọc event logs (historical) | Backend | Web3j `eth_getLogs` | `eth_getLogs` |
| Đọc event logs (polling) | Backend | Web3j `eth_getLogs` | `eth_getLogs` |

**RPC URL:** `https://eth-sepolia.g.alchemy.com/v2/anGVqS5jz_jojA1PyCq6b`

---

## 6. Smart Contract Addresses (Sepolia)

| Contract | Address |
|----------|---------|
| NFTMarketplace | `0x42aD78a0B2763688eBab29DD3260BCE64098352b` |
| NFTCollection | `0xEA995b3df84552A177CC56D2dda6e508Ffd5da92` |

---

## 7. Directory Structure

```
NFT-project/
├── contracts/                         # Smart Contracts (Solidity + Foundry)
│   ├── src/
│   │   ├── NFTMarketplace.sol          # Main marketplace contract
│   │   └── NFTCollection.sol           # ERC-721 NFT contract
│   ├── lib/openzeppelin-contracts/    # OpenZeppelin dependency
│   └── out/                           # Compiled artifacts (ABI, bytecode)
│
├── backend/                            # Spring Boot Backend
│   └── src/main/java/com/nft/marketplace/
│       ├── config/                    # Web3j, Contract, Indexer configs
│       ├── controller/                # REST controllers
│       ├── dto/                       # Data Transfer Objects
│       ├── entity/                    # JPA Entities
│       ├── repository/                # Spring Data JPA Repositories
│       └── service/
│           ├── TradeHistoryService.java      # Trade history CRUD
│           └── NftEventIndexerService.java   # Blockchain event polling
│
└── frontend/                          # React + Vite Frontend
    └── src/
        ├── abi/                      # Contract ABIs (from Etherscan)
        ├── components/              # React components
        ├── config/
        │   ├── contracts.js          # Contract addresses
        │   └── wagmi.js             # wagmi v2 config
        ├── hooks/
        │   └── useContract.js       # All contract interaction hooks
        ├── pages/                    # Page components
        ├── services/
        │   ├── api.js               # Backend REST API client
        │   └── pinata.js           # IPFS upload service
        └── utils/
            └── index.js             # Helpers (price, address, IPFS)
```

---

## 8. Môi Trường (Environment)

### Backend (`application.properties`)
```
web3j.contract.marketplace-address=0x42aD78a0B2763688eBab29DD3260BCE64098352b
web3j.contract.nft-collection-address=0xEA995b3df84552A177CC56D2dda6e508Ffd5da92
web3j.rpc.http-url=https://eth-sepolia.g.alchemy.com/v2/anGVqS5jz_jojA1PyCq6b
nft.indexer.from-block=0
nft.indexer.poll-interval-ms=15000
```

### Frontend (`.env`)
```
VITE_CHAIN_ID=11155111
VITE_MARKETPLACE_ADDRESS=0x42aD78a0B2763688eBab29DD3260BCE64098352b
VITE_NFT_COLLECTION_ADDRESS=0xEA995b3df84552A177CC56D2dda6e508Ffd5da92
VITE_API_BASE_URL=http://localhost:8080/api/v1
VITE_PINATA_JWT=<your_pinata_jwt>
```
