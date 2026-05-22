# Công Nghệ và Thư Viện Sử Dụng

## Tổng Quan

Dự án gồm 3 phần chính, mỗi phần dùng các công nghệ riêng:

| Phần | Công nghệ | Ngôn ngữ/Framework |
|-------|-----------|------------------|
| Smart Contracts | Solidity + Foundry | Solidity 0.8.20 |
| Backend | Spring Boot + Java | Java 17 |
| Frontend | React + Vite | JavaScript (ES2022) |

---

## 1. Smart Contracts

### 1.1. Solidity `^0.8.20`

Ngôn ngữ lập trình smart contract cho Ethereum VM.

**Các tính năng quan trọng được dùng:**
- `struct` — định nghĩa cấu trúc dữ liệu tự tạo
- `enum` — định nghĩa kiểu liệt kê
- `mapping` — key-value storage
- `event` — ghi log vào blockchain
- `modifier` — kiểm tra quyền trước khi thực thi
- `try/catch` — xử lý lỗi khi gọi external contract
- `emit` — phát event
- `payable` — cho phép nhận ETH
- `view` / `pure` — chỉ đọc, không tốn gas

**Solidity `^0.8.20`** có built-in overflow/underflow protection.

### 1.2. Foundry / Forge

Framework development, testing, và deployment cho Solidity.

```bash
# Compile
forge build

# Deploy lên Sepolia
forge create --rpc-url $SEPOLIA_RPC \
  --private-key $PK src/NFTMarketplace.sol:NFTMarketplace

# Test
forge test

# Call contract function từ CLI
forge call <address> getAllActiveDirectListings --rpc-url $SEPOLIA_RPC
```

**Các tool của Foundry:**
| Tool | Mục đích |
|------|----------|
| `forge` | Compile, test, deploy, script |
| `cast` | Call contract, encode data, compute selectors |
| `anvil` | Local development chain |

### 1.3. OpenZeppelin Contracts

Thư viện contracts đã được audit cho Solidity.

```solidity
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";
import { ReentrancyGuard } from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import { IERC165 } from "@openzeppelin/contracts/utils/introspection/IERC165.sol";
import { IERC2981 } from "@openzeppelin/contracts/interfaces/IERC2981.sol";
import { IERC721 } from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
```

| Contract | Mục đích |
|---------|----------|
| `Ownable` | Giới hạn quyền gọi hàm admin cho owner |
| `ReentrancyGuard` | Ngăn reentrancy attack |
| `ERC2981` | Per-token royalty standard |
| `ERC721URIStorage` | ERC-721 với mutable tokenURI |
| `IERC721` | Interface cho NFT transfers |

---

## 2. Backend

### 2.1. Spring Boot `3.3.5`

Framework Java enterprise.

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.5'
    id 'java'
}
```

**Key dependencies:**
```groovy
implementation 'org.springframework.boot:spring-boot-starter-web'       // REST API
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'  // MySQL ORM
implementation 'org.springframework.boot:spring-boot-starter-validation' // Input validation
```

### 2.2. Java `17`

LTS version. Dùng:
- Records (`record Dto(...) {}`)
- Pattern matching
- Text blocks
- Switch expressions

### 2.3. Web3j `5.0.0`

Thư viện Java để tương tác với Ethereum blockchain.

```java
import org.web3j.protocol.Web3j;
import org.web3j.protocol.core.DefaultBlockParameter;
import org.web3j.protocol.core.methods.response.EthCall;
import org.web3j.protocol.core.methods.response.EthLog;

// Tạo Web3j instance
Web3j web3j = Web3j.build(new HttpService(rpcUrl));

// Gọi view function
EthCall response = web3j.ethCall(
    Transaction.createFunctionCallTransaction(...),
    DefaultBlockParameter.valueOf("latest")
).send();

// Đọc event logs
EthFilter filter = new EthFilter(from, to, contractAddress);
EthLog ethLog = web3j.ethGetLogs(filter).send();
```

### 2.4. MySQL + JPA

```groovy
implementation 'com.mysql:mysql-connector-j:8.3.0'
```

```java
// Entity
@Entity @Table @Id @GeneratedValue @Column

// Repository
public interface TradeHistoryRepository extends JpaRepository<TradeHistory, Long>

// Query methods tự động
tradeHistoryRepository.findByTxHash(txHash)
tradeHistoryRepository.findByTokenIdAndNftContractOrderByCreatedAtDesc(tokenId, nftContract)
```

### 2.5. Lombok

```java
@Data @Builder @NoArgsConstructor @AllArgsConstructor
@Getter @Setter
@RequiredArgsConstructor
```

Tự động sinh boilerplate code (getter, setter, constructor, builder).

---

## 3. Frontend

### 3.1. React `18`

UI library.

```jsx
import { useState, useEffect, useQuery } from 'react'
import { Outlet, Route, Routes } from 'react-router-dom'
```

Key patterns:
- `useState` / `useEffect` — state management
- `useQuery` (TanStack Query) — server state
- React Router — routing

### 3.2. Vite `^5.4.8`

Build tool và dev server.

```json
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "preview": "vite preview"
}
```

Ưu điểm: **HMR nhanh**, build nhanh hơn webpack.

### 3.3. wagmi `^2.12.22`

React hooks cho Ethereum.

```js
import { useAccount, useReadContract, useWriteContract,
         useWaitForTransactionReceipt } from 'wagmi'
```

**Tại sao dùng wagmi thay vì ethers.js trực tiếp?**

| | wagmi | ethers.js |
|--|-------|-----------|
| API | React hooks | Class-based |
| Caching | Tự động (TanStack Query) | Phải tự quản lý |
| Wallet | Tích hợp connectors | Phải tự cấu hình |
| Type safety | Tốt hơn | Cần manual typing |
| Bundle size | Nhỏ hơn (viem bên dưới) | Lớn hơn |

### 3.4. viem `^2.21.21`

Thư viện Ethereum cho JavaScript.

```js
import { encodeFunctionData, decodeFunctionResult,
         parseAbiItem, concat } from 'viem'
```

Dùng bởi wagmi. Cung cấp:
- ABI encoding/decoding
- RPC type-safe calls
- Chain definitions (sepolia, mainnet, etc.)
- Utility functions (formatEther, parseEther, etc.)

### 3.5. ethers.js `^6.13.2`

Dùng trong dự án cho utilities:

```js
import { ethers } from 'ethers'

ethers.formatEther('1000000000000000000')  // "1.0"
ethers.parseEther('1.0')                  // BigInt
```

### 3.6. TanStack Query `^5.56.2`

Server state management.

```js
import { useQuery } from '@tanstack/react-query'

useQuery({
  queryKey: ['nft-metadata', uri],
  queryFn: () => fetchNftMetadata(uri),
  enabled: !!uri,
  staleTime: 1000 * 60 * 5,  // 5 phút
})
```

**Caching**: Query results được cache. Nếu component re-mount, data từ cache được trả ngay.

### 3.7. Pinata SDK (HTTP API)

Pinata không có official SDK, dùng **axios** gọi REST API:

```js
import axios from 'axios'

axios.post('https://api.pinata.cloud/pinning/pinFileToIPFS', formData, {
  headers: {
    Authorization: `Bearer ${VITE_PINATA_JWT}`,
  }
})
```

### 3.8. Tailwind CSS `^3.4.13`

Utility-first CSS framework.

```jsx
<button className="bg-violet-600 hover:bg-violet-700
           text-white px-4 py-2 rounded-lg
           transition-colors duration-200">
  Connect Wallet
</button>
```

### 3.9. Framer Motion `^11.9.0`

Animation library cho React.

```jsx
import { motion } from 'framer-motion'

<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.5 }}
>
  <NFTCard />
</motion.div>
```

### 3.10. React Hot Toast `^2.4.1`

Toast notifications.

```js
import toast from 'react-hot-toast'

toast.success('NFT minted successfully!')
toast.error('Transaction failed')
toast.loading('Confirming...')
```

### 3.11. React Router `^6.26.2`

Client-side routing.

```jsx
<Routes>
  <Route path="/" element={<HomePage />} />
  <Route path="/nft/:contract/:tokenId" element={<NftDetailPage />} />
  <Route path="/profile/:address" element={<ProfilePage />} />
</Routes>
```

### 3.12. Axios

HTTP client cho backend REST API và IPFS.

```js
import axios from 'axios'

const api = axios.create({ baseURL: API_BASE_URL, timeout: 15000 })
const response = await api.get(`/trades/token/${contract}/${tokenId}`)
```

### 3.13. Lucide React

Icon library.

```jsx
import { Wallet, ShoppingCart, Gavel, Clock } from 'lucide-react'

<Wallet className="w-5 h-5" />
```

---

## 4. RPC Provider — Alchemy

**Alchemy** là RPC provider cho Sepolia (free tier).

```
URL: https://eth-sepolia.g.alchemy.com/v2/<API_KEY>
```

Alchemy cung cấp:
- JSON-RPC endpoint cho `eth_call`, `eth_getLogs`, `eth_sendTransaction`
- Enhanced APIs (debug namespace, traces)
- WebSocket support (cho real-time updates)
- Dashboard để xem usage và debug transactions

---

## 5. Tổng Hợp Dependencies

### package.json (frontend)
```json
{
  "dependencies": {
    "@tanstack/react-query": "^5.56.2",
    "axios": "^1.7.7",
    "ethers": "^6.13.2",
    "framer-motion": "^11.9.0",
    "lucide-react": "^0.447.0",
    "react": "^18.3.1",
    "react-hot-toast": "^2.4.1",
    "react-router-dom": "^6.26.2",
    "viem": "^2.21.21",
    "wagmi": "^2.12.22"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "autoprefixer": "^10.4.20",
    "tailwindcss": "^3.4.13",
    "vite": "^5.4.8"
  }
}
```

### build.gradle (backend)
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web:3.3.5'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa:3.3.5'
    implementation 'org.springframework.boot:spring-boot-starter-validation:3.3.5'
    implementation 'org.web3j:core:5.0.0'
    implementation 'com.mysql:mysql-connector-j:8.3.0'
    implementation 'org.projectlombok:lombok:1.18.34'
    annotationProcessor 'org.projectlombok:lombok:1.18.34'
}
```
