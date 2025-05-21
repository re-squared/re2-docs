# Setup

First, pull the protocol repo:

```bash
git clone https://github.com/re-squared/resquared-contracts
```

and install the dependencies:

```bash
yarn install
```

#### Local Network Setup Guide

#### Initial Setup

Start the local network:

```bash
# Start local network
yarn hardhat node --network hardhat
```

#### Deployment

Deploy the full protocol (includes mock contracts):

```bash
# Deploy full protocol
yarn hardhat deploy --tags local --network hardhat
```

This:

1. Deploys MockBeaconRoots at EIP-4788 address
2. Sets up two virtual chains (A & B) with:
   * Cross-chain gateways
   * Message routers
   * AVS registries
   * Stake managers
3. Configures cross-chain messaging
4. Deploys test tokens

#### Testing

```bash
# Run all tests
yarn hardhat test

# Run specific test
yarn hardhat test test/hardhat/ValidatorAccount.ts
```

#### Mock Contracts & Tools

**MockBeaconRoots**

Simulates EIP-4788 beacon roots contract for local testing:

```typescript
// Get contract instance
const beaconRoots = await ethers.getContractAt("MockBeaconRoots", BEACON_ROOTS);

// Set block root
const timestamp = BigInt(Math.floor(Date.now() / 1000));
const mockRoot = ethers.keccak256(ethers.toUtf8Bytes("mock_block_root"));
await beaconRoots.setBlockRoot(timestamp, mockRoot);
```

#### Working with Validator Accounts

1. Deploy validator account:

```typescript
await stakeManager.connect(validator).deployValidatorAccount();
```

2. Setup validator data:

```typescript
const validatorData = {
    pubkey: "0x932530a536ce5536651d7739b1fccfed76bf0308f92ca28826b03e90a3da871ba7b84863239bafb8456c0c4337b5bd0c",
    withdrawalCredentials: ethers.zeroPadValue(validator.address, 32),
    effectiveBalance: 32000000000n,
    slashed: false,
    activationEligibilityEpoch: 339233n,
    activationEpoch: 339251n,
    exitEpoch: BigInt(2n ** 64n - 1n),
    withdrawableEpoch: BigInt(2n ** 64n - 1n)
};
```

3. Verify withdrawal credentials:

```typescript
await validatorAccount.verifyWithdrawalAddress(
    validatorData,
    mockProofs,
    validatorIndex,
    timestamp
);
```

#### AVS Operations

1. Create AVS:

```typescript
const options = {
    minSlash: 5,
    maxSlash: 100,
    minReward: 0,
    maxReward: 100,
    lockPeriod: 7 * 24 * 3600
};

const tx = await avsRegistry.connect(avsOwner).createAVS(options);
```

2. Enroll validator in AVS:

```typescript
// Local enrollment
await validatorAccount.connect(validator).enroll(
    avsId,
    localChainId,
    "0x",
    "0x"
);

// Cross-chain enrollment
const sendOptions = Options.newOptions()
    .addExecutorLzReceiveOption(2000000, ethers.parseEther("0.3"))
    .toHex()
    .toString();

await validatorAccount.connect(validator).enroll(
    avsId,
    remoteChainId,
    sendOptions,
    returnOptions,
    { value: ethers.parseEther("1") }
);
```

#### Deployed Addresses

After local deployment, you'll get contract addresses:

```json
{
    "chainA": {
        "endpoint": "0x...",
        "messageRouter": "0x...",
        "avsRegistry": "0x...",
        "stakeManager": "0x..."
    },
    "chainB": {
        "endpoint": "0x...",
        "messageRouter": "0x...",
        "avsRegistry": "0x...",
        "stakeManager": "0x..."
    },
    "mockToken": "0x...",
    "beaconRoots": "0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02"
}
```
