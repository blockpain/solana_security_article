# Remaining Accounts

`ctx.remaining_accounts` provides a way to pass additional accounts into a function that weren't specified upfront in the Accounts struct. This gives flexiblility to the developer, allowing them to handle scenarios requiring a dynamic number of accounts, such as processing a variable number of users or interacting with different contracts. However, this flexibility comes with a caveat: accounts passed through `ctx.remaining_accounts` do not undergo the same validationan applied to accounts defined in the Accounts struct. This lack of validation opens a door to potential misuse and security vulnerabilities.

Specifically, because ctx.remaining_accounts doesn't validate the accounts, a malicious user could exploit this by passing in accounts that the program did not intend to interact with, leading to unauthorized actions or access. 

In the example below, the program uses `ctx.remaining_accounts` to dynamically receive user PDAs to calculate rewards. However, due to the absence of explicit checks on these accounts, it fails to ensure that only valid and eligible users' accounts are processed. This can be exploitedby a user who passes in accounts they do not own, or ones they created themself,in order to gain more rewards than they are actually owed.

## Code Example

```rust
use anchor_lang::prelude::*;

#[program]
pub mod rewards_distribution {
    use super::*;

    // function to calculate user rewards via passed in remaining_accounts

    pub fn calculate_rewards(ctx: Context<CalculateRewards>) -> Result<()> {
        let rewards_account = &ctx.accounts.rewards_account;
        let authority = &ctx.accounts.authority;

        // iterate over accounts passed in via `ctx.remaining_accounts`
        // UNSAFE: Assumes all accounts are valid accounts without validation.

        for user_pda_info in ctx.remaining_accounts.iter() {

            // logic to check user activity and calculate rewards
            // could be exploited by passing in incorrect or malicious PDAs,        }

        // Logic to distribute calculated rewards       

        Ok(())
    }
}

#[derive(Accounts)]
pub struct CalculateRewards<'info> {
    // Account to store total rewards information
    #[account(mut)]
    pub rewards_account: Account<'info, RewardsAccount>,
    // Authority responsible for rewards distribution
    pub authority: Signer<'info>,
}

// rewards account 
pub struct RewardsAccount {
    // rewards to distribute
    pub total_rewards: u64,
    // other relevant fields...
}
```
