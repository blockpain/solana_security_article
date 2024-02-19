## improper initialization

Unlike contracts deployed to the EVM, Solana programs are not deployed with a constructorto set state variables, and instead are manually initialized (normally by a function called `initialize` or something similar). Initization functions typically set data such as the programs authrority, or create accounts that form the base of the program being deployed (i.e. a central state account or something of the sort). 

Since the initialization function is called manually, and not automatically on program deployment, it is important that this instruction is instruction is called by a known address under the control of the programs development team. Otherwise, it is possible for an attacker to frontrun initialization, possibly setting up the program using accounts under the attacker's control.

A common practice is to use the programs `upgrade_authority` as the authorized address to call the `initialize` function, if the program has one.

### insecure initialization (can be frontrun)

```rust
pub fn initialize(ctx: Context<Initialize>) -> ProgramResult {
  ctx.accounts.central_state.authority = authority.key();
  ...  
}


#[derive(Accounts)]
pub struct Initialize<'info> {
  authority: Signer<'info>,
  #[account(mut,
    init,
    payer = authority,
    space = CentralState::SIZE,
    seeds = CentralState::SEEDS,
  )]
  central_state: Account<'info, CentralState>,
  ...
}

#[account]
pub struct CentralState {
  authority: Pubkey,
  ...
}
```

The example above is a stripped down initialize function, which sets the authority of a `CentralState` account to the caller of the instruction. However, this could be any account who calls initialize! As previously mentioned, a common way to secure an initialization function is to use the programs `upgrade_authority`, which is known at deployment.

Below is an example from the anchor documentation using constraints, which ensures only the program's upgrade authority can call the initialize function


**TODO** add anchor example here
