# Frontend Giao Tiếp Smart Contract

## Tổng Quan

Frontend dùng **wagmi v2** + **viem** để giao tiếp với smart contract trên Sepolia. wagmi cung cấp các React hooks, viem xử lý low-level encoding/decoding, và ethers.js được dùng ở một số nơi.

---

## 1. Kiến Trúc Thư Viện

```
Frontend (React)
    │
    ├── wagmi v2         — React hooks + wallet management + chuỗi
    ├── viem v2           — Low-level RPC encoding/decoding
    ├── ethers.js v6      — Contract ABI parsing + utilities
    │
    ▼
Alchemy RPC (https://eth-sepolia.g.alchemy.com)
    │
    ▼
Smart Contract (0x42aD78...)
```

---

## 2. Cấu Hình wagmi (`wagmi.js`)

```js
import { createConfig, http } from 'wagmi'
import { mainnet, sepolia } from 'viem/chains'
import { injected, metaMask, coinbaseWallet } from 'wagmi/connectors'

export const wagmiConfig = createConfig({
  chains: [mainnet, sepolia],
  connectors: [
    injected(),          // Trình duyệt có sẵn (MetaMask, Rabby,...)
    metaMask(),          // MetaMask riêng
    coinbaseWallet({ appName: 'ArtVerse NFT Marketplace' }),
  ],
  transports: {
    [mainnet.id]:    http(),
    [sepolia.id]:    http('https://eth-sepolia.g.alchemy.com/v2/anGVqS5jz_jojA1PyCq6b'),
  },
})
```

**Chuỗi được hỗ trợ**: Sepolia (testnet) là chuỗi chính. Mainnet được cấu hình nhưng không active.

---

## 3. ABI — Application Binary Interface

### 3.1. ABI Là Gì?

ABI định nghĩa **cách encode/decode** dữ liệu khi giao tiếp với contract. Nó cho JavaScript biết:
- Contract có những hàm nào
- Hàm nhận tham số gì, trả về gì
- Cách encode request và decode response

### 3.2. ABI File Trong Project

```js
// frontend/src/abi/nftMarketplace.js
export const NFT_MARKETPLACE_ABI = [
  {
    "type": "function",
    "name": "buyItem",
    "inputs": [{ "name": "listingId", "type": "uint256" }],
    "outputs": [],
    "stateMutability": "payable",   // ← quan trọng: có thể nhận ETH
    "type": "function"
  },
  {
    "type": "function",
    "name": "getAllActiveDirectListings",
    "inputs": [],
    "outputs": [{ "type": "DirectListing[]", "name": "", "components": [...] }],
    "stateMutability": "view",
    "type": "function"
  },
  // ... tất cả functions và events
]
```

### 3.3. Custom ABI Parser (`parseAbiStrings.js`)

viem chuẩn không hiểu **custom struct types** như `DirectListing`, `AuctionListing`. Dự án có custom parser để handle:

```js
import { parseAbiItems } from './parseAbiStrings'

// Thay vì dùng viem's parseAbi, dùng parseAbiItems
const abiItems = parseAbiItems(nftMarketplaceABI)
// → parseAbiItems hiểu "DirectListing[]" và tự tạo type definition
```

---

## 4. Gọi Hàm Đọc (Read Functions) — View Calls

### 4.1. Cách Hoạt Động

```
Frontend → useReadContract hook
    │
    ▼
wagmi encodeFunctionData(functionName, args)
    │
    ▼
viem eth_call → RPC
    │
    ▼
Contract executes read-only (không tốn gas)
    │
    ▼
viem decodeFunctionResult → JavaScript object
    │
    ▼
React component re-render
```

### 4.2. Ví Dụ: Đọc Tất Cả Direct Listings

```js
// hooks/useContract.js
import { useReadContract } from 'wagmi'

export function useDirectListings() {
  return useReadContract({
    address: CONTRACTS.NFT_MARKETPLACE,
    abi: NFT_MARKETPLACE_ABI,
    functionName: 'getAllActiveDirectListings',
  })
}
```

```jsx
// pages/DirectListingsPage.jsx
function DirectListingsPage() {
  const { data: listings, isLoading } = useDirectListings()

  if (isLoading) return <Skeleton />
  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
      {listings?.map(listing => (
        <NFTCard key={listing.listingId} listing={listing} />
      ))}
    </div>
  )
}
```

### 4.3. Các Read Hooks Trong Dự Án

| Hook | Function | Trả về |
|------|----------|---------|
| `useDirectListings()` | `getAllActiveDirectListings()` | `DirectListing[]` |
| `useActiveAuctions()` | `getAllActiveAuctions()` | `AuctionListing[]` |
| `useOffersByToken(c,t)` | `getOffersByToken(c,t)` | `Offer[]` |
| `useOffersByBuyer(a)` | `getOffersByBuyer(a)` | `Offer[]` |
| `useTokenURI(id)` | `tokenURI(id)` | `string` (IPFS URI) |
| `useTokenOwner(id)` | `ownerOf(id)` | `address` |
| `useUserTokens(addr)` | `getTokensByOwner(addr)` | `uint256[]` |
| `useProceeds(addr)` | `proceeds(addr)` | `uint256` (wei) |

---

## 5. Gọi Hàm Ghi (Write Functions) — Transaction Calls

### 5.1. Cách Hoạt Động

```
User click "Buy NFT" button
    │
    ▼
useBuyItem() hook
    │
    ▼
wagmi prepareWriteContract({ ... })
    │
    ▼
MetaMask popup → User ký transaction
    │
    ▼ eth_sendTransaction
Blockchain executes buyItem()
    │
    ▼
Transaction receipt (async)
    │
    ▼
useBuyItem() state: isConfirming → isConfirmed
```

### 5.2. Write Hook Pattern

```js
// hooks/useContract.js
import { useWriteContract, useWaitForTransactionReceipt } from 'wagmi'

export function useBuyItem() {
  const { writeContract, data: hash, ... } = useWriteContract()
  const { isLoading: isConfirming, isSuccess: isConfirmed } =
    useWaitForTransactionReceipt({ hash })

  const action = (listingId, { value: priceInWei }) => {
    writeContract({
      address: CONTRACTS.NFT_MARKETPLACE,
      abi: NFT_MARKETPLACE_ABI,
      functionName: 'buyItem',
      args: [listingId],
      value: priceInWei,  // ← quan trọng: ETH gửi kèm transaction
    })
  }

  return { action, hash, isSubmitting, isConfirming, isConfirmed, error }
}
```

```jsx
// NftDetailPage.jsx
function NftDetailPage() {
  const { action, isConfirming, isConfirmed, error } = useBuyItem()

  return (
    <Button
      onClick={() => action(listingId, { value: price })}
      disabled={isConfirming}
    >
      {isConfirming ? 'Confirming...' : isConfirmed ? 'Purchased!' : 'Buy Now'}
    </Button>
  )
}
```

### 5.3. Các Write Hooks

| Hook | Function | Value Param |
|------|----------|-------------|
| `useCreateDirectListing()` | `createDirectListing()` | — |
| `useBuyItem()` | `buyItem(listingId)` | `value: price` ✅ |
| `useCancelDirectListing()` | `cancelDirectListing()` | — |
| `useUpdateDirectPrice()` | `updateDirectPrice()` | — |
| `useCreateAuction()` | `createAuctionListing()` | — |
| `useStartAuction()` | `startAuction()` | — |
| `usePlaceBid()` | `placeBid(listingId)` | `value: bidAmount` ✅ |
| `useFinalizeAuction()` | `finalizeAuction()` | — |
| `useMakeOffer()` | `makeOffer(...)` | `value: price` ✅ |
| `useCancelOffer()` | `cancelOffer(offerId)` | — |
| `useAcceptOffer()` | `acceptOffer(offerId)` | — |
| `useWithdrawProceeds()` | `withdrawProceeds()` | — |
| `useMintNFT()` | `mintNFT(...)` | — |

---

## 6. Value — Khi Nào Cần Gửi ETH?

```js
// Mua NFT → gửi ETH = giá
const { action } = useBuyItem()
action(listingId, { value: listing.price })

// Đặt Bid → gửi ETH = số tiền bid
const { action } = usePlaceBid()
action(auctionListingId, { value: bidAmount })

// Tạo Offer → gửi ETH = giá offer (escrow)
const { action } = useMakeOffer()
action(nftContract, tokenId, price, duration, { value: price })
```

**Quy tắc**: Nếu function Solidity có `payable`, user phải gửi ETH. Wagmi `value` param cho phép set số ETH gửi kèm transaction.

---

## 7. Contract Addresses

```js
// frontend/src/config/contracts.js
export const CONTRACTS = {
  NFT_MARKETPLACE: import.meta.env.VITE_MARKETPLACE_ADDRESS,
  NFT_COLLECTION:  import.meta.env.VITE_NFT_COLLECTION_ADDRESS,
}
```

Cả ABI và Address phải **khớp nhau** — nếu address trỏ đến contract khác, ABI không hợp lệ.

---

## 8. Cách Đọc ABI Từ Etherscan

Khi contract được verify trên Etherscan, ABI có thể copy trực tiếp:

1. Vào `https://sepolia.etherscan.io/address/0x42aD78a...#code`
2. Scroll xuống "Contract ABI" → Copy
3. Paste vào file trong `frontend/src/abi/`

ABI từ Etherscan bao gồm **tất cả functions, events, errors** của contract.

---

## 9. ethers.js Trong Dự Án

ethers.js được dùng cho một số utility functions:

```js
// utils/index.js
import { ethers } from 'ethers'

// Format wei → ETH string
formatEther(value) {
  return ethers.formatEther(value)
}

// Format ETH string → wei BigInt
parseEther(value) {
  return ethers.parseEther(value).toString()
}
```

---

## 10. Ví Dụ: Luồng Mua NFT Hoàn Chỉnh

```
1. User mở trang NFT: /nft/0xNFT/5
       │
       ▼
2. useDirectListings() → đọc blockchain
       │                 → listing = { listingId: 1, price: 100000000000000000, ... }
       ▼
3. NFTCard hiển thị giá 0.1 ETH
       │
4. User click "Buy Now"
       │
5. MetaMask popup: "Buy NFT for 0.1 ETH"
       │
6. User ký transaction
       │
7. Blockchain xử lý buyItem(1):
       │  listing.isActive = false
       │  proceeds[seller] += 0.0975 ETH
       │  safeTransferFrom(seller, buyer, tokenId)
       │  emit ItemSold(1, buyer, nftContract, 5, 0.1 ETH)
       │
8. Transaction confirmed (block mới)
       │
9. Backend indexer (15s sau):
       │  eth_getLogs ItemSold
       │  decode nftContract, tokenId, price
       │  INSERT trade_history
       │
10. Frontend hiển thị "Purchased!" ✅
```
