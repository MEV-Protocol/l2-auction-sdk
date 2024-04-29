# L2 Auction SDK

SDK for L2 auction bidders. To participate in the auction, bidders must be registered. Please contact Mev Protocol for registration. This guide provides information on how to interact with the auction contracts and the L2 RPC endpoints.

## Table of Contents

- [RPC Endpoints](#rpc-endpoints)
- [Bundle JSON Requests and Responses](#json-requests-and-responses)
- [Contracts Addresses](#Contracts-Addresses)
- [Bidding](#Bidding)
    - [Manual Bidding](#Manual-bidding)
    - [Contract Bidding](#Contract-bidding)
- [Send Tx and Bundle](#Send-Tx-and-Bundles)
- [Bidder Contracts](#Bidder-Contracts)
- [Bundler Examples](#bundler-examples)
- [Contributing](#contributing)
- [License](#license)


## RPC Endpoints

- **L2 RPC (TESTNET):**
  - Description: L2 Node RPC (Testnet)
  - URL: [https://holesky-api.securerpc.com/l2](https://holesky-api.securerpc.com/l2/)
  - Methods: eth_*
  - ChainId: 42169

- **Beta bundle RPC (TESTNET):**
  - Description: Beta bundle submission RPC
  - URL: [https://holesky-api.securerpc.com/v2](https://holesky-api.securerpc.com/v2)
  - Method: mev_sendBetaBundle
  - Parameters:
    - `txs`: List of txs as bundle e.g. [0x2323...,]
    - `slot`: slot number e.g. "11282389"
  - ChainId: 17000

## Bundle JSON Requests and Responses

### Example JSON request

```jsonc
{
    "jsonrpc": "2.0",
    "method": "mev_sendBetaBundle",
    "params": [
      {
        "txs": [0x... ],
        "slot": "1001"
      }
    ],
    "id": 8
}
```

### Example JSON response
```jsonc
{
    'jsonrpc': '2.0',
    'id': 1,
    'method': 'mev_sendBetaBundle',
    'result': '0x79e5cba7876f532218ac35a357209800be2362dd2e3f1e6dc5974698f0d7cee4'
}
```

## Contracts Addresses

### L1 Addresses (Testnet)
| Contract         | Address                                                                                                                       |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| L1StandardBridge | [0x3Ae5Ca0B05bE12d4FF9983Ed70D86de9C34e820C](https://holesky.etherscan.io/address/0x3Ae5Ca0B05bE12d4FF9983Ed70D86de9C34e820C) |

### L2 Addresses (Testnet)
| Contract        | Address                                                                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| WETH            | [0x4200000000000000000000000000000000000006](https://holesky-blockscout.securerpc.com/address/0x4200000000000000000000000000000000000006) |
| Auctioneer      | [0x10DeC79E201FE7b27b8c4A1524d9733727D60ea4](https://holesky-blockscout.securerpc.com/address/0x10DeC79E201FE7b27b8c4A1524d9733727D60ea4) |
| SettlementHouse | [0x46bb4fE80C04b5C28C761ADdd43FD10fFCcB57CE](https://holesky-blockscout.securerpc.com/address/0x46bb4fE80C04b5C28C761ADdd43FD10fFCcB57CE) |
| Accountant      | [0x6B5021E079a3941cac287a4F24F411B1Ee87222f](https://holesky-blockscout.securerpc.com/address/0x6B5021E079a3941cac287a4F24F411B1Ee87222f) |


## Bridge
Before bidding, you will need to bridge ETH to L2 for gas fees.
```bash
# Fund L2 address by sending ETH to the bridge address.
cast send -i $L1_BRIDGE --value 0.01ether --rpc-url $L1_RPC_URL --private-key $PRIVATE_KEY

# Confirming the bridged Holesky ETH on L2
cast balance $WALLET_ADDRESS --rpc-url $L2_RPC_URL
```

## Bidding
***Only registered bidders can participate in the auction. Operators can onboard new bidders through the contract. Please contact Mev Protocol to proceed.***

The following code snippet uses [Foundry](https://book.getfoundry.sh/getting-started/installation) as example.





### Manual bidding

#### Wrap ETH for bidding
```bash
cast send $WETH "deposit()" --value 1000gwei --private-key $PRIVATE_KEY --rpc-url $L2_RPC_URL

# allow Auctioneer to spend WETH
cast send $WETH "approve(address,uint256)" $AUCTIONEER $(cast max-uint) --private-key $PRIVATE_KEY --rpc-url $L2_RPC_URL
```

#### Get slot number of open auctions
```bash
cast call $AUCTIONEER "getOpenAuctions()" --rpc-url $L2_RPC_URL
```

#### Place a bid
To check for bidderId when registered, call `IdMap` on the contract:

```solidity
function IdMap(address bidder) external view returns (uint8 id);
```

Bids are packed by price, amount, bidderId
```solidity
/**
* @dev Packed Bid details into a uint256 for submission.
*
* @param bidPrice Price per item.
* @param itemsToBuy Items to buy in the auction.
* @param bidderId Id for bidder
* @return packedBid for auction submission
*/
function packBid(uint256 bidPrice, uint256 itemsToBuy, uint256 bidderId)
    external
    pure
    returns (uint256 packedBid);
```

For example:
```bash
# Packing a bid for 21,000 gas and 0.01 gwei per gas
cast call $AUCTIONEER \
    "packBid(uint128,uint120,uint8)" 0.01gwei 21000 $(cast call $AUCTIONEER "IdMap(address)" $WALLET_ADDRESS --rpc-url $L2_RPC_URL) \
  --rpc-url $L2_RPC_URL

# Placing a bid
cast send $AUCTIONEER \
  "bid(uint256,uint256[] memory)" SLOT_NUMBER ["0x...(Packed Bid)"] \
  --private-key $PRIVATE_KEY \
  --rpc-url $L2_RPC_URL
```

#### Winning bid info

After an auction is closed, bidders can query their bid results:
```solidity
/**
 * @dev Retrieve information about a bidder after auction settlement.
 *
 * @param slot The slot identifier of the auction.
 * @param bidder The address of the bidder for whom information is requested.
 * @return itemsBought The number of items bought by the bidder in the specified auction.
 * @return amountOwed The amount owed by the bidder for the items bought in the specified auction.
 *
 * Requirements:
 * - The auction must have been settled.
 * - The provided `bidder` address must be valid and have participated in the auction.
 *
 */
function getBidderInfo(uint256 slot, address bidder)
    external
    view
    returns (uint120 itemsBought, uint128 amountOwed);
```

```bash
cast call $AUCTIONEER \
  "getBidderInfo(uint256,address)" SLOT_NUMBER $WALLET_ADDRESS \
  --rpc-url $L2_RPC_URL
```

---


### Contract bidding
Bidder contract requirements:
- Sufficient WETH balance and approval
- Registered by operator
- Implement `Bidder` interface


A minimal viable bidder is provided below, this contract will bid for every opened auctions:
```solidity
/// SPDX-License-Identifier: UPL-1.0
pragma solidity ^0.8.25;

import {WETH} from "solmate/tokens/WETH.sol";
import {ERC6909} from "solmate/tokens/ERC6909.sol";

interface Bidder {
    /**
     * @dev Get the bid from a bidder for a specific slot and round.
     * @param slot The auction slot.
     * @return packedBids Array of bids (in a packed format). uint256(uint128(bidPrice),uint120(itemsToBuy),uint8(bidderId))
     */
    function getBid(uint256 slot) external view returns (uint256[] memory packedBids);
}

interface SettlementHouse {
    function submitBundle(uint256 slot, uint256 amount, bytes32[] calldata hashes) external;
}

/// @title MockBidder
contract MockBidder is Bidder {
    uint256[] public bids;
    ERC6909 auctioneer;
    SettlementHouse house;
    WETH weth;

    constructor(WETH _weth, address _auctioneer, address settlement) {
        weth = _weth;
        auctioneer = ERC6909(_auctioneer);
        house = SettlementHouse(settlement);
        weth.approve(_auctioneer, type(uint256).max);
    }

    function setBids(uint256[] memory newBids) public {
        bids = newBids;
    }

    function getBid(uint256) external view returns (uint256[] memory packedBids) {
        return bids;
    }

    function submit(uint256 slot, uint256 amount, bytes32[] calldata hashes) external {
        auctioneer.approve(address(house), slot, amount);
        house.submitBundle(slot, amount, hashes);
    }
}
```

## Send Tx and Bundles

#### Check gas token balance for the slot
```bash
cast call $AUCTIONEER \
  "balanceOf(address,uint256)" $WALLET_ADDRESS SLOT_NUMBER \
  --rpc-url $L2_RPC_URL
```

#### Build and sign transaction on L1
- Prioriy fee **MUST** be **ZERO**
- RPC URL should be L1
- Return signed transaction
```bash
# trasnfer 0.01 ether to self
cast mktx $WALLET_ADDRESS \
  --value 0.01ether \
  --priority-gas-price 0 \
  --private-key $PRIVATE_KEY \
  --rpc-url $L1_RPC_URL
```

#### Submit Tx to Beta Bundle RPC
Use the `mev_sendBetaBundle` method to send the bundle to L1 RPC. The bundle will be stored in L1 RPC, and L1 RPC will return the `bundle hash`.

```bash
curl -X POST \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"mev_sendBetaBundle","params":[{"slot": "SLOT_NUMBER","txs":["0x..SIGNED TX"]}]}' \
  $BETA_BUNDLE_RPC
```

#### Pay gas token for the bundle on L2
```solidity
/*
 * @dev Function to submit a bundle of transactions.
 * @param slot The slot of the futures token being deposited
 * @param amountOfGas Amount of futures tokens to deposit
 * @param bundleHashes List of bundle hashes
 */
function submitBundle(uint256 slot, uint256 amountOfGas, bytes32[] calldata bundleHashes) external
```
Submit the `bundle hash` to `SettlementHouse`. During this process, the `submitBundle` in `SettlementHouse` will check and burn future Gas tokens.
```bash
cast send $SETTLEMENT_HOUSE \
  "submitBundle(uint256,uint256,bytes32[])" SLOT_NUMBER GAS_TOKEN_AMOUNT ["0x..BUNDLE_HASH"] \
  --private-key $PRIVATE_KEY \
  --rpc-url $L2_RPC_URL
```
The block builder will query the bundle hash in the `SettlementHouse` contract. If a bundle in the bundle pool is in the `SettlementHouse` contract, it will be included in the beta block by the block builder.


## Bidder Contracts

[Source code for a fully functional open bidder is provided](open-bidder-contracts/)

## Bundler Examples

[Source code for a fully functional bundler is provided](beta-bundles-py/)

## Contributing

Contributions are welcome! If you find any issues or have suggestions for improvements, feel free to open an issue or submit a pull request.

## License

This project is licensed under the [MIT License](LICENSE).
