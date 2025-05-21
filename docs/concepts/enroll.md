# Enrollment System

## Overview

The ReSquared enrollment system enables validators to permissionlessly register with Autonomous Verifiable Services (AVSs) across different blockchains. This system defines how validators can enroll in AVSs, how token requirements work, and how the cross-chain communication occurs to facilitate this enrollment.

## Permissionless Validator Enrollment

The core principle is open participation—any validator meeting specified token requirements can enroll without central authority approval. This approach maximizes decentralization while maintaining security through token staking requirements.

### Enrollment Process

When a validator wants to enroll with an AVS, they initiate the process by calling the `enroll` function:

```solidity
function enroll(
    bytes32 avsId,
    uint256 chainId,
    ValidatorRecord memory validator,
    bytes calldata sendOptions,
    bytes calldata returnOptions
) external payable {
    // Validation checks
    if (avsId == bytes32(0)) revert InvalidAVS();
    bytes32 validatorId = validatorAccountToId[msg.sender];
    bytes32 enrollId = keccak256(abi.encode(avsId, validatorId));
    if (validatorId == bytes32(0)) revert NoValidatorAccount();
    if (pendingAVSEnrollments[enrollId]) revert EnrollmentPending();
    if (validatorState[validatorId].avsList[validator.token].contains(avsId)) 
        revert AlreadyEnrolled();
    
    // Process enrollment based on chain
    if (chainId == block.chainid) {
        // Local enrollment
        // ...
    } else {
        // Cross-chain enrollment
        // ...
    }
}
```

The system handles two types of enrollments:

**Local Enrollment:**
When enrolling with an AVS on the same blockchain, the process executes immediately:

```solidity
// Inside the enroll function for local enrollment
IAVSRegistry(avsRegistry).enroll(avsId, validator);
if (!validatorState[validatorId].avsEnrolledTokens.contains(validator.token)) {
    validatorState[validatorId].avsEnrolledTokens.add(validator.token);
}
validatorState[validatorId].avsList[validator.token].add(avsId);
emit AVSEnrollmentConfirmed(validatorId, avsId);
```

**Cross-Chain Enrollment:**
For AVSs on different blockchains, a more interactive process is needed:

```solidity
// Inside the enroll function for cross-chain enrollment
bytes memory payload = abi.encode(
    EnrollMessage({
        avsId: avsId,
        validator: validator,
        returnOptions: returnOptions
    })
);
// Mark enrollment as pending
pendingAVSEnrollments[enrollId] = true;
IMessageRouter(messageRouter).sendMessage{value: msg.value}(
    uint32(chainId),
    uint8(MessageType.ENROLL),
    payload,
    sendOptions
);
emit AVSEnrollmentInitiated(validatorId, avsId, chainId);
```

The destination chain then handles the incoming enrollment message:

```solidity
function handleIncomingEnrollment(
    bytes32 avsId,
    ValidatorRecord calldata validator,
    bytes calldata returnOptions
) external payable {
    // Only message router can call this function
    require(msg.sender == messageRouter, "Only message router");
    
    // Validation and execution
    if (avsAccounts[avsId] == address(0)) {
        _sendEnrollmentFailure(
            validator.chainId,
            avsId,
            validator.id,
            "InvalidAVS",
            returnOptions
        );
        return;
    }
    
    if (!validateEnrollment(avsId, validator)) {
        _sendEnrollmentFailure(
            validator.chainId,
            avsId,
            validator.id,
            "ValidatorRequirementsNotMet",
            returnOptions
        );
        return;
    }
    
    // Execute enrollment and send confirmation
    _executeEnrollment(avsId, validator);
    _sendEnrollmentConfirmation(
        validator.chainId,
        avsId,
        validator.id,
        validator.token,
        returnOptions
    );
}
```

After successful cross-chain enrollment, a confirmation message is sent back to update the source chain's state:

```solidity
function handleEnrollConfirmed(
    bytes32 avsId,
    bytes32 validatorId,
    address token
) external {
    require(msg.sender == messageRouter, "Only message router");
    bytes32 enrollId = keccak256(abi.encode(avsId, validatorId));
    require(pendingAVSEnrollments[enrollId], "No pending enrollment");
    
    // Update validator state
    if (!validatorState[validatorId].avsEnrolledTokens.contains(token)) {
        validatorState[validatorId].avsEnrolledTokens.add(token);
    }
    validatorState[validatorId].avsList[token].add(avsId);
    delete pendingAVSEnrollments[enrollId];
    emit AVSEnrollmentConfirmed(validatorId, avsId);
}
```

## Token Requirements System

This system allows AVSs to define economic security parameters for validator participation. Each AVS can create custom token requirements to enforce minimum stake amounts.

### Token Requirement Definition

Each AVS can specify a token address (with address(0) representing native ETH), the chain ID where the token exists, and a minimum amount threshold required for staking. These requirements are stored with unique identifiers based on token and chain ID combinations.

```solidity
function addTokenRequirement(
    address token,
    uint256 chainId,
    uint256 minAmount
) external onlyAdmin {
    // Allow address(0) for native ETH
    if (minAmount == 0) revert InvalidAmount();
    bytes32 key = _getTokenRequirementKey(token, chainId);
    if (_tokenRequirementKeys.contains(key))
        revert TokenRequirementAlreadyExists();
    
    // Store the requirement
    _tokenRequirements[key] = TokenRequirement({
        token: token,
        chainId: chainId,
        minAmount: minAmount
    });
    _tokenRequirementKeys.add(key);
    emit TokenRequirementAdded(token, chainId, minAmount);
}
```

### Validator Eligibility

During enrollment, the system evaluates whether a validator meets the token requirements through a clear eligibility check process:

```solidity
function checkValidatorEligibility(
    address validatorToken,
    uint256 validatorChainId,
    uint256 amount
) external view returns (bool) {
    // If no requirements exist, any validator is eligible
    if (_tokenRequirementKeys.length() == 0) {
        return true;
    }
    
    // Check if the validator meets the specific requirement for their token
    bytes32 key = _getTokenRequirementKey(validatorToken, validatorChainId);
    if (_tokenRequirementKeys.contains(key)) {
        return amount >= _tokenRequirements[key].minAmount;
    }
    
    // If no matching requirement exists for this token/chainId, validator is not eligible
    return false;
}
```

This function follows a simple but effective logic:
1. If the AVS has no token requirements, any validator is automatically eligible
2. If requirements exist, it checks if the validator's token/chain combination matches any requirement
3. If a match is found, it compares the validator's staked amount against the minimum required
4. If there's no matching requirement for the validator's token/chain, they cannot enroll with that token

## AVS Account System

Each AVS is represented by a dedicated account contract deployed through a Create2 factory pattern, ensuring deterministic addresses and gas efficiency.

### Account Creation and Management

AVS accounts are created through a factory process initiated by the AVS owner. They are deployed with a salt value for deterministic addressing, ensuring consistent identification across chains.

```solidity
function createAVS(
    Options memory options,
    string calldata description
) external verifyOptions(options) returns (address, bytes32) {
    bytes32 salt = _computeSalt(msg.sender);
    bytes memory deploymentData = _getAVSInitCode(msg.sender, description);
    address avsAccount = Create2.deploy(0, salt, deploymentData);
    bytes32 avsId = keccak256(
        abi.encode(salt, block.chainid, avsAccount, avsNonce[msg.sender])
    );
    
    // Register the AVS
    avsAccounts[avsId] = avsAccount;
    avsAccountToId[avsAccount] = avsId;
    avsNonce[msg.sender]++;
    avsOptions[avsId] = options;
    emit AVSCreated(avsId, avsAccount, msg.sender, description);
    return (avsAccount, avsId);
}
```

Each account is initialized with owner information and AVS description, then assigned a unique identifier derived from salt, chain ID, address, and nonce. The owner receives administrative privileges, allowing them to manage token requirements, handle slashing events, and update AVS metadata as needed.

### Validator Tracking

These accounts maintain comprehensive records of enrolled validators, storing unique validator identifiers, origin chain IDs, and token-specific enrollment records with stake amounts and slashing history. This structured data enables efficient validation checks during enrollment and slashing events.

```solidity
function _executeEnrollment(
    bytes32 avsId,
    ValidatorRecord calldata validator
) internal {
    // Add validator to AVS if not already present
    if (!avsValidators[avsId].contains(validator.id)) {
        avsValidators[avsId].add(validator.id);
        validators[validator.id].id = validator.id;
        validators[validator.id].chainId = validator.chainId;
    }
    
    // Ensure validator isn't already enrolled with this token
    require(
        validators[validator.id].enrollments[validator.token].token == address(0),
        "Validator already enrolled"
    );
    
    // Record enrollment details
    validators[validator.id].enrollments[validator.token] = ValidatorEnrollRecord({
        totalStake: validator.totalStake,
        totalSlashed: 0,
        token: validator.token
    });
    emit ValidatorEnrolled(avsId, validator.id, validator.chainId);
}
```

When validators enroll, their records are added to the AVS's validator set, and when they detach, these records are properly removed, maintaining an accurate representation of the active validator set at all times.

## Detachment Process

Validators can gracefully exit from AVSs while maintaining system security through a carefully designed detachment process with timelock protection.

### Timelock Mechanism

When a validator wishes to detach, they must first initiate a detachment request that triggers a timelock period before the actual detachment can be executed:

```solidity
function initiateDetach(
    bytes32 avsId,
    uint256 chainId,
    address token,
    bytes calldata sendOptions,
    bytes calldata returnOptions
) external payable {
    // Validation checks
    if (avsId == bytes32(0)) revert InvalidAVS();
    bytes32 validatorId = validatorAccountToId[msg.sender];
    if (validatorId == bytes32(0)) revert NoValidatorAccount();
    if (!validatorState[validatorId].avsList[token].contains(avsId))
        revert NotEnrolled();
    
    bytes32 detachId = keccak256(abi.encodePacked(avsId, validatorId));
    
    if (chainId == block.chainid) {
        // Local detachment initiation
        uint256 unlockTime = IAVSRegistry(avsRegistry).initiateDetach(
            avsId,
            validatorId,
            token
        );
        // Store the unlock time for later verification
        pendingDetachUnlockTimes[detachId] = unlockTime;
    } else {
        // Cross-chain detachment initiation
        // ...
    }
}
```

This timelock period (typically 24 hours on mainnet) serves as a security buffer, preventing validators from immediately withdrawing their stake after committing a violation but before being slashed.

### Detachment Execution

After the timelock expires, the validator can execute the detachment, which finalizes the removal from the AVS:

```solidity
function executeDetach(
    bytes32 avsId,
    uint256 chainId,
    address token,
    bytes calldata sendOptions,
    bytes calldata returnOptions
) external payable {
    // Validation checks
    // ...
    
    bytes32 detachId = keccak256(abi.encodePacked(avsId, validatorId));
    uint256 unlockTime = pendingDetachUnlockTimes[detachId];
    if (unlockTime == 0) revert InvalidChainId();
    if (block.timestamp < unlockTime) revert InvalidChainId();
    
    if (chainId == block.chainid) {
        // Local detachment execution
        IAVSRegistry(avsRegistry).executeDetach(avsId, validatorId, token);
        validatorState[validatorId].avsList[token].remove(avsId);
        if (validatorState[validatorId].avsList[token].length() == 0) {
            validatorState[validatorId].avsEnrolledTokens.remove(token);
        }
        delete pendingDetachUnlockTimes[detachId];
    } else {
        // Cross-chain detachment execution
        // ...
    }
}
```

This execution removes the validator from the AVS's validator set, clears their enrollment record for the specific token, and updates their state to reflect the detachment. All relevant state variables are cleaned up to prevent inconsistencies.

## Edge Case Handling

The system robustly handles various scenarios to ensure consistency and reliability. For enrollment failures, cross-chain processes trigger return messages with specific reasons:

```solidity
function _sendEnrollmentFailure(
    uint256 chainId,
    bytes32 avsId,
    bytes32 validatorId,
    string memory reason,
    bytes calldata returnOptions
) internal {
    bytes memory payload = abi.encode(
        EnrollFailureMessage({
            avsId: avsId,
            validatorId: validatorId,
            reason: reason
        })
    );
    
    IMessageRouter(messageRouter).sendMessage(
        uint32(chainId),
        uint8(MessageType.ENROLL_FAILURE),
        payload,
        returnOptions
    );
}
```

Similarly, detachment failures return error messages to the source chain with the system cleaning up pending detachment states. 

Chain reorganizations are managed through state changes that use block numbers or timestamps rather than transaction hashes:

```solidity
// Example of using timestamps for timelock instead of transaction hashes
uint256 unlockTime = block.timestamp + timelockDuration;
pendingDetachUnlockTimes[detachId] = unlockTime;
```

For transaction failures due to gas issues or other temporary problems, the system maintains consistent states allowing for retry attempts, with pending operations remaining in that state until explicitly processed or cleared.

## Key Considerations

The ReSquared enrollment system features a truly permissionless design where any validator can attempt enrollment, with success depending solely on meeting token requirements. Its requirement flexibility allows AVSs to accept various tokens across different chains while maintaining security thresholds appropriate for their needs.

```solidity
// Example of flexible token requirements
// Native ETH on chain 1
addTokenRequirement(address(0), 1, 32 ether);
// USDC on chain 10
addTokenRequirement(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48, 10, 50000 * 10**6);
// Custom token on chain 42161
addTokenRequirement(0x1234567890123456789012345678901234567890, 42161, 1000 * 10**18);
```

The system provides seamless cross-chain compatibility through the ReSquared message routing system, enabling validators to participate in AVSs regardless of blockchain boundaries. Security measures are built into every aspect of the system, including detachment timelocks, comprehensive validation checks, and pending transaction tracking.

This architecture achieves an optimal balance between openness and security, creating a dynamic multi-chain validator ecosystem that can adapt to the evolving needs of decentralized networks.