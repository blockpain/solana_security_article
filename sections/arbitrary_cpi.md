# Arbitrary CPI
## The Vulnerability
Arbitrary Cross-Program Invocations (CPIs) occur when a program invokes another program without verifying the target program's identity. This vulnerability
exists because the Solana runtime allows any program to call an other program if the caller has the callee's program ID and adheres to the callee's 
interface. If a program performs CPIs based on user input without validating the callee's program ID, it could execute code in an attacker-controlled program.

## Example
Consider a program that distributes awards to participants based on their contributions to a project. After distributing the rewards, the program records the
details in a seperate ledger program for auditing and tracking purposes. The ledger program is assumed to be a trusted program, providing a public interface
for keeping track of specific entries from authorized programs. The program includes a function to distribute and record rewards, which takes in the ledger
program as an account. However, the function fails to verify the provided `ledger_program` before making a CPI to it:
```rust
pub fn distribute_and_record_rewards(ctx: Context<DistributeAndRecord>, reward_amount: u64) -> ProgramResult {
    // Reward distribution logic

    let instruction = custom_ledger_program::instruction::record_transaction(
        &ctx.accounts.ledger_program.key(),
        &ctx.accounts.reward_account.key(),
        reward_amount,
    )?;

    invoke(
        &instruction,
        &[
            ctx.accounts.reward_account.clone(),
            ctx.accounts.ledger_program.clone(),
        ],
    )
}
#[derive(Accounts)]
pub struct DistributeAndRecord<'info> {
    reward_account: AccountInfo<'info>,
    ledger_program: AccountInfo<'info>,
}
```
An attacker could exploit this by passing a malicious program's ID as the `ledger_program`, leading to unintended consequences.

## Recommended Mitigation
To secure against this issue, developers can add a check that verifies the ledger program's identity before performing the CPI. This
check would ensure that the CPI call is made to the intended program, preventing arbitrary CPIs:
```rust
pub fn distribute_and_record_rewards(ctx: Context<DistributeAndRecord>, reward_amount: u64) -> ProgramResult {
    // Reward distribution logic

    // Verify the ledger_program is the expected custom ledger program
    if ctx.accounts.ledger_program.key() != &custom_ledger_program::ID {
        return Err(ProgramError::IncorrectProgramId.into())
    }
    
    let instruction = custom_ledger_program::instruction::record_transaction(
        &ctx.accounts.ledger_program.key(),
        &ctx.accounts.reward_account.key(),
        reward_amount,
    )?;

    invoke(
        &instruction,
        &[
            ctx.accounts.reward_account.clone(),
            ctx.accounts.ledger_program.clone(),
        ],
    )
}
#[derive(Accounts)]
pub struct DistributeAndRecord<'info> {
    reward_account: AccountInfo<'info>,
    ledger_program: AccountInfo<'info>,
}
```

Note that if a program was written in Anchor then it may have a publicly available CPI module. This makes invoking the program from another
Anchor program easy and secure. The Anchor CPI module automatically checks that the address of the program passed in matches the address of the
program stored in the module. Alternatively, hardcoding the address instead of having the user pass it in can be a possible solution.