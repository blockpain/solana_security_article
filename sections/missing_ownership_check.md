# Missing Owner Check
## The Vulnerability
Ownership checks are crucial to validate that an account involved in a transaction or operation is owned by the expected program. Accounts
include an [`owner`](https://docs.rs/solana-program/latest/solana_program/account_info/struct.AccountInfo.html#structfield.owner) field, which 
indicates the program that has the authority to write to the account's data. This field ensures that only authorized programs can modify an 
account's state. Moreover, this field is useful for making sure that accounts passed in to an instruction are owned by the expected program. 
Missing ownership checks can lead to severe vulnerabilities, including unauthorized fund transfers and the execution of privileged operations.

## Example
Consider a program function defined to allow admin-only withdrawals from a vault. The function takes in a configuration account (i.e., `config`)
and uses its `admin` field to check whether the provided admin account's public key is the same as the one stored in the `config` account. However, it
fails to verify the ownership of the `config` account, assuming it to be trustworthy:
```rust
pub fn admin_token_withdraw(program_id: &Pubkey, accounts: &[AccountInfo], amount: u64) -> ProgramResult {
    // Account setup

    if config.admin != admin.pubkey() {
        return Err(ProgramError::InvalidAdminAccount)
    }

    // Transfer funds logic
}
```
A malicious actor could exploit this by supplying a `config` account they control with a matching `admin` field, effectively tricking the program into 
executing the withdrawal

## Recommended Mitigation
To mitigate this, perform an ownership check that verifies the `owner` field of the account:
```rust
pub fn admin_token_withdraw(program_id: &Pubkey, accounts: &[AccountInfo], amount: u64) -> ProgramResult {
    // Account setup

    if config.admin != admin.pubkey() {
        return Err(ProgramError::InvalidAdminAccount)
    }

    if config.owner != program_id {
        return Err(ProgramError::InvalidConfigAccount)
    }

    // Transfer funds logic
}
```

Anchor streamlines this check with the `Account` type. `Account<'info, T>` is a wrapper around `AccountInfo`, which verifies program ownership and deserializes the
underlying data into `T` (i.e., the specified account type). This allows developers to use `Account<'info, T>` to validate account ownership easily. Developers
can also use the `#[account]` attribute to add the [`Owner`](https://docs.rs/anchor-lang/latest/anchor_lang/trait.Owner.html) trait to a given account. This trait
defines an address expected to own the account. In addition to this, developers can use the `owner` constraint to define the program that should own a given 
account, if it's different from the currently executing one. This is useful, for example, when writing an instruction that expects an account to be a PDA derived
from a different program. The `owner` constraint is defined as: `#[account(owner = <expr>)]`, where `<expr>` is an arbitrary expression.