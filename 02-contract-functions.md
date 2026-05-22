# Giải Thích Chi Tiết Các Contract Functions

## Mục Lục
1. [NFTMarketplace.sol](#1-nftmarketplacesol)
2. [NFTCollection.sol](#2-nftcollectionsol)
3. [Các Điểm Quan Trọng Cần Lưu Ý](#3-các-điểm-quan-trọng-cần-lưu-ý)

---

## 1. NFTMarketplace.sol

### 1.1. Cấu Trúc Dữ Liệu (Structs & Enums)

```solidity
enum ListingType { DIRECT, AUCTION }
enum AuctionStatus { NOT_STARTED, ACTIVE, ENDED, CANCELED }

struct DirectListing {
    uint256 listingId;
    address seller;
    address nftContract;
    uint256 tokenId;
    uint256 price;      // giá cố định (wei)
    bool isActive;
}

struct AuctionListing {
    uint256 listingId;
    address seller;
    address nftContract;
    uint256 tokenId;
    uint256 startingBid;     // giá khởi điểm
    uint256 buyoutPrice;     // giá mua ngay (0 = không có)
    uint256 startTime;      // unix timestamp
    uint256 endTime;        // unix timestamp
    address highestBidder;
    uint256 highestBid;
    AuctionStatus status;
}

struct Offer {
    uint256 offerId;
    address buyer;
    address nftContract;
    uint256 tokenId;
    uint256 price;         // wei, đồng thời là số tiền escrow
    uint256 expiresAt;     // unix timestamp
    bool isActive;
}
```

### 1.2. Biến Trạng Thái Quan Trọng

```solidity
uint256 public platformFeeBasisPoints = 250;  // 250/10000 = 2.5%
uint256 public constant BASIS_POINTS = 10_000; // 100%
address public feeRecipient;                    // ví nhận phí platform
uint256 private _listingIdCounter = 1;          // ID tự tăng cho listing

// Mapping chính
mapping(uint256 => DirectListing) public directListings;
mapping(uint256 => AuctionListing) public auctionListings;

// Track NFT nào đang có listing active
mapping(address => mapping(uint256 => uint256)) public activeListingId;       // [nftContract][tokenId] => listingId
mapping(address => mapping(uint256 => ListingType)) public activeListingType;   // [nftContract][tokenId] => DIRECT|AUCTION

// Tiền của seller chờ rút
mapping(address => uint256) public proceeds;      // [seller] => số wei chờ rút

// Lưu bid của từng user trong auction
mapping(uint256 => mapping(address => uint256)) public auctionBids; // [listingId][bidder] => bidAmount
```

**Giải thích `activeListingId` và `activeListingType`**: Mỗi NFT (contract + tokenId) chỉ có thể có **1 listing active tại một thời điểm**. Tracking này ngăn việc tạo nhiều listing trùng lặp cho cùng 1 NFT, và dùng trong hàm `_clearActiveListing` để deactivate listing cũ khi mua thành công.

---

### 1.3. Direct Listing — Mua Bán Giá Cố Định

#### `createDirectListing(nftContract, tokenId, price)`

**Ý nghĩa**: Seller tạo một listing bán NFT với giá cố định.

```solidity
function createDirectListing(address nftContract, uint256 tokenId, uint256 price) external {
    // 1. Validation
    if (price == 0) revert PriceMustBeGreaterThanZero();

    // 2. Kiểm tra người gọi là chủ sở hữu NFT
    IERC721 nft = IERC721(nftContract);
    if (nft.ownerOf(tokenId) != msg.sender) revert NotTokenOwner();

    // 3. Kiểm tra marketplace đã được approve để transfer NFT
    // Điều này yêu cầu user phải gọi NFT.approve(marketplace, tokenId) TRƯỚC
    if (!_isApprovedForMarketplace(nftContract, tokenId, msg.sender)) revert MarketplaceNotApproved();

    // 4. Kiểm tra NFT chưa có listing active
    if (activeListingId[nftContract][tokenId] != 0) revert ListingAlreadyActive();

    // 5. Lưu listing vào mapping
    uint256 listingId = _listingIdCounter++;
    directListings[listingId] = DirectListing({ listingId, seller: msg.sender,
        nftContract, tokenId, price, isActive: true });

    // 6. Track active listing để prevent duplicate listings
    activeListingId[nftContract][tokenId] = listingId;
    activeListingType[nftContract][tokenId] = ListingType.DIRECT;

    emit DirectListingCreated(listingId, msg.sender, nftContract, tokenId, price);
}
```

**Điều kiện tiên quyết**: User phải approve marketplace trước:
```solidity
await nftContract.approve(marketplaceAddress, tokenId)
// hoặc
await nftContract.setApprovalForAll(marketplaceAddress, true)
```

---

#### `buyItem(listingId)` — Mua NFT

**Ý nghĩa**: Buyer mua NFT trực tiếp, ETH chuyển cho marketplace.

```solidity
function buyItem(uint256 listingId) external payable nonReentrant {
    DirectListing storage listing = directListings[listingId];

    // 1. Validate
    if (listing.seller == address(0)) revert ListingNotFound();
    if (!listing.isActive) revert ListingNotActive();
    if (msg.value < listing.price) revert BidTooLow();

    // ── 2. EFFECTS FIRST ──────────────────────────────────────────────────
    // Đóng listing TRƯỚC khi gọi external contract để prevent reentrancy
    listing.isActive = false;
    _clearActiveListing(listing.nftContract, listing.tokenId);

    // ── 3. Tính phân chia tiền ─────────────────────────────────────────
    // platformFee = price * 250 / 10000
    // sellerAmount = price - platformFee - royalty
    (uint platformFee, address royaltyReceiver, uint royaltyAmount, uint sellerAmount) =
        _calculatePayout(listing.nftContract, listing.tokenId, listing.price);

    // Cộng tiền vào mapping proceeds (chưa chuyển ngay)
    proceeds[listing.seller] += sellerAmount;
    proceeds[feeRecipient] += platformFee;
    if (royaltyAmount > 0) {
        proceeds[royaltyReceiver] += royaltyAmount;
    }

    // ── 4. INTERACTIONS — NFT Transfer ────────────────────────────────────
    // Transfer NFT từ seller cho buyer
    IERC721(listing.nftContract).safeTransferFrom(listing.seller, msg.sender, listing.tokenId);

    // ── 5. Refund nếu buyer gửi dư ───────────────────────────────────────
    if (msg.value > listing.price) {
        uint refund = msg.value - listing.price;
        (bool success,) = payable(msg.sender).call{value: refund}("");
        if (!success) revert TransferFailed();
    }

    emit ItemSold(listingId, msg.sender, listing.nftContract, listing.tokenId, listing.price);
}
```

**Luồng tiền**:
```
Buyer gửi: price ETH
  ├── platformFee (2.5%) → proceeds[feeRecipient]
  ├── royalty (ví dụ 5%) → proceeds[royaltyReceiver]
  └── sellerAmount → proceeds[seller]
       → seller gọi withdrawProceeds() để rút
```

**Tại sao dùng `proceeds` thay vì chuyển thẳng?**
Marketplace giữ tiền trong contract → nếu chuyển thẳng có thể bị reentrancy. Dùng `proceeds` mapping + `withdrawProceeds()` là pattern an toàn.

---

### 1.4. Auction — Đấu Giá

#### `createAuctionListing(...)` — Tạo Đấu Giá

```solidity
function createAuctionListing(
    address nftContract,
    uint256 tokenId,
    uint256 startingBid,    // giá khởi điểm (phải > 0)
    uint256 buyoutPrice,    // 0 = không có mua ngay
    uint256 startTime,      // phải > block.timestamp
    uint256 endTime         // phải > startTime + 1 giờ
) external {
    // Validations...
    // buyoutPrice > startingBid nếu buyoutPrice != 0

    auctionListings[listingId] = AuctionListing({
        listingId, seller: msg.sender, nftContract, tokenId,
        startingBid, buyoutPrice, startTime, endTime,
        highestBidder: address(0), highestBid: 0,
        status: AuctionStatus.NOT_STARTED  // chưa bắt đầu
    });

    activeListingId[nftContract][tokenId] = listingId;
    activeListingType[nftContract][tokenId] = ListingType.AUCTION;
}
```

**Điểm quan trọng**: Auction tạo ra với `status = NOT_STARTED`, nghĩa là chưa ai bid được cho đến khi `block.timestamp >= startTime`. Seller có thể gọi `startAuction()` để activate sớm.

---

#### `placeBid(listingId)` — Đặt Giá

```solidity
function placeBid(uint256 listingId) external payable nonReentrant {
    AuctionListing storage auction = auctionListings[listingId];

    // Auto-start nếu đã đến giờ
    if (auction.status == AuctionStatus.NOT_STARTED) {
        if (block.timestamp < auction.startTime) revert AuctionNotStartedYet();
        auction.status = AuctionStatus.ACTIVE;
    }

    // Đây là BID MỚI = tổng bid cũ + msg.value
    // Ví dụ: đã bid 1 ETH, gửi thêm 0.5 ETH → tổng = 1.5 ETH
    uint newTotalBid = auctionBids[listingId][msg.sender] + msg.value;
    if (newTotalBid <= auction.highestBid || newTotalBid < auction.startingBid)
        revert BidTooLow();

    // Cập nhật
    auctionBids[listingId][msg.sender] = newTotalBid;
    auction.highestBid = newTotalBid;
    auction.highestBidder = msg.sender;

    // BUYOUT CHECK — nếu bid >= buyoutPrice → instant win
    if (auction.buyoutPrice > 0 && newTotalBid >= auction.buyoutPrice) {
        _finalizeAuction(listingId);         // Settle ngay
        emit BuyoutTriggered(listingId, msg.sender, newTotalBid);
        return;
    }

    emit BidPlaced(listingId, msg.sender, newTotalBid);
}
```

**Điểm quan trọng**: `newTotalBid = existingBid + msg.value` — nghĩa là mỗi bid mới **cộng dồn** vào bid cũ. Đây là cumulative bidding, không phải replacement.

---

#### `finalizeAuction(listingId)` — Kết Thúc Đấu Giá

```solidity
function finalizeAuction(uint256 listingId) external nonReentrant {
    // Chỉ gọi được khi block.timestamp >= endTime
    // Ai cũng có thể gọi (permissionless), không chỉ seller

    _finalizeAuction(listingId);
    emit AuctionFinalized(listingId, auction.highestBidder, auction.highestBid);
}
```

---

### 1.5. Offer — Chào Mua Non-Custodial

**Điểm khác biệt quan trọng so với Auction**: Offer dùng **ETH escrow** — buyer gửi tiền vào contract ngay khi tạo offer. Contract giữ tiền cho đến khi seller accept hoặc buyer cancel.

#### `makeOffer(nftContract, tokenId, price, durationSeconds)`

```solidity
function makeOffer(
    address nftContract,
    uint256 tokenId,
    uint256 price,           // = số ETH gửi vào contract (escrow)
    uint256 durationSeconds  // offer có hiệu lực trong bao lâu
) external payable nonReentrant {
    if (msg.value < price) revert InsufficientOfferValue();  // phải gửi đủ
    // ETH được tự động giữ trong contract balance

    offers[offerId] = Offer({
        offerId, buyer: msg.sender, nftContract, tokenId,
        price, expiresAt: block.timestamp + durationSeconds, isActive: true
    });

    // Track để query theo token và buyer
    offerIdsByToken[nftContract][tokenId].push(offerId);
    offerIdsByBuyer[msg.sender].push(offerId);

    emit OfferCreated(...);
}
```

**Tại sao "non-custodial"?** Buyer không gửi ETH cho ai — ETH được **khóa trong contract**. Seller không thể lừa đảo vì NFT chỉ transfer khi ETH đã nằm trong contract.

#### `acceptOffer(offerId)`

```solidity
function acceptOffer(uint256 offerId) external nonReentrant {
    // 1. Seller phải là chủ sở hữu NFT hiện tại
    // 2. Offer phải còn active và chưa hết hạn

    // ETH đã nằm trong contract từ makeOffer — không cần buyer làm gì thêm

    // Chia tiền
    (platformFee, royaltyReceiver, royaltyAmount, sellerAmount) =
        _calculatePayout(...);
    proceeds[seller] += sellerAmount;
    proceeds[feeRecipient] += platformFee;

    // Transfer NFT seller → buyer
    IERC721(offer.nftContract).safeTransferFrom(seller, buyer, offer.tokenId);

    emit OfferAccepted(offerId, buyer, seller, salePrice);
}
```

**Luồng tiền của Offer**:
```
ETH đã nằm trong contract (escrow từ makeOffer)
  ├── platformFee → proceeds[feeRecipient]
  ├── royalty → proceeds[royaltyReceiver]
  └── sellerAmount → proceeds[seller] (seller rút bằng withdrawProceeds)
```

---

### 1.6. Các Hàm Nội Bộ Quan Trọng

#### `_calculatePayout` — Chia Tiền Bán

```solidity
function _calculatePayout(address nftContract, uint256 tokenId, uint256 salePrice)
    internal view returns (platformFee, royaltyReceiver, royaltyAmount, sellerAmount)
{
    // Platform fee: 2.5%
    platformFee = (salePrice * platformFeeBasisPoints) / BASIS_POINTS;  // 250/10000
    sellerAmount = salePrice - platformFee;

    // Royalty từ ERC-2981
    (royaltyReceiver, royaltyAmount) = _getRoyaltyInfo(nftContract, tokenId, salePrice);

    // Royalty không thể lớn hơn sellerAmount (safety cap)
    if (royaltyAmount > sellerAmount) {
        royaltyAmount = sellerAmount;
    }
    sellerAmount -= royaltyAmount;
}
```

#### `_isApprovedForMarketplace` — Kiểm Tra Approval

```solidity
function _isApprovedForMarketplace(address nftContract, uint256 tokenId, address tokenOwner)
    internal view returns (bool)
{
    IERC721 nft = IERC721(nftContract);
    // Cách 1: approve riêng từng token
    // Cách 2: setApprovalForAll cho toàn bộ collection
    return nft.getApproved(tokenId) == address(this)
        || nft.isApprovedForAll(tokenOwner, address(this));
}
```

#### `_clearActiveListing` — Dọn Active Listing

```solidity
function _clearActiveListing(address nftContract, uint256 tokenId) internal {
    uint256 listingId = activeListingId[nftContract][tokenId];

    // Xóa reference
    delete activeListingId[nftContract][tokenId];
    delete activeListingType[nftContract][tokenId];

    // ⚠️ QUAN TRỌNG: acceptOffer gọi _clearActiveListing nhưng
    // acceptOffer không tạo direct listing, nên directListings[listingId].isActive
    // sẽ là giá trị mặc định (false) → an toàn
    if (listingId != 0) {
        delete directListings[listingId].isActive;
    }
}
```

#### `_getRoyaltyInfo` — Lấy Royalty ERC-2981

```solidity
function _getRoyaltyInfo(...) internal view returns (...) {
    if (!_supportsRoyalty(nftContract)) return (address(0), 0);

    // Dùng try/catch vì contract không hỗ trợ ERC-2981 sẽ revert
    try IERC2981(nftContract).royaltyInfo(tokenId, salePrice)
        returns (address royaltyReceiver, uint256 royaltyAmount) {
        return (royaltyReceiver, royaltyAmount);
    } catch {
        return (address(0), 0);
    }
}
```

#### `_supportsRoyalty` — ERC-165 Check

```solidity
function _supportsRoyalty(address nftContract) internal view returns (bool) {
    // ERC-165: gọi supportsInterface với interfaceId của ERC-2981
    (bool success, bytes memory data) = nftContract.staticcall(
        abi.encodeWithSelector(IERC165.supportsInterface.selector,
            type(IERC2981).interfaceId)
    );
    return success && data.length >= 32 && abi.decode(data, (bool));
}
```

---

### 1.7. Events Emit

| Event | Khi nào | Dữ liệu |
|-------|----------|----------|
| `ItemSold` | `buyItem` thành công | listingId, buyer, nftContract, tokenId, price |
| `AuctionFinalized` | `finalizeAuction` thành công | listingId, winner, winningBid |
| `OfferAccepted` | `acceptOffer` thành công | offerId, buyer, seller, price |

**Ba event này chính là những gì backend indexer đọc để ghi vào MySQL.**

---

## 2. NFTCollection.sol

### 2.1. Mint NFT

```solidity
function mintNFT(address to, string memory tokenURI_, uint96 royaltyFee)
    public returns (uint256 tokenId)
{
    _tokenIdCounter++;
    tokenId = _tokenIdCounter;

    _safeMint(to, tokenId);                    // ERC-721 mint
    _setTokenURI(tokenId, tokenURI_);          // Lưu IPFS URI
    _setTokenRoyalty(tokenId, to, royaltyFee); // Set ERC-2981 royalty cho token này

    emit NFTMinted(tokenId, to, tokenURI_, royaltyFee);
}
```

**Các bước mint một NFT**:
1. Upload ảnh lên IPFS → nhận `imageHash`
2. Tạo JSON metadata `{ name, description, image: ipfs://imageHash }`
3. Upload JSON lên IPFS → nhận `metadataHash`
4. Gọi `mintNFT(to, "ipfs://metadataHash", royaltyFeeInBasisPoints)`
5. Token được mint, tokenURI trỏ đến IPFS metadata

### 2.2. Ownership Tracking

Contract dùng `_update` hook (override từ ERC-721) để track danh sách token mỗi address:

```solidity
function _update(address to, uint256 tokenId, address auth)
    internal override returns (address from)
{
    from = super._update(to, tokenId, auth);

    // Bỏ token khỏi danh sách chủ cũ
    if (from != address(0)) {
        _ownedTokens[from].pop() // swap-with-last và pop
        delete _ownedTokensIndex[tokenId];
    }

    // Thêm token vào danh sách chủ mới
    if (to != address(0)) {
        _ownedTokensIndex[tokenId] = _ownedTokens[to].length;
        _ownedTokens[to].push(tokenId);
    }
}
```

**Tại sao cần tracking riêng?** ERC-721 `balanceOf` đọc từ mapping `balance`, nhưng không có cách hiệu quả để liệt kê TẤT CẢ token IDs của một address. `_ownedTokens` mapping giải quyết vấn đề này.

### 2.3. ERC-2981 Royalty

Mỗi NFT có thể set royalty riêng qua `_setTokenRoyalty(tokenId, receiver, feeNumerator)`. Marketplace đọc royalty khi bán qua `_getRoyaltyInfo`.

**Ví dụ**: Mint với `royaltyFee = 500` (5%) → mỗi lần bán, 5% giá bán được chuyển cho người mint (creator).

---

## 3. Các Điểm Quan Trọng Cần Lưu Ý

### 3.1. Reentrancy Protection

Tất cả hàm thay đổi state đều có `nonReentrant` modifier (OpenZeppelin `ReentrancyGuard`). Pattern áp dụng: **Effects → Interactions** (update state trước, sau đó mới gọi external contracts như NFT transfer).

### 3.2. Tại Sao `ItemSold` Không Có Seller?

Event `ItemSold` chỉ có `buyer` trong topic (để filter theo buyer), không có `seller` trong event data. Đây là limitation của Solidity events — chỉ có thể đánh index cho 3 topics. **Seller address không thể phục hồi từ event logs**, nên trade_history sẽ có seller = `0x0000...` khi dùng `ItemSold`.

**Giải pháp**: Dùng `getDirectListing(listingId)` để lấy seller từ contract state (backend gọi `ethCall`).

### 3.3. Method Selectors

Method selector = 4 bytes đầu của `keccak256(functionSignature)`. Ví dụ:
```
keccak256("buyItem(uint256)") = 0xd2b8648ec6ff6bd9... → selector = 0xd2b8648e
```

Khi gọi contract bằng raw `eth_call` (như backend làm), phải dùng đúng selector.

### 3.4. Direct Listing vs Offer

| | Direct Listing | Offer |
|--|---------------|-------|
| Ai gửi ETH? | Buyer gửi khi mua | Buyer gửi khi tạo offer |
| ETH ở đâu? | Gửi vào contract, chia ngay | **Giữ trong contract** (escrow) |
| Khi nào seller nhận tiền? | Sau khi `buyItem` thành công | Sau khi `acceptOffer` |
| Offer hết hạn? | Không có expiry | Có (`expiresAt`) |

### 3.5. Auction Buyout

Khi bid >= `buyoutPrice`:
```solidity
if (auction.buyoutPrice > 0 && newTotalBid >= auction.buyoutPrice) {
    _finalizeAuction(listingId);  // Settle ngay lập tức
    emit BuyoutTriggered(...);
    return;
}
```
Không cần chờ `endTime`. Highest bidder ngay lập tức nhận NFT.

### 3.6. Phí Platform

```solidity
platformFee = (salePrice * 250) / 10000  // = 2.5%
```
Tối đa 10% (`require(newFeeBasisPoints <= 1000)`). Fee chuyển vào `proceeds[feeRecipient]`, feeRecipient là address được set trong constructor = deployer.
