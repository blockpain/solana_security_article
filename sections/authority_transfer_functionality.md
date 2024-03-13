# Authority Transfer Functionality
## The Vulnerability
Solana programs often designate specific public keys as authorities for critical functions, such as updating program parameters
or withdrawing funds. However, lacking the functionality to transfer this authority to another address can pose significant risks.
This limitation becomes problematic in scenarios such as team changes, protocol sales, or if the authority becomes compromised.

## Example
Consider a program where a global admin authority is responsible for setting specific protocol parameters through a `set_params`
function. The program does not include a mechanism to change the global admin:
```rust
pub fn set_params(ctx: Context<SetParams>, /* parameters to be set */) -> Result<()> {
    require_keys_eq!(
        ctx.accounts.current_admin.key(),
        ctx.accounts.global_admin.authority,
    );

    // Logic to set parameters
}
```
Here, the authority is statically defined, without the ability to update it to a new public key

## Recommended Mitigation
A secure approach to mitigating this issue is to create a two-step process for transferring authority. This process would allow the
current authority to nominate a new `pending_authority`, which must explicitly accept the role. Not only would this provide authority
transfer functionality, but it would also protect against accidental transfers or malicious takeovers. The flow would be as follows:
- **Nomination by the Current Authority**: the current authority would nominate a new `pending_authority` by calling `nominate_new_authority`,
which sets the `pending_authority` field in the program state
- **Acceptance by New Authority**: the nominated `pending_authority` calls `accept_authority` to take on their new role, transferring
authority from the current authority to `pending_authority`

This would look something like:
```rust
pub fn nominate_new_authority(ctx: Context<NominateAuthority>, new_authority: Pubkey) -> Result<()> {
    let state = &mut ctx.accounts.state;
    require_keys_eq!(
        state.authority, 
        ctx.accounts.current_authority.key()
    );

    state.pending_authority = Some(new_authority);
    Ok(())
}

pub fn accept_authority(ctx: Context<AcceptAuthority>) -> Result<()> {
    let state = &mut ctx.accounts.state;
    require_keys_eq!(
        Some(ctx.accounts.new_authority.key()), 
        state.pending_authority
    );

    state.authority = ctx.accounts.new_authority.key();
    state.pending_authority = None;
    Ok(())
}

#[derive(Accounts)]
pub struct NominateAuthority<'info> {
    #[account(
        mut,
        has_one = authority,
    )]
    pub state: Account<'info, ProgramState>,
    pub current_authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct AcceptAuthority<'info> {
    #[account(
        mut,
        constraint = state.pending_authority == Some(new_authority.key())
    )]
    pub state: Account<'info, ProgramState>,
    pub new_authority: Signer<'info>,
}

#[account]
pub struct ProgramState {
    pub authority: Pubkey,
    pub pending_authority: Option<Pubkey>,
    // Other relevant program state fields
}
```
In this example, the `ProgramState` account structure holds the current `authority` and an optional `pending_authority`. The `NominateAuthority`
context ensures that the transaction is signed by the current authority, allowing them to nominate a new authority. The `AcceptAuthority` context
checks that the `pending_authority` matches the signer of the transaction, allowing them to accept and become the new authority. This setup ensures
a secure and controlled transition of authority within the program.