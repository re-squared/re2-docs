# Rewards System

## Overview

The ReSquared rewards system provides a cross-chain mechanism for distributing rewards to validators and their delegators. Rewards are created by Autonomous Verifiable Services (AVSs), claimed by validators, and then made available to delegators through a Merkle proof verification system.

## Reward Creation by AVSs

When an AVS wants to reward validators for their service, they initiate a formalized reward submission process. The AVS, through its managed account, submits a structured package of payments to the protocol. Each payment explicitly identifies which validator is being rewarded, which token is being used for payment, and the precise amount being awarded.

This process involves several critical security checks. First, the system verifies that the entity submitting rewards is indeed an authorized AVS account. Only registered AVS accounts can create rewards, preventing unauthorized distributions. Second, the system confirms that each validator receiving rewards is actually enrolled with the AVS. This prevents rewards from being allocated to validators who are not participating in the service.

Once these validations are passed, the rewards are not immediately distributed but rather recorded in the validator's reward balance for each specific token. This staged approach allows for proper accounting and later distribution between validators and their delegators. The entire transaction is recorded through an event emission, creating a transparent audit trail of reward submissions.

### How AVSs Create Rewards

1. **Reward Submission**:
   - AVSs submit rewards by calling `submitReward` on the AVS Registry
   - Each submission contains arrays of payment structures with:
     - Validator IDs that earned rewards
     - Token addresses used for reward payments
     - Reward amounts

```solidity
function submitReward(Payment[] memory payments) external onlyAdmin {
    // Only AVS accounts can call submitReward
    bytes32 avsId = avsAccountToId[msg.sender];
    if (avsId == bytes32(0)) revert UnauthorizedAccess();
    
    for (uint256 i = 0; i < payments.length; i++) {
        Payment memory payment = payments[i];
        if (!avsValidators[avsId].contains(payment.validatorId))
            revert NotEnrolled();
        
        // Add rewards to validator's balance
        validators[payment.validatorId].rewards[payment.token] += payment.amount;
    }
    
    emit RewardsSubmitted(avsId, payments);
}
```

2. **Reward Distribution Creation**:
   - The system generates a unique distribution ID
   - A rewards distribution record is created with:
     - Total reward amount
     - Token address
     - Timestamp
     - Initially empty Merkle root

```solidity
function createRewardsDistribution(
    bytes32 validatorId,
    address token,
    uint256 totalAmount
) internal returns (bytes32 distributionId) {
    distributionId = keccak256(
        abi.encodePacked(validatorId, token, totalAmount, block.timestamp)
    );
    
    rewardsDistributions[distributionId] = RewardsDistribution({
        merkleRoot: bytes32(0),
        totalAmount: totalAmount,
        timestamp: block.timestamp,
        token: token
    });
    
    emit RewardsDistributionCreated(
        distributionId,
        validatorId,
        token,
        totalAmount,
        block.timestamp
    );
    
    return distributionId;
}
```

## Validator Reward Claiming

Validators have a flexible system for claiming their accumulated rewards, which works both within a single blockchain and across multiple chains.

### Local Reward Claiming

When a validator wishes to claim rewards on the same blockchain where they were earned, the process is straightforward. The validator initiates a claim, specifying which tokens they want to collect rewards for, where to send those rewards, and what percentage should be set aside for their delegators.

The system verifies the validator's identity through their registered ID and processes each token claim individually. For each token, the validator's entire reward balance is cleared, and the specified amount is transferred to the designated recipient address. If the validator has specified a delegator share (for example, 90% of rewards), the system automatically calculates this split, transferring the validator's portion directly while setting aside the delegator's portion in a special distribution.

### Cross-Chain Reward Claiming

The ReSquared system truly shines in its support for cross-chain reward claiming. When validators have earned rewards on one blockchain but wish to claim them from another, the system employs a sophisticated message routing mechanism.

The validator initiates the claim from their current chain, providing the same information as in a local claim plus additional parameters for cross-chain messaging. The system creates a formatted message containing the validator's ID, recipient address, delegator share percentage, and tokens to claim. This message is then sent through the Re2Gateway to the destination chain where the rewards exist.

On the destination chain, the message is received and processed similarly to a local claim. The rewards are divided according to the specified delegator share, with the validator's portion being transferred directly and the delegator's portion being set aside. A confirmation is sent back to the originating chain, completing the cross-chain claiming process.

This cross-chain capability allows validators to manage their rewards across multiple blockchains without having to maintain active nodes or wallets on each chain, significantly simplifying their operational requirements.

## Validator Reward Claiming

### Cross-Chain Reward Claiming Process

1. **Initiate Claim**:
   - Validators call `claimReward` through the StakeManager's RewardsManager
   - They specify:
     - Chain ID where rewards exist
     - Recipient address
     - Token addresses to claim
     - Delegator share percentage
     - Send options for cross-chain messaging

```solidity
function claimReward(
    uint256 chainId,
    address recipient,
    address[] calldata tokens,
    uint256 delegatorShare,
    bytes calldata sendOptions
) external payable {
    // Get validator ID from the sender's validator account
    bytes32 validatorId = validatorAccountToId[msg.sender];
    if (validatorId == bytes32(0)) revert NoValidatorAccount();
    if (chainId == 0) revert InvalidChainId();

    if (chainId == block.chainid) {
        // Local reward claim
        _handleLocalRewardClaim(validatorId, recipient, delegatorShare, tokens);
    } else {
        // Cross-chain reward claim
        _handleCrossChainRewardClaim(validatorId, recipient, tokens, chainId, delegatorShare, sendOptions);
    }
}
```

2. **Cross-Chain Message Processing**:
   - For cross-chain claims, a message is sent via Re2Gateway
   - The message includes:
     - Validator ID
     - Recipient address
     - Delegator share percentage
     - Tokens to claim

```solidity
function _handleCrossChainRewardClaim(
    bytes32 validatorId,
    address recipient,
    address[] calldata tokens,
    uint256 chainId,
    uint256 delegatorShare,
    bytes calldata sendOptions
) internal {
    // Prepare the cross-chain message payload
    bytes memory payload = abi.encode(
        ClaimRewardMessage({
            validatorId: validatorId,
            recipient: recipient,
            delegatorShare: delegatorShare,
            tokens: tokens
        })
    );

    // Send the message to the destination chain
    IMessageRouter(messageRouter).sendMessage{value: msg.value}(
        uint32(chainId),
        uint8(MessageType.CLAIM_REWARD),
        payload,
        sendOptions
    );

    emit RewardClaimInitiated(validatorId, recipient, chainId);
}
```

3. **Reward Processing on Destination Chain**:
   - The RewardsPool (AVS Registry) processes the claim
   - For each token:
     - Validator's reward balance is cleared
     - If delegator share is specified:
       - Delegator portion is calculated
       - Distribution is created for delegators to claim
       - Validator portion is transferred to recipient
     - If no delegator share, full amount goes to validator

```solidity
function claimReward(
    bytes32 validatorId,
    address recipient,
    uint256 delegatorShare,
    address[] calldata tokens
) external onlyManagerOrRouter {
    bytes32 distributionId;
    for (uint256 i = 0; i < tokens.length; i++) {
        address token = tokens[i];
        uint256 amount = validators[validatorId].rewards[token];
        validators[validatorId].rewards[token] = 0;

        if (delegatorShare > 0 && amount > 0) {
            // Calculate delegator portion
            uint256 delegatorPortion = (amount * delegatorShare) / 10000;
            uint256 validatorPortion = amount - delegatorPortion;

            distributionId = createRewardsDistribution(
                validatorId,
                token,
                delegatorPortion
            );
            // Only transfer the validator's portion to the recipient
            IERC20(token).safeTransfer(recipient, validatorPortion);
        } else {
            // No delegator share, transfer the full amount
            IERC20(token).safeTransfer(recipient, amount);
        }
    }
    emit RewardClaimed(distributionId, validatorId, recipient);
}
```


## Delegator Reward Distribution Mechanism

The delegator reward distribution employs a Merkle tree-based verification system that is both gas-efficient and cryptographically secure.

### Merkle Tree Generation

After a validator claims rewards and sets aside a portion for delegators, an off-chain process begins to organize the distribution. This process calculates each delegator's share based on their proportion of delegation to the validator during the relevant period. The calculation takes into account:

- The total amount delegated by each delegator
- The duration of delegation during the reward period
- The relative weight of different strategy types (if applicable)

The system then constructs a Merkle tree where each leaf represents a delegator's claim, containing their address and share percentage. The root of this tree – a single 32-byte hash that cryptographically represents all delegator claims – is published to the blockchain by an authorized entity.

### Claim Process for Delegators

Once the Merkle root is published on-chain, delegators can begin claiming their rewards. Each delegator submits a claim transaction that includes:

- Their delegator record (address and share percentage)
- The index of their entry in the Merkle tree
- The ID of the distribution they're claiming from
- A Merkle proof verifying their inclusion in the distribution

The system first verifies that the distribution exists and has a valid Merkle root. It then checks that the delegator hasn't already claimed from this distribution, preventing double-claiming. The claim amount is calculated based on the delegator's share percentage and the total distribution amount.

The critical security step comes next: the system verifies the Merkle proof against the stored root. This mathematical verification proves that the delegator's record was indeed included in the original distribution calculation without requiring the entire distribution list to be stored on-chain. If the proof is valid, the system marks the claim as processed and transfers the appropriate amount of tokens directly to the delegator's address.

This Merkle-based approach allows for efficient and secure distribution of rewards to potentially thousands of delegators while minimizing on-chain storage and computation costs.

## Comprehensive Reward Flow Examples

### Example 1: Simple Reward Flow

Let's track a reward through the entire system:

1. An AVS submits a reward of 1000 USDC for a validator who has successfully processed transactions for a week.
2. The validator claims this reward, specifying a 90% delegator share and their own wallet as the recipient.
3. The system transfers 100 USDC (10%) directly to the validator's wallet.
4. The remaining 900 USDC (90%) is set aside in a distribution with a unique ID.
5. An off-chain process calculates that the validator has three delegators with the following shares:
   - Delegator A: 50% of delegation (450 USDC)
   - Delegator B: 30% of delegation (270 USDC)
   - Delegator C: 20% of delegation (180 USDC)
6. A Merkle tree is created with these allocations, and the root is published on-chain.
7. Each delegator submits a claim with their Merkle proof:
   - Delegator A receives 450 USDC
   - Delegator B receives 270 USDC
   - Delegator C receives 180 USDC

### Example 2: Cross-Chain Reward with Multiple Tokens

1. An AVS operating on Chain A submits multiple rewards for a validator:
   - 500 USDC for fee processing
   - 20 ETH for security services
   - 1000 AVS_TOKEN for participation
2. The validator, operating primarily on Chain B, initiates a cross-chain claim.
3. A message is sent from Chain B to Chain A requesting the rewards.
4. On Chain A, the system processes each token separately:
   - For each token, 10% goes to the validator and 90% to delegators
   - Three distribution IDs are created, one for each token
5. The validator's portions (50 USDC, 2 ETH, and 100 AVS_TOKEN) are transferred to their address.
6. Three separate Merkle trees are created off-chain, one for each token distribution.
7. Delegators can claim their share of each token distribution independently by providing the appropriate proofs.

### How Delegators Claim Their Share

1. **Merkle Root Setting**:
   - After validator claims, an off-chain process calculates:
     - Each delegator's share based on their delegation percentage
     - A Merkle tree with leaves containing delegator data
   - The Merkle root is set on-chain by an authorized entity

```solidity
function setMerkleRoot(
    bytes32 distributionId,
    bytes32 merkleRoot
) external onlyRewardsManager {
    require(
        rewardsDistributions[distributionId].timestamp > 0,
        "Distribution does not exist"
    );
    require(
        rewardsDistributions[distributionId].merkleRoot == bytes32(0),
        "Merkle root already set"
    );

    // Set the Merkle root
    rewardsDistributions[distributionId].merkleRoot = merkleRoot;

    emit MerkleRootAdded(distributionId, merkleRoot, block.timestamp);
}
```

2. **Delegator Claim Process**:
   - Delegators call `claimDelegatorReward` with:
     - Their delegator record
     - Leaf index in the Merkle tree
     - Distribution ID
     - Merkle proof

```solidity
function claimDelegatorReward(
    DelegatorRecord calldata delegator,
    uint256 index,
    bytes32 distributionId,
    bytes32[] calldata merkleProof
) external {
    RewardsDistribution memory distribution = rewardsDistributions[distributionId];
    require(distribution.timestamp > 0, "Distribution does not exist");
    require(distribution.merkleRoot != bytes32(0), "Merkle root not set");

    // Check if already claimed
    uint256 claimedAmount = delegatorClaims[distributionId][delegator.delegatorAddress];
    require(claimedAmount == 0, "Already claimed");

    uint256 amount = (delegator.share * distribution.totalAmount) / 10000;

    // Verify the merkle proof
    bytes32 leaf = sha256(abi.encode(delegator));
    require(
        MerkleLib.verify(merkleProof, distribution.merkleRoot, leaf, index),
        "Invalid proof"
    );

    // Mark as claimed and send tokens
    delegatorClaims[distributionId][delegator.delegatorAddress] = amount;

    // Transfer tokens to delegator
    IERC20(distribution.token).safeTransfer(
        delegator.delegatorAddress,
        amount
    );

    emit DelegatorRewardClaimed(
        distributionId,
        msg.sender,
        amount,
        distribution.token
    );
}
```

3. **Merkle Proof Verification**:
   - The system verifies the provided proof against the stored Merkle root
   - If valid, the delegator receives their portion of rewards
   - The system marks the rewards as claimed to prevent double-claiming




## Key Considerations

1. **Reward Distribution Transparency**:
   - All reward distributions are recorded on-chain with unique identifiers
   - Distribution details including total amounts and tokens are publicly visible

2. **Cross-Chain Efficiency**:
   - Validators can claim rewards from any chain where they've registered
   - The ReSquared message routing system handles cross-chain communication

3. **Fair Delegator Distribution**:
   - Delegator rewards are proportional to their delegation amount
   - The Merkle proof system ensures secure and efficient distribution

4. **Security Measures**:
   - Only authorized AVS accounts can submit rewards
   - Distribution IDs are cryptographically generated to prevent tampering
   - Merkle proofs ensure only legitimate delegators can claim rewards
   - Double-claim prevention through tracking of claimed amounts
