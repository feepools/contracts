# Feepools Contracts

Feepools is a service that allows projects to offer staking rewards to their token holders. Projects can set up feepools (anyone can create them) and direct their stakers to stake tokens into these pools to earn rewards.

## Overview

The system operates with two layers:

- **Permissioned Layer (Optional)**: The owner can auto-include feepools for projects using the `listPool` function. When a pool is listed, all stakers of the stake token automatically join that pool unless they specify custom pools.

- **Custom Pools**: Projects can create their own feepools and specify them directly. Stakers can choose to join specific pools when staking.

### Deployments

Staker: `0x9d851487C5d3B22E60dCed1021edA88171047862`

## Critical Functions for Projects

### Creating a Feepool

**Function**: `createPool(address stakeToken, address rewardToken, uint rewardDurationDays)`

**Contract**: [`Staker.sol`](src/Staker.sol)

**Purpose**: Creates a new feepool for staking rewards.

**Parameters**:
- `stakeToken`: The token that users will stake
- `rewardToken`: The token used for rewards (use `address(0)` for WETH)
- `rewardDurationDays`: Duration of the reward period in days

**Returns**: Address of the created pool

**Notes**:
- Anyone can create a pool
- Automatically creates a StakeCredit token if it doesn't exist for the stake token
- Stakes 1 wei to prevent trapped rewards

### Listing a Pool (Optional - Owner Only)

**Function**: `listPool(address pool)`

**Contract**: [`Staker.sol`](src/Staker.sol)

**Purpose**: Lists a pool so it's automatically included for all stakers of the stake token.

**Access**: Owner only

**Effect**: When stakers stake tokens, they automatically join listed pools unless they specify custom pools.

**Note**: This is optional. Projects can create pools without listing them, and stakers can still join them by specifying custom pools.

### Adding Rewards to a Pool

Once a pool is created, projects need to add rewards. There are three functions available in the [`FeePool.sol`](src/FeePool.sol) contract:

#### 1. addEthReward

**Function**: `addEthReward(bytes memory data)`

**Purpose**: Add ETH rewards to the pool.

**Usage**: Send ETH along with the call.

**Parameters**:
- `data`: Swap data if the reward token is different from WETH (can be empty bytes if reward token is WETH)

**Note**: Automatically converts to WETH if reward token is WETH, or swaps to the reward token if different.

#### 2. addTokenReward

**Function**: `addTokenReward(address token, bytes memory data)`

**Purpose**: Add rewards from tokens already in the pool contract.

**Parameters**:
- `token`: Token address to add as rewards
- `data`: Swap data if token needs to be swapped to reward token

**Note**: Uses balance already in the pool contract. The tokens must have been transferred to the pool contract beforehand.

#### 3. addTokenRewardWithTransfer

**Function**: `addTokenRewardWithTransfer(address from, address token, uint quantity, bytes memory data)`

**Purpose**: Transfer tokens from caller and add as rewards.

**Parameters**:
- `from`: Address to transfer from (typically `msg.sender`)
- `token`: Token address
- `quantity`: Amount to transfer and add
- `data`: Swap data if token needs to be swapped to reward token

**Note**: This is the most common method for projects to add rewards, as it handles the transfer in a single transaction.

## Setup Flow for Projects

To set up staking rewards for your project:

1. **Create a feepool** using `createPool`:
   - Specify your stake token (the token users will stake)
   - Specify the reward token (use `address(0)` for WETH)
   - Set the reward duration in days

2. **(Optional) Request owner to list the pool** using `listPool`:
   - This enables auto-inclusion for all stakers
   - If not listed, stakers can still join by specifying your pool as a custom pool

3. **Add rewards** using one of the reward functions:
   - Use `addEthReward` to send ETH rewards
   - Use `addTokenRewardWithTransfer` to transfer and add token rewards
   - Use `addTokenReward` if tokens are already in the pool contract

4. **Direct stakers** to stake tokens into the feepool:
   - Stakers call `stake` on the Staker contract
   - If your pool is listed, stakers automatically join it
   - If not listed, stakers can specify your pool in the `customPools` parameter

## Integration Guide

### Staker Contract vs FeePool Contract

Understanding the architecture is crucial for integrators:

**Staker Contract** ([`Staker.sol`](src/Staker.sol)):
- Main entry point for all staking operations
- Manages user accounts, token stakes, and pool memberships
- Handles StakeCredit token minting/burning
- Coordinates interactions with multiple FeePool contracts
- Functions: `stake`, `unstake`, `joinPools`, `leavePools`, `claimRewards`, etc.

**FeePool Contract** ([`FeePool.sol`](src/FeePool.sol)):
- Individual reward pool for a specific stake token, reward token, and duration
- Manages reward distribution and accrual
- Tracks user stakes and rewards within that specific pool
- Only callable by the Staker contract (enforced via `onlyStaker` modifier)
- Functions: `stake`, `unstake`, `claimReward`, reward query functions, etc.

**Important**:
- **User operations** (stake, unstake, claim rewards) should go through the **Staker contract**
- **Adding rewards** (addEthReward, addTokenReward, addTokenRewardWithTransfer) should be called directly on the **FeePool contract** by projects/integrators
- **Read-only/view functions** on FeePool contracts can and should be called directly by integrators to query reward information for displaying user rewards in their applications

### Unstaking and Reward Claims

**Automatic Reward Claiming on Unstake**: When a user unstakes all of their tokens (full unstake), the system automatically claims and pays out all unclaimed rewards from all pools the user was participating in. This happens via the `leavePools` function which calls `claimPoolRewards` after unstaking.

**Partial Unstake**: When unstaking a partial amount, rewards are not automatically claimed, but the user's reward accrual is updated. Users can claim rewards separately using `claimRewards` or `claimPoolRewards`.

### Reward Query Functions (FeePool Contract)

These functions are available on individual FeePool contracts to query reward information:

#### getUnpaidRewards

**Function**: `getUnpaidRewards(address user)`

**Returns**: `uint256` - The amount of unclaimed rewards for a user

**Purpose**: Get the current unclaimed reward balance for a user in this pool.

#### getRewardPerToken

**Function**: `getRewardPerToken()`

**Returns**: `uint256` - The current reward per token (scaled by 1e18)

**Purpose**: Get the current reward rate per staked token. This value accumulates over time as rewards are distributed.

#### getPaidRewards

**Function**: `getPaidRewards(address user)`

**Returns**: `uint256` - The total amount of rewards already claimed by the user

**Purpose**: Get the historical total of rewards that have been claimed by a user in this pool.

#### getRewardForDuration

**Function**: `getRewardForDuration()`

**Returns**: `uint256` - The total reward amount for the current reward period

**Purpose**: Get the total reward amount that will be distributed over the current reward duration period.

#### getTotalRewardsAdded

**Function**: `getTotalRewardsAdded()`

**Returns**: `uint256` - The total amount of rewards added to the pool over its lifetime

**Purpose**: Get the cumulative total of all rewards that have been added to this pool since its creation. This is useful for tracking the pool's reward history.

### Accessing FeePool Contracts

To interact with FeePool functions, you need the pool address. You can get it using:
- `getPoolAddress(stakeToken, rewardToken, rewardDurationDays)` - Reverts if pool doesn't exist
- `getPoolAddressUnchecked(stakeToken, rewardToken, rewardDurationDays)` - Returns address(0) if pool doesn't exist

## Helper Functions

Useful view functions for querying pool information:

### Pool Information

- **`getPoolAddress(address stakeToken, address rewardToken, uint rewardDurationDays)`**: Get pool address by parameters
- **`getPoolAddressUnchecked(address stakeToken, address rewardToken, uint rewardDurationDays)`**: Get pool address without reverting if not found
- **`getPool(address pool)`**: Get pool information (stakeToken, rewardToken, rewardDuration, stakes, errors)
- **`getPoolDestructured(address pool)`**: Get pool information as separate return values

### Listed Pools

- **`isListedToken(address token)`**: Check if a token has any listed pools
- **`getListedTokens()`**: Get all tokens with listed pools
- **`getListedPools(address token)`**: Get all listed pools for a token
- **`isListedPool(address token, address pool)`**: Check if a specific pool is listed for a token

### User Information

- **`getStake(address user, address token)`**: Get user's stake amount for a token
- **`getJoinedPools(address user, address token)`**: Get pools a user has joined for a token
- **`hasJoinedPool(address user, address token, address pool)`**: Check if a user has joined a specific pool

### Staker Information

- **`isStaker(address token, address user)`**: Check if a user is a staker of a token
- **`getStakers(address token)`**: Get all stakers for a token
- **`getStakerAt(address token, uint index)`**: Get staker at a specific index for a token
- **`getStakerCount(address token)`**: Get the total count of stakers for a token

### Setup

```
forge install OpenZeppelin/openzeppelin-contracts@v4.9.5
```

### Build

```shell
$ forge build
```

### Test

```shell
$ forge test
```

### Help

```shell
$ forge --help
$ anvil --help
$ cast --help
```
