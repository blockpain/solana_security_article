# Loss of Precision
## The Vulnerability
Loss of precision, albeit minuscule in appearance, can pose a significant threat to a program. It can lead to incorrect calculations, 
arbitrage opportunities, and unexpected program behavior.

Precision loss in arithmetic operations is a common source of errors. With Solana programs, fixed-point arithmetic is recommended whenever
possible. This is because programs only support a [limited subset of Rust's float operations](https://solana.com/docs/programs/limitations#float-rust-types-support).
If a program attempts to use an unsupported float operation, the runtime will return an unresolved symbol error. Additionally, float operations
take more instructions compared to their integer equivalents. The use of fixed-point arithmetic and the need to handle large numbers of tokens
and fractional amounts accurately can exacerbate precision loss.

### Multiplication After Division
While the [associative property](https://en.wikipedia.org/wiki/Associative_property) holds for most mathematical operations, its application in
computer arithmetic can lead to unexpected precision loss. A classic example of precision loss occurs when performing multiplication after division, which
can yield different results from performing multiplication before division. For example, consider the following expressions: `(a/c) * b` and `(a*b) / c`. 
Mathematically, these expressions are associative - they *should* yield the same result. However, in the context of Solana and fixed-point arithmetic, the order 
of operations matters significantly. Performing division first `(a/c)` may result in a loss of precision due to rounding down the quotient before it's
multiplied by `b`. This could result in a smaller result than expected. Conversely, multiplying `(a*b)` before dividing by `c` could preserve more of 
the original precision. This difference can lead to incorrect calculations, creating unexpected program behavior and/or arbitrage opportunities.

### `saturating_*` Arithmetic Functions
While `saturating_*` arithmetic functions prevent overflow and underflow by capping values at their maximum or minimum possible values, they can lead to
subtle bugs and precision loss if this cap is reached unexpectedly. This occurs when the program's logic assumes that saturation alone will guarantee an
accurate result and ignores handling the potential loss of precision or accuracy. 

For example, imagine a program designed to calculate and distribute rewards to users based on the amount of tokens they trade within a specific period:
```rust
pub fn calculate_reward(transaction_amount: u64, reward_multiplier: u64) -> u64 {
    transaction_amount.saturating_mul(reward_multiplier)
}
```
Consider the scenario where the `transaction_amount` is 100 000 tokens and the `reward_multiplier` is 100 tokens per transaction. Multiplying the two will exceed the
maximum value a `u64` can hold. This means their product will be capped and lead to a substantial loss of precision by under-rewarding the user.

### Rounding Errors
Rounding operations are a common loss of precision in programming. The choice of rounding method can significantly impact the accuracy of calculations and the behavior of 
Solana programs. The `try_round_u64()` function is used to round decimal values to the nearest whole number. Rounding up is problematic as it can artificially inflate
values, leading to discrepancies between the actual and expected calculations.

Consider a Solana program that converts collateral into liquidity based on market conditions. The program uses `try_round_u64()` to round the result of a division operation:
```rust
pub fn collateral_to_liquidity(&self, collateral_amount: u64) -> Result<u64, ProgramError> {
    Decimal::from(collateral_amount)
        .try_div(self.0)?
        .try_round_u64()
}
```
In this scenario, rounding up can lead to the issuance of more liquidity tokens than justified by the collateral amount. This discrepancy can be exploited by malicious
actors to perform arbitrage attacks to extract value from the protocol via favorably influenced rounding outcomes. To mitigate, use `try_floor_u64` to round down to the
nearest whole number. This approach minimizesthe risk of artificially inflating values and ensures that any rounding does not give the user an advantage at the expense
of the system. Alternatively, implement logic to explicitly handle scenarios where rounding could significantly impact the outcome. This might include setting specific
thresholds fpr rounding decisions or apply different logic based on the size of the values involved.