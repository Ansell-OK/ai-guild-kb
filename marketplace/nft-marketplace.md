# NFT Marketplace Implementation Guide

## 📖 Overview

An NFT marketplace allows users to list, buy, and sell NFTs. This guide covers building a complete marketplace with listing management, purchasing, bidding, and royalty systems.

### What You'll Learn
- Marketplace contract architecture
- Listing and delisting NFTs
- Purchase mechanisms
- Bidding systems
- Royalty implementation
- Frontend marketplace UI

### Prerequisites
- Understanding of SIP-009 NFTs
- Wallet integration knowledge
- Basic Clarity contract development

---

## 🎯 Key Concepts

### Marketplace Components

**1. Listings**
- NFT ownership verification
- Price setting
- Expiration management

**2. Purchases**
- Payment processing
- NFT transfer
- Fee distribution

**3. Offers/Bids**
- Bid placement
- Bid acceptance
- Bid cancellation

**4. Royalties**
- Creator fees
- Platform fees
- Payment splitting

---

## 💻 Basic Marketplace Contract

### Complete Implementation

```clarity
;; NFT Marketplace Contract
;; Allows listing, buying, and bidding on NFTs

;; Import SIP-009 trait
(use-trait nft-trait 'SP2PABAF9FTAJYNFZH93XENAJ8FVY99RRM50D2JG9.nft-trait.nft-trait)

;; Constants
(define-constant contract-owner tx-sender)
(define-constant marketplace-fee u250) ;; 2.5% (250/10000)

;; Error codes
(define-constant err-not-owner (err u100))
(define-constant err-listing-not-found (err u101))
(define-constant err-nft-not-listed (err u102))
(define-constant err-insufficient-payment (err u103))
(define-constant err-nft-transfer-failed (err u104))
(define-constant err-payment-failed (err u105))
(define-constant err-listing-expired (err u106))
(define-constant err-unauthorized (err u107))

;; Data structures

;; Listing information
(define-map listings
    { nft-contract: principal, token-id: uint }
    {
        seller: principal,
        price: uint,
        royalty: uint, ;; Percentage to creator (e.g., 500 = 5%)
        creator: principal,
        expiry: uint ;; Block height
    }
)

;; Bid information
(define-map bids
    { nft-contract: principal, token-id: uint, bidder: principal }
    { amount: uint, expiry: uint }
)

;; -----------------------------------------------------------
;; Listing Functions
;; -----------------------------------------------------------

;; List an NFT for sale
(define-public (list-nft
    (nft-contract <nft-trait>)
    (nft-asset {nft-contract: principal, token-id: uint})
    (price uint)
    (royalty uint)
    (creator principal)
    (expiry uint)
)
    (let
        (
            (token-owner (unwrap! (contract-call? nft-contract get-owner (get token-id nft-asset)) err-nft-transfer-failed))
        )
        ;; Verify sender owns the NFT
        (asserts! (is-eq (some tx-sender) token-owner) err-not-owner)
        
        ;; Royalty can't exceed 25%
        (asserts! (<= royalty u2500) err-unauthorized)
        
        ;; Store listing
        (ok (map-set listings
            nft-asset
            {
                seller: tx-sender,
                price: price,
                royalty: royalty,
                creator: creator,
                expiry: expiry
            }
        ))
    )
)

;; Delist an NFT
(define-public (delist-nft (nft-asset {nft-contract: principal, token-id: uint}))
    (let
        (
            (listing (unwrap! (map-get? listings nft-asset) err-listing-not-found))
        )
        ;; Only seller can delist
        (asserts! (is-eq tx-sender (get seller listing)) err-unauthorized)
        
        ;; Remove listing
        (ok (map-delete listings nft-asset))
    )
)

;; Update listing price
(define-public (update-price
    (nft-asset {nft-contract: principal, token-id: uint})
    (new-price uint)
)
    (let
        (
            (listing (unwrap! (map-get? listings nft-asset) err-listing-not-found))
        )
        ;; Only seller can update
        (asserts! (is-eq tx-sender (get seller listing)) err-unauthorized)
        
        ;; Update price
        (ok (map-set listings
            nft-asset
            (merge listing { price: new-price })
        ))
    )
)

;; -----------------------------------------------------------
;; Purchase Functions
;; -----------------------------------------------------------

;; Purchase a listed NFT
(define-public (purchase-nft
    (nft-contract <nft-trait>)
    (nft-asset {nft-contract: principal, token-id: uint})
)
    (let
        (
            (listing (unwrap! (map-get? listings nft-asset) err-listing-not-found))
            (price (get price listing))
            (seller (get seller listing))
            (creator (get creator listing))
            (royalty-amount (/ (* price (get royalty listing)) u10000))
            (marketplace-amount (/ (* price marketplace-fee) u10000))
            (seller-amount (- (- price royalty-amount) marketplace-amount))
        )
        ;; Check listing hasn't expired
        (asserts! (< block-height (get expiry listing)) err-listing-expired)
        
        ;; Transfer payment to seller
        (try! (stx-transfer? seller-amount tx-sender seller))
        
        ;; Transfer royalty to creator
        (if (> royalty-amount u0)
            (try! (stx-transfer? royalty-amount tx-sender creator))
            true
        )
        
        ;; Transfer marketplace fee
        (try! (stx-transfer? marketplace-amount tx-sender contract-owner))
        
        ;; Transfer NFT to buyer
        (try! (contract-call? nft-contract transfer 
            (get token-id nft-asset) 
            seller 
            tx-sender
        ))
        
        ;; Remove listing
        (map-delete listings nft-asset)
        
        ;; Print event
        (print {
            event: "nft-purchased",
            nft-contract: (get nft-contract nft-asset),
            token-id: (get token-id nft-asset),
            buyer: tx-sender,
            seller: seller,
            price: price
        })
        
        (ok true)
    )
)

;; -----------------------------------------------------------
;; Bidding Functions
;; -----------------------------------------------------------

;; Place a bid on an NFT
(define-public (place-bid
    (nft-asset {nft-contract: principal, token-id: uint})
    (amount uint)
    (expiry uint)
)
    (begin
        ;; Lock STX for the bid
        (try! (stx-transfer? amount tx-sender (as-contract tx-sender)))
        
        ;; Store bid
        (ok (map-set bids
            {
                nft-contract: (get nft-contract nft-asset),
                token-id: (get token-id nft-asset),
                bidder: tx-sender
            }
            { amount: amount, expiry: expiry }
        ))
    )
)

;; Cancel a bid
(define-public (cancel-bid
    (nft-asset {nft-contract: principal, token-id: uint})
)
    (let
        (
            (bid (unwrap! (map-get? bids {
                nft-contract: (get nft-contract nft-asset),
                token-id: (get token-id nft-asset),
                bidder: tx-sender
            }) err-listing-not-found))
        )
        ;; Return locked STX
        (try! (as-contract (stx-transfer? (get amount bid) tx-sender (get bidder bid))))
        
        ;; Remove bid
        (ok (map-delete bids {
            nft-contract: (get nft-contract nft-asset),
            token-id: (get token-id nft-asset),
            bidder: tx-sender
        }))
    )
)

;; Accept a bid (seller only)
(define-public (accept-bid
    (nft-contract <nft-trait>)
    (nft-asset {nft-contract: principal, token-id: uint})
    (bidder principal)
)
    (let
        (
            (listing (unwrap! (map-get? listings nft-asset) err-listing-not-found))
            (bid (unwrap! (map-get? bids {
                nft-contract: (get nft-contract nft-asset),
                token-id: (get token-id nft-asset),
                bidder: bidder
            }) err-listing-not-found))
            (bid-amount (get amount bid))
            (royalty-amount (/ (* bid-amount (get royalty listing)) u10000))
            (marketplace-amount (/ (* bid-amount marketplace-fee) u10000))
            (seller-amount (- (- bid-amount royalty-amount) marketplace-amount))
        )
        ;; Only seller can accept
        (asserts! (is-eq tx-sender (get seller listing)) err-unauthorized)
        
        ;; Check bid hasn't expired
        (asserts! (< block-height (get expiry bid)) err-listing-expired)
        
        ;; Transfer payment to seller from contract
        (try! (as-contract (stx-transfer? seller-amount tx-sender (get seller listing))))
        
        ;; Transfer royalty to creator
        (if (> royalty-amount u0)
            (try! (as-contract (stx-transfer? royalty-amount tx-sender (get creator listing))))
            true
        )
        
        ;; Transfer marketplace fee
        (try! (as-contract (stx-transfer? marketplace-amount tx-sender contract-owner)))
        
        ;; Transfer NFT to bidder
        (try! (contract-call? nft-contract transfer 
            (get token-id nft-asset)
            (get seller listing)
            bidder
        ))
        
        ;; Remove listing and bid
        (map-delete listings nft-asset)
        (map-delete bids {
            nft-contract: (get nft-contract nft-asset),
            token-id: (get token-id nft-asset),
            bidder: bidder
        })
        
        (print {
            event: "bid-accepted",
            nft-contract: (get nft-contract nft-asset),
            token-id: (get token-id nft-asset),
            seller: (get seller listing),
            buyer: bidder,
            price: bid-amount
        })
        
        (ok true)
    )
)

;; -----------------------------------------------------------
;; Read-only Functions
;; -----------------------------------------------------------

;; Get listing details
(define-read-only (get-listing (nft-asset {nft-contract: principal, token-id: uint}))
    (ok (map-get? listings nft-asset))
)

;; Get bid details
(define-read-only (get-bid 
    (nft-asset {nft-contract: principal, token-id: uint})
    (bidder principal)
)
    (ok (map-get? bids {
        nft-contract: (get nft-contract nft-asset),
        token-id: (get token-id nft-asset),
        bidder: bidder
    }))
)

;; Check if NFT is listed
(define-read-only (is-listed (nft-asset {nft-contract: principal, token-id: uint}))
    (ok (is-some (map-get? listings nft-asset)))
)

;; Calculate fees for a price
(define-read-only (calculate-fees (price uint) (royalty uint))
    (let
        (
            (royalty-amount (/ (* price royalty) u10000))
            (marketplace-amount (/ (* price marketplace-fee) u10000))
            (seller-amount (- (- price royalty-amount) marketplace-amount))
        )
        (ok {
            total: price,
            seller-receives: seller-amount,
            royalty-amount: royalty-amount,
            marketplace-fee: marketplace-amount
        })
    )
)
```

---

## 🌐 Frontend Marketplace UI

### Complete Marketplace Component

```javascript
import React, { useState, useEffect } from 'react';
import { useWallet } from './WalletContext';
import {
  openContractCall,
  makeStandardSTXPostCondition,
  makeStandardNonFungiblePostCondition,
  FungibleConditionCode,
  NonFungibleConditionCode,
  createAssetInfo,
  uintCV,
  standardPrincipalCV,
  contractPrincipalCV,
  tupleCV
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';

function NFTMarketplace() {
  const { address, isConnected } = useWallet();
  const [listings, setListings] = useState([]);
  const [loading, setLoading] = useState(true);

  const MARKETPLACE_CONTRACT = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM';
  const NFT_CONTRACT = 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM';

  // Fetch all listings
  useEffect(() => {
    fetchListings();
  }, []);

  const fetchListings = async () => {
    // In production, you'd fetch from an indexer API
    // For now, this is a placeholder
    setLoading(false);
  };

  return (
    <div className="marketplace">
      <h1>NFT Marketplace</h1>
      
      {isConnected && (
        <div className="user-actions">
          <ListNFTModal address={address} />
        </div>
      )}

      <div className="listings-grid">
        {loading ? (
          <p>Loading listings...</p>
        ) : listings.length === 0 ? (
          <p>No NFTs listed</p>
        ) : (
          listings.map((listing) => (
            <NFTCard key={`${listing.nftContract}-${listing.tokenId}`} listing={listing} />
          ))
        )}
      </div>
    </div>
  );
}

// Component for listing an NFT
function ListNFTModal({ address }) {
  const [isOpen, setIsOpen] = useState(false);
  const [tokenId, setTokenId] = useState('');
  const [price, setPrice] = useState('');
  const [royalty, setRoyalty] = useState('5'); // 5%
  const [expiryDays, setExpiryDays] = useState('30');

  const listNFT = async () => {
    const priceInMicroSTX = parseFloat(price) * 1000000;
    const royaltyBps = parseFloat(royalty) * 100; // Convert % to basis points
    const expiryBlock = 144 * parseInt(expiryDays); // Roughly 144 blocks per day
    const currentBlock = 100000; // You'd get this from API

    const nftAsset = tupleCV({
      'nft-contract': contractPrincipalCV(NFT_CONTRACT, 'my-nft'),
      'token-id': uintCV(parseInt(tokenId))
    });

    const options = {
      network: new StacksTestnet(),
      contractAddress: MARKETPLACE_CONTRACT,
      contractName: 'nft-marketplace',
      functionName: 'list-nft',
      functionArgs: [
        contractPrincipalCV(NFT_CONTRACT, 'my-nft'),
        nftAsset,
        uintCV(priceInMicroSTX),
        uintCV(royaltyBps),
        standardPrincipalCV(address), // creator
        uintCV(currentBlock + expiryBlock)
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`NFT listed! TX: ${data.txId}`);
        setIsOpen(false);
      },
    };

    await openContractCall(options);
  };

  if (!isOpen) {
    return (
      <button onClick={() => setIsOpen(true)} className="list-btn">
        List NFT
      </button>
    );
  }

  return (
    <div className="modal">
      <div className="modal-content">
        <h2>List Your NFT</h2>
        
        <input
          type="number"
          placeholder="Token ID"
          value={tokenId}
          onChange={(e) => setTokenId(e.target.value)}
        />

        <input
          type="number"
          step="0.01"
          placeholder="Price (STX)"
          value={price}
          onChange={(e) => setPrice(e.target.value)}
        />

        <input
          type="number"
          step="0.1"
          placeholder="Royalty (%)"
          value={royalty}
          onChange={(e) => setRoyalty(e.target.value)}
        />

        <input
          type="number"
          placeholder="Expiry (days)"
          value={expiryDays}
          onChange={(e) => setExpiryDays(e.target.value)}
        />

        <div className="modal-actions">
          <button onClick={listNFT} className="submit-btn">
            List NFT
          </button>
          <button onClick={() => setIsOpen(false)} className="cancel-btn">
            Cancel
          </button>
        </div>
      </div>
    </div>
  );
}

// Component for displaying an NFT card
function NFTCard({ listing }) {
  const { address, isConnected } = useWallet();
  const [bidAmount, setBidAmount] = useState('');
  const [showBidInput, setShowBidInput] = useState(false);

  const purchaseNFT = async () => {
    const nftAsset = tupleCV({
      'nft-contract': contractPrincipalCV(listing.nftContract, 'my-nft'),
      'token-id': uintCV(listing.tokenId)
    });

    // Create post conditions
    const stxPostCondition = makeStandardSTXPostCondition(
      address,
      FungibleConditionCode.Equal,
      listing.price
    );

    const nftPostCondition = makeStandardNonFungiblePostCondition(
      listing.seller,
      NonFungibleConditionCode.Sends,
      createAssetInfo(listing.nftContract, 'my-nft', 'my-nft'),
      uintCV(listing.tokenId)
    );

    const options = {
      network: new StacksTestnet(),
      contractAddress: MARKETPLACE_CONTRACT,
      contractName: 'nft-marketplace',
      functionName: 'purchase-nft',
      functionArgs: [
        contractPrincipalCV(listing.nftContract, 'my-nft'),
        nftAsset
      ],
      postConditions: [stxPostCondition, nftPostCondition],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Purchase successful! TX: ${data.txId}`);
      },
    };

    await openContractCall(options);
  };

  const placeBid = async () => {
    const bidInMicroSTX = parseFloat(bidAmount) * 1000000;
    const expiryBlock = 144 * 7; // 7 days
    const currentBlock = 100000;

    const nftAsset = tupleCV({
      'nft-contract': contractPrincipalCV(listing.nftContract, 'my-nft'),
      'token-id': uintCV(listing.tokenId)
    });

    const options = {
      network: new StacksTestnet(),
      contractAddress: MARKETPLACE_CONTRACT,
      contractName: 'nft-marketplace',
      functionName: 'place-bid',
      functionArgs: [
        nftAsset,
        uintCV(bidInMicroSTX),
        uintCV(currentBlock + expiryBlock)
      ],
      postConditionMode: PostConditionMode.Deny,
      onFinish: (data) => {
        alert(`Bid placed! TX: ${data.txId}`);
        setShowBidInput(false);
      },
    };

    await openContractCall(options);
  };

  return (
    <div className="nft-card">
      <img src={listing.imageUrl} alt={`NFT #${listing.tokenId}`} />
      
      <div className="nft-info">
        <h3>NFT #{listing.tokenId}</h3>
        <p className="price">{(listing.price / 1000000).toFixed(2)} STX</p>
        <p className="seller">Seller: {listing.seller.slice(0, 8)}...</p>
        <p className="royalty">Royalty: {(listing.royalty / 100).toFixed(1)}%</p>
      </div>

      {isConnected && address !== listing.seller && (
        <div className="nft-actions">
          <button onClick={purchaseNFT} className="buy-btn">
            Buy Now
          </button>
          
          {!showBidInput ? (
            <button onClick={() => setShowBidInput(true)} className="bid-btn">
              Place Bid
            </button>
          ) : (
            <div className="bid-input">
              <input
                type="number"
                step="0.01"
                placeholder="Bid amount (STX)"
                value={bidAmount}
                onChange={(e) => setBidAmount(e.target.value)}
              />
              <button onClick={placeBid} className="submit-bid-btn">
                Submit
              </button>
            </div>
          )}
        </div>
      )}

      {isConnected && address === listing.seller && (
        <div className="seller-actions">
          <button className="delist-btn">Delist</button>
          <button className="view-bids-btn">View Bids</button>
        </div>
      )}
    </div>
  );
}

export default NFTMarketplace;
```

---

## 🧪 Testing Marketplace

```typescript
import { Clarinet, Tx, Chain, Account, types } from 'https://deno.land/x/clarinet@v1.0.0/index.ts';

Clarinet.test({
  name: "Can list NFT for sale",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const deployer = accounts.get('deployer')!;
    const seller = accounts.get('wallet_1')!;
    
    // First mint an NFT
    chain.mineBlock([
      Tx.contractCall(
        'my-nft',
        'mint',
        [types.principal(seller.address)],
        deployer.address
      )
    ]);
    
    // List the NFT
    let block = chain.mineBlock([
      Tx.contractCall(
        'nft-marketplace',
        'list-nft',
        [
          types.principal(`${deployer.address}.my-nft`),
          types.tuple({
            'nft-contract': types.principal(`${deployer.address}.my-nft`),
            'token-id': types.uint(1)
          }),
          types.uint(1000000), // 1 STX
          types.uint(500), // 5% royalty
          types.principal(seller.address),
          types.uint(1000) // expiry block
        ],
        seller.address
      )
    ]);
    
    block.receipts[0].result.expectOk();
  }
});

Clarinet.test({
  name: "Can purchase listed NFT",
  async fn(chain: Chain, accounts: Map<string, Account>) {
    const deployer = accounts.get('deployer')!;
    const seller = accounts.get('wallet_1')!;
    const buyer = accounts.get('wallet_2')!;
    
    // Mint and list NFT
    chain.mineBlock([
      Tx.contractCall('my-nft', 'mint', [types.principal(seller.address)], deployer.address)
    ]);
    
    chain.mineBlock([
      Tx.contractCall(
        'nft-marketplace',
        'list-nft',
        [
          types.principal(`${deployer.address}.my-nft`),
          types.tuple({
            'nft-contract': types.principal(`${deployer.address}.my-nft`),
            'token-id': types.uint(1)
          }),
          types.uint(1000000),
          types.uint(500),
          types.principal(seller.address),
          types.uint(1000)
        ],
        seller.address
      )
    ]);
    
    // Purchase NFT
    let block = chain.mineBlock([
      Tx.contractCall(
        'nft-marketplace',
        'purchase-nft',
        [
          types.principal(`${deployer.address}.my-nft`),
          types.tuple({
            'nft-contract': types.principal(`${deployer.address}.my-nft`),
            'token-id': types.uint(1)
          })
        ],
        buyer.address
      )
    ]);
    
    block.receipts[0].result.expectOk();
    
    // Verify new owner
    let owner = chain.callReadOnlyFn(
      'my-nft',
      'get-owner',
      [types.uint(1)],
      deployer.address
    );
    
    owner.result.expectOk().expectSome().expectPrincipal(buyer.address);
  }
});
```

---

## ⚠️ Common Pitfalls

### 1. **Not Verifying NFT Ownership**

Always check the seller owns the NFT before listing

### 2. **Forgetting Expiry Checks**

Check block height against expiry in purchases and bids

### 3. **Incorrect Fee Calculations**

Test fee math thoroughly with edge cases

### 4. **Missing Post Conditions**

Always protect buyers and sellers with post conditions

---

## 🚀 Advanced Features

- Auction system with time-based bidding
- Bundle sales (multiple NFTs)
- Collection offers
- Dutch auctions
- Lazy minting

---

## 🎓 Next Steps

**Related Guides:**
- → Advanced Bidding Systems
- → Royalty Standards
- → Marketplace Analytics