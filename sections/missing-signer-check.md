# Missing Signer Check
## The Vulnerability
Transactions are signed with a wallet's private key to ensure authentication, integrity, non-repudiation, and the
authorization of a specific transaction by a specific wallet. By requiring transactions to be signed with the sender's
private key, Solana's runtime can verify that a transaction is initiated by the proper account and has not been tampered 
with. This mechanism underpins the trustless nature of decentralized networks. Without this verification, any account that 
supplies the correct account as an argument can execute a transaction. This could lead to unauthorized access to privileged 
information, funds, or functionality. This vulnerability arises from the failure to validate whether an operation is signed 
by the appropriate account's private key before executing certain privileged functionality.

## Example
Take the following function,
```
pub fn update_admin(program_id: &Pubkey, accounts &[AccountInfo]) -> ProgramResult {
    let account_iter = &mut accounts.iter();
    let config = ConfigAccount::unpack(next_account_info(account_iter)?)?;
    let admin = next_account_info(account_iter)?;
    let new_admin = next_account_info (account_iter)?;

    if admin.pubkey() != config.admin {
        return Err(ProgramError::InvalidAdminAccount);
    }

    config.admin = new_admin.pubkey();

    Ok(())
}
```
This function intends to update the program's admin. It includes a check to ensure that the operation is initiated by the current
admin, which is good access control. However, the function fails to verify that the transaction was signed by the current admin's
private key. Thus, anyone calling this function can pass in the proper `admin` account such that `admin.pubkey() = config.admin`,
irrespective of whether the account calling this function is actually the current admin. This allows a malicious actor to execute
the instruction with their account passed in as the new admin, directly bypassing the need for the current admin's authorization.

## Recommended Mitigation
Programs must include checks to verify that an account has been signed by the appropriate wallet. This can be done by checking the
[`AccountInfo::is_signer`](https://docs.rs/solana-program/latest/solana_program/account_info/struct.AccountInfo.html#structfield.is_signer) 
field of the accounts involved in the transaction. The program can enforce that only authorized accounts can perform certain actions
by checking whether the account executing the privileged operation has the `is_signer` flag set to `true`.

The updated code example would look like this:
```
pub fn update_admin(program_id: &Pubkey, accounts &[AccountInfo]) -> ProgramResult {
    let account_iter = &mut accounts.iter();
    let config = ConfigAccount::unpack(next_account_info(account_iter)?)?;
    let admin = next_account_info(account_iter)?;
    let new_admin = next_account_info (account_iter)?;

    if admin.pubkey() != config.admin {
        return Err(ProgramError::InvalidAdminAccount);
    }

    // Add in a check for the admin's signature
    if !admin.is_signer {
        return Err(ProgramError::NotSigner);
    }

    config.admin = new_admin.pubkey();

    Ok(())
}
```