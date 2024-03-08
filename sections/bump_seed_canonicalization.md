# Bump Seed Canonicalization
## The Vulnerability
Bump seed canonicalization refers to the practice of using the highest valid bump seed (i.e., canonical bump) when deriving Program Derived
Addresses (PDAs). Using the canonical bump is a deterministic and secure way to find an address given a set of seeds. Failing to use the
canonical bump can lead to vulnerabilities, such as malicious actors creating or manipulating PDAs that would compromise program logic
or data integrity.

## Example
Consider a program designed to create unique user profiles, each with an associated PDA derived explicitly using `create_program_address`. The 
program allows for the creation of a profile by taking a user-provided bump. However, this is problematic as it introduces the risk of using
the non-canonical bump: 
```rust
pub fn create_profile(ctx: Context<CreateProfile>, user_id: u64, attributes: Vec<u8>, bump: u8) -> Result<()> {
    // Explicitly derive the PDA using create_program_address and a user-provided bump
    let seeds: &[&[u8]] = &[b"profile", &user_id.to_le_bytes(),&[bump]];
    let (derived_address, _bump) = Pubkey::create_program_address(seeds, &ctx.program_id)?;

    if derived_address != ctx.accounts.profile.key() {
        return Err(ProgramError::InvalidSeeds);
    }

    let profile_pda = &mut ctx.accounts.profile;
    profile_pda.user_id = user_id;
    profile_pda.attributes = attributes;

    Ok(())
}

#[derive(Accounts)]
pub struct CreateProfile<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    /// The profile account, expected to be a PDA derived with the user_id and a user-provided bump seed
    #[account(mut)]
    pub profile: Account<'info, UserProfile>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct UserProfile {
    pub user_id: u64,
    pub attributes: Vec<u8>,
}
```
In this scenario, the program derives a `UserProfile` PDA using `create_program_address` with seeds that include a user-provided bump. This is
problematic because it does not ensure the use of the canonical bump. This would allow a malicious actor to create multiple PDAs for the same
user ID with different bumps.

## Recommended Mitigation
To mitigate this issue, we can refactor our example to derive PDAs using `find_program_address` and validate the bump seed explicitly:
```rust
pub fn create_profile(ctx: Context<CreateProfile>, user_id: u64, attributes: Vec<u8>) -> Result<()> {
    // Securely derive the PDA using find_program_address to ensure the canonical bump is used
    let seeds: &[&[u8]] = &[b"profile", user_id.to_le_bytes()];
    let (derived_address, bump) = Pubkey::find_program_address(seeds, &ctx.program_id);

    // Store the canonical bump in the profile for future validations
    let profile_pda = &mut ctx.accounts.profile;
    profile_pda.user_id = user_id;
    profile_pda.attributes = attributes;
    profile_pda.bump = bump;

    Ok(())
}

#[derive(Accounts)]
#[instruction(user_id: u64)]
pub struct CreateProfile<'info> {
    #[account(
        init, 
        payer = user, 
        space = 8 + 1024 + 1, 
        seeds = [b"profile", user_id.to_le_bytes().as_ref()], 
        bump
    )]
    pub profile: Account<'info, UserProfile>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct UserProfile {
    pub user_id: u64,
    pub attributes: Vec<u8>,
    pub bump: u8,
}
```
Here, `find_program_address` is used to derive the PDA with the canonical bump seed to ensure a deterministic and secure PDA creation. The
canonical bump is stored in the `UserProfile` account, allowing for efficient and secure validation in susbequent operations. The reason we
prefer `find_program_address` over `create_program_address` is because the latter creates a valid PDA *without searching for a bump seed*. Because
it doesn't search for a bump seed, it may unpredictably return an error for any given set of seeds and is not generally suitable for creating
PDAs. `find_program_address` will *always* use the canonical bump when creating a PDA. This is because it iterates through various `create_program_address`
calls, starting with a bump of 255 and decrementing the bump with each iteration. Once a valid address is found, the function returns the derived
PDA and the canonical bump used to derive it. 

Note, Anchor enforces the use of the canonical bump for PDA derivations through its `seeds` and `bump` constraints, streamlining this entire process to
ensure secure and deterministic PDA creation and validation.