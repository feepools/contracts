# Feepools Contracts

Feepools is a service that allows projects to offer staking rewards to their token holders. Projects can set up feepools (anyone can create them) and direct their stakers to stake tokens into these pools to earn rewards.

## Overview

The system operates with two layers:

- **Permissioned Layer (Optional)**: The owner can auto-include feepools for projects using the `listPool` function. When a pool is listed, all stakers of the stake token automatically join that pool unless they specify custom pools.

- **Custom Pools**: Projects can create their own feepools and specify them directly. Stakers can choose to join specific pools when staking.

### Deployments

Staker: `0xcf7400244d0fbD33752d4B51A0415038CAA6bE1D`

SwapConfig: `0x7C4299647e3FBD9f7f40A2b39372fedd7ceD3f4D`

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
