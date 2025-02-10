# Subgraph Setup Guide

## Why Use a Subgraph?

The Graph Protocol provides a better way to query blockchain data compared to direct RPC calls:

- **Pre-indexed data**: No need to paginate through blocks
- **GraphQL queries**: More flexible and efficient data fetching
- **Real-time updates**: Automatic indexing of new events
- **No rate limits**: Unlike RPC providers like Alchemy
- **Complex filtering**: Easy to query specific conditions
- **Aggregated data**: Can pre-compute analytics

## Setup Steps

### 1. Create a Subgraph

```bash
# Install The Graph CLI
npm install -g @graphprotocol/graph-cli

# Initialize subgraph
graph init --studio your-subgraph-name

# Navigate to subgraph directory
cd your-subgraph-name
```

### 2. Configure `subgraph.yaml`

```yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: PredictionMarket
    network: base
    source:
      address: "YOUR_CONTRACT_ADDRESS"
      abi: PredictionMarket
      startBlock: YOUR_START_BLOCK
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Market
        - SharesPurchased
        - MarketResolved
      abis:
        - name: PredictionMarket
          file: ./abis/PredictionMarket.json
      eventHandlers:
        - event: SharesPurchased(indexed uint256,indexed address,bool,uint256)
          handler: handleSharesPurchased
        - event: MarketCreated(indexed uint256,string,uint256)
          handler: handleMarketCreated
        - event: MarketResolved(indexed uint256,bool)
          handler: handleMarketResolved
      file: ./src/mapping.ts
```

### 3. Define Schema (`schema.graphql`)

```graphql
type Market @entity {
  id: ID!
  marketId: BigInt!
  question: String!
  endTime: BigInt!
  resolved: Boolean!
  winner: Boolean
  totalVolume: BigInt!
  totalTrades: BigInt!
  createdAt: BigInt!
  sharesPurchaseds: [SharesPurchased!]! @derivedFrom(field: "market")
}

type SharesPurchased @entity {
  id: ID!
  market: Market!
  marketId: BigInt!
  buyer: Bytes!
  isOptionA: Boolean!
  amount: BigInt!
  blockNumber: BigInt!
  blockTimestamp: BigInt!
  transactionHash: Bytes!
}

type DailyMarketStats @entity {
  id: ID!
  market: Market!
  date: BigInt!
  optionAVolume: BigInt!
  optionBVolume: BigInt!
  totalVolume: BigInt!
  totalTrades: BigInt!
  priceA: BigDecimal!
  priceB: BigDecimal!
}
```

### 4. Create Mapping (`src/mapping.ts`)

```typescript
import { BigInt, BigDecimal } from "@graphprotocol/graph-ts";
import {
  SharesPurchased as SharesPurchasedEvent,
  MarketCreated as MarketCreatedEvent,
  MarketResolved as MarketResolvedEvent,
} from "../generated/PredictionMarket/PredictionMarket";
import { Market, SharesPurchased, DailyMarketStats } from "../generated/schema";

export function handleMarketCreated(event: MarketCreatedEvent): void {
  let market = new Market(event.params.marketId.toString());
  market.marketId = event.params.marketId;
  market.question = event.params.question;
  market.endTime = event.params.endTime;
  market.resolved = false;
  market.totalVolume = BigInt.fromI32(0);
  market.totalTrades = BigInt.fromI32(0);
  market.createdAt = event.block.timestamp;
  market.save();
}

export function handleSharesPurchased(event: SharesPurchasedEvent): void {
  let id = event.transaction.hash.toHex() + "-" + event.logIndex.toString();
  let purchase = new SharesPurchased(id);

  purchase.market = event.params.marketId.toString();
  purchase.marketId = event.params.marketId;
  purchase.buyer = event.params.buyer;
  purchase.isOptionA = event.params.isOptionA;
  purchase.amount = event.params.amount;
  purchase.blockNumber = event.block.number;
  purchase.blockTimestamp = event.block.timestamp;
  purchase.transactionHash = event.transaction.hash;

  purchase.save();

  // Update market totals
  let market = Market.load(event.params.marketId.toString());
  if (market) {
    market.totalVolume = market.totalVolume.plus(event.params.amount);
    market.totalTrades = market.totalTrades.plus(BigInt.fromI32(1));
    market.save();
  }

  // Update daily stats
  updateDailyStats(event);
}

export function handleMarketResolved(event: MarketResolvedEvent): void {
  let market = Market.load(event.params.marketId.toString());
  if (market) {
    market.resolved = true;
    market.winner = event.params.winner;
    market.save();
  }
}

function updateDailyStats(event: SharesPurchasedEvent): void {
  let timestamp = event.block.timestamp;
  let dayID = timestamp.toI32() / 86400; // 86400 seconds in a day
  let dayStartTimestamp = dayID * 86400;
  let id = event.params.marketId.toString() + "-" + dayID.toString();

  let dailyStats = DailyMarketStats.load(id);
  if (!dailyStats) {
    dailyStats = new DailyMarketStats(id);
    dailyStats.market = event.params.marketId.toString();
    dailyStats.date = BigInt.fromI32(dayStartTimestamp);
    dailyStats.optionAVolume = BigInt.fromI32(0);
    dailyStats.optionBVolume = BigInt.fromI32(0);
    dailyStats.totalVolume = BigInt.fromI32(0);
    dailyStats.totalTrades = BigInt.fromI32(0);
    dailyStats.priceA = BigDecimal.fromString("0.5");
    dailyStats.priceB = BigDecimal.fromString("0.5");
  }

  if (event.params.isOptionA) {
    dailyStats.optionAVolume = dailyStats.optionAVolume.plus(
      event.params.amount
    );
  } else {
    dailyStats.optionBVolume = dailyStats.optionBVolume.plus(
      event.params.amount
    );
  }

  dailyStats.totalVolume = dailyStats.totalVolume.plus(event.params.amount);
  dailyStats.totalTrades = dailyStats.totalTrades.plus(BigInt.fromI32(1));

  // Calculate prices based on volume
  let totalVol = dailyStats.optionAVolume.plus(dailyStats.optionBVolume);
  if (totalVol.gt(BigInt.fromI32(0))) {
    dailyStats.priceA = dailyStats.optionAVolume
      .toBigDecimal()
      .div(totalVol.toBigDecimal());
    dailyStats.priceB = dailyStats.optionBVolume
      .toBigDecimal()
      .div(totalVol.toBigDecimal());
  }

  dailyStats.save();
}
```

### 5. Deploy to The Graph Studio

```bash
# Build the subgraph
graph build

# Deploy to The Graph Studio
graph deploy --studio your-subgraph-name
```

### 6. Update Environment Variables

Add your subgraph URL to `.env.local`:

```bash
NEXT_PUBLIC_SUBGRAPH_URL=https://api.studio.thegraph.com/query/your-subgraph-name/your-subgraph-name/version/latest
```

## Benefits

After setting up the subgraph, your app will:

- ✅ **Load faster** - No more waiting for RPC pagination
- ✅ **Be more reliable** - No more rate limiting issues
- ✅ **Scale better** - Handle more users without RPC costs
- ✅ **Provide richer data** - Pre-computed analytics and aggregations
- ✅ **Stay in sync** - Real-time updates as events occur

## Next Steps

1. Set up your subgraph following the steps above
2. Update your `NEXT_PUBLIC_SUBGRAPH_URL` in `.env.local`
3. The app will automatically use the subgraph instead of RPC calls
4. Monitor your subgraph's sync status in The Graph Studio

The subgraph approach is the industry standard for DeFi applications and will provide a much better user experience compared to direct RPC calls.
