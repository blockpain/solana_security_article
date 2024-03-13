# PDA Sharing
## The Vulnerability
PDA sharing is a common vulnerability that arises when the same PDA is used across multiple authority domains or roles. This
could allow a malicious actor to access data or funds taht do not belong to them via the misuse of PDAs as a signer without
the proper access control checks in place.

## Example
Consider a program designed to faciliate token staking and distributing rewards. The program uses a single PDA for staking tokens
into a given pool and withdrawing rewards. The PDA is derived using a static seed (e.g., the name of the staking pool), making it
common across all operations.
```rust
pub fn stake_tokens(ctx: Context<StakeTokens>, amount: u64) -> ProgramResult {
    // Logic to stake tokens
    Ok(())
}

pub fn withdraw_rewards(ctx: Context<WithdrawRewards>, amount: u64) -> ProgramResult {
    // Logic to withdraw rewards
    Ok(())
}

#[derive(Accounts)]
pub struct StakeTokens<'info> {
    #[account(
        mut, 
        seeds = [b"staking_pool_pda"], 
        bump
    )]
    staking_pool: AccountInfo<'info>,
    // Other staking-related accounts
}

#[derive(Accounts)]
pub struct WithdrawRewards<'info> {
    #[account(
        mut, 
        seeds = [b"staking_pool_pda"], 
        bump
    )]
    rewards_pool: AccountInfo<'info>,
    // Other rewards withdrawal-related accounts
}
```
This is problematic as the staking and rewards withdrawal functionalities rely on the same PDA derived from `staking_pool_pda`. This could
potentially allow users to manipulate the contract into unauthorized rewards withdrawal or staking manipulation.

## Recommended Mitigation
To mitigate against this vulnerability, use distinct PDAs for different functionalities. Ensure that each PDA serves a specific context and
is derived using unique, operation-specific seeds:
```rust
pub fn stake_tokens(ctx: Context<StakeTokens>, amount: u64) -> ProgramResult {
    // Logic to stake tokens
    Ok(())
}

pub fn withdraw_rewards(ctx: Context<WithdrawRewards>, amount: u64) -> ProgramResult {
    // Logic to withdraw rewards
    Ok(())
}

#[derive(Accounts)]
pub struct StakeTokens<'info> {
    #[account(
        mut,
        seeds = [b"staking_pool", &staking_pool.key().as_ref()],
        bump
    )]
    staking_pool: AccountInfo<'info>,
    // Other staking-related accounts
}

#[derive(Accounts)]
pub struct WithdrawRewards<'info> {
    #[account(
        mut,
        seeds = [b"rewards_pool", &rewards_pool.key().as_ref()],
        bump
    )]
    rewards_pool: AccountInfo<'info>,
    // Other rewards withdrawal-related accounts
}
```
In the example above, the PDAs for staking tokens and withdrawing rewards are derived using distinct seeds (`staking_pool` and `rewards_pool`, respectively), 
combined with the specific account's key. This ensures that the PDAs are uniquely tied to their intended functionalities, mitigating the risk of
unauthorized actions.