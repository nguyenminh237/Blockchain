# Luồng Dữ Liệu — Từ Blockchain Đến Giao Diện

## Tổng Quan

Dữ liệu trong hệ thống ArtVerse chảy theo nhiều hướng song song:

1. **Blockchain (Source of Truth)** — mọi giao dịch mua/bán được ghi trực tiếp vào smart contract
2. **Backend Event Indexer** — đọc events từ blockchain → ghi vào MySQL
3. **Frontend** — đọc trực tiếp từ blockchain (RPC) + đọc từ backend REST API (lịch sử)

---

## 1. Hai Loại Dữ Liệu

### 1.1. Dữ Liệu Hiện Tại (Current State) — Đọc Trực Tiếp Từ Blockchain

Là những gì **đang tồn tại** trên blockchain tại thời điểm hiện tại.

| Dữ liệu | Nguồn | Ai đọc |
|----------|--------|---------|
| Danh sách listing active | `getAllActiveDirectListings()` | Frontend |
| Danh sách auction active | `getAllActiveAuctions()` | Frontend |
| Chi tiết một listing | `getDirectListing(id)` / `getAuctionListing(id)` | Frontend |
| Offers của một token | `getOffersByToken(nftContract, tokenId)` | Frontend |
| NFT owner | `ownerOf(tokenId)` / `getTokensByOwner(address)` | Frontend |
| Tiền chờ rút | `proceeds(address)` | Frontend |

→ Frontend dùng **wagmi/viem** gọi trực tiếp qua Ethereum RPC.

### 1.2. Lịch Sử Giao Dịch (Trade History) — Đọc Từ MySQL

Là những gì **đã xảy ra** trong quá khứ.

| Dữ liệu | Nguồn | Ai đọc |
|----------|--------|---------|
| Lịch sử giao dịch một token | `trade_history` table | Frontend (qua REST API) |
| Hoạt động của user | `trade_history` table | Frontend (qua REST API) |
| Hoạt động của collection | `trade_history` table | Frontend (qua REST API) |

→ Backend đọc từ blockchain events → ghi vào MySQL → cung cấp REST API.

---

## 2. Sơ Đồ Luồng Chi Tiết

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        BLOCKCHAIN (Sepolia)                                  │
│                                                                          │
│  User A ──createDirectListing──→ directListings[id]  ←─────────────┐   │
│                                             ↑                             │   │
│  User B ───────buyItem(id)───────────── NFT Transfer → User B         │   │
│                                             │                             │   │
│                                    emit ItemSold(...)                    │   │
│                                             │                             │   │
│                                    emit AuctionFinalized(...)            │   │
│                                             │                             │   │
│                                    emit OfferAccepted(...)                │   │
│                                             │                             │   │
└─────────────────────────────────────────────┼─────────────────────────────┘
                                              │ eth_getLogs (polling 15s)
                                              │ eth_call (on-demand)
                                              ▼
                               ┌──────────────────────────────┐
                               │      BACKEND EVENT INDEXER     │
                               │  NftEventIndexerService.java  │
                               │                               │
                               │  1. Poll eth_getLogs          │
                               │     filter: ItemSold          │
                               │     filter: AuctionFinalized  │
                               │     filter: OfferAccepted     │
                               │                               │
                               │  2. Decode log data           │
                               │     - nftContract, tokenId    │
                               │     - price (wei)             │
                               │     - seller, buyer           │
                               │                               │
                               │  3. Deduplicate by txHash     │
                               │     findByTxHash(txHash)      │
                               │                               │
                               │  4. saveFromEvent()          │
                               │                               │
                               │  5. TradeHistoryService       │
                               │     wei → ETH conversion      │
                               └──────────────┬───────────────┘
                                              │
                                              │ JDBC
                                              ▼
                               ┌──────────────────────────────┐
                               │           MySQL               │
                               │                               │
                               │  Table: trade_history        │
                               │  ┌──────────────────────┐  │
                               │  │ id (PK)             │  │
                               │  │ nftContract         │  │
                               │  │ tokenId             │  │
                               │  │ price (BIGINT wei)  │  │
                               │  │ seller              │  │
                               │  │ buyer               │  │
                               │  │ tradeType (DIRECT/ │  │
                               │  │           AUCTION/ │  │
                               │  │           OFFER)   │  │
                               │  │ listingId          │  │
                               │  │ offerId            │  │
                               │  │ txHash (UNIQUE)   │  │
                               │  │ blockNumber        │  │
                               │  │ createdAt          │  │
                               │  └──────────────────────┘  │
                               └──────────────────────────────┘
                                              │
                                              │ REST API
                                              ▼
                               ┌──────────────────────────────┐
                               │     FRONTEND (React)          │
                               │                               │
                               │  tradeApi.getTokenActivity() │
                               │  tradeApi.getUserActivity()  │
                               │  tradeApi.getCollectionActivity│
                               │                               │
                               │  ┌──────────────────────┐   │
                               │  │ TokenActivityDto     │   │
                               │  │  latestPrice (ETH)  │   │
                               │  │  totalTrades         │   │
                               │  │  history[]           │   │
                               │  └──────────────────────┘   │
                               │                               │
                               │  NftDetailPage ─ History Tab  │
                               │  ProfilePage ─ Activity Tab │
                               └──────────────────────────────┘
```

---

## 3. Luồng Đọc Trực Tiếp Từ Blockchain (Frontend → RPC)

Frontend đọc dữ liệu hiện tại mà không cần backend.

```
User mở trang /nft/0x.../123
        │
        ▼
useDirectListings() hook
        │
        ▼
wagmi useReadContract({
  address: MARKETPLACE_ADDRESS,
  abi: NFT_MARKETPLACE_ABI,
  functionName: 'getAllActiveDirectListings',
})
        │
        ▼
viem encodeFunctionData({
  data: '0xd2b8648e' + '0x0000...0007'  // selector + listingId param
})
        │
        ▼
eth_call → Alchemy RPC
        │
        ▼
viem decodeFunctionResult() → DirectListing[]
        │
        ▼
useDirectListings() trả về { data: DirectListing[] }
        │
        ▼
NFTCard components render danh sách NFT
```

### Tất Cả Các Read Operations Của Frontend

```
getAllActiveDirectListings()     → Danh sách listing đang active
getDirectListing(listingId)       → Chi tiết 1 listing
getAllActiveAuctions()           → Danh sách auction đang active
getAuctionListing(listingId)      → Chi tiết 1 auction
getOffersByToken(nftContract, tokenId)   → Offers cho 1 NFT
getOffersByBuyer(address)         → Offers của 1 user
getTokensByOwner(address)         → NFTs của 1 user
ownerOf(tokenId)                  → Chủ sở hữu NFT
tokenURI(tokenId)                  → IPFS URI của NFT
proceeds(address)                 → Số ETH chờ rút
```

---

## 4. Luồng Event Indexing (Backend)

Backend polling blockchain mỗi 15 giây để tìm các giao dịch mới.

```
┌─────────────────────────────────────────────────────────────┐
│         NftEventIndexerService.indexEvents()              │
│         (runs every 15 seconds via @Scheduled)            │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  Bước 1: Lấy current block                               │
│  eth_blockNumber → currentBlock                            │
│  Ví dụ: 6012345                                           │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  Bước 2: Xác định block range                            │
│  from = lastProcessedBlock + 1  (VD: 6012000)            │
│  to   = min(currentBlock, lastProcessedBlock + 500)      │
│  to   = 6012345                                           │
│  Range: [6012000, 6012345] — 345 blocks                  │
└─────────────────────────────────────────────────────────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
    processItemSold   processAuction processOfferAccepted
        Events        Finalized        Events
          │             │             │
          ▼             ▼             ▼
    eth_getLogs   eth_getLogs    eth_getLogs
    (filter:      (filter:       (filter:
     ItemSold)     AuctionFinalized) OfferAccepted)
          │             │             │
          ▼             ▼             ▼
    decode log   decode log    decode log
    (nftContract (listingId,   (offerId,
     tokenId,     buyer, bid)    buyer, seller,
     price)                   price)
          │             │             │
          │      fetchAuctionListing   fetchOffer(offerId)
          │      (eth_call)      (eth_call)
          │        │                   │
          │     seller,            nftContract,
          │     nftContract,        tokenId
          │     tokenId
          │        │                   │
          └────────┼─────────────────┘
                   ▼
          saveFromEvent() → MySQL
                   │
                   ▼
          Deduplicate by txHash
          (findByTxHash → skip if exists)
                   │
                   ▼
          lastProcessedBlock = to (6012345)
```

### Chi Tiết Decoding Event

**ItemSold Event** (`contracts/src/NFTMarketplace.sol`):
```solidity
event ItemSold(
    uint256 indexed listingId,  // Topic[1] → lấy từ topics.get(1)
    address indexed buyer,       // Topic[2] → lấy từ topics.get(2)
    address nftContract,         // Data[0:64]  → lấy từ event.getData()
    uint256 tokenId,             // Data[64:128]
    uint256 price                // Data[128:192]
);
```

Trong Java (NftEventIndexerService.java):
```java
// Topic parsing — address padded left in 32 bytes
String buyer = "0x" + event.getTopics().get(2).substring(26);

// Data parsing — mỗi value là 32 bytes (64 hex chars)
String nftContract = "0x" + data.substring(24, 64);  // address: 20 bytes → padded left
BigInteger tokenId = new BigInteger(data.substring(64, 128), 16);
BigInteger price   = new BigInteger(data.substring(128, 192), 16);
```

---

## 5. Cách Dữ Liệu Đến Frontend

### 5.1. Dữ Liệu Hiện Tại → Frontend (Trực Tiếp)

```
Blockchain Contract State
        │
        ▼ eth_call (wagmi)
    Frontend React
        │
        ▼
    useDirectListings() / useActiveAuctions() / useOffersByToken()
        │
        ▼
    Components: NFTCard, AuctionCard, OfferCard
```

### 5.2. Lịch Sử → Backend (MySQL) → Frontend (REST API)

```
Blockchain Event (ItemSold)
        │
        ▼ eth_getLogs (Backend)
    NftEventIndexerService
        │
        ▼
    MySQL trade_history
        │
        ▼ HTTP REST API
    Frontend tradeApi
        │
        ▼
    TradeHistoryDto[]
        │
        ▼
    Components: HistoryTab, ProfilePage, ActivitySection
```

---

## 6. Ví Dụ Cụ Thể: Một Giao Dịch Mua NFT

**Tình huống**: User B mua NFT token #5 từ User A qua direct listing với giá 0.1 ETH.

```
Bước 1: User A tạo listing
────────────────────────────────
User A → createDirectListing(0xNFT, 5, 100000000000000000)  // 0.1 ETH = 1e17 wei
        │
        ▼ eth_sendTransaction (signed by A)
    Blockchain: directListings[1] = { ..., isActive: true }
        │
        ▼
    Blockchain emit: DirectListingCreated(...)

Bước 2: User B mua
────────────────────────────────
User B frontend → useBuyItem() hook
        │
        ▼ eth_sendTransaction (signed by B, value: 0.1 ETH)
    buyItem(1) { payable }
        │
        ▼
    Blockchain:
    1. listing.isActive = false
    2. proceeds[feeRecipient] += 2500000000000000    // 2.5% = 0.0025 ETH
    3. proceeds[0xA] += 97500000000000000           // 97.5% = 0.0975 ETH
    4. safeTransferFrom(A, B, 5)                   // NFT: A → B
    5. emit ItemSold(1, buyer=B, nftContract, tokenId=5, price=0.1 ETH)

Bước 3: Backend Indexer đọc event (15 giây sau)
────────────────────────────────────────────────
Backend polls eth_getLogs(filter: ItemSold, fromBlock=X, toBlock=Y)
        │
        ▼
    Log: { topics: [..., listingId=1, buyer=B], data: [nftContract, 5, 100000000000000000] }
        │
        ▼
    decode nftContract, tokenId, price
        │
        ▼
    findByTxHash(txHash) → empty → continue
        │
        ▼
    tradeHistoryService.saveFromEvent(
        nftContract, tokenId=5, price=100000000000000000, // wei
        seller="0xA...", buyer="0xB...", type="DIRECT",
        listingId="1", null, txHash, blockNumber
    )
        │
        ▼
    MySQL INSERT trade_history { price=100000000000000000, seller=A, buyer=B, type=DIRECT, ... }

Bước 4: Frontend hiển thị lịch sử
────────────────────────────────────
User mở NftDetailPage của token #5
        │
        ▼
    tradeApi.getTokenActivity(nftContract, 5)
        │
        ▼ GET /api/v1/trades/token/{nftContract}/5
        │
        ▼
    TradeHistoryService → MySQL
        │
        ▼
    TokenActivityDto {
        nftContract: "0xNFT",
        tokenId: 5,
        latestPrice: "0.1000",    // converted from wei → ETH
        totalTrades: 1,
        history: [TradeHistoryDto{ seller=A, buyer=B, price="0.1000", type=DIRECT, ... }]
    }
        │
        ▼
    HistoryTab component → bảng lịch sử giao dịch
```

---

## 7. Các RPC Calls Chi Tiết

### 7.1. eth_getLogs — Đọc Event Logs (Backend)

```java
// NftEventIndexerService.java
EthFilter filter = new EthFilter(fromBlock, toBlock, marketplaceAddress);
filter.addSingleTopic(EVENT_ITEM_SOLD);  // "0xd2b8648e..."

EthLog ethLog = web3j.ethGetLogs(filter).send();
List<EthLog.LogResult> logs = ethLog.getLogs();

// Mỗi LogResult chứa:
// - address: contract address
// - topics: indexed event params
// - data: non-indexed event params
// - transactionHash: hash của tx
// - blockNumber: block chứa tx
```

### 7.2. eth_call — Đọc Contract State (Backend + Frontend)

```java
// Backend: fetchOffer(offerId)
// Lấy nftContract và tokenId từ offer struct
String encodedCall = "0x3ca211ad"  // keccak("offers(uint256)")[0:4]
        + String.format("%064x", BigInteger.valueOf(offerId));

EthCall response = web3j.ethCall(
    Transaction.createFunctionCallTransaction(
        "0x0000...0000",  // from (không cần signer cho view call)
        BigInteger.ZERO,  // gas
        BigInteger.ZERO,  // gas price
        BigInteger.ZERO,  // value
        marketplaceAddress,
        BigInteger.ZERO,  // nonce
        encodedCall
    ),
    DefaultBlockParameter.valueOf("latest")  // đọc block mới nhất
).send();

// Response là hex data cần decode thủ công:
// Offer struct = [offerId][buyer][nftContract][tokenId][price][expiresAt][isActive]
// Mỗi slot = 32 bytes = 64 hex chars
```

### 7.3. eth_sendTransaction — Gọi Hàm Write (Frontend)

```
User click "Buy" → useBuyItem() hook
        │
        ▼
wagmi prepareWriteContract({
    address: MARKETPLACE_ADDRESS,
    abi: NFT_MARKETPLACE_ABI,
    functionName: 'buyItem',
    args: [listingId],
    value: priceInWei,  // ← quan trọng: phải gửi ETH
})
        │
        ▼
wagmi → Wallet (MetaMask) → User ký transaction
        │
        ▼ eth_sendTransaction
    Blockchain xử lý buyItem()
        │
        ▼
    Transaction receipt (async)
        │
        ▼
useBuyItem() → { isConfirming: true } → spinner
        │
        ▼
Transaction confirmed
        │
        ▼
useBuyItem() → { isConfirmed: true } → toast "Purchase successful!"
```

---

## 8. Cấu Trúc Dữ Liệu Trong MySQL

```sql
CREATE TABLE trade_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    nft_contract VARCHAR(64) NOT NULL,
    token_id BIGINT NOT NULL,
    price DECIMAL(65,0) NOT NULL,          -- lưu bằng wei (BigInteger)
    seller VARCHAR(64) NOT NULL,           -- address
    buyer VARCHAR(64) NOT NULL,            -- address
    trade_type VARCHAR(10) NOT NULL,      -- DIRECT | AUCTION | OFFER
    listing_id VARCHAR(78),                -- string của listing ID
    offer_id VARCHAR(78),                  -- string của offer ID
    tx_hash VARCHAR(66) NOT NULL UNIQUE,  -- ngăn duplicate
    block_number BIGINT NOT NULL,
    created_at DATETIME NOT NULL,

    INDEX idx_token_contract (token_id, nft_contract),
    INDEX idx_seller (seller),
    INDEX idx_buyer (buyer),
    INDEX idx_created_at (created_at),
    INDEX idx_trade_type (trade_type)
);
```

**Giá trị quan trọng**: `price` lưu bằng **wei** (BigInteger), khi trả về cho frontend được convert sang **ETH decimal string** trong `TradeHistoryService.convertToEthString()`.
