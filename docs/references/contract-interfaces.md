# Contract Interfaces

### Enums & Structs

```solidity
enum SlashType {
</strong>    SOFT,
    HARD
}

// Define a struct to represent token requirements
struct TokenRequirement {
    address token;
    uint256 chainId;
    uint256 minAmount;
}

struct Options {
    uint256 minSlash; // in percentage of the total stake
    uint256 maxSlash; // in percentage of the total stake
    uint256 minReward;
    uint256 maxReward;
    uint256 lockPeriod;
}

struct ValidatorRecord {
    bytes32 id;
    uint256 chainId;
    uint256 totalStake;
    address token;
}

struct ValidatorEnrollRecord {
    uint256 totalStake;
    uint256 totalSlashed;
    address token;
}

struct ValidatorData {
    bytes32 id;
    uint256 chainId;
    mapping(address => ValidatorEnrollRecord) enrollments;
    mapping(address => uint256) rewards;
}

// As defined in phase0/beacon-chain.md:356
struct Validator {
    bytes pubkey;
    bytes32 withdrawalCredentials;
    uint64 effectiveBalance;
    bool slashed;
    uint64 activationEligibilityEpoch;
    uint64 activationEpoch;
    uint64 exitEpoch;
    uint64 withdrawableEpoch;
}

// As defined in phase0/beacon-chain.md:436
struct BeaconBlockHeader {
    uint64 slot;
    uint64 proposerIndex;
    bytes32 parentRoot;
    bytes32 stateRoot;
    bytes32 bodyRoot;
}

struct Reconciliation {
    uint256 initiatedTimestamp;
    uint256 pendingBalanceWei;
    bool verified;
}

enum MessageType {
    ENROLL,
    ENROLL_CONFIRMED,
    ENROLL_FAILED,
    DETACH_INITIATED,
    DETACH_INITIATED_CONFIRMED,
    DETACH_EXECUTE,
    DETACH_COMPLETED,
    SLASH,
    CLAIM_REWARD
}

struct Payment {
    bytes32 validatorId;
    uint256 amount;
    address token;
}
struct RewardsDistribution {
    bytes32 merkleRoot;
    uint256 totalAmount;
    uint256 timestamp;
    address token;
}
struct DelegatorRecord {
    address delegatorAddress;
    uint256 share;
    bytes32 delegatorId;
}
struct DelegateMessage {
    bytes32 validatorId;
    address delegator;
    address token;
    uint256 amount;
    bytes32 delegationId;
    bytes returnOptions;
}

struct UndelegateMessage {
    bytes32 validatorId;
    address delegator;
    bytes32 delegationId;
    bytes returnOptions;
}

struct UndelegateConfirmation {
    bytes32 validatorId;
    address delegator;
    bytes32 delegationId;
}

struct DelegationConfirmation {
    bytes32 validatorId;
    address delegator;
    bytes32 delegationId;
}

struct DelegationFailure {
    address delegator;
    bytes32 delegationId;
    string reason;
}

// AVS messages
struct EnrollMessage {
    bytes32 avsId;
    ValidatorRecord validator;
    bytes returnOptions;
}

struct EnrollConfirmation {
    bytes32 avsId;
    bytes32 validatorId;
    address token;
}

struct EnrollFailure {
    bytes32 avsId;
    bytes32 validatorId;
    string reason;
}

// Detach messages
struct DetachMessage {
    bytes32 avsId;
    bytes32 validatorId;
    bytes returnOptions;
}

// New detach message types for A->B->A pattern with time lock
struct DetachInitiateMessage {
    bytes32 avsId;
    bytes32 validatorId;
    address token;
    bytes returnOptions;
}

struct DetachInitiateConfirmation {
    bytes32 avsId;
    bytes32 validatorId;
    uint256 unlockTime;
}

struct DetachExecuteMessage {
    bytes32 avsId;
    bytes32 validatorId;
    address token;
    bytes returnOptions;
}

struct DetachCompletedMessage {
    bytes32 avsId;
    bytes32 validatorId;
    address token;
}

struct DetachFailureMessage {
    bytes32 avsId;
    bytes32 validatorId;
    string reason;
}

// Slashing messages
struct SlashMessage {
    address token;
    uint256 amount;
    uint256 slashedBy;
    uint256 timestamp;
    bytes32 slashId;
    bytes32 avsId;
    bytes32 validatorId;
}

struct ClaimRewardMessage {
    bytes32 validatorId;
    address recipient;
    uint256 delegatorShare;
    address[] tokens;
}
```

### IAVSAccount

```solidity
interface IAVSAccount {
    /**
     * @notice Sets the AVS address
     * @param addr The new AVS address
     */
    function setAVSAddress(address addr) external;

    /**
     * @notice Updates the AVS description
     * @param _description The new description
     */
    function setDescription(string calldata _description) external;

    /**
     * @notice Initiates a slash operation via AVS Registry
     * @param validatorId The ID of the validator to slash
     * @param slash type of slash soft or hard
     * @param reason The reason for slashing
     */
    function slash(
        bytes32 validatorId,
        address token,
        SlashType slash,
        string calldata reason,
        bytes calldata sendOptions
    ) external;

    /**
     * @notice Adds a token requirement for validator enrollment
     * @param token The token address
     * @param chainId The chain ID where the token exists
     * @param minAmount The minimum amount required
     */
    function addTokenRequirement(
        address token,
        uint256 chainId,
        uint256 minAmount
    ) external;

    /**
     * @notice Removes a token requirement
     * @param token The token address
     * @param chainId The chain ID where the token exists
     */
    function removeTokenRequirement(address token, uint256 chainId) external;

    /**
     * @notice Checks if a validator is eligible for enrollment based on token requirements
     * @param validatorToken The token address used by the validator
     * @param validatorChainId The chain ID of the validator's token
     * @param amount The amount of tokens the validator has
     * @return true if the validator meets any of the token requirements, false otherwise
     */
    function checkValidatorEligibility(
        address validatorToken,
        uint256 validatorChainId,
        uint256 amount
    ) external view returns (bool);

    /**
     * @notice Gets all token requirements
     * @return Array of token requirements
     */
    function getTokenRequirements()
        external
        view
        returns (TokenRequirement[] memory);
}
```

### IAVSManager

```solidity
interface IAVSManager {
    // Events
    event AVSEnrollmentInitiated(
        bytes32 indexed validatorId,
        bytes32 indexed avsId,
        uint256 chainId
    );
    event AVSEnrollmentConfirmed(
        bytes32 indexed validatorId,
        bytes32 indexed avsId
    );
    event AVSEnrollmentFailed(
        bytes32 indexed validatorId,
        bytes32 indexed avsId,
        string reason
    );
    event AVSDetachmentInitiated(
        bytes32 indexed validatorId,
        bytes32 indexed avsId
    );

    // Events for enhanced detachment flow
    event DetachmentExecutionInitiated(
        bytes32 indexed validatorId,
        bytes32 indexed avsId
    );
    event DetachmentFailed(
        bytes32 indexed validatorId,
        bytes32 indexed avsId,
        string reason
    );

    // Errors
    error InvalidAVS();
    error EnrollmentPending();
    error NotEnrolled();
    error AlreadyEnrolled();

    // Original methods
    function enroll(
        bytes32 avsId,
        uint256 chainId,
        ValidatorRecord memory validatorRecord,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    function handleEnrollFailed(
        bytes32 avsId,
        bytes32 validatorId,
        string calldata reason
    ) external;

    function handleEnrollConfirmed(
        bytes32 avsId,
        bytes32 validatorId,
        address token
    ) external;

    /**
     * @notice Initiates detachment with a time lock
     * @param avsId The ID of the AVS to detach from
     * @param chainId The chain ID where the AVS registry exists
     * @param sendOptions Options for cross-chain messaging
     * @param returnOptions Options for the return message
     */
    function initiateDetach(
        bytes32 avsId,
        uint256 chainId,
        address token,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    /**
     * @notice Handles confirmation of detachment initiation
     * @param avsId The ID of the AVS
     * @param validatorId The ID of the validator
     * @param unlockTime The time when detachment can be executed
     */
    function handleDetachInitiateConfirmed(
        bytes32 avsId,
        bytes32 validatorId,
        uint256 unlockTime
    ) external;

    /**
     * @notice Executes detachment after time lock has passed
     * @param avsId The ID of the AVS to detach from
     * @param chainId The chain ID where the AVS registry exists
     * @param sendOptions Options for cross-chain messaging
     * @param returnOptions Options for the return message
     */
    function executeDetach(
        bytes32 avsId,
        uint256 chainId,
        address token,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    /**
     * @notice Handles confirmation of detachment completion
     * @param avsId The ID of the AVS
     * @param validatorId The ID of the validator
     */
    function handleDetachCompleted(
        bytes32 avsId,
        bytes32 validatorId,
        address token
    ) external;

    /**
     * @notice Handles failure of detachment process
     * @param avsId The ID of the AVS
     * @param validatorId The ID of the validator
     * @param reason The reason for failure
     */
    function handleDetachFailed(
        bytes32 avsId,
        bytes32 validatorId,
        string calldata reason
    ) external;

    /**
     * @notice Checks if a validator is enrolled to any avs using the given token
     * @param validatorAccount The address of the validator
     * @param token The address of the token
     * @return bool True if the validator is enrolled to any AVS using the token
     */
    function canUnstake(
        address validatorAccount,
        address token
    ) external view returns (bool);
}
```

### IAVSRegistry

```solidity
interface IAVSRegistry {
    // Events
    event AVSCreated(
        bytes32 indexed avsId,
        address indexed avsAccount,
        address indexed owner,
        string description
    );

    event ValidatorEnrolled(
        bytes32 indexed avsId,
        bytes32 indexed validatorId,
        uint256 sourceChainId
    );
    event ValidatorDetached(bytes32 indexed avsId, bytes32 indexed validatorId);
    event ValidatorSlashed(
        bytes32 indexed validatorId,
        uint256 amount,
        string reason
    );
    event CrossChainEnrollmentInitiated(
        bytes32 indexed avsId,
        bytes32 indexed validatorId
    );
    event CrossChainDetachmentInitiated(
        bytes32 indexed avsId,
        bytes32 indexed validatorId
    );

    // New events for the enhanced detachment flow
    event DetachmentInitiated(
        bytes32 indexed avsId,
        bytes32 indexed validatorId,
        uint256 unlockTime
    );
    event DetachmentCompleted(
        bytes32 indexed avsId,
        bytes32 indexed validatorId
    );

    // Errors
    error InvalidAVS();
    error UnauthorizedAccess();
    error AlreadyDeployed();
    error InvalidValidator();
    error NotEnrolled();
    error SlashExceedsLimit();
    error InvalidMessageRouter();
    error ValidatorRequirementsNotMet();
    error DetachmentNotInitiated();
    error DetachmentTimeLockNotMet();

    function createAVS(
        Options memory options,
        string memory description
    ) external returns (address, bytes32);

    function slash(
        bytes32 validatorID,
        address token,
        SlashType slash,
        string calldata reason,
        bytes calldata sendOptions
    ) external payable;

    function setStakeManager(address _stakeManager) external;

    function setDetachTimeLock(uint256 _timeLock) external;

    // handles both cross chain and local enroll
    function enroll(bytes32 avsId, ValidatorRecord calldata validator) external;

    function handleIncomingEnrollment(
        bytes32 avsId,
        ValidatorRecord memory validator,
        bytes calldata returnOptions
    ) external payable;

    // New detachment methods with time lock
    function initiateDetach(
        bytes32 avsId,
        bytes32 validatorId,
        address token
    ) external returns (uint256);

    function executeDetach(
        bytes32 avsId,
        bytes32 validatorId,
        address token
    ) external;

    function handleIncomingDetachInitiation(
        bytes32 avsId,
        bytes32 validatorId,
        address token,
        bytes calldata returnOptions
    ) external payable;

    function handleIncomingDetachExecution(
        bytes32 avsId,
        bytes32 validatorId,
        address token,
        bytes calldata returnOptions
    ) external payable;

    // Legacy methods - maintained for backwards compatibility
    function handleIncomingDetachment(
        bytes32 avsId,
        bytes32 validatorId
    ) external;
    function detach(bytes32 avsId, bytes32 validatorId) external;
}
```

### IAccountManager

```solidity
/**
 * @title IAccountManager
 * @notice Interface for managing validator accounts using the BeaconProxy pattern
 * @dev Supports creation and management of validator accounts with one account per EOA
 */
interface IAccountManager {
    // Custom errors
    error AccountAlreadyDeployed();
    error InvalidValidatorConfiguration();

    // Events
    event ValidatorAccountDeployed(
        address indexed owner,
        bytes32 indexed validatorId,
        address account
    );

    /**
     * @notice Computes the address of a validator account for a given EOA
     * @param validator The validator owner address
     * @return The computed account address
     */
    function computeValidatorAccountAddress(
        address validator
    ) external view returns (address);

    /**
     * @notice Deploy a new validator account
     * @dev Each EOA can only deploy one validator account
     * @return The deployed account address
     */
    function deployValidatorAccount() external returns (address);
}
```

### IBaseValidatorAccount

```solidity
/**
 * @title IBaseValidatorAccount
 * @notice Base interface for all validator account types
 * @dev Contains common functionality that all validator types must implement
 */
interface IBaseValidatorAccount {
    /**
     * @notice Emitted when the account is locked
     */
    event Locked();

    /**
     * @notice Emitted when the account is unlocked
     */
    event Unlocked();

    /**
     * @notice Emitted when a withdrawal occurs
     * @param amount The amount withdrawn
     */
    event Withdrawal(uint256 amount);

    /**
     * @notice Emitted when the account is slashed
     * @param amount The amount slashed
     */
    event Slashed(uint256 amount);

    /**
     * @notice Emitted when ETH is received
     * @param sender The address that sent the ETH
     * @param amount The amount of ETH received
     */
    event ETHReceived(address sender, uint256 amount);

    /**
     * @notice Emitted when tokens are staked
     * @param token The token address being staked
     * @param amount The amount of tokens staked
     */
    event TokenStaked(address token, uint256 amount);

    /**
     * @notice Emitted when Ether is staked
     * @param amount The amount of Ether staked
     */
    event EtherStaked(uint256 amount);

    /**
     * @notice Emitted when tokens are unstaked
     * @param token The token address being unstaked
     * @param amount The amount of tokens unstaked
     */
    event TokenUnstaked(address token, uint256 amount);

    /**
     * @notice Withdraws ETH to specified recipient
     * @param recipient Address to receive the withdrawal
     * @param amount Amount to withdraw
     */
    function withdrawETH(address recipient, uint256 amount) external;

    /**
     * @notice Withdraws ERC20 tokens to specified recipient
     * @param token Address of the ERC20 token
     * @param recipient Address to receive the tokens
     * @param amount Amount of tokens to withdraw
     */
    function withdrawERC20Tokens(
        address token,
        address recipient,
        uint256 amount
    ) external;

    /**
     * @notice Enrolls the validator in an AVS
     * @param avsId The ID of the AVS to enroll in
     * @param chainId The chain ID where the AVS exists
     * @param sendOptions Options for sending the message
     * @param returnOptions Options for the return message
     */
    function enroll(
        bytes32 avsId,
        uint256 chainId,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    /**
     * @notice Initiates detachment from an AVS with time lock
     * @param avsId The ID of the AVS to detach from
     * @param chainId The chain ID where the AVS exists
     * @param sendOptions Options for sending the message
     * @param returnOptions Options for the return message
     */
    function initiateDetach(
        bytes32 avsId,
        uint256 chainId,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    /**
     * @notice Executes detachment from an AVS after time lock has passed
     * @param avsId The ID of the AVS to detach from
     * @param chainId The chain ID where the AVS exists
     * @param sendOptions Options for sending the message
     * @param returnOptions Options for the return message
     */
    function executeDetach(
        bytes32 avsId,
        uint256 chainId,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    /**
     * @notice Claims rewards for the validator
     * @param chainId The chain ID where the rewards exist
     * @param recipient The address to receive the rewards
     * @param tokens Array of token addresses to claim rewards for
     * @param sendOptions Options for sending the message
     */
    function claimReward(
        uint256 chainId,
        address recipient,
        address[] calldata tokens,
        bytes calldata sendOptions
    ) external payable;

    /**
     * @notice Slashes the validator's stake
     * @param amount Amount to slash
     */
    function slash(uint256 amount) external;

    /**
     * @notice Returns the validator's total stake
     * @return The total stake amount
     */
    function totalStake() external view returns (uint256);

    /**
     * @notice Returns the validator's total slashed amount
     * @return The total slashed amount
     */
    function totalSlashed() external view returns (uint256);

    /**
     * @notice Returns whether the validator account is active
     * @return True if active, false otherwise
     */
    function active() external view returns (bool);

    /**
     * @notice Returns whether the validator account is locked
     * @return True if locked, false otherwise
     */
    function locked() external view returns (bool);

    /**
     * @notice Returns the token ID associated with this validator
     * @return The validator's token ID
     */
    function stakingToken() external view returns (address);
}
```

### IBeaconManager

```solidity
interface IBeaconManager {
    function verifyValidatorState(
        Validator memory validator,
        bytes32[] calldata proof,
        uint256 validatorIndex,
        uint64 ts
    ) external;

    function verifyValidatorBalance(
        bytes32 balanceRecord,
        bytes32[] calldata proof,
        uint256 validatorIndex,
        uint64 ts
    ) external returns (uint64);
}
```

### IBeaconRoots

```solidity
interface IBeaconRoots {
    function getParentBlockRoot(uint64) external view returns (bytes32);
}
```

### IBeaconValidatorAccount

```solidity
/**
 * @title IBeaconValidator
 * @notice Interface for native Ethereum staking validators
 * @dev Extends the base validator interface with beacon chain specific functionality
 */
interface IBeaconValidatorAccount is IBaseValidatorAccount {
    /**
     * @notice Emitted when withdrawal address is verified
     */
    event WithdrawAddressVerified();

    /**
     * @notice Emitted when balance is verified
     * @param sender The transaction sender
     * @param beaconTimestamp The timestamp of the beacon block used
     * @param balance The verified balance
     */
    event BalanceVerified(
        address sender,
        uint256 beaconTimestamp,
        uint256 balance
    );

    /**
     * @notice Emitted when reconciliation is started
     * @param timestamp The timestamp when reconciliation was started
     */
    event ReconciliationStarted(uint256 timestamp);

    /**
     * @notice Verifies the withdrawal address against validator credentials
     * @param validator Validator data structure
     * @param proof Merkle proof for verification
     * @param validatorIndex Index of the validator
     * @param ts Timestamp for verification
     */
    function verifyWithdrawalAddress(
        Validator memory validator,
        bytes32[] calldata proof,
        uint256 validatorIndex,
        uint64 ts
    ) external;

    /**
     * @notice Initiates the reconciliation process
     */
    function startReconcile() external;

    /**
     * @notice Verifies the balance during reconciliation
     * @param balanceRecord Balance record for verification
     * @param proof Merkle proof for verification
     * @param validatorIndex Index of the validator
     * @param ts Timestamp for verification
     */
    function verifyBalance(
        bytes32 balanceRecord,
        bytes32[] calldata proof,
        uint256 validatorIndex,
        uint64 ts
    ) external;
}
```

### IDelegateManager

```solidity
interface IDelegateManager {
    // Events
    event DelegationInitiated(
        bytes32 indexed validatorId,
        bytes32 indexed delegationId,
        address indexed delegator,
        uint256 amount
    );
    event DelegationPending(
        bytes32 indexed validatorId,
        bytes32 indexed delegationId,
        address indexed delegator,
        uint256 amount
    );
    event DelegationFinalized(
        bytes32 indexed validatorId,
        bytes32 indexed delegationId,
        address indexed delegator,
        uint256 amount
    );
    event DelegationConfirmed(
        bytes32 indexed validatorId,
        bytes32 indexed delegationId,
        address indexed delegator,
        uint256 amount
    );
    event DelegationFailed(
        bytes32 indexed validatorId,
        bytes32 indexed delegationId,
        address indexed delegator,
        string reason
    );
    event RefundInitiated(
        bytes32 indexed delegationId,
        address indexed delegator,
        address indexed token,
        uint256 amount,
        uint256 timestamp
    );
    event RefundProcessed(
        bytes32 indexed delegationId,
        address indexed delegator,
        address indexed token,
        uint256 amount
    );

    event UndelegateInitiated(
        bytes32 delegationId,
        bytes32 validatorId,
        address delegator
    );

    event UndelegateCompleted(
        bytes32 delegationId,
        bytes32 validatorId,
        address delegator,
        uint256 amount
    );

    // Errors
    error InactiveDelegation();
    error InvalidValidator();
    error ValidatoNotRegistered();
    error InactiveValidator();
    error InvalidAmount();
    error InvalidDelegation();
    error AlreadyDelegated();
    error NotDelegationOwner();
    error RefundNotInitiated();
    error RefundAlreadyProcessed();
    error RefundDelayNotMet();
    error ValidatorMismatch();
    error RefundAlreadyInitiated();

    function delegate(
        bytes32 validatorId,
        address token,
        uint256 amount
    ) external payable;

    function refund(address delegator, bytes32 delegationId) external;
}
```

### IMessageRouter

```solidity
interface IMessageRouter {
    function sendMessage(
        uint32 dstChainId,
        uint8 messageType,
        bytes calldata payload,
        bytes calldata options
    ) external payable returns (bytes32 messageId);

    function avsRegistry() external view returns (address);
    function stakeManager() external view returns (address);
    function re2Gateway() external view returns (address);
}
```

### IRe2Gateway

```solidity
interface IRe2Gateway {
    function r2send(
        address initiator,
        address destAddr,
        address refundAddr,
        uint256 destChainId,
        uint256 amount,
        uint256 destGasAmount,
        bytes calldata message
    ) external payable returns (bytes32 bridgeId);

    function calculateTotalPayment(
        uint256 amount,
        uint256 destGasAmount
    ) external view returns (uint256);
}
```

### IRe2Receiver

```solidity
interface IRe2Receiver {
    /**
     * @notice Handles the reception of a message from the source chain.
     * @param bridgeId The unique identifier for the bridge.
     * @param sender The address of the sender on the source chain.
     * @param sourceChainId The ID of the source chain.
     * @param message The message received from the source chain.
     */
    function r2receiver(
        bytes32 bridgeId,
        address sender,
        uint256 sourceChainId,
        bytes memory message
    ) external payable;
}
```

### IRewardsManager

```solidity
interface IRewardsManager {
    // Events
    event RewardClaimInitiated(
        bytes32 indexed validatorId,
        address indexed recipient,
        uint256 chainId
    );

    function claimReward(
        uint256 chaindId,
        address recipient,
        address[] calldata tokens,
        uint256 delegatorShare,
        bytes calldata sendOptions
    ) external payable;
}
```

### IRewardsPool

```solidity
interface IRewardsPool {
    event RewardsSubmitted(bytes32 indexed avsId, Payment[] payments);
    event RewardClaimed(
        bytes32 indexed delegatorId,
        bytes32 indexed validatorId,
        address recipient
    );
    event RewardsDistributionCreated(
        bytes32 indexed distributionId,
        bytes32 indexed validatorId,
        address indexed token,
        uint256 totalAmount,
        uint256 timestamp
    );

    event MerkleRootAdded(
        bytes32 indexed distributionId,
        bytes32 merkleRoot,
        uint256 timestamp
    );

    event DelegatorRewardClaimed(
        bytes32 indexed distributionId,
        address indexed delegator,
        uint256 amount,
        address token
    );

    function submitReward(Payment[] memory payments) external;

    function claimReward(
        bytes32 validatorId,
        address recipient,
        uint256 deletegatorShare,
        address[] calldata tokens
    ) external;
}
```

### ISlashingManager

```solidity
interface ISlashingManager {
    // Events
    event ValidatorSlashed(
        address indexed validator,
        bytes32 avsID,
        uint256 amount
    );

    event SlashingInitiated(
        bytes32 indexed validatorId,
        bytes32 indexed avsId,
        bytes32 slashId
    );
    event SlashingExecuted(
        bytes32 indexed validatorId,
        bytes32 indexed slashId,
        address token,
        uint256 amount,
        uint256 slashedBy
    );
    event CrossChainSlashInitiated(
        bytes32 indexed validatorId,
        uint256 indexed chainId,
        bytes32 slashId,
        uint256 amount,
        address token,
        uint256 slashedBy
    );

    // Custom errors
    error ZeroAmount();
    error InvalidChainID();
    error SlashAlreadyExecuted();

    function slash(
        uint256 chainID,
        bytes32 validatorID,
        bytes32 avsID,
        address token,
        uint256 amount,
        uint256 slashedBy,
        bytes calldata sendOptions
    ) external payable;

    function handleIncomingSlash(
        address token,
        uint256 amount,
        uint256 slashedBy,
        uint256 timestamp,
        bytes32 slashId,
        bytes32 avsId,
        bytes32 validatorId
    ) external;
}
```

### IStakeManager

```solidity
/**
 * @title IStakeManager
 * @notice Interface for managing validator stakes, AVS enrollments, and delegators
 * @dev Combines account management, slashing, delegator, and cross-chain functionality
 */
interface IStakeManager {
    function getValidatorState(
        bytes32 validatorId
    )
        external
        view
        returns (
            address,
            uint256,
            address[] memory,
            bytes32[] memory,
            bytes32[] memory,
            address[] memory,
            address[] memory
        );

    function getDelegationState(
        address delegator
    ) external view returns (bytes32, bytes32[] memory, address[] memory);

    function setAVSRegistry(address _avsRegistry) external;

    function setMessageRouter(address _messageRouter) external;

    function setForwarder(address _forwarder) external;
}
```

### IValidatorAccount

```solidity
/**
 * @title IValidatorAccount
 * @notice Unified interface for validator accounts supporting both token staking and beacon chain restaking
 * @dev Combines features for all validator types with enhanced ETH management
 */
interface IValidatorAccount {
    /**
     * @notice Emitted when the account is locked
     */
    event Locked();

    /**
     * @notice Emitted when the account is unlocked
     */
    event Unlocked();

    /**
     * @notice Emitted when a withdrawal occurs
     * @param token The token address (address(0) for ETH)
     * @param amount The amount withdrawn
     */
    event Withdrawal(address token, uint256 amount);

    /**
     * @notice Emitted when the account is slashed
     * @param token The token being slashed
     * @param amount The amount slashed
     * @param percentage If slashing by percentage, the percentage used
     */
    event Slashed(address token, uint256 amount, uint256 percentage);

    /**
     * @notice Emitted when ETH is received
     * @param sender The address that sent the ETH
     * @param amount The amount of ETH received
     */
    event ETHReceived(address sender, uint256 amount);

    /**
     * @notice Emitted when tokens are staked
     * @param token The token address being staked
     * @param amount The amount of tokens staked
     */
    event TokenStaked(address token, uint256 amount);

    /**
     * @notice Emitted when tokens are unstaked
     * @param token The token address being unstaked
     */
    event TokenUnstaked(address token);

    /**
     * @notice Emitted when withdrawal address is verified
     */
    event WithdrawAddressVerified();

    /**
     * @notice Emitted when balance is verified
     * @param sender The transaction sender
     * @param beaconTimestamp The timestamp of the beacon block used
     * @param balance The verified balance
     */
    event BalanceVerified(
        address sender,
        uint256 beaconTimestamp,
        uint256 balance
    );

    /**
     * @notice Emitted when reconciliation is started
     * @param timestamp The timestamp when reconciliation was started
     */
    event ReconciliationStarted(uint256 timestamp);

    /**
     * @notice Emitted when beacon validator is unstaked before balance was verified and after reconciliation was started
     */
    event PendingBalanceMismatch();

    event DelegatorShareUpdated(uint256 share);

    /**
     * @notice Stake any token in the validator
     * @param token The token address (address(0) for ETH)
     * @param amount The amount to stake
     */
    function stakeToken(address token, uint256 amount) external payable;

    /**
     * @notice Withdraws a token to specified recipient
     * @param token The token address (address(0) for ETH)
     * @param recipient Address to receive the withdrawal
     * @param amount Amount to withdraw
     */
    function withdrawToken(
        address token,
        address recipient,
        uint256 amount
    ) external;

    /**
     * @notice Unstake a token
     * @param token The token address (address(0) for ETH)
     * @param fromBeacon Whether to unstake from beacon chain (for ETH only)
     */
    function unstakeToken(address token, bool fromBeacon) external;

    /**
     * @notice Verifies the withdrawal address against validator credentials
     * @param validator Validator data structure
     * @param proof Merkle proof for verification
     * @param validatorIndex Index of the validator
     * @param ts Timestamp for verification
     */
    function verifyWithdrawalAddress(
        Validator memory validator,
        bytes32[] calldata proof,
        uint256 validatorIndex,
        uint64 ts
    ) external;

    /**
     * @notice Initiates the reconciliation process
     */
    function startReconcile() external;

    /**
     * @notice Verifies the balance during reconciliation
     * @param balanceRecord Balance record for verification
     * @param proof Merkle proof for verification
     * @param validatorIndex Index of the validator
     * @param ts Timestamp for verification
     */
    function verifyBalance(
        bytes32 balanceRecord,
        bytes32[] calldata proof,
        uint256 validatorIndex,
        uint64 ts
    ) external;

    /**
     * @notice Enrolls the validator in an AVS
     * @param avsId The ID of the AVS to enroll in
     * @param chainId The chain ID where the AVS exists
     * @param token The token to use for this enrollment
     * @param sendOptions Options for cross-chain messaging
     * @param returnOptions Options for return messaging
     */
    function enroll(
        bytes32 avsId,
        uint256 chainId,
        address token,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    /**
     * @notice Initiates detachment from an AVS with time lock
     * @param avsId The ID of the AVS to detach from
     * @param chainId The chain ID where the AVS exists
     * @param sendOptions Options for sending the message
     * @param returnOptions Options for the return message
     */
    function initiateDetach(
        bytes32 avsId,
        uint256 chainId,
        address token,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    /**
     * @notice Executes detachment from an AVS after time lock has passed
     * @param avsId The ID of the AVS to detach from
     * @param chainId The chain ID where the AVS exists
     * @param sendOptions Options for sending the message
     * @param returnOptions Options for the return message
     */
    function executeDetach(
        bytes32 avsId,
        uint256 chainId,
        address token,
        bytes calldata sendOptions,
        bytes calldata returnOptions
    ) external payable;

    /**
     * @notice Claims rewards for the validator
     * @param chainId The chain ID where the rewards exist
     * @param recipient The address to receive the rewards
     * @param tokens Array of token addresses to claim rewards for
     * @param sendOptions Options for sending the message
     */
    function claimReward(
        uint256 chainId,
        address recipient,
        address[] calldata tokens,
        bytes calldata sendOptions
    ) external payable;

    /**
     * @notice Slashes the validator's stake for a specific token by amount
     * @param token The token address to slash
     * @param amount The amount to slash
     */
    function slash(address token, uint256 amount) external;

    /**
     * @notice Check if a token is actively staked by the validator
     * @param token The token address to check
     * @return True if the token is actively staked
     */
    function isTokenStaked(address token) external view returns (bool);

    /**
     * @notice Get all staked tokens
     * @return Array of staked token addresses
     */
    function getStakedTokens() external view returns (address[] memory);

    /**
     * @notice Get token stake details
     * @param token The token address
     * @return amount The staked amount
     * @return slashedAmount The slashed amount
     * @return active Whether the stake is active
     */
    function getTokenStake(
        address token
    )
        external
        view
        returns (uint256 amount, uint256 slashedAmount, bool active);

    /**
     * @notice Check if the validator has any active token stakes
     * @return True if any token is active
     */
    function hasActiveTokens() external view returns (bool);

    /**
     * @notice Returns whether the validator account is locked
     * @return True if locked, false otherwise
     */
    function locked() external view returns (bool);

    /**
     * @notice Checks if the validator is a beacon validator
     * @return True if this is a beacon validator
     */
    function isBeaconValidator() external view returns (bool);

    /**
     * @notice Returns the beacon stake amount (in Gwei)
     * @return The beacon stake amount
     */
    function beaconEthStaked() external view returns (uint256);

    /**
     * @notice Returns the beacon ETH slashed amount (in Wei)
     * @return The beacon ETH slashed amount
     */
    function beaconEthSlashed() external view returns (uint256);

    /**
     * @notice Returns the withdrawal ETH balance
     * @return The withdrawal ETH balance
     */
    function withdrawalETHBalance() external view returns (uint256);

    /**
     * @notice Returns the unstakeable ETH balance
     * @return The unstakeable ETH balance
     */
    function unStakeableETHBalance() external view returns (uint256);

    /**
     * @notice Checks if ETH transfers are disabled
     * @return True if ETH transfers are disabled
     */
    function ethTransferDisabled() external view returns (bool);

    /**
     *  @notice Allows the stake manager to lock the validator account
     */
    function lock() external;

    /**
     *  @notice Allows the stake manager to unlock the validator account
     */
    function unlock() external;
}

```
