# Validator Setup

## Ethereum Validator Onboarding Documentation

### Overview

This document provides a comprehensive guide for onboarding an Ethereum validator to our platform. The platform enables unified validator management for both regular token staking and Beacon Chain validation, with additional features for participating in Actively Validated Services (AVS).

### System Architecture

```mermaid
flowchart TD
    subgraph Contracts
        VA[ValidatorAccount]
        SM[StakeManager]
        AVS[AVSRegistry]
        MR[MessageRouter]
        BB[BeaconRoots]
    end
    
    subgraph User Accounts
        Validator[Validator]
        Delegator[Delegator]
    end
    
    subgraph External
        BC[Beacon Chain]
        ET[ERC20 Tokens]
    end
    
    Validator -->|Deploys| VA
    VA -->|Interacts with| SM
    SM -->|Manages Registry| AVS
    VA -->|Verifies against| BB
    BB -->|Connects to| BC
    VA -->|Stakes| ET
    Delegator -->|Delegates to| VA
    MR -->|Communicates with| SM
    SM -->|Uses| MR
```

### Validator Onboarding Process

The process of onboarding a validator involves several steps, from account deployment to verification and staking.

```mermaid
sequenceDiagram
    actor Validator
    participant SM as StakeManager
    participant VA as ValidatorAccount
    participant BC as Beacon Chain
    participant AVS as AVSRegistry
    
    Validator->>SM: deployValidatorAccount()
    SM->>VA: Deploy and initialize
    Note over VA: Account created with validator as admin
    
    Validator->>VA: stakeToken(token, amount)
    Note over VA: Token can be ETH or ERC20
    
    alt If Beacon Chain Validator
        Validator->>BC: Register Validator (external)
        Validator->>VA: verifyWithdrawalAddress(validatorData, proof, ...)
        VA->>BC: Verify through BeaconRoots
        Note over VA: Registered as Beacon validator
    end
    
    opt AVS Registration
        Validator->>VA: enroll(avsId, chainId, token, ...)
        VA->>SM: Forward enrollment request
        SM->>AVS: Register validator
        AVS-->>VA: Confirm enrollment
    end
```

### Beacon Chain Verification

For validators operating on the Ethereum Beacon Chain, additional verification is required.

```mermaid
sequenceDiagram
    actor Validator
    participant VA as ValidatorAccount
    participant BR as BeaconRoots
    participant BC as Beacon Chain
    
    Note over Validator: Already registered on Beacon Chain with ValidatorAccount as withdrawal address
    
    Validator->>VA: verifyWithdrawalAddress(validatorData, proof, validatorIndex, timestamp)
    VA->>BR: verifyValidatorState(validatorData, proof, validatorIndex, timestamp)
    BR->>BC: Verify against Beacon Chain data
    BC-->>BR: Confirmation
    BR-->>VA: Verified
    VA->>VA: Set isBeaconValidator = true
    VA->>VA: Set beaconEthStaked = validator.effectiveBalance
    VA-->>Validator: WithdrawAddressVerified event
```

### Reconciliation Process

Beacon Chain validators need to periodically reconcile their on-chain balances.

```mermaid
sequenceDiagram
    actor Validator
    participant VA as ValidatorAccount
    participant BC as Beacon Chain
    
    Validator->>VA: startReconcile()
    Note over VA: Calculate pending balance
    VA->>VA: Lock account
    VA-->>Validator: ReconciliationStarted event
    
    Validator->>VA: verifyBalance(balanceRecord, proof, validatorIndex, timestamp)
    VA->>BC: Verify current balance
    BC-->>VA: Balance verified
    VA->>VA: Update beaconEthStaked
    VA->>VA: Adjust withdrawalETHBalance
    VA->>VA: Unlock account
    VA-->>Validator: BalanceVerified event
```

### Key Functions

#### ValidatorAccount Functions

| Function                         | Purpose                                         |
| -------------------------------- | ----------------------------------------------- |
| `initialize`                     | Sets up the validator account with proper roles |
| `stakeToken`                     | Stakes ETH or ERC20 tokens                      |
| `withdrawToken`                  | Withdraws available tokens or rewards           |
| `unstakeToken`                   | Unstakes tokens after checking AVS enrollment   |
| `verifyWithdrawalAddress`        | Verifies and activates Beacon Chain validation  |
| `startReconcile`/`verifyBalance` | Handles Beacon Chain balance reconciliation     |
| `enroll`                         | Enrolls the validator in an AVS                 |
| `initiateDetach`/`executeDetach` | Handles AVS detachment with time lock           |
| `claimReward`                    | Claims rewards from validation activities       |

### Onboarding Checklist

1. **Preparation**:
   * Set up validator infrastructure
   * Generate Ethereum keys
2. **Account Creation**:
   * Call `deployValidatorAccount()` on StakeManager
   * Verify account creation and role assignments
3. **Staking**:
   * Stake ETH or ERC20 tokens via `stakeToken()`
   * For Beacon Chain: complete registration on Beacon Chain
   * Verify withdrawal address with `verifyWithdrawalAddress()`
4. **AVS Enrollment (Optional)**:
   * Research available AVS services
   * Enroll using `enroll()` with appropriate parameters
   * Monitor rewards and performance
5. **Maintenance**:
   * Regularly reconcile Beacon Chain balances
   * Claim rewards as necessary
   * Monitor for slashing events

### Troubleshooting

| Issue                 | Resolution                                         |
| --------------------- | -------------------------------------------------- |
| Unstaking failed      | Check AVS enrollment status                        |
| Verification failed   | Ensure validator data and proofs are correct       |
| Cannot withdraw       | Check if tokens are staked or slashed              |
| Reconciliation issues | Verify that pending balance matches expected value |

### Glossary

* **AVS**: Actively Validated Services - services that validators can provide
* **Beacon Chain**: Ethereum's proof-of-stake consensus layer
* **Reconciliation**: Process of verifying and updating Beacon Chain balances

***
