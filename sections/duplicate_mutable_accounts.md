# Duplicate Mutable Accounts
## The Vulnerability
Duplicate mutable accounts refers to a scenario where the same account is passed more than once as a mutable parameter to an instruction.
This occurs when an instruction requires two mutable accounts of the same type. A malicious actor could pass in the same account twice,
causing the account to be mutated in unintended ways (e.g., overwriting data), depending on the program's logic. The severity of this
vulnerability vaires based on the specific scenario, ranging from minor data overwrites to major security risks.

## Example
Consider a program designed to reward users based on their participation in a certain on-chain activity. The program has an instruction to
update the balance of two accounts: a reward account and a bonus account. A user is supposed to receive a standard reward in one account and
a potential bonus in another account, based on specific criteria:
```rust
pub fn distribute_rewards(ctx: Context<DistributeRewards>, reward_amount: u64, bonus_amount: u64) -> Result<()> {
    let reward_account = &mut ctx.accounts.reward_account;
    let bonus_reward = &mut ctx.accounts.bonus_account;

    // Intended to increment the reward and bonus accounts separately
    reward_account.balance += reward_amount;
    bonus_account.balance += bonus_amount;

    Ok(())
}

#[derive(Accounts)]
pub struct DistributeRewards<'info> {
    #[account(mut)]
    reward_account: Account<'info, RewardAccount>,
    #[account(mut)]
    bonus_account: Account<'info, RewardAccount>,
}

#[account]
pub struct RewardAccount {
    pub balance: u64,
}
```

If a malicious actor passes the same account for `reward_account` and `bonus_account`, the account's balance would be incorrectly updated twice.

## Recommended Mitigation
To mitigate this issue, add a check within the instruction logic to verify that the public keys of the two accounts are not identical:
```rust
pub fn distribute_rewards(ctx: Context<DistributeRewards>, reward_amount: u64, bonus_amount: u64) -> Result<()> {
    if ctx.accounts.reward_account.key() == ctx.accounts.bonus_account.key() {
        return Err(ProgramError::InvalidArgument.into())
    }
    
    let reward_account = &mut ctx.accounts.reward_account;
    let bonus_reward = &mut ctx.accounts.bonus_account;

    // Intended to increment the reward and bonus accounts separately
    reward_account.balance += reward_amount;
    bonus_account.balance += bonus_amount;

    Ok(())
}
```

Developers can use Anchor's account constraints to add a more explicit constraint on the account, ensuring that it's not the same as
another account. This can be done using the `#[account]` attribute and the `constraint` keyword:
```rust
pub fn distribute_rewards(ctx: Context<DistributeRewards>, reward_amount: u64, bonus_amount: u64) -> Result<()> {
    let reward_account = &mut ctx.accounts.reward_account;
    let bonus_reward = &mut ctx.accounts.bonus_account;

    // Intended to increment the reward and bonus accounts separately
    reward_account.balance += reward_amount;
    bonus_account.balance += bonus_amount;

    Ok(())
}

#[derive(Accounts)]
pub struct DistributeRewards<'info> {
    #[account(
        mut,
        constraint = reward_account.key() != bonus_account.key()
    )]
    reward_account: Account<'info, RewardAccount>,
    #[account(mut)]
    bonus_account: Account<'info, RewardAccount>,
}

#[account]
pub struct RewardAccount {
    pub balance: u64,
}
```