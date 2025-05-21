# Delegation System

The ReSquared delegation system enables stakers to delegate their assets to validators, contributing to network security while earning rewards without running validator infrastructure themselves. This technical document details the delegation and undelegation mechanisms.

## Delegation Architecture

### Delegation Process

The ReSquared protocol implements a multi-token delegation system through a series of smart contracts that handle delegation state tracking, token custody, and validator management. Delegation occurs in three phases:

1. Validator deployment of specialized ValidatorAccount contracts that can receive delegated assets
2. Staker delegation to validator via `delegate()` function, transferring token custody
3. State synchronization across local and cross-chain contexts through the MessageRouter system

### Initiating a Delegation

```solidity
function delegate(bytes32 validatorId, address token, uint256 amount) external payable {
    bytes32 delegationId = keccak256(abi.encodePacked(validatorId, msg.sender, block.timestamp));
    require(delegations[delegationId].amount == 0, "Duplicate delegation");
    
    if (validatorState[validatorId].validatorAccount == address(0)) revert InvalidValidator();
    require(IValidatorAccount(validatorAccount).isTokenStaked(token), "InactiveValidator");

    _handleDelegation(validatorId, delegationId, token, amount);
}
```

The delegation system generates a unique delegationId by hashing the validator ID, sender address, and block timestamp. This creates an immutable reference for the delegation record and prevents duplicate delegations.

### Token Transfer and State Management

```solidity
function _handleDelegation(bytes32 validatorId, bytes32 delegationId, address token, uint256 amount) internal {
    require(delegatorState[msg.sender].currentValidator == bytes32(0) || 
            delegatorState[msg.sender].currentValidator == validatorId,
            "Cannot delegate to multiple validators");

    if (token == address(0)) {
        require(msg.value == amount, "ETH amount mismatch");
    } else {
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
    }

    _processDelegate(validatorId, msg.sender, delegationId, token, amount);
}
```

The system enforces a single-validator constraint for each delegator, preventing delegation splitting while still supporting multiple token types for the same validator. Token transfers use different mechanisms based on type - ETH via transaction value and ERC20 via safeTransferFrom.

### State Persistence Model

```solidity
function _processDelegate(bytes32 validatorId, address delegator, bytes32 delegationId, address token, uint256 amount) internal {
    if (!validatorState[validatorId].delegatedTokens.contains(token)) {
        validatorState[validatorId].delegatedTokens.add(token);
    }
    validatorState[validatorId].delegators[token].add(delegator);

    delegations[delegationId] = Delegation({
        token: token,
        amount: amount,
        delegatedAt: block.timestamp,
        isActive: true,
        isRefunded: false,
        refundInitiatedAt: 0
    });

    if (!delegatorState[delegator].delegatedTokens.contains(token)) {
        delegatorState[delegator].delegatedTokens.add(token);
    }
    delegatorState[delegator].delegations[token].add(delegationId);
    
    if (delegatorState[delegator].currentValidator == bytes32(0)) {
        delegatorState[delegator].currentValidator = validatorId;
    }

    emit DelegationFinalized(validatorId, delegationId, delegator, amount);
}
```

The state persistence model uses a multi-mapping approach that:
1. Maps validators to their delegators (per token)
2. Maps delegators to their delegations (per token)
3. Tracks an explicit current validator for each delegator
4. Maintains delegation records with status flags for active/refund state

This enables efficient querying of delegation relationships while supporting the protocol's security model constraints.

### Architectural Constraints

The ReSquared protocol enforces these key architectural constraints:

1. **Single-Validator Relationship**: A staker can only delegate to one validator at a time, creating a strong alignment of interests.
2. **Chain Specificity**: Delegations are bound to delete to validators on the same chain.
3. **Multi-Token Support**: Delegations are tracked per token, allowing validators to participate in diverse security models but delegators to only delegate in tokens their validator staked in.
4. **No Partial Undelegation**: A delegator must undelegate their entire position for a given token.
5. **Time-Locked Undelegation**: Undelegation requests are subject to a time-lock period to prevent abuse.
   

## Undelegation Mechanism

### Two-Phase Undelegation

ReSquared implements a two-phase time-locked undelegation mechanism:

1. **Initiation Phase**: Delegator requests undelegation, which calculates slashable amount and starts timelock
2. **Execution Phase**: After timelock expires, delegator executes actual token transfer

This approach prevents immediate withdrawals while providing economic security through proper slash accounting.

### Initiating Undelegation

```solidity
function initiateRefund(bytes32 delegationId) external {
    Delegation storage delegation = delegations[delegationId];
    if (delegation.isRefunded) revert RefundAlreadyProcessed();
    if (delegation.refundInitiatedAt > 0) revert RefundAlreadyInitiated();
    
    if (!delegatorState[msg.sender].delegations[delegation.token].contains(delegationId)) 
        revert NotDelegationOwner();

    bytes32 validatorId = delegatorState[msg.sender].currentValidator;
    uint256 effectiveAmount = calculateRefundableAmount(validatorId, delegationId);
    _queueRefund(msg.sender, delegation.token, effectiveAmount, delegationId);
}
```

The undelegation process begins with ownership verification and ensures that a delegation can only be refunded once. The system calculates the refundable amount at initiation time, accounting for any slashing events.

### Slash-Adjusted Refund Calculation

```solidity
function calculateRefundableAmount(bytes32 validatorId, bytes32 delegationId) internal view returns (uint256) {
    Delegation storage delegation = delegations[delegationId];
    uint256 effectiveAmount = delegation.amount;
    
    EnumerableSet.Bytes32Set storage slashSet = validatorState[validatorId].slashRecords;
    for (uint256 i = 0; i < slashSet.length(); i++) {
        SlashRecord storage slash = slashRecords[slashSet.at(i)];
        
        if (slash.token == delegation.token && 
            slash.timestamp > delegation.delegatedAt &&
            (delegation.refundInitiatedAt == 0 || slash.timestamp < delegation.refundInitiatedAt)) {
            
            uint256 slashedAmount = (effectiveAmount * slash.slashedByBips) / 10000;
            effectiveAmount -= slashedAmount;
        }
    }
    return effectiveAmount;
}
```

The refund calculation implements a proportional slashing model that:
1. Only applies slashes for the specific token being undelegated
2. Only considers slashes that occurred during the delegation lifetime
3. Applies each slash proportionally using basis points (1/100 of a percent)

### Refund Queueing

```solidity
function _queueRefund(address delegator, address token, uint256 amount, bytes32 delegationId) internal {
    Delegation storage delegation = delegations[delegationId];
    delegation.refundInitiatedAt = block.timestamp;
    delegation.isActive = false;
    emit RefundInitiated(delegationId, delegator, token, amount, block.timestamp);
}
```

The queueing process marks the delegation as inactive and records the initiation timestamp, which is used for enforcing the time-lock period.

### Completing Undelegation

```solidity
function refund(address delegator, bytes32 delegationId) external {
    Delegation storage delegation = delegations[delegationId];
    require(delegation.refundInitiatedAt > 0, "Invalid delegation");
    if (delegation.isRefunded) revert RefundAlreadyProcessed();
    if (block.timestamp < delegation.refundInitiatedAt + REFUND_DELAY)
        revert RefundDelayNotMet();

    delegation.isRefunded = true;
    uint256 effectiveAmount = delegation.amount;

    if (delegation.token == address(0)) {
        payable(delegator).sendValue(effectiveAmount);
    } else {
        IERC20(delegation.token).safeTransfer(delegator, effectiveAmount);
    }

    emit RefundProcessed(delegationId, delegator, delegation.token, effectiveAmount);
}
```

The refund completion has three validation checks:
1. Verify the refund was properly initiated
2. Ensure the refund hasn't already been processed
3. Confirm the time-lock period (REFUND_DELAY) has elapsed

The system then transfers the tokens directly to the delegator using the appropriate transfer mechanism for the token type.

### Security Mechanisms

The undelegation system implements these security mechanisms:

1. **Time-Lock Enforcement**: Mandatory delay period (typically 24 hours) between initiation and execution
2. **Slashing Awareness**: Automatically applies validator slashes to undelegations
3. **State Validation**: Multi-stage process with state checks to prevent exploits
4. **Reentrancy Protection**: State changes before external calls to prevent reentrancy attacks

## Supported Token Types

The ReSquared delegation system supports three categories of tokens:

1. **Native ETH**: Identified by the zero address (`address(0)`). Requires sending ETH with the delegation transaction.

2. **ERC20 Tokens**: Any token implementing the ERC20 interface can be used for delegation.

3. **Beacon Chain ETH**: Validators can connect their ReSquared  validator account to beacon chain validator account by verifying withdrawal credentials.

The protocol's smart contracts handle each token type with specialized logic, enabling a unified delegation system across diverse asset types.