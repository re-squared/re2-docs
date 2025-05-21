# Slashing System

## Overview

The ReSquared slashing system enables Autonomous Verifiable Services (AVSs) to penalize validators for malicious behavior or broken commitments. This system allows AVSs to slash a portion of a validator's staked tokens across different blockchains, providing a secure and efficient way to enforce protocol rules and maintain system integrity.

## Slashing Process Flow

1. **Slashing Initiation**: An AVS identifies misbehavior and calls the `slash` function
2. **AVS Registry Processing**: Validates the request and calculates slashing amount
3. **Slashing Manager Execution**: Determines if slashing should be local or cross-chain
4. **Validator Account Response**: Implements the actual stake reduction

```solidity
// AVS initiates slashing
function slash(
    bytes32 validatorId,
    address token,
    SlashType slashType,  // SOFT or HARD
    string calldata reason,
    bytes calldata sendOptions
) external payable {
    if (validatorId == bytes32(0)) revert InvalidValidatorId();
    // Call AVS Registry to handle slashing
    IAVSRegistry(avsRegistry).slash(validatorId, token, slashType, reason, sendOptions);
    emit SlashInitiated(validatorId, slashType);
}
```

## Slashing Parameters Using Basis Points (BIPS)

The system uses basis points (BIPS) for precise penalty calculation:
- 1 BIP = 0.01%
- 10000 BIPS = 100%
- Each AVS defines `minSlash` (soft slashing) and `maxSlash` (hard slashing) in BIPS

```solidity
// AVS Registry determines slashing amount based on type
uint256 slashedBy = slashType == SlashType.SOFT ? options.minSlash : options.maxSlash;
// Calculate actual amount to slash based on percentage
uint256 amount = (validator.totalStake * slashedBy) / 100;
```

## Local Slashing Execution

For slashing on the same blockchain:

```solidity
function _executeLocalSlashing(
    bytes32 validatorId,
    bytes32 slashId,
    bytes32 avsId,
    address token,
    uint256 amount,
    uint256 slashedByBips,
    uint256 timestamp
) internal {
    // Create unique slash ID to prevent double-slashing
    if (slashRecords[slashId].validatorId != bytes32(0)) revert SlashAlreadyExecuted();

    // Verify validator is enrolled with this AVS
    ValidatorState storage vState = validatorState[validatorId];
    require(vState.avsList[token].contains(avsId), "AVS not enrolled for this token");

    // Record slash details and update validator's records
    slashRecords[slashId] = SlashRecord({
        amount: amount, slashedByBips: slashedByBips, timestamp: timestamp,
        avsId: avsId, validatorId: validatorId, token: token
    });
    validatorState[validatorId].slashRecords.add(slashId);
    validatorState[validatorId].totalSlashedByAVS[avsId] += amount;

    // Execute the actual slashing on validator's account
    IValidatorAccount(vState.validatorAccount).slash(token, amount);
    emit SlashingExecuted(validatorId, slashId, token, amount, slashedByBips);
}
```

## Cross-Chain Slashing Mechanism

For slashing across different blockchains:

```solidity
function _executeCrossChainSlashing(
    uint256 chainID,
    bytes32 avsId,
    bytes32 validatorId,
    bytes32 slashId,
    address token,
    uint256 amount,
    uint256 slashedBy,
    bytes memory sendOptions
) internal {
    // Create message with all necessary slashing information
    bytes memory payload = abi.encode(
        SlashMessage(token, amount, slashedBy, block.timestamp, 
                    slashId, avsId, validatorId)
    );

    // Send message to destination chain via message router
    IMessageRouter(messageRouter).sendMessage{value: msg.value}(
        uint32(chainID), uint8(MessageType.SLASH), payload, sendOptions
    );

    emit CrossChainSlashInitiated(
        validatorId, chainID, slashId, amount, token, slashedBy
    );
}
```

## Token-Specific Slashing Behavior

The system handles different token types with specialized logic:

### ETH Slashing (Multiple Forms)

```solidity
// Inside ValidatorAccount.slash for ETH
if (token == address(0)) {
    // Calculate available directly staked ETH
    uint256 directEthAvailable = tokenStakes[address(0)].amount -
        tokenStakes[address(0)].slashedAmount;

    if (slashAmount <= directEthAvailable) {
        // If enough directly staked ETH, slash only that
        tokenStakes[address(0)].slashedAmount += slashAmount;
    } else {
        // Mark all directly staked ETH as slashed
        tokenStakes[address(0)].slashedAmount = tokenStakes[address(0)].amount;
        tokenStakes[address(0)].active = false;

        // Slash remaining amount from beacon ETH
        uint256 remainingSlash = slashAmount - directEthAvailable;
        beaconEthSlashed += remainingSlash;
        
        // Deactivate beacon validator if fully slashed
        if (beaconEthSlashed >= beaconEthStaked * GWEI_FACTOR) {
            isBeaconValidator = false;
        }
    }
}
```

### ERC20 Token Slashing

```solidity
// For ERC20 tokens
else {
    // Calculate available unslashed amount
    uint256 availableAmount = tokenStakes[token].amount - tokenStakes[token].slashedAmount;
    require(slashAmount <= availableAmount, "Insufficient stake to slash");

    // Increase slashed amount counter
    tokenStakes[token].slashedAmount += slashAmount;

    // Deactivate token if completely slashed
    if (tokenStakes[token].slashedAmount >= tokenStakes[token].amount) {
        tokenStakes[token].active = false;
    }
}
```

## Security Features and Protections

1. **Unique Slashing IDs**: Generated from validator ID, AVS ID, chain ID, and timestamp to prevent double-slashing
2. **Authorization Controls**: Only registered AVS accounts can initiate slashing
3. **Validation Checks**: Multiple checks ensure validators are enrolled with the AVS and have sufficient stake
4. **Transparent Audit Trail**: Events emitted at each stage provide accountability
5. **Persistent Slashing Records**: All slashing actions are permanently recorded on-chain

## Key Benefits

1. **Cross-Chain Enforcement**: Ensures consistent rule enforcement across multiple blockchains
2. **Proportional Penalties**: Slashing amounts calibrated to violation severity
3. **Token-Aware Design**: Specialized handling for different asset types
4. **Economic Security**: Creates tangible deterrent against malicious behavior
5. **Transparent Accountability**: All slashing actions documented with clear justifications

The ReSquared slashing system balances effective enforcement with fair treatment of validators, serving as a cornerstone of trust in the decentralized network ecosystem.