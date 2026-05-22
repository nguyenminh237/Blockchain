# IPFS và Metadata NFT

## Tổng Quan

NFT metadata và hình ảnh được lưu trên **IPFS** (InterPlanetary File System), một hệ thống lưu trữ phân tán. Frontend sử dụng **Pinata** làm IPFS gateway để upload và truy cập.

---

## 1. IPFS Là Gì?

```
Traditional:  https://example.com/image.jpg  (server đơn, có thể die)
IPFS:          ipfs://QmXxx...abc            (content-addressed, decentralized)
```

IPFS dùng **content addressing**:
- Ảnh được hash bằng **CID** (Content Identifier)
- `ipfs://QmABC123...` = hash SHA-256 của nội dung file
- Ai cũng có thể serve cùng một file bằng cùng CID
- Không thể thay đổi nội dung mà không đổi CID

---

## 2. Cấu Trúc Metadata NFT (ERC-721)

Theo chuẩn OpenSea metadata format:

```json
{
  "name": "Cosmic Journey #42",
  "description": "A beautiful journey through space...",
  "image": "ipfs://QmXxx.../image.png",
  "attributes": [
    {
      "trait_type": "Background",
      "value": "Deep Space"
    },
    {
      "trait_type": "Rarity",
      "value": "Legendary"
    }
  ]
}
```

| Field | Ý nghĩa |
|--------|---------|
| `name` | Tên NFT |
| `description` | Mô tả |
| `image` | IPFS URI của hình ảnh |
| `attributes` | Traits/properties (tuỳ chọn) |

---

## 3. Luồng Upload Toàn Bộ

```
User chọn ảnh
        │
        ▼
1. Upload ảnh lên IPFS → ipfs://QmImage...
        │
        ▼
2. Tạo JSON metadata
{
  "name": "My NFT",
  "description": "...",
  "image": "ipfs://QmImage...",
  "attributes": [...]
}
        │
        ▼
3. Upload JSON lên IPFS → ipfs://QmMeta...
        │
        ▼
4. Gọi mintNFT(to, "ipfs://QmMeta...", royaltyFee)
        │
        ▼
5. Blockchain lưu tokenURI = "ipfs://QmMeta..."
        │
        ▼
6. Frontend đọc tokenURI → fetch metadata → hiển thị
```

---

## 4. Pinata Service

### 4.1. Upload Ảnh

```js
// frontend/src/services/pinata.js
export async function pinFileToIPFS(file) {
  const formData = new FormData()
  formData.append('file', file)

  const response = await axios.post(
    'https://api.pinata.cloud/pinning/pinFileToIPFS',
    formData,
    {
      headers: {
        'Content-Type': 'multipart/form-data',
        Authorization: `Bearer ${import.meta.env.VITE_PINATA_JWT}`,
      },
    }
  )

  // response.data = { IpfsHash, PinSize, Timestamp, isDuplicate }
  const ipfsHash = response.data.IpfsHash
  const ipfsUri = `ipfs://${ipfsHash}`
  const gatewayUrl = `https://gateway.pinata.cloud/ipfs/${ipfsHash}`

  return { ipfsHash, ipfsUri, gatewayUrl }
}
```

### 4.2. Upload JSON Metadata

```js
export async function pinJSONToIPFS(metadata) {
  const response = await axios.post(
    'https://api.pinata.cloud/pinning/pinJSONToIPFS',
    {
      pinataContent: metadata,
      pinataMetadata: { name: metadata.name },
    },
    {
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${import.meta.env.VITE_PINATA_JWT}`,
      },
    }
  )

  const ipfsHash = response.data.IpfsHash
  return `ipfs://${ipfsHash}`  // Lưu vào contract
}
```

---

## 5. Mint NFT Flow Trong Frontend

```jsx
// pages/MintNFTPage.jsx
async function handleMint() {
  // 1. Upload ảnh
  const { ipfsUri: imageUri } = await pinFileToIPFS(imageFile)
  // imageUri = "ipfs://QmXxx..."

  // 2. Tạo metadata JSON
  const metadata = {
    name: nftName,
    description: nftDescription,
    image: imageUri,
    attributes: [
      { trait_type: 'Artist', value: artistName },
      { trait_type: 'Rarity', value: 'Rare' },
    ],
  }

  // 3. Upload metadata
  const tokenURI = await pinJSONToIPFS(metadata)
  // tokenURI = "ipfs://QmYyy..."

  // 4. Mint NFT
  const { hash } = await mintNFT({
    args: [connectedAccount, tokenURI, royaltyFeeBasisPoints],
  })

  toast.success('NFT minted successfully!')
}
```

---

## 6. Đọc Metadata NFT Trong Frontend

### 6.1. Resolve IPFS URL

```js
// frontend/src/utils/index.js
export function resolveIpfsUrl(uri) {
  if (!uri) return ''
  if (uri.startsWith('ipfs://')) {
    const cid = uri.replace('ipfs://', '')
    return `https://gateway.pinata.cloud/ipfs/${cid}`
    // Hoặc dùng Cloudflare: https://cloudflare-ipfs.com/ipfs/${cid}
  }
  return uri
}
```

### 6.2. Fetch Metadata

```js
export async function fetchNftMetadata(tokenURI) {
  const url = resolveIpfsUrl(tokenURI)
  const response = await fetch(url)
  if (!response.ok) throw new Error('Failed to fetch metadata')
  return response.json()
}
```

### 6.3. Hiển Thị NFT

```jsx
function NFTCard({ listing }) {
  const { data: uri } = useTokenURI(listing.tokenId)  // từ contract
  const { data: metadata, isLoading } = useQuery({
    queryKey: ['nft-metadata', uri],
    queryFn: () => fetchNftMetadata(uri),
    enabled: !!uri,
  })

  if (isLoading) return <Skeleton />

  return (
    <div>
      <img
        src={resolveIpfsUrl(metadata?.image)}
        alt={metadata?.name}
      />
      <h3>{metadata?.name}</h3>
      <p>{metadata?.description}</p>
      <div className="flex flex-wrap gap-2">
        {metadata?.attributes?.map((attr, i) => (
          <span key={i} className="badge">
            {attr.trait_type}: {attr.value}
          </span>
        ))}
      </div>
    </div>
  )
}
```

---

## 7. Ví Dụ Thực Tế

```
1. Upload ảnh "cosmic-journey.png" (2MB)
   → Pinata trả: QmXyz123... (CID v1)
   → URL: https://gateway.pinata.cloud/ipfs/QmXyz123...

2. Tạo metadata:
   {
     "name": "Cosmic Journey #42",
     "description": "...",
     "image": "ipfs://QmXyz123...",
     "attributes": [...]
   }

3. Upload metadata JSON
   → Pinata trả: QmAbc456... (CID mới)
   → tokenURI = "ipfs://QmAbc456..."

4. Gọi mintNFT(minter, "ipfs://QmAbc456...", 500)
   → Blockchain: tokenId #42 được tạo với tokenURI = "ipfs://QmAbc456..."

5. Frontend đọc tokenURI(#42) → "ipfs://QmAbc456..."
   → fetch() → JSON metadata
   → Hiển thị tên, ảnh, attributes
```

---

## 8. Tại Sao Dùng Pinata?

```
Public IPFS gateway (ipfs.io, cloudflare-ipfs.com):
✓ Miễn phí
✗ Có thể chậm hoặc unavailable
✗ Không đảm bảo pin vĩnh viễn

Pinata (pinata.cloud):
✓ Dedicated gateway nhanh
✓ Pin vĩnh viễn (re-pinning tự động)
✓ Quản lý files qua dashboard
✗ Cần API key (JWT token)

API Key lấy ở: https://app.pinata.cloud/developers/api-keys
Cần: VITE_PINATA_JWT=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```
