# Account Reloading
## The Vulnerability
Account reloading is a vulnerability that arises when developers fail to update deserialized accounts after performing a CPI. Anchor does
not automatically refresh the state of deserialized accounts after a CPI. This could lead to scenarios where program logic operates on stale
data, leading to logical errors or incorrect calculations.

## Example
Consider a protocol where users can stake tokens to earn rewards over time. The program facilitating this includes functionality to update a 
user's staking rewards based on certain conditions being met or external triggers. A user's rewards are calculated and updated through a CPI to
a rewards distribution program. However, the program fails to update the original staking account after the CPI to reflect the new rewards 
balance:
```rust
pub fn update_rewards(ctx: Context<UpdateStakingRewards>, amount: u64) -> Result<()> {
    let staking_seeds = &[b"stake", ctx.accounts.staker.key().as_ref(), &[ctx.accounts.staking_account.bump]];

    let cpi_accounts = UpdateRewards {
        staking_account: ctx.accounts.staking_account.to_account_info(),
    };
    let cpi_program = ctx.accounts.rewards_distribution_program.to_account_info();
    let cpi_ctx = CpiContext::new_with_signer(cpi_program, cpi_accounts, staking_seeds);

    rewards_distribution::cpi::update_rewards(cpi_ctx, amount)?;

    // Attempt to log the "updated" reward balance
    msg!("Rewards: {}", ctx.accounts.staking_account.rewards);
    
    // Logic that uses the stale ctx.accounts.staking_account.rewards

    Ok(())
}

#[derive(Accounts)]
pub struct UpdateStakingRewards<'info> {
    #[account(mut)]
    pub staker: Signer<'info>,
    #[account(
        mut,
        seeds = [b"stake", staker.key().as_ref()],
        bump,
    )]
    pub staking_account: Account<'info, StakingAccount>,
    pub rewards_distribution_program: Program<'info, RewardsDistribution>,
}

#[account]
pub struct StakingAccount {
    pub staker: Pubkey,
    pub stake_amount: u64,
    pub rewards: u64,
    pub bump: u8,
}
```
In this example, the `update_rewards` function attempts to update the rewards for a user's staking account through a CPI call to a rewards 
distribution program. Initially, the program logs `ctx.accounts.staking_account.rewards` (i.e., the rewards balance) after the CPI, and then
continues onto logic that uses the stale `ctx.accounts.staking_account.rewards` data. The issue is that the staking account's state is not
automatically updated post-CPI, which is why the data is stale.

## Recommended Mitigation
To mitigate this issue, explicitly call Anchor's [`reload`](https://docs.rs/anchor-lang/latest/src/anchor_lang/accounts/account.rs.html#271-275) 
method to reload a given account from storage. Reloading an account post-CPI will accurately reflect its state:
```rust
pub fn update_rewards(ctx: Context<UpdateStakingRewards>, amount: u64) -> Result<()> {
    let staking_seeds = &[b"stake", ctx.accounts.staker.key().as_ref(), &[ctx.accounts.staking_account.bump]];

    let cpi_accounts = UpdateRewards {
        staking_account: ctx.accounts.staking_account.to_account_info(),
    };
    let cpi_program = ctx.accounts.rewards_distribution_program.to_account_info();
    let cpi_ctx = CpiContext::new_with_signer(cpi_program, cpi_accounts, staking_seeds);

    rewards_distribution::cpi::update_rewards(cpi_ctx, amount)?;

    // Reload the staking account to reflect the updated reward balance
    ctx.accounts.staking_account.reload()?;

    // Log the updated reward balance
    msg!("Rewards: {}", ctx.accounts.staking_account.rewards);
    
    // Logic that uses ctx.accounts.staking_account.rewards

    Ok(())
}
```