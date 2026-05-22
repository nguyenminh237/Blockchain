# Cơ Chế Auction và Direct Listing

## Tổng Quan

NFTMarketplace hỗ trợ hai cách bán: **Direct Listing** (giá cố định) và **Auction** (đấu giá). Cả hai đều emit events mà backend indexer đọc để ghi lịch sử giao dịch.

---

## 1. Direct Listing — Giá Cố Định

### 1.1. Luồng Hoàn Chỉnh

```
User A (seller)                    Contract                    User B (buyer)
     │                               │                            │
     │  1. approve(marketplace, id) │                            │
     │───────────────────────────────→│                            │
     │                               │                            │
     │  2. createDirectListing(     │                            │
     │     nftContract, tokenId,    │                            │
     │     priceInWei)              │                            │
     │──────────────────────────────→│  Listing created           │
     │                               │  isActive = true          │
     │                               │                            │
     │                               │  3. buyer gọi buyItem()    │
     │                               │  với value = price        │
     │                               │←──────────────────────────│
     │                               │                            │
     │                               │  4. Transfer NFT          │
     │                               │  A → B                   │
     │                               │                            │
     │  proceeds[A] += 97.5%        │                            │
     │←──────────────────────────────│                            │
     │                               │                            │
     │  5. withdrawProceeds()       │                            │
     │──────────────────────────────→│  ETH → A                  │
```

### 1.2. Điều Kiện Cần Thiết

Trước khi `createDirectListing`, user phải approve marketplace:

```jsx
// Frontend: approve marketplace để transfer NFT
await nftContract.approve(marketplaceAddress, tokenId)
// hoặc
await nftContract.setApprovalForAll(marketplaceAddress, true)
```

Contract kiểm tra:
```solidity
if (!_isApprovedForMarketplace(nftContract, tokenId, msg.sender))
    revert MarketplaceNotApproved();
```

### 1.3. Khi Nào Listing Bị Đóng?

Listing `isActive = false` khi:
- Buyer gọi `buyItem()` thành công
- Seller gọi `cancelDirectListing()`
- Seller gọi `updateDirectPrice()` (không đóng listing, chỉ đổi giá)
- Seller chuyển NFT đi chỗ khác (NFT không còn trong marketplace)

---

## 2. Auction — Đấu Giá

### 2.1. Kiến Trúc Auction

```
AuctionListing struct:
├── listingId, seller, nftContract, tokenId
├── startingBid      ← giá khởi điểm
├── buyoutPrice      ← giá mua ngay (0 = không có)
├── startTime       ← khi nào bắt đầu bid được
├── endTime         ← khi nào kết thúc
├── highestBidder    ← ai đang cao nhất
├── highestBid      ← giá cao nhất hiện tại
└── status          ← NOT_STARTED | ACTIVE | ENDED | CANCELED
```

### 2.2. Luồng Hoàn Chỉnh

```
T = seller tạo auction (status = NOT_STARTED)
        │
        ▼
AUCTION CREATED
        │
        ▼ T < startTime
No bidding allowed (revert AuctionNotStartedYet)
        │
        ├─ Seller gọi startAuction() → status = ACTIVE
        │    hoặc block.timestamp ≥ startTime (auto start)
        │
        ▼
AUCTION ACTIVE
        │
        ├─ User C bid 1 ETH
        │   auctionBids[C] = 1 ETH, highestBid = 1 ETH, highestBidder = C
        │
        ├─ User D bid 1.5 ETH
        │   auctionBids[D] = 1.5 ETH, highestBid = 1.5 ETH, highestBidder = D
        │   (User C's 1 ETH vẫn lưu, nhưng không còn highest nữa)
        │
        ├─ User E bid ≥ buyoutPrice (VD: 3 ETH)
        │   → BUYOUT → instant finalize → NFT → E → proceeds → A
        │   (kết thúc ngay, không cần đợi endTime)
        │
        └─ block.timestamp ≥ endTime → auction ENDED
             (chưa finalize)

        ▼
ANYONE calls finalizeAuction()
        │
        ├─ Nếu có highestBidder
        │   NFT → highestBidder
        │   ETH (trừ fees) → proceeds[seller]
        │   emit AuctionFinalized(listingId, winner, winningBid)
        │
        └─ Nếu không ai bid (highestBidder = address(0))
             emit AuctionEndedNoBids(listingId)
             NFT → vẫn của seller
```

### 2.3. Cumulative Bidding

```solidity
// Mỗi bid mới = bid cũ + msg.value
uint256 newTotalBid = auctionBids[listingId][msg.sender] + msg.value;
```

**Ví dụ**:
```
User C bid 1 ETH:
  auctionBids[C] = 0 + 1 ETH = 1 ETH
  highestBid = 1 ETH, highestBidder = C

User C bid thêm 0.5 ETH:
  auctionBids[C] = 1 ETH + 0.5 ETH = 1.5 ETH
  highestBid = 1.5 ETH, highestBidder = C

User D bid 0.3 ETH:
  auctionBids[D] = 0 + 0.3 ETH = 0.3 ETH
  highestBid = 1.5 ETH (C vẫn cao nhất)
  (D's 0.3 ETH vẫn lưu, D có thể rút lại)
```

### 2.4. Buyout — Instant Win

```solidity
if (auction.buyoutPrice > 0 && newTotalBid >= auction.buyoutPrice) {
    _finalizeAuction(listingId);    // Settle ngay
    emit BuyoutTriggered(...);
    return;
}
```

**Ví dụ**: Auction có `buyoutPrice = 2 ETH`
```
User A bid 1 ETH (highestBid = 1 ETH)
User B bid 2 ETH (2 ETH >= buyoutPrice)
→ Auction finalize ngay lập tức
→ NFT → User B
→ proceeds → seller
→ emit BuyoutTriggered(..., 2 ETH)
```

### 2.5. Refunding Failed Bids

Sau khi auction kết thúc, ai không phải winner có thể rút lại tiền:

```solidity
function withdrawFailedBid(uint256 listingId) external nonReentrant {
    // Chỉ gọi được khi auction ENDED hoặc CANCELED
    // Winner không được rút bid
    uint256 amount = auctionBids[listingId][msg.sender];
    auctionBids[listingId][msg.sender] = 0;  // Đặt về 0 TRƯỚC khi transfer
    payable(msg.sender).call{value: amount}("");
}
```

**Tại sao phải đặt về 0 trước?** Để ngăn reentrancy.

### 2.6. Cancel Auction

```solidity
if (auction.highestBidder != address(0))
    revert CannotCancelAuctionWithBids();
```
→ Auction chỉ cancel được khi **chưa ai bid**.

---

## 3. So Sánh Chi Tiết

| | Direct Listing | Auction |
|--|--------------|---------|
| Giá | Fixed price | Bidding, có buyout |
| Khi nào kết thúc? | Ngay khi có buyer mua | `endTime` hoặc buyout |
| Ai nhận NFT? | Buyer | Highest bidder |
| Ai gọi finalize? | — | Bất kỳ ai |
| Bid refund? | — | `withdrawFailedBid()` |
| Cancel? | Seller bất kỳ lúc nào | Chỉ khi chưa có bid |

---

## 4. Cùng Lúc Có Direct và Auction?

Không. Mỗi NFT chỉ có thể có **một listing active** tại một thời điểm:

```solidity
if (activeListingId[nftContract][tokenId] != 0)
    revert ListingAlreadyActive();
```

`activeListingId` và `activeListingType` mapping đảm bảo điều này.

---

## 5. Phí Platform

Cả hai đều trả **2.5% platform fee**:

```
Giá bán = 1 ETH
  ├── Platform fee (2.5%) = 0.025 ETH → feeRecipient
  ├── Royalty (5%)       = 0.05 ETH  → NFT creator (ERC-2981)
  └── Seller proceeds     = 0.925 ETH → proceeds[seller]
                              → seller gọi withdrawProceeds()
```

---

## 6. Events Emit

| Action | Event | Backend Indexer |
|--------|-------|---------------|
| `buyItem()` | `ItemSold` | ✅ Indexes |
| `finalizeAuction()` | `AuctionFinalized` | ✅ Indexes |
| `acceptOffer()` | `OfferAccepted` | ✅ Indexes |
| `createDirectListing()` | `DirectListingCreated` | ❌ Không index |
| `createAuctionListing()` | `AuctionCreated` | ❌ Không index |
| `placeBid()` | `BidPlaced` / `BuyoutTriggered` | ❌ Không index |
