## Frontrunning

With the rising popularity of transaction bundlers such as Jito and priority transactions, frontrunning is a concern that should be taken seriously by protocols built on the Solana blockchain. Imagine a protocol that handles purchasing and bidding for a product, storing the sellers pricing information in an account named `SellInfo`. 

```
#[account]
pub struct SellInfo {
  sale_price: u64,
  product_for_sale: Product,
  sale_token_mint: Mint,
  sale_token_destination: Pubkey
}
```

To purchase a `Product` that is listed, a buyer would need to pass in the `SellInfo` account related to the product they want. But what if the seller is able to change the `sale_price` of their listing? 

```
pub fn change_sale_price(Ctx<ChangeSalePrice>, new_price: u64) {...}
```

This would introduce a serious frontrunning opportunity for the seller, especially if the buyers purchasing transaction doesnt include `expected_price` checks that would ensure they are paying no more than the `expected_price`. If the purchaser submits a transaction to buy the given `Product` is would be possible for the seller to call `change_sale_price`, and using Jito, ensure this transaction is included ~before~ the purchasers transaction. A malicious seller could change the price of the `Product` to an exorbitant amount, unbeknownst to the purchaser, forcing them to pay much more than expected for the `Product`! 

#### Recommended Mitigation

A simple solution  would be including `expected_price` checks on the purchasing side of the deal, preventing the buyer from paying more than expected for the `Product` they want to buy.


