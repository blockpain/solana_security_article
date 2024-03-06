# Type Cosplay
## The Vulnerability
Type cosplay is a vulnerability where one account type is misrepresented as another due to a lack of type checks during deserialization. This
can lead to the execution of unauthorized actions or data corruption, as the program would operate based on the incorrect assumption of the
account's role or permissions. It is vital to always explicitly check the account's intended type during deserialization.

## Example
Consider a program that manages access to admin operations based on a user's role. Each user account includes a role discriminator to
distinguish between regular users and administrators. The program contains a function to update admin settings, intended only for administrators.
However, the program fails to check the account's discriminator and deserializes user account data without confirming whether the account is
an administrator:
```rust
pub fn update_admin_settings(ctx: Context<UpdateSettings>) -> ProgramResult {
    // Deserialize without checking the discriminator
    let user = User::try_from_slice(&ctx.accounts.user.data.borrow()).unwrap();

    // Sensitive update logic

    msg!("Admin settings updated by: {}", user.authority)
    Ok(())
}

#[derive(Accounts)]
pub struct UpdateSettings<'info> {
    user: AccountInfo<'info>
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct User {
    authority: Pubkey,
}
```
The issue is that `update_admin_settings` deserializes the user account passed in without checking the account's role discriminator, partly due to the
fact that the `User` struct is missing a discriminator field!

## Recommended Mitigation
To mitigate against this issue, developers can introduce a discriminator field in the `User` struct, and verify it during the deserialization process:
```rust
pub fn update_admin_settings(ctx: Context<UpdateSettings>) -> ProgramResult {
    let user = User::try_from_slice(&ctx.accounts.user.data.borrow()).unwrap();

    // Verify the user's discriminator
    if user.discriminant != AccountDiscriminant::Admin {
        return Err(ProgramError::InvalidAccountData.into())
    }
    
    // Sensitive update logic

    msg!("Admin settings updated by: {}", user.authority)
    Ok(())
}

#[derive(Accounts)]
pub struct UpdateSettings<'info> {
    user: AccountInfo<'info>
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct User {
    discriminant: AccountDiscriminant,
    authority: Pubkey,
}

#[derive(BorshSerialize, BorshDeserialize, PartialEq)]
pub enum AccountDiscriminant {
    Admin,
    // Other account types
}
```

Anchor simplifies the mitigation of type cosplay vulnerabilities by automatically managing discriminators for account times. This is done
via the `Account<'info, T>` wrapper, where Anchor ensures type safety by automatically checking the discriminator during deserialization. This
allows developers to focus more on the buisness logic of their program, rather than implementing various type checks manually.