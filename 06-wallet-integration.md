# Tích Hợp Ví MetaMask

## Tổng Quan

Frontend sử dụng **wagmi v2** để quản lý kết nối ví. wagmi hỗ trợ nhiều connectors, trong đó MetaMask là phổ biến nhất.

---

## 1. Cách Hoạt Động

```
User click "Connect Wallet"
        │
        ▼
wagmi connect({ connector: metaMask() })
        │
        ▼
MetaMask popup → "Connect to ArtVerse?"
        │
        ▼
User approve
        │
        ▼
MetaMask gửi accountsChanged event
        │
        ▼
wagmi lưu { address, chainId, ... } trong React context
        │
        ▼
useAccount() → { address: "0x1234..." }
```

---

## 2. Cấu Hình Connectors

```js
// frontend/src/config/wagmi.js
import { metaMask, injected, coinbaseWallet } from 'wagmi/connectors'

export const wagmiConfig = createConfig({
  chains: [mainnet, sepolia],
  connectors: [
    injected(),       // Trình duyệt (MetaMask, Rabby, ...)
    metaMask(),      // MetaMask riêng (tách biệt)
    coinbaseWallet({ appName: 'ArtVerse NFT Marketplace' }),
  ],
  transports: {
    [sepolia.id]: http('https://eth-sepolia.g.alchemy.com/...'),
  },
})
```

---

## 3. Sử Dụng Trong Component

### 3.1. Connect Button

```jsx
import { useAccount, useConnect, useDisconnect } from 'wagmi'

function ConnectButton() {
  const { address, isConnected } = useAccount()
  const { connect, connectors, isPending } = useConnect()
  const { disconnect } = useDisconnect()

  if (isConnected) {
    return (
      <div className="flex items-center gap-3">
        <span className="text-sm text-gray-400">
          {address.slice(0, 6)}...{address.slice(-4)}
        </span>
        <Button variant="outline" onClick={() => disconnect()}>
          Disconnect
        </Button>
      </div>
    )
  }

  return (
    <Button onClick={() => connect({ connector: connectors[0] })}
            disabled={isPending}>
      {isPending ? 'Connecting...' : 'Connect Wallet'}
    </Button>
  )
}
```

### 3.2. Lấy Thông Tin Ví

```jsx
import { useAccount, useBalance, useChainId } from 'wagmi'

function WalletInfo() {
  const { address, isConnected } = useAccount()
  const { data: balance } = useBalance({ address })
  const chainId = useChainId()

  if (!isConnected) return <p>Not connected</p>

  return (
    <div>
      <p>Address: {address}</p>
      <p>Balance: {balance?.formatted} {balance?.symbol}</p>
      <p>Network: {chainId === 11155111 ? 'Sepolia' : 'Other'}</p>
    </div>
  )
}
```

---

## 4. Wagmi Provider Setup

```jsx
// frontend/src/main.jsx
import { WagmiProvider } from 'wagmi'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { wagmiConfig } from './config/wagmi'

const queryClient = new QueryClient()

ReactDOM.createRoot(document.getElementById('root')).render(
  <WagmiProvider config={wagmiConfig}>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </WagmiProvider>
)
```

---

## 5. Wallet Connectors Chi Tiết

| Connector | Cách hoạt động |
|-----------|---------------|
| `injected()` | Dùng ví đã cài trong trình duyệt (MetaMask, Rabby, etc.) |
| `metaMask()` | Kết nối riêng MetaMask (tách biệt khỏi injected) |
| `coinbaseWallet()` | Coinbase Wallet browser extension |

### 5.1. injected()

```js
// Dùng ví bất kỳ đã cài trong trình duyệt
connectors: [injected()]
// User không cần MetaMask riêng — ví nào có dùng đó
```

### 5.2. MetaMask Riêng

```js
// Chỉ dùng MetaMask
connectors: [metaMask()]
// Nếu không có MetaMask → lỗi
```

---

## 6. Cách wagmi Gửi Transaction

```
User gọi writeContract (ví dụ: buyItem)
        │
        ▼
wagmi → connector (MetaMask)
        │
        ▼
MetaMask popup hiển thị:
"ArtVerse wants to:
- Buy NFT for 0.1 ETH
- Estimated gas: 0.002 ETH"
        │
        ▼
User ký (Approve) HOẶC Reject
        │
        ├─ Reject → error returned
        │
        └─ Approve → eth_sendTransaction
                │
                ▼
          Blockchain nhận tx
                │
                ▼
          Transaction receipt
                │
                ▼
          useWaitForTransactionReceipt() → isConfirmed
```

---

## 7. Chain Detection

wagmi tự động kiểm tra user đang ở đúng network:

```jsx
import { useAccount, useChainId } from 'wagmi'
import { sepolia } from 'viem/chains'

function NetworkWarning() {
  const chainId = useChainId()
  const isWrongNetwork = chainId !== sepolia.id

  if (isWrongNetwork) {
    return (
      <div className="bg-yellow-500 text-black p-3">
        ⚠️ Please switch to Sepolia testnet
      </div>
    )
  }
}
```

### Chuyển Network Tự Động

```jsx
import { useSwitchChain } from 'wagmi'

function SwitchButton() {
  const { switchChain } = useSwitchChain()

  return (
    <Button onClick={() => switchChain({ chainId: 11155111 })}>
      Switch to Sepolia
    </Button>
  )
}
```

---

## 8. Multicall — Đọc Nhiều Contract Cùng Lúc

wagmi hỗ trợ multicall để đọc nhiều contract state trong một RPC call:

```jsx
import { useReadContracts } from 'wagmi'

function NftInfo({ tokenId }) {
  const { data } = useReadContracts({
    contracts: [
      { address: NFT_COLLECTION, abi, functionName: 'ownerOf', args: [tokenId] },
      { address: NFT_COLLECTION, abi, functionName: 'tokenURI', args: [tokenId] },
      { address: MARKETPLACE, abi, functionName: 'getActiveListingId',
        args: [NFT_COLLECTION, tokenId] },
    ],
  })

  const [owner, uri, activeListing] = data
  // ...
}
```

---

## 9. Các Hook Quan Trọng

| Hook | Mục đích |
|------|----------|
| `useAccount()` | Lấy address, isConnected |
| `useBalance({ address })` | Lấy ETH balance |
| `useConnect()` | Connect ví |
| `useDisconnect()` | Disconnect ví |
| `useChainId()` | Lấy chain ID |
| `useSwitchChain()` | Chuyển network |
| `useReadContract()` | Gọi view function |
| `useWriteContract()` | Gọi write function (kí transaction) |
| `useWaitForTransactionReceipt()` | Đợi transaction confirmed |
| `useReadContracts()` | Multicall nhiều reads |
