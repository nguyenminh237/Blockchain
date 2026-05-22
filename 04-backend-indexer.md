# Cơ Chế Đọc Events Blockchain — Backend Event Indexer

## Tổng Quan

Backend sử dụng **block polling** để đọc các events từ smart contract và ghi vào MySQL. Mỗi 15 giây, `NftEventIndexerService` hỏi blockchain về các events mới xảy ra trong khoảng block kể từ lần polling trước.

---

## 1. Tại Sao Cần Event Indexer?

Blockchain là **append-only** — không có bảng giao dịch có cấu trúc như MySQL. Để truy vấn "lịch sử giao dịch của token #5", ta phải:

1. Biết block number của tx đầu tiên liên quan đến token #5
2. Đọc event logs trong khoảng block đó → hiện tại

Event Indexer giải quyết bằng cách:
- **Tự động đọc** mỗi event mới từ blockchain
- **Decode** raw log data thành dữ liệu có cấu trúc
- **Ghi** vào MySQL để truy vấn nhanh

---

## 2. Event Signatures

Mỗi event có một **signature** duy nhất = `keccak256` hash của tên event + params:

```java
// NftEventIndexerService.java
// cast keccak "ItemSold(uint256,address,address,uint256,uint256)"
private static final String EVENT_ITEM_SOLD =
        "0xd2b8648ec6ff6bd9ed10162d6ec424ae792c12820a475e8ef52f23fd7da4f1eb";

// cast keccak "AuctionFinalized(uint256,address,uint256)"
private static final String EVENT_AUCTION_FINALIZED =
        "0x4d9113a1377d665eaa1f9168a9c9080f2e488cb820b10149de3d6d2e0f2780c7";

// cast keccak "OfferAccepted(uint256,address,address,uint256)"
private static final String EVENT_OFFER_ACCEPTED =
        "0xc67f49229cab3f614ad390c12432e0e671552dcfd15e24ddc005cf5a01a02135";
```

**Cách lấy selector**: 
```bash
cast keccak "ItemSold(uint256,address,address,uint256,uint256)"
# Output: 0xd2b8648ec6ff6bd9ed10162d6ec424ae792c12820a475e8ef52f23fd7da4f1eb
```

---

## 3. Polling Flow

### 3.1. Khởi Tạo (`init()`)

```java
@PostConstruct
public void init() {
    // Kiểm tra config: nft.indexer.from-block
    if (indexerConfig.getFromBlock() > 0) {
        lastProcessedBlock = indexerConfig.getFromBlock();
    } else {
        // Không có config → lấy block hiện tại trừ 1
        EthBlockNumber blockNumber = web3j.ethBlockNumber().send();
        lastProcessedBlock = Math.max(0, blockNumber.getBlockNumber().longValue() - 1);
    }
}
```

**Lưu ý**: `fromBlock` được set bằng **block number**, không phải timestamp. `lastProcessedBlock = latestBlock - 1` đảm bảo không miss event nào trong block hiện tại.

### 3.2. Polling Loop (`indexEvents()`)

```java
@Scheduled(fixedDelayString = "${nft.indexer.poll-interval-ms:15000}")
public void indexEvents() {
    // 1. Lấy block hiện tại
    EthBlockNumber blockNumber = web3j.ethBlockNumber().send();
    long currentBlock = blockNumber.getBlockNumber().longValue();

    if (currentBlock <= lastProcessedBlock) {
        return; // Không có block mới
    }

    // 2. Tính block range
    // Ví dụ: lastProcessedBlock = 6000000, currentBlock = 6000350
    // from = 6000001, to = 6000301 (batch max 500 blocks)
    long from = lastProcessedBlock + 1;
    long to   = Math.min(currentBlock, lastProcessedBlock + BATCH_SIZE); // 500

    // 3. Xử lý 3 loại event song song
    processItemSoldEvents(fromBlock, toBlock);
    processAuctionFinalizedEvents(fromBlock, toBlock);
    processOfferAcceptedEvents(fromBlock, toBlock);

    // 4. Cập nhật checkpoint
    lastProcessedBlock = Math.max(lastProcessedBlock, to);
}
```

### 3.3. Batch Processing

```java
private static final int BATCH_SIZE = 500; // max blocks per poll
```

Mỗi lần polling xử lý tối đa **500 blocks**. Với block time ~12 giây (Sepolia), 500 blocks = ~100 phút. Scheduler chạy mỗi 15 giây nên luôn theo kịp.

**Tại sao batch?** RPC provider (Alchemy) giới hạn query range. Batch 500 blocks là sweet spot giữa performance và không quá tải.

---

## 4. Đọc Event Logs — `eth_getLogs`

### 4.1. Cú Pháp

```java
EthFilter filter = new EthFilter(fromBlock, toBlock, marketplaceAddress);
filter.addSingleTopic(EVENT_ITEM_SOLD);  // Lọc theo event signature

EthLog ethLog = web3j.ethGetLogs(filter).send();

// Mỗi log chứa:
// - address: contract address (đã filter)
// - topics[0]: event signature
// - topics[1..n]: indexed params ( VD: listingId, buyer)
// - data: non-indexed params (VD: nftContract, tokenId, price)
// - transactionHash: hash của transaction
// - blockNumber: block chứa event
```

### 4.2. EthFilter — Cách Lọc

```
eth_getLogs với filter:
{
  "fromBlock": 6000001,
  "toBlock": 6000301,
  "address": "0x42aD78a...",
  "topics": ["0xd2b8648e..."]  // ItemSold signature
}

Trả về: TẤT CẢ logs của ItemSold event trong range đó
```

### 4.3. Indexed vs Non-Indexed Params

Trong Solidity, params có `indexed` keyword sẽ nằm trong **topics[]**, không có sẽ nằm trong **data**:

```solidity
event ItemSold(
    uint256 indexed listingId,    // → topics[1]
    address indexed buyer,         // → topics[2]
    address nftContract,          // → data[0:64]
    uint256 tokenId,              // → data[64:128]
    uint256 price                 // → data[128:192]
);
```

---

## 5. Chi Tiết Từng Event

### 5.1. ItemSold — Direct Listing

```java
private void processItemSoldEvents(DefaultBlockParameter from, DefaultBlockParameter to) {
    EthFilter filter = new EthFilter(from, to, marketplaceAddress);
    filter.addSingleTopic(EVENT_ITEM_SOLD);
    EthLog ethLog = web3j.ethGetLogs(filter).send();

    for (EthLog.LogResult result : ethLog.getLogs()) {
        org.web3j.protocol.core.methods.response.Log event = result.get();
        DecodedItemSold decoded = decodeItemSold(event);
        if (decoded == null) continue;

        String txHash = event.getTransactionHash();

        // ── Deduplication ──────────────────────────────────────────
        // Một transaction có thể emit nhiều ItemSold nếu marketplace
        // gọi inner transfers (không xảy ra trong contract này, nhưng phòng)
        if (tradeHistoryRepository.findByTxHash(txHash).isPresent()) {
            continue;
        }

        // ItemSold đã có đủ data trong event → không cần ethCall
        tradeHistoryService.saveFromEvent(
            decoded.nftContract(),
            decoded.tokenId(),
            decoded.price(),
            "0x0000...0000",  // ⚠️ seller không có trong event
            decoded.buyer(),
            "DIRECT",
            decoded.listingId(),
            null,
            txHash,
            event.getBlockNumber().longValue()
        );
    }
}
```

**Decoding ItemSold:**
```java
private DecodedItemSold decodeItemSold(Log event) {
    // topics[1] = listingId (uint256 indexed)
    BigInteger listingId = new BigInteger(event.getTopics().get(1).substring(2), 16);

    // topics[2] = buyer (address indexed)
    String buyer = "0x" + event.getTopics().get(2).substring(26);

    // data = nftContract(32bytes) + tokenId(32bytes) + price(32bytes)
    String data = event.getData().substring(2);

    String nftContract = "0x" + data.substring(24, 64);  // 20 bytes address padded left
    BigInteger tokenId  = new BigInteger(data.substring(64, 128), 16);  // hex → BigInteger
    BigInteger price    = new BigInteger(data.substring(128, 192), 16);

    return new DecodedItemSold(listingId.toString(), nftContract, price, tokenId.longValue(), buyer);
}
```

### 5.2. AuctionFinalized — Auction

```java
private void processAuctionFinalizedEvents(...) {
    // 1. Đọc event logs
    EthLog ethLog = web3j.ethGetLogs(filter).send();
    // ...

    for (EthLog.LogResult result : ethLog.getLogs()) {
        DecodedAuctionFinalized decoded = decodeAuctionFinalized(event);
        // topics[1] = listingId, topics[2] = winner, data = winningBid

        // 2. AuctionFinalized KHÔNG có nftContract/tokenId/seller
        //    trong event data → phải gọi ethCall để lấy từ contract state
        String[] listingInfo = fetchAuctionListing(decoded.listingId());
        if (listingInfo == null) {
            log.warn("Could not fetch auction {} for AuctionFinalized tx {}",
                decoded.listingId(), txHash);
            continue;
        }
        // listingInfo = [seller, nftContract, tokenId]

        tradeHistoryService.saveFromEvent(
            listingInfo[1], listingInfo[2], decoded.price(),
            listingInfo[0], decoded.buyer(), "AUCTION",
            decoded.listingId(), null, txHash, blockNumber
        );
    }
}
```

### 5.3. OfferAccepted — Offer

```java
private void processOfferAcceptedEvents(...) {
    for (EthLog.LogResult result : ethLog.getLogs()) {
        DecodedOfferAccepted decoded = decodeOfferAccepted(event);
        // topics[1] = offerId, topics[2] = buyer, topics[3] = seller, data = price

        // 2. OfferAccepted KHÔNG có nftContract/tokenId trong event
        //    → gọi ethCall fetchOffer(offerId)
        String[] offerInfo = fetchOffer(decoded.offerId());
        if (offerInfo == null) {
            log.warn("Could not fetch offer {} for OfferAccepted tx {}",
                decoded.offerId(), txHash);
            continue;
        }
        // offerInfo = [nftContract, tokenId]

        tradeHistoryService.saveFromEvent(
            offerInfo[0], Long.parseLong(offerInfo[1]),
            decoded.price(), decoded.seller(), decoded.buyer(),
            "OFFER", null, String.valueOf(decoded.offerId()),
            txHash, blockNumber
        );
    }
}
```

---

## 6. Đọc Contract State — `ethCall`

### 6.1. Tại Sao Cần ethCall?

Một số event KHÔNG chứa đủ thông tin trong log. Ví dụ:

| Event | Có trong log | Phải ethCall để lấy |
|-------|--------------|---------------------|
| `ItemSold` | nftContract, tokenId, price, buyer | — |
| `AuctionFinalized` | listingId, winner, winningBid | seller, nftContract, tokenId |
| `OfferAccepted` | offerId, buyer, seller, price | nftContract, tokenId |

### 6.2. Cách Gọi ethCall

```java
// Lấy thông tin offer từ contract state
private String[] fetchOffer(long offerId) {
    // Method selector: keccak("offers(uint256)")[0:4]
    // keccak("offers(uint256)") = 0x3ca211ad...
    String encodedCall = "0x3ca211ad"
            + String.format("%064x", BigInteger.valueOf(offerId));

    // ethCall: đọc contract storage tại block "latest"
    EthCall response = web3j.ethCall(
        Transaction.createFunctionCallTransaction(
            "0x0000...0000",  // from: không cần signer cho view call
            BigInteger.ZERO,   // gas
            BigInteger.ZERO,  // gasPrice
            BigInteger.ZERO,  // nonce
            marketplaceAddress,
            BigInteger.ZERO,  // value
            encodedCall
        ),
        DefaultBlockParameter.valueOf("latest")  // ← Dùng "latest"!
    ).send();

    if (response.hasError() || "0x".equals(response.getValue())) {
        return null; // Offer không tồn tại hoặc đã deactivate
    }

    // Decode response hex
    String hexData = response.getValue().substring(2);
    // Offer struct = [offerId][buyer][nftContract][tokenId][price][expiresAt][isActive]
    // Mỗi field = 32 bytes = 64 hex chars
    String nftContract = "0x" + hexData.substring(2*64+24, 3*64);  // skip 2 slots
    long tokenId = new BigInteger(hexData.substring(3*64+2, 4*64), 16).longValue();
    return new String[]{nftContract, String.valueOf(tokenId)};
}
```

### 6.3. Tại Sao Dùng `"latest"` Thay Vì `lastProcessedBlock`?

```java
// SAI — đọc block 0 → storage rỗng → trả về 0x
DefaultBlockParameter.valueOf(BigInteger.valueOf(lastProcessedBlock))

// ĐÚNG — đọc block mới nhất → luôn có data
DefaultBlockParameter.valueOf("latest")
```

Contract state (storage) tồn tại vĩnh viễn trên blockchain. Dùng `"latest"` đảm bảo luôn đọc được data, không phụ thuộc vào `lastProcessedBlock`.

### 6.4. Offer Struct Layout (Decoded)

```
hexData = 0x[slot0][slot1][slot2][slot3][slot4][slot5][slot6]
          0      1      2       3       4      5      6
         64ch   64ch   64ch    64ch    64ch   64ch   64ch

slot 0: offerId     = 0x00...0001
slot 1: buyer      = 0x00...00 + 20-byte-address  → lấy 24→64
slot 2: nftContract= 0x00...00 + 20-byte-address  → lấy 24→64
slot 3: tokenId   = 0x00...2a                    → lấy 2→64
slot 4: price      = 0x00...00...1bc16d674ec80000 (100000000000000000 wei)
slot 5: expiresAt  = 0x00...00...0000018f46e9e00
slot 6: isActive   = 0x00...01
```

---

## 7. Deduplication — Ngăn Duplicate Entry

```java
String txHash = event.getTransactionHash();
if (tradeHistoryRepository.findByTxHash(txHash).isPresent()) {
    seen.add(txHash);  // Đã có trong DB → skip
    continue;
}
```

`trade_history.tx_hash` có UNIQUE constraint. Nếu indexer restart và đọc lại block range trùng lặp, `saveFromEvent` sẽ bị lỗi unique constraint. Kiểm tra trước bằng `findByTxHash` an toàn hơn.

---

## 8. Wei → ETH Conversion

```java
// TradeHistoryService.java
public static String convertToEthString(BigInteger wei) {
    BigInteger eth = wei.divide(BigInteger.TEN.pow(14)); // bỏ 14 số 0 cuối
    String ethStr = eth.toString();
    if (ethStr.length() <= 4) {
        return "0." + ethStr.replaceAll("0", "0"); // pad left với 0
    }
    String intPart = ethStr.substring(0, ethStr.length() - 4);
    String decPart = ethStr.substring(ethStr.length() - 4);
    return intPart + "." + decPart;
}
```

**Ví dụ**:
```
wei = 100000000000000000 (0.1 ETH)
÷ 10^14 = 1000000 → "1000000"
→ "10" + "." + "0000" = "0.1000"
```

---

## 9. Reindex — Backfill Events Cũ

```java
public void reindexFromBlock(long from, long to) {
    // Gọi thủ công để điền data cho block range cũ
    // Ví dụ: reindexFromBlock(6000000, 6010000)
    processItemSoldEvents(fromBlock, toBlock);
    processAuctionFinalizedEvents(fromBlock, toBlock);
    processOfferAcceptedEvents(fromBlock, toBlock);
}
```

Dùng khi backend mới start hoặc muốn backfill dữ liệu từ trước khi indexer chạy.
