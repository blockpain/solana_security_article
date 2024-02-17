# Arithmetic Overflows and Underflows
## The Vulnerability
An integer is a number without a fractional component. Rust stores integers as fixed-size variables.  These variables are defined by their
[signedness](https://en.wikipedia.org/wiki/Signedness) (i.e., signed or unsigned) and the amount of space they occupy in memory. For example, 
the `u8` type denotes an unsigned integer that occupies 8 bits of space. It's capable of holding values from 0 to 255. Trying to store a value 
outside of that range would result in an integer overflow or underflow. An integer overflow is when a variable exceeds its maximum capacity and 
wraps around to its minimum value. An integer underflow is when a variable drops below its minimum capacity and wraps around to its maximum value.

Rust includes checks for integer overflows and underflows when compiling in debug mode. These checks will cause the program to *panic* at 
runtime, if such a condition is detected. However, Rust does not include checks that panic for integer overflows and underflows when compiling
in release mode with the `--release` flag. This behavior can introduce subtle vulnerabilities as the overflow or underflow occurs silently.
The [Berkley Packet Filter (BPF)](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter) toolchain is an integral part of Solana's development 
environment as it is used to compile Solana programs. The `cargo build-bpf` command is used to compile Rust projects into BPF bytecode for 
deployment. The issue with this is that it compiles programs in release mode by default. Thus, Solana programs are vulnerable to integer overflows 
and underflows.

## Example
An attacker can exploit this vulnerability by taking advantage of the silent overflow / underflow behavior in release mode, especially functions that
handle token balances. Take the following example,
```
pub fn process_instruction(
    _program_id: & Pubkey,
    accounts: [&AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let account = next_account_info(account_info_iter)?;

    let mut balance: u8 = account.data.borrow()[0];
    let tokens_to_subtract: u8 = 100;

    balance = balance - tokens_to_subtract;

    account.data.borrow_mut()[0] = balance;
    msg!("Updated balance to {}", balance);
    
    Ok(())
}
```
This function assumes the balance is stored in the first byte for simplicity. It takes the account's balance and subtracts `tokens_to_subtract` from it.
If the user's balance is less than `tokens_to_subtract`, it'll cause an underflow. For example, a user with 10 tokens would underflow to a total balance
of 165 tokens.

## Recommended Mitigation
### `overflow-checks`
The easiest way to mitigate this vulnerability is to set the key `overflow-checks` to `true` in the project's `Cargo.toml` file. Here, Rust will add overflow
and underflow checks in the compiler. However, adding overflow and underflow checks increases the [compute cost](https://solana.com/docs/core/runtime#compute-budget) 
of a transaction. In cases where compute needs to be optimized for, it may be more beneficial to set `overflow-checks` to `false`.

### `checked_*` Arithmetic
Use Rust's `checked_*` arithmetic functions on each integer type to strategically check for overflows and underflows throughout your program. These functions will
return `None` if an overflow or underflow would occur. This allows the program to handle the error gracefully. For example, you could refactor the previous code to:
```
pub fn process_instruction(
    _program_id: & Pubkey,
    accounts: [&AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let account = next_account_info(account_info_iter)?;

    let mut balance: u8 = account.data.borrow()[0];
    let tokens_to_subtract: u8 = 100;

    match balance.checked_sub(tokens_to_subtract) {
        Some(new_balance) => {
            account.data.borrow_mut()[0] = new_balance;
            msg!("Updated balance to {}", new_balance);
        },
        None => {
            return Err(ProgramErrorr::InsufficientFunds);
        }
    }

    Ok(())
}
```
In the revised example, `checked_sub` is used to subtract `tokens_to_subtract` from `balance`. Thus, if `balance` is sufficient to cover the subtraction, 
`checked_sub` will return `Some(new_balance)`. The program continues to update the account's balance safely and logs it. However, if the subtraction 
would result in an underflow, `checked_sub` returns `None`, which we can handle by returning an error.

## Casting
Similarly, casting between integer types using the `as` keyword without proper checks can introduce an integer overflow or underflow vulnerabilities. This is 
because casting can either truncate or extend values in unintended ways. When casting from a larger integer type to a smaller one (e.g., `u64` to `u32`), 
Rust will truncate the higher bits of the orignal value that do not fit into the target type. This is problematic when the original value exceeds the maximum 
value that the target type can store. When casting from a smaller integer type to a larger one (e.g., `i16` to `i32`), Rust will extend the value. This is 
straightforward for unsigned types. However, this can lead to [sign extension](https://en.wikipedia.org/wiki/Sign_extension) with signed integers to introduce
unintended negative values.

### Recommended Mitigation
Use Rust's safe casting methods to mitigate this vulnerability. This includes methods such as [`try_from`](https://doc.rust-lang.org/std/convert/trait.TryFrom.html#tymethod.try_from) 
and [`from`](https://doc.rust-lang.org/std/convert/trait.From.html#tymethod.from). Using `try_from` returns a `Result` type, allowing for the explicit handling 
of cases where the value does not fit into the target type gracefully. Using Rust's `from` method can be used for a safe, implicit conversion for conversions that
are guaranteed to be lossless (e.g., `u8` to `u32`). For example, if a program needed to safely convery a `u64` token amount to a `u32` type for processing, it can 
do the following:
```
pub fn convert_token_amount(amount: u64) -> Result<u32, ProgramError> {
    u32::try_from(amount).map_err(|_| ProgramError::InvalidArgument)
}
```
In this example, if `amount` exceeds the maximum value a `u32` can hold (i.e., 4 294 967 295), the conversion fails and the program returns an error. This prevents a
potential overflow/underflow from occurring.
